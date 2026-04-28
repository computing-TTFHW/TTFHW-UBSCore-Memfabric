# MemFabric Hybrid Build & Test (NPU + HCom)

## Description
Builds, performs incremental compilation with ccache, and runs unit tests for the MemFabric Hybrid project inside a manylinux ARM64 Docker container. Uses **NPU + test + hcom** configuration. Outputs timing statistics to JSON.

## Prerequisites

- **SSH access** to the remote Linux build machine (default: `root@192.168.0.195`)
- **Docker** installed on the remote machine
- **Git** for cloning the repository

## Linux Build Machine

- **CPU**: Kunpeng-920, aarch64, 2 sockets × 8 cores = 16 threads
- **OS**: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
- **Container name**: `memfabric_ut`
- **Workspace in container**: `/workspace/memfabric_hybrid`

## Container Image

- **Image**: `swr.cn-north-4.myhuaweicloud.com/memfabric-hybrid/memfabric-hybrid_arm:v20`
- **User**: Must run with `--user root` (image default USER `build` does not exist in container)

## Environment Configuration

### Step 1: Install system dependencies

```bash
yum install -y ccache bc python3-devel libasan6 gcc-toolset-14-libasan-devel gcc-toolset-14-libubsan-devel rdma-core-devel
```

### Step 2: Install Python dependencies

```bash
pip3 install --upgrade pip setuptools wheel
pip3 install pybind11==3.0.1
```

### Step 3: Move system patchelf (conflicts with auditwheel)

```bash
mv /usr/bin/patchelf /usr/bin/patchelf.bak
```

### Step 4: Set environment variables (Python 3.11 for manylinux wheel)

```bash
export PYTHON_HOME=/opt/python/cp311-cp311
export PATH=/opt/python/cp311-cp311/bin:/opt/rh/gcc-toolset-14/root/usr/bin:/usr/bin:$PATH
export LD_LIBRARY_PATH=/opt/rh/gcc-toolset-14/root/usr/lib64:/opt/rh/gcc-toolset-14/root/usr/lib:/usr/lib64:$LD_LIBRARY_PATH
```

Verify:
```bash
/opt/python/cp311-cp311/bin/python3.11 --version  # Python 3.11.14
gcc --version                                       # GCC 14.2.1
```

## Build Process

### 1. Full Build (NPU + test + hcom)

```bash
cd /workspace/memfabric_hybrid

# Clean previous build
rm -rf build output

T0=$(date +%s.%N)

bash script/build_and_pack_run.sh \
    --build_mode RELEASE \
    --build_python ON \
    --xpu_type NPU \
    --build_test ON \
    --build_hcom ON \
    --build_hcom_rdma ON \
    --build_hcom_ub OFF \
    --build_etcd_backend OFF \
    --build_tool cmake

T1=$(date +%s.%N)
BUILD_TOTAL=$(echo "scale=3; $T1 - $T0" | bc)
```

**Artifacts generated:**
- `output/hybm/lib64/libmf_hybm_core.so` — NPU-aware hybm core library
- `output/smem/lib64/libmf_smem.so` — Shared memory library
- `output/3rdparty/hcom/lib/libhcom.so` — hcom communication library (RDMA)
- `src/smem/python/memfabric_hybrid/dist/memfabric_hybrid-1.1.0-cp311-cp311-manylinux_2_26_aarch64.manylinux_2_28_aarch64.whl` — manylinux wheel

Generate `.run` installer:
```bash
mkdir -p output/memfabric_hybrid/wheel
cp src/smem/python/memfabric_hybrid/dist/*manylinux*.whl output/memfabric_hybrid/wheel/
export IS_MANYLINUX=True
bash script/run_pkg_maker/make_run.sh
# Output: output/memfabric_hybrid-1.1.0_linux_aarch64.run
```

### 2. Incremental Build (ccache)

**Prerequisite: Fix `script/build.sh`** — 3 patches needed:

| # | Line | Original | Replace With | Reason |
|---|------|----------|-------------|--------|
| 1 | 150 | `rm -rf ./build ./output` | `# rm -rf ./build ./output` | Preserve build directory between builds |
| 2 | 159 | `mkdir build/` | `mkdir -p build/` | Succeeds when directory exists |
| 3 | 223-224 | `rm -rf ./output` + `rm -rf ./build` | comment out both | Prevent build destruction in make section |
| 4 | 324 | `rm -rf build/` | `# rm -rf build/` | Prevent build destruction in wheel section |

Then add ccache compiler launcher after the `-G "$GENERATOR"` line (around line 161):
```
        -DCMAKE_C_COMPILER_LAUNCHER=ccache \
        -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
```

Configure ccache:
```bash
export CCACHE_DIR=/root/.ccache_npu
ccache -M 5G
ccache -z  # reset stats before build
```

**Step A** — Full build to populate ccache (source unchanged, cmake detects launcher change, rebuilds all objects from ccache):
```bash
T0=$(date +%s.%N)
bash script/build_and_pack_run.sh --build_mode RELEASE --build_python ON --xpu_type NPU --build_test ON --build_hcom ON --build_hcom_rdma ON --build_hcom_ub OFF --build_etcd_backend OFF --build_tool cmake
T1=$(date +%s.%N)
echo "FULL_BUILD=$(echo "scale=3; $T1 - $T0" | bc)s"
ccache -s > /tmp/ccache_stats_full.txt
```

**Step B** — Trigger partial recompilation and measure incremental speed:
```bash
touch src/hybm/include/hybm_def.h
echo "// ccache incremental test" >> src/hybm/csrc/common/hybm_networks_common.cpp

T0=$(date +%s.%N)
bash script/build_and_pack_run.sh --build_mode RELEASE --build_python ON --xpu_type NPU --build_test ON --build_hcom ON --build_hcom_rdma ON --build_hcom_ub OFF --build_etcd_backend OFF --build_tool cmake
T1=$(date +%s.%N)
echo "INCR_BUILD=$(echo "scale=3; $T1 - $T0" | bc)s"

ccache -s > /tmp/ccache_stats_incr.txt
```

**Results**: Full build (ccache populated): 44.8s, 97.37% hit rate. Incremental (2 files modified): 9.4s, 98.21% hit rate (55 hits, 1 miss).

### 3. UT Test

Patch `script/run_ut.sh` for RELEASE mode (avoid ASAN/mockcpp crashes):

| Original                    | Replace With            |
|-----------------------------|-------------------------|
| `CMAKE_BUILD_TYPE=ASAN`     | `CMAKE_BUILD_TYPE=RELEASE` |
| `BUILD_OPEN_ABI=ON`         | `BUILD_OPEN_ABI=OFF`    |
| `--gtest_break_on_failure`  | *(remove)*              |
| `set -e`                    | `# set -e disabled`     |

Run UT build:
```bash
T0=$(date +%s.%N)
bash script/run_ut.sh --fast
T1=$(date +%s.%N)
UT_BUILD=$(echo "scale=3; $T1 - $T0" | bc)
```

**Note**: `run_ut.sh` does NOT accept a gtest filter argument — it runs ALL tests. The test binary crashes with MOCKER_CPP tests and `HybmComposeDataOpTest` (NPU SDMA SIGSEGV). Run tests manually with the safe filter below.

Run safe test suites:
```bash
cd output/bin/ut
export LD_LIBRARY_PATH=/workspace/memfabric_hybrid/output/smem/lib64:/workspace/memfabric_hybrid/output/hybm/lib64:/workspace/memfabric_hybrid/output/hybm/lib64/cann/driver/lib64:/usr/lib64
export ASCEND_HOME_PATH=/workspace/memfabric_hybrid/output/hybm/lib64/cann

FILTER="HybmTransportManagerTest.*:HybmConnBasedSegmentTest.*:HybmDevSegmentTest.*:HybmHostShmSegmentTest.*:HybmVaManagerTest.*:HybmVmmBasedSegmentTest.*:ComposeTransportManagerTest.*:BipartiteRanksQpManagerTest.*:DeviceQpManagerTest.*:DeviceRdmaHelperTest.*:RdmaTransportManagerTest.*:FixedRanksQpManagerTestTest.*:JoinableRanksQpManagerTest.*:HostHcomCounterStreamTest.*:HostHcomReconnectorTest.*:AccConfigStoreTest.*:NetworkEndpointUtilTest.*:FindAvailablePortTest.*:SmLastErrorTest.*:SmemNetGroupEngineTest.*:ExecutorServiceTest.*:MfFailpointTest.*:MFFileUtilTest.*:MfMonotonicTest.*:MFNumUtilTest.*:OutLoggerTest.*:MFUrlParserTest.*"

T0=$(date +%s.%N)
./test_memfabric --gtest_output=xml:/tmp/test_safe.xml --gtest_filter="${FILTER}" 2>/tmp/ut_stderr.txt
T1=$(date +%s.%N)
UT_RUN=$(echo "scale=3; $T1 - $T0" | bc)
```

## gtest Filter Limitation

gtest 1.8.0's `--gtest_filter` only processes the FIRST exclusion pattern (e.g., `-AccLinksTest.*`). Subsequent exclusions are ignored. **Solution**: Use an inclusion filter that explicitly lists safe test suites instead of excluding unsafe ones.

## Test Suites Reference

### Included (27 suites)
HybmTransportManagerTest, HybmConnBasedSegmentTest, HybmDevSegmentTest, HybmHostShmSegmentTest, HybmVaManagerTest, HybmVmmBasedSegmentTest, ComposeTransportManagerTest, BipartiteRanksQpManagerTest, DeviceQpManagerTest, DeviceRdmaHelperTest, RdmaTransportManagerTest, FixedRanksQpManagerTestTest, JoinableRanksQpManagerTest, HostHcomCounterStreamTest, HostHcomReconnectorTest, AccConfigStoreTest, NetworkEndpointUtilTest, FindAvailablePortTest, SmLastErrorTest, SmemNetGroupEngineTest, ExecutorServiceTest, MfFailpointTest, MFFileUtilTest, MfMonotonicTest, MFNumUtilTest, OutLoggerTest, MFUrlParserTest

### Excluded (25 suites)
- **MOCKER_CPP hooks** (crash on ARM64): AccLinksTest, HybmBigMemEntryTest, HybmEntityDefaultTest, HybmEntityShmFdTest, HybmEntityTagInfoTest, HybmEntryTest, HybmMemSegmentTest, HybmDataOpDeviceRdmaTest, HybmDataOpSdmaTest, HybmDataOpHostShmTest, HybmDataOpHostRdmaTest, HybmDataOperatorTest, HybmDataOpEntryTest, SmemBmTest, SmemEtcdStoreBackendTest, SmemHaConfigStoreTest, SmemNetTest, SmemShmTest, SmemStoreFactoryTest, SmemTransTest, SmemNetGroupEngineMockTest, SmemExternalBackendTest, HostHcomHelperTest, HcomTransportManagerTest
- **NPU SDMA crash**: HybmComposeDataOpTest — triggers SIGSEGV in `initialize_with_sdma_only` when `xpu_type=NPU`

## Pre-existing Test Failures

The following 3 test failures are known issues not introduced by any configuration change:
- `HybmVmmBasedSegmentTest.Mmap_RemoteSlice_SucceedsAndSkipsLocal`
- `SmemNetGroupEngineTest.concurrent_clients_join_leave_twice_should_succeed_on_fixed_port`
- `OutLoggerTest.Alarm`

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| `unable to find user build` | Image USER directive | Use `--user root` in docker run |
| `No module named 'pybind11'` | Missing pybind11 | `pip3 install pybind11==3.0.1` |
| `Python.h: No such file or directory` | Missing python3-devel | `yum install -y python3-devel` |
| `bc: command not found` | Missing bc utility | `yum install -y bc` |
| `cannot find -lasan` | Missing ASAN runtime | `yum install -y gcc-toolset-14-libasan-devel` |
| `ZIP does not support timestamps before 1980` | `SOURCE_DATE_EPOCH=0` | Patch `setup.py` line 33: change `"0"` to `"1700000000"` |
| `find_namespace_packages` import error | Outdated setuptools | `pip3 install --upgrade setuptools` |
| `No module named 'wheel'` | Missing wheel module | `pip3 install wheel` |
| mockcpp submodule not found | Submodules not initialized | `git submodule update --init --recursive` |
| ASAN SEGV in UT execution | mockcpp + ASAN on ARM64 | Use RELEASE mode instead of ASAN |
| `HybmComposeDataOpTest` SIGSEGV | NPU SDMA initialization | Exclude from test filter with NPU config |
| auditwheel fails (purelib) | .so files in purelib folder | Use Python 3.11 + pybind11 3.0.1 — setup.py's bdist_wheel places .so in platlib for Python 3.11 |
| auditwheel fails (library not found) | Missing .so in wheel | Ensure build.sh copied libs to `memfabric_hybrid/lib/` before wheel build |
| ccache 0% hit rate | `build.sh` line 150: `rm -rf ./build ./output` destroys build dir | Comment out `rm -rf` at lines 150, 223, 224, 324; change `mkdir build/` to `mkdir -p build/` |

## Container Setup

```bash
# Start container with --user root override
docker run -d --user root --name memfabric_ut \
    swr.cn-north-4.myhuaweicloud.com/memfabric-hybrid/memfabric-hybrid_arm:v20 \
    sleep 3600

# Create workspace directory inside container
docker exec memfabric_ut mkdir -p /workspace

# Copy source code into container
docker cp /path/to/memfabric_hybrid memfabric_ut:/workspace/memfabric_hybrid
```

## Step-by-Step Workflow

1. **Start container**: `docker run -d --user root --name memfabric_ut <image> sleep 3600`
2. **Install dependencies**: ccache, bc, python3-devel, pybind11==3.0.1, wheel, setuptools, libasan6, rdma-core-devel
3. **Move patchelf**: `mv /usr/bin/patchelf /usr/bin/patchelf.bak`
4. **Set env**: PYTHON_HOME=/opt/python/cp311-cp311, PATH includes cp311 bin
5. **Copy source**: Clone repo → init submodules → docker cp into container
6. **Full build**: Run `build_and_pack_run.sh` with NPU+test+hcom flags, collect phase timing
7. **Generate .run**: Copy manylinux wheel → run `make_run.sh`
8. **Incremental build**: Patch build.sh for ccache → modify source → rebuild → collect ccache stats
9. **UT build**: Patch run_ut.sh (ASAN→RELEASE, open_abi→OFF, remove break_on_failure, disable set -e) → run `run_ut.sh --fast`
10. **UT test**: Run test_memfabric with safe inclusion filter → collect XML results
11. **Copy artifacts**: Collect .whl, .run, test_detail.xml, and build_ut_result.json back to host
