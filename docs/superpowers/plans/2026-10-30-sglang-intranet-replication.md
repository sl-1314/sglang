# sglang 内网复刻 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在完全独立的全新隔离环境（`/opt/sglang*`）里把 sglang（钉死 `4349538c02`）跑起来（server + 一次推理），全过程零污染当前机器默认环境，**每执行一次安装/修改操作立即在记录文档中追加一条流水**，最终把验证过的命令序列固化为单阶段 `Dockerfile.dev` / `Dockerfile.serving`。

**Architecture:** 全程在当前构建容器（Ubuntu 22.04.5 + CUDA 12.9 + 4×H20-3e）内的独立前缀目录 `/opt/sglang*` 下手工执行、逐段验证、**单步同步记录**；每个阶段的已验证命令直接成为 Dockerfile 的行。容器内无 docker daemon，Dockerfile 本身不可构建验证——以"每行命令都被亲手执行过并有流水号可查"作为正确性来源。

**Tech Stack:** 内网 `nvidia-cuda:12.9.1-devel-ubuntu22.04`、conda（仅用 `/usr/local/miniconda3/bin/conda` 创建独立 `-p /opt/sglang` 环境，不动 `base`）、pip + PyPI（走代理）、`torch==2.13.0+cu129` wheel 链、`sglang_kernel` cu129 wheel（GitHub Releases）、`flashinfer_python[cu12]`、Rust（独立安装到 `/opt/sglang-rust`）、`Qwen/Qwen3-0.6B`（HF，走代理）。

**Spec:** `docs/superpowers/specs/2026-10-30-sglang-intranet-image-replication-design.md`

---

## Global Constraints & Isolation Rules（硬约束与零污染纪律）

1. **base 固定**：`registry-offlinebiz.sankuai.com/custom_prod/com.sankuai.data.hadoop.gpu/nvidia-cuda:12.9.1-devel-ubuntu22.04`（imageId 642392771），不换。
2. **sglang 钉死 `4349538c02`**（2026-09-04，最后含 cu12 构建路径的提交）；`torch==2.13.0+cu129`、`requires-python >= 3.10`。
3. **彻底环境隔离（绝不污染默认环境与 `$HOME`，实体落盘 `/tmp/sglang-isolated/` 5.8TB NVMe + `/opt/sglang*` 软链）**：
   - 物理存储根目录：`/tmp/sglang-isolated/{env,src,rust,cache}`（位于 `/dev/nvme1n1p1`，可用 5.8 TB，不占 `/` 的 46 GB overlay 空间）
   - 软链映射（保持与最终 Dockerfile 路径 100% 一致）：
     - `/opt/sglang` → `/tmp/sglang-isolated/env`
     - `/opt/sglang-src` → `/tmp/sglang-isolated/src`
     - `/opt/sglang-rust` → `/tmp/sglang-isolated/rust`
     - `/opt/sglang-cache` → `/tmp/sglang-isolated/cache`
   - Python/conda 环境：`/opt/sglang`（`conda create -y -p /opt/sglang python=3.10`，全程以 `/opt/sglang/bin/python`、`/opt/sglang/bin/pip` 绝对路径调用，绝不修改 `/usr/local/miniconda3` base 环境或系统 Python）
   - sglang 独立源码副本：`/opt/sglang-src`（从本地 `/home/hadoop-djst-algoplat/sglang` 克隆并 checkout `4349538c02`；所有对 `pyproject.toml` 的修改仅在 `/opt/sglang-src` 内进行）
   - Rust 独立工具链：`/opt/sglang-rust`（`CARGO_HOME=/opt/sglang-rust/cargo`、`RUSTUP_HOME=/opt/sglang-rust/rustup`，加 `--no-modify-path`，绝不写 `~/.cargo`、`~/.rustup`、`~/.bashrc`）
   - 独立缓存目录：`/opt/sglang-cache/{pip,conda,hf}`（显式导出 `PIP_CACHE_DIR`、`CONDA_PKGS_DIRS`、`HF_HOME`，不写 `~/.cache`）
4. **无免密 sudo 下的工具策略**：系统已自带 `gcc`/`g++`/`cmake`/`ninja`/`git`/`libnuma1`；缺少的 `protoc`（`sglang-grpc` Rust 扩展编译必需）通过 `conda install -y -p /opt/sglang -c conda-forge libprotobuf` 装入独立环境 `/opt/sglang/bin/protoc`（对应 Dockerfile 中记录为 `apt-get install -y protobuf-compiler` 并在差异表中注明原因）。
5. **每步必记纪律（Execute-One-Record-One）**：
   - 任何安装、文件修改、配置变更操作，**每执行一次，必须在同一回合内立刻向 `docker/intranet/build-notes.md` 的「操作流水记录」追加一条**（含流水号 `#01, #02...`、阶段、目的、完整命令、结果/版本、失败与修复过程）。禁止攒多步后再补记。
6. **代理**：`http://10.177.8.11:3128`（`http_proxy` / `https_proxy`）；所有外网命令显式带代理。
7. **单阶段 Dockerfile**：无 BuildKit、无多阶段、无 cache mount（对齐 mlp `image create --components`）。
8. **反目标**：Dockerfile 里出现没有亲手跑过、在 `build-notes.md` 中找不到对应流水号的行；静默 hack 不记录。
9. **交付物落位**：`/home/hadoop-djst-algoplat/sglang/docker/intranet/`（`build-notes.md`、`versions.md`、`Dockerfile.serving`、`Dockerfile.dev`），新分支 `intranet-replication`。

---

## Review Focus & 对照 `4349538c02` 已核实的四个关键修正

1. **`pyproject.toml` 在 `4349538c02` 默认写的是 `cu13` 依赖**：
   - 原文件含 `cuda-python>=13.0`、`flashinfer_python[cu13]==0.6.18`、`nvidia-cutlass-dsl[cu13]==4.6.2`、`humming-kernels[cu13]==0.1.12`，以及单机推理跳过的 `sgl-deep-ep==0.1.2`、`sgl-deep-gemm==0.1.7`。
   - 对齐上游 Dockerfile cu12 分支做法 + 单机裁剪决策：在 `/opt/sglang-src/python/pyproject.toml` 中将 `cu13` 改写为 `cu12` 对应项，并注释掉 `sgl-deep-ep` / `sgl-deep-gemm`（及 `humming-kernels[cu13]` 若无 cu12 对应则核查或按需裁剪），全过程记入流水与差异表。
2. **`sglang-kernel` 的 cu129 wheel 来自 GitHub Releases 而非 PyPI**：
   - PyPI 上的 `sglang-kernel==0.4.6.post1` 是 cu13 版；cu129 必须按上游 Dockerfile 第 207 行安装：
     `https://github.com/sgl-project/whl/releases/download/v0.4.6.post1/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl`（`cp310-abi3`，兼容 Python 3.10）。
3. **Editable 安装必须加 `--no-build-isolation` 并具备 `protoc`**：
   - 否则 pip 会在隔离环境中从默认 PyPI 重新拉 cu13 版 `torch==2.13.0`。先在 `/opt/sglang` 装好 `setuptools>=61.0 setuptools-rust>=1.11 setuptools-scm>=8.0 wheel` 与 `protoc`，再执行 `pip install --no-build-isolation --no-deps -e python/`。
4. **`kernels` cubin 预热命令与 cu129 预期行为**：
   - 正确命令是 `/opt/sglang/bin/kernels lock python && /opt/sglang/bin/kernels download python`；按上游 Dockerfile 第 654 行注释，cu129 无匹配的预编译 `sgl-flash-attn3` cubin，预期回退运行时 JIT。

---

### Task 1: 建分支、初始化记录文档与环境基线核查（S0）

**Files:**
- Create: `docker/intranet/build-notes.md`（分支 `intranet-replication`）

- [x] **Step 1: 切新分支并建立 `docker/intranet/build-notes.md` 骨架**

```bash
cd /home/hadoop-djst-algoplat/sglang && git switch -c intranet-replication
mkdir -p docker/intranet
```

`build-notes.md` 包含五个核心区块：
1. 环境基线（OS / GPU / 驱动 / CUDA / 磁盘 / 代理 / 系统自带工具）
2. 隔离目录约定表（`/opt/sglang`、`/opt/sglang-src`、`/opt/sglang-rust`、`/opt/sglang-cache`）
3. **操作流水记录（Operation Log，每执行一步追加一条 `#01, #02...`）**
4. 踩坑与排障记录
5. 与上游 `4349538c02:docker/Dockerfile` 差异表 & MLP components 映射

- [x] **Step 2: 核查 S0 基线（只读）并记入流水 `#01`、`#02`**

```bash
nvidia-smi
df -h / /home
curl -fsS -x http://10.177.8.11:3128 https://pypi.org/simple/ -o /dev/null && echo PYPI_OK
curl -fsS -x http://10.177.8.11:3128 https://github.com -o /dev/null && echo GITHUB_OK
curl -fsS -x http://10.177.8.11:3128 https://huggingface.co -o /dev/null && echo HF_OK
```

每跑完立即将实测结果写入 `build-notes.md`。

- [x] **Step 3: 创建隔离缓存目录 `/opt/sglang-cache` 并记入流水**

```bash
mkdir -p /opt/sglang-cache/{pip,conda,hf}
```

立即记录到 `build-notes.md`。

---

### Task 2: 独立 Python 环境与编译工具准备（S1 & S2）

**Files:**
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: 创建独立 conda 环境 `/opt/sglang`（Python 3.10）→ 立即记录流水**

```bash
CONDA_PKGS_DIRS=/opt/sglang-cache/conda \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/usr/local/miniconda3/bin/conda create -y -p /opt/sglang python=3.10
/opt/sglang/bin/python -V
```

- [x] **Step 2: 在 `/opt/sglang` 内安装 `protoc`（替代 `apt install protobuf-compiler`）→ 立即记录流水**

```bash
CONDA_PKGS_DIRS=/opt/sglang-cache/conda \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/usr/local/miniconda3/bin/conda install -y -p /opt/sglang -c conda-forge libprotobuf
/opt/sglang/bin/protoc --version
```

（若 conda-forge 渠道慢或受限，备选：从 GitHub Releases 下载 `protoc-*-linux-x86_64.zip` 解压到 `/opt/sglang/bin/protoc`，同样完全隔离且立即记录。）

- [x] **Step 3: 升级 `/opt/sglang` 内的 `pip` / `setuptools` / `wheel` / `setuptools-rust` / `setuptools-scm` → 立即记录流水**

```bash
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install --upgrade "pip" "setuptools>=61.0" "setuptools-rust>=1.11" "setuptools-scm>=8.0" "wheel" "html5lib" "six"
```

---

### Task 3: 独立 Rust 工具链安装（S3 前置）

**Files:**
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: 安装 Rust 到 `/opt/sglang-rust`（不碰 `$HOME`）→ 立即记录流水**

```bash
mkdir -p /opt/sglang-rust
CARGO_HOME=/opt/sglang-rust/cargo \
RUSTUP_HOME=/opt/sglang-rust/rustup \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
sh -c 'curl --proto "=https" --tlsv1.2 --retry 3 -sSf https://sh.rustup.rs | sh -s -- -y --no-modify-path --profile minimal --default-toolchain stable'
/opt/sglang-rust/cargo/bin/rustc --version && /opt/sglang-rust/cargo/bin/cargo --version
```

（需 `rustc >= 1.85` 以支持 `sglang-grpc` 的 Rust edition 2024。）

---

### Task 4: 独立源码副本与核心 GPU 依赖分步安装（S3 主体）

**Files:**
- Create: `docker/intranet/versions.md`
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: 从本地仓库克隆独立源码到 `/opt/sglang-src` 并切到 `4349538c02` → 立即记录流水**

```bash
git clone /home/hadoop-djst-algoplat/sglang /opt/sglang-src
git -C /opt/sglang-src checkout 4349538c02
git -C /opt/sglang-src rev-parse HEAD
```

- [x] **Step 2: 整理 `versions.md` 与系统依赖裁剪表 → 立即记录流水**

对照 `4349538c02` 的 `python/pyproject.toml` 与 `docker/Dockerfile` 生成 `docker/intranet/versions.md`，并把上游 `base` 段 apt 包裁剪判定表写入 `build-notes.md`。

- [x] **Step 3: 单独安装 `torch==2.13.0` + `torchvision==0.28.0` + `torchaudio==2.11.0`（cu129）并验证 CUDA/cuDNN → 立即记录流水**

```bash
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install --index-url https://download.pytorch.org/whl/cu129 \
    torch==2.13.0 torchvision==0.28.0 torchaudio==2.11.0
/opt/sglang/bin/python -c "import torch; print(torch.__version__, 'cuda:', torch.cuda.is_available(), 'cudnn:', torch.backends.cudnn.is_available(), 'device:', torch.cuda.get_device_name(0))"
```

- [x] **Step 4: 单独安装 `sglang-kernel`（cu129 GitHub wheel）并验证 → 立即记录流水**

```bash
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install --no-deps --force-reinstall \
    https://github.com/sgl-project/whl/releases/download/v0.4.6.post1/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl
/opt/sglang/bin/python -c "import sgl_kernel; print('sgl_kernel ok')"
```

- [x] **Step 5: 单独安装 `flashinfer_python[cu12]==0.6.18` 与 `flashinfer-cubin==0.6.18` 并验证 → 立即记录流水**

```bash
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install "flashinfer_python[cu12]==0.6.18"
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install "flashinfer-cubin==0.6.18" --index-url https://flashinfer.ai/whl
/opt/sglang/bin/python -c "import flashinfer; print('flashinfer:', getattr(flashinfer, '__version__', 'ok'))"
```

- [x] **Step 6: 在 `/opt/sglang-src/python/pyproject.toml` 中执行 cu13→cu12 改写与单机裁剪，并安装其余依赖 → 立即记录流水**

對 `/opt/sglang-src/python/pyproject.toml`：
1. 按上游 Dockerfile 改写：`cuda-python>=13.0` → `cuda-python>=12,<13`、`flashinfer_python[cu13]` → `flashinfer_python[cu12]`、`nvidia-cutlass-dsl[cu13]` → `nvidia-cutlass-dsl`；
2. 注释掉单机跳过的 `sgl-deep-ep` / `sgl-deep-gemm`（以及检查 `humming-kernels[cu13]` 在 cu12 下的处理方式）；
3. 安装其余依赖并确认没有 `-cu13` 包混入（若有则按上游 Dockerfile 卸载并保持 cu129 torch）。
每一步修改和安装都立即记录 `git diff` 与结果到 `build-notes.md`。

---

### Task 5: sglang 源码 Editable 安装与编译验证（S4）

**Files:**
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: `--no-build-isolation --no-deps -e python/` 安装（含 Rust 扩展编译）→ 立即记录流水**

```bash
cd /opt/sglang-src
PATH="/opt/sglang/bin:/opt/sglang-rust/cargo/bin:$PATH" \
CARGO_HOME=/opt/sglang-rust/cargo \
RUSTUP_HOME=/opt/sglang-rust/rustup \
PIP_CACHE_DIR=/opt/sglang-cache/pip \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/pip install --no-build-isolation --no-deps -e python/
```

- [x] **Step 2: 验证 `import sglang` 与 `launch_server --help` → 立即记录流水**

```bash
/opt/sglang/bin/python -c "import sglang; print(sglang.__version__)"
/opt/sglang/bin/python -m sglang.launch_server --help | head -10
```

- [x] **Step 3: 验证 `kernels lock/download` 行为（预期 cu129 fallback JIT）→ 立即记录流水**

```bash
PATH="/opt/sglang/bin:$PATH" \
HF_HOME=/opt/sglang-cache/hf \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/kernels lock /opt/sglang-src/python || true
```

---

### Task 6: Server 起服与真实推理验收（S5）

**Files:**
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: 下载 `Qwen/Qwen3-0.6B` 到 `/opt/sglang-cache/hf` → 立即记录流水**

```bash
HF_HOME=/opt/sglang-cache/hf \
http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
/opt/sglang/bin/python -c "from huggingface_hub import snapshot_download; print(snapshot_download('Qwen/Qwen3-0.6B'))"
```

- [x] **Step 2: 启动 sglang server（绑定单卡 GPU 0）+ 健康检查 + 推理请求 → 立即记录流水**

```bash
CUDA_VISIBLE_DEVICES=0 HF_HOME=/opt/sglang-cache/hf \
/opt/sglang/bin/python -m sglang.launch_server \
    --model-path Qwen/Qwen3-0.6B --host 127.0.0.1 --port 30000 > /tmp/sglang_server.log 2>&1 &
```

等待就绪后执行 `/health` 与 `/generate` 请求，把完整响应写入 `build-notes.md`，随后停止 server 进程。

---

### Task 7: 固化 `Dockerfile.serving` / `Dockerfile.dev` 与 MLP 映射（S6）

**Files:**
- Create: `docker/intranet/Dockerfile.serving`
- Create: `docker/intranet/Dockerfile.dev`
- Modify: `docker/intranet/build-notes.md`

- [x] **Step 1: 从 `build-notes.md` 操作流水逐行导出 `Dockerfile.serving` 与 `Dockerfile.dev`**（每条指令标注对应的流水号 `#XX`）
- [x] **Step 2: 反目标自检（逐行核对是否有未亲手跑过的行）并补齐上游差异表与 MLP `components` 映射**
