# gem5 x86 — First Project (SE Mode)

> **Goal:** Stand up a clean, reproducible gem5 x86 setup on Ubuntu 24.04 using a Python virtualenv, run a minimal SE-mode “hello world,” then run a few **mini-experiments** (CPU model + cache size sweeps) and capture stats/figures for a professional portfolio write-up.

---

## Table of Contents
   [Repository Layout](#repository-layout)
1. [What I Build](#what-i-build)
2. [Prerequisites](#prerequisites)
3. [Quick Start](#quick-start)
4. [Detailed Setup](#detailed-setup)

   * [4.1 System Packages (Ubuntu 24.04)](#41-system-packages-ubuntu-2404)
   * [4.2 Workspace + Virtualenv](#42-workspace--virtualenv)
   * [4.3 Clone gem5](#43-clone-gem5)
   * [4.4 SCons Pin (fixes configure probe)](#44-scons-pin-fixes-configure-probe)
   * [4.5 Build gem5 (X86)](#45-build-gem5-x86)
   * [4.6 Smoke Test (modern tutorial)](#46-smoke-test-modern-tutorial)
   * [4.7 Standard Library “Hello” Script](#47-standard-library-hello-script)
5. [Mini-Experiments](#mini-experiments)

   * [5.1 CPU Model Sweep](#51-cpu-model-sweep)
   * [5.2 L1 Cache Size Sweep](#52-l1-cache-size-sweep)
   * [5.3 Memory Latency Stress (DDR variants)](#53-memory-latency-stress-ddr-variants)
   * [5.4 Workload Character: Compute vs Memory Bound](#54-workload-character-compute-vs-memory-bound)
6. [Collecting & Comparing Stats](#collecting--comparing-stats)
7. [Repository Layout](#repository-layout)
8. [Troubleshooting](#troubleshooting)
9. [What to Screenshot (Portfolio Proof)](#what-to-screenshot-portfolio-proof)


---

## Repository Layout

```
repo-root/
├─ my/
│  ├─ 01_se_hello_stdlib.py
│  └─ 02_param_stdlib.py
├─ tools/
│  └─ harvest.sh
├─ out/                 # experiment outputs via --outdir
├─ Images/              # screenshots/figures for the README
└─ gem5/                # gem5 source 
```


---

## What I Build

* **Clean gem5 environment** on Ubuntu 24.04 using a **Python venv**.
* **X86 SE-mode** run of a built-in hello binary.
* Two standard-library configs:

  * `my/01_se_hello_stdlib.py` — simple TimingSimpleCPU + L1/L2.
  * `my/02_param_stdlib.py` — configurable CPU (Atomic/Timing/O3) and cache sizes.
* **Mini-experiments** with reproducible `--outdir` runs and simple scripts to harvest key metrics from `stats.txt`.

---

## Prerequisites

* Ubuntu 24.04 (or 22.04) with sudo access
* \~10–15 GB free disk space, stable network

> **Image placeholders:**
>
> * *System info screenshot* → ![Host Info](Images/Hardware.png)

---

## Quick Start

```bash
# 1) System packages (Ubuntu 24.04)
sudo apt update
sudo apt install -y \
  build-essential scons python3-dev git pre-commit zlib1g zlib1g-dev \
  libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
  libboost-all-dev libhdf5-serial-dev python3-pydot python3-venv python3-tk mypy \
  m4 libcapstone-dev libpng-dev libelf-dev pkg-config wget cmake doxygen

# 2) Workspace + venv
mkdir -p ~/sim/gem5-x86 && cd ~/sim/gem5-x86
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel

# 3) Pin SCons (resolves a configure-probe quirk on some hosts)
pip install "SCons==4.5.2" mako pyyaml pybind11

# 4) Get gem5 and build X86
git clone https://gem5.googlesource.com/public/gem5
cd gem5
scons -j"$(nproc)" --config=force build/X86/gem5.opt

# 5) Smoke test (modern tutorial config)
./build/X86/gem5.opt configs/learning_gem5/part1/simple.py
```

Expected console output includes the hello message and a finished simulation. Stats appear in `m5out/stats.txt`.

> * *Smoke test run* → ![Smoke Test Console](Images/SmokeTest.png)

---

## Detailed Setup

### 4.1 System Packages (Ubuntu 24.04)

See the [Quick Start](#quick-start) step (same commands). This installs compilers, SCons, protobuf (optional), Boost (optional), etc.

### 4.2 Workspace + Virtualenv

```bash
mkdir -p ~/sim/gem5-x86 && cd ~/sim/gem5-x86
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel
```

### 4.3 Clone gem5

```bash
cd ~/sim/gem5-x86
git clone https://gem5.googlesource.com/public/gem5
cd gem5
# Optional: pick a stable tag
# git fetch --tags && git checkout v24.0.0.0
```

### 4.4 SCons Pin (fixes configure probe)

Some host setups trip over a configure probe when using newer SCons. Pinning to **SCons 4.5.2** inside the venv has been reliable.

```bash
source ~/sim/gem5-x86/.venv/bin/activate
pip install --upgrade "SCons==4.5.2" mako pyyaml pybind11
scons --version  # should report 4.5.2
```

### 4.5 Build gem5 (X86)

```bash
scons -j"$(nproc)" --config=force build/X86/gem5.opt
```

> If you see a pre-commit prompt, press **Enter** to let it install hooks (not required for building, but harmless).

### 4.6 Smoke Test (modern tutorial)

`se.py` is deprecated. Use the maintained tutorial config:

```bash
./build/X86/gem5.opt configs/learning_gem5/part1/simple.py
```

You should see the hello-world message, and `m5out/stats.txt` will be created.

### 4.7 Standard Library “Hello” Script

Create `my/01_se_hello_stdlib.py` (TimingSimpleCPU + L1/L2 caches):

```python
# my/01_se_hello_stdlib.py
from pathlib import Path
from gem5.isas import ISA
from gem5.components.processors.cpu_types import CPUTypes
from gem5.components.processors.simple_processor import SimpleProcessor
from gem5.components.cachehierarchies.classic.private_l1_private_l2_cache_hierarchy import \
    PrivateL1PrivateL2CacheHierarchy
from gem5.components.memory.single_channel import SingleChannelDDR3_1600
from gem5.components.boards.x86_board import X86Board
from gem5.simulate.simulator import Simulator

cache = PrivateL1PrivateL2CacheHierarchy(
    l1i_size="32kB", l1i_assoc=2,
    l1d_size="32kB", l1d_assoc=2,
    l2_size="256kB", l2_assoc=8,
)

processor = SimpleProcessor(cpu_type=CPUTypes.TIMING, num_cores=1, isa=ISA.X86)
memory = SingleChannelDDR3_1600(size="512MB")
board = X86Board(clk_freq="1GHz", processor=processor, memory=memory, cache_hierarchy=cache)

hello = Path("tests/test-progs/hello/bin/x86/linux/hello").resolve()
board.set_se_binary_workload(hello)

sim = Simulator(board=board)
sim.run()
```

Run it with an explicit results directory:

```bash
./build/X86/gem5.opt my/01_se_hello_stdlib.py --outdir=out/timing_l1l2
```

---

## Mini-Experiments

All experiments write to unique `--outdir` folders for clean comparisons.

Create `my/02_param_stdlib.py`:

```python
# my/02_param_stdlib.py
import argparse
from pathlib import Path
from gem5.isas import ISA
from gem5.components.processors.cpu_types import CPUTypes
from gem5.components.processors.simple_processor import SimpleProcessor
from gem5.components.cachehierarchies.classic.no_cache import NoCache
from gem5.components.cachehierarchies.classic.private_l1_private_l2_cache_hierarchy import \
    PrivateL1PrivateL2CacheHierarchy
from gem5.components.memory.single_channel import SingleChannelDDR3_1600
from gem5.components.boards.x86_board import X86Board
from gem5.simulate.simulator import Simulator

CPU_MAP = {"atomic": CPUTypes.ATOMIC, "timing": CPUTypes.TIMING, "o3": CPUTypes.O3}

p = argparse.ArgumentParser()
p.add_argument("--cpu", choices=CPU_MAP.keys(), default="timing")
p.add_argument("--l1i", default="32kB")
p.add_argument("--l1d", default="32kB")
p.add_argument("--l2",  default="256kB")
p.add_argument("--nocache", action="store_true")
args = p.parse_args()

cache = NoCache() if args.nocache else PrivateL1PrivateL2CacheHierarchy(
    l1i_size=args.l1i, l1i_assoc=2,
    l1d_size=args.l1d, l1d_assoc=2,
    l2_size=args.l2,  l2_assoc=8,
)

processor = SimpleProcessor(cpu_type=CPU_MAP[args.cpu], num_cores=1, isa=ISA.X86)
memory = SingleChannelDDR3_1600(size="512MB")
board = X86Board(clk_freq="1GHz", processor=processor, memory=memory, cache_hierarchy=cache)

binary = Path("tests/test-progs/hello/bin/x86/linux/hello").resolve()
board.set_se_binary_workload(binary)

sim = Simulator(board=board)
sim.run()
```

