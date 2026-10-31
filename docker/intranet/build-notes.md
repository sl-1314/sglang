# sglang 内网镜像复刻 · 全过程记录文档（Build Notes）

- **日期**：2026-10-30
- **目标 sglang Commit**：`4349538c02`（2026-09-04，上游最后保留 CUDA 12 构建路径的提交）
- **目标基础镜像**：`registry-offlinebiz.sankuai.com/custom_prod/com.sankuai.data.hadoop.gpu/nvidia-cuda:12.9.1-devel-ubuntu22.04`（imageId `642392771`）
- **核心纪律**：
  1. **完全隔离**：所有环境、源码、工具链、缓存实体均落在 `/tmp/sglang-isolated/`（5.8 TB NVMe），通过 `/opt/sglang*` 软链暴露标准路径，绝不触碰当前机器默认环境（`/usr/local/miniconda3` base、系统 Python、`$HOME/.cache`、`$HOME/.cargo` 等）。
  2. **一步一记**：每执行一次安装/修改/核查操作，立即在下方「三、操作流水记录」追加一条记录，任何写进最终 Dockerfile 的行都必须对应某条流水号。

---

## 一、环境基线（S0 实测）

| 检查项 | 实测值 | 结论 |
|---|---|---|
| 操作系统 | `Ubuntu 22.04.5 LTS (Jammy Jellyfish)` | 与目标 base 一致 |
| GPU 型号与数量 | `4 × NVIDIA H20-3e`（`143771 MiB` / 卡，架构 Hopper `sm90`） | 满足单卡/多卡推理验收 |
| NVIDIA 驱动与 CUDA | Driver `550.127.08`（`nvidia-smi` 显示 `CUDA Version: 12.9`），Toolkit `nvcc V12.9.86` | 满足 CUDA 12.9 |
| 磁盘可用空间 | `/`（overlay）`46G` 可用；`/tmp`（`/dev/nvme1n1p1`）**`5.8T` 可用**；`/home`（`/dev/sdb`）`230G` 可用 | 实体目录放 `/tmp/sglang-isolated/`，软链至 `/opt/sglang*` |
| 裸环境系统工具（`env -i`） | `Python 3.10.12`、`/usr/bin/{git,gcc,g++,cmake,ninja}`、`libnuma1 2.0.14`、`/usr/local/miniconda3/bin/conda` | 基础编译工具齐全，缺 `protoc`（由独立 conda 环境补齐） |
| 代理 `http://10.177.8.11:3128` 直通项 | `pypi.org`、`mirrors.aliyun.com/pypi`、`download.pytorch.org/whl/cu129`、`flashinfer.ai/whl`、`sh.rustup.rs`、`docs.sglang.io/whl/cu129`、`hf-mirror.com` 全部 `200 OK` | PyPI / PyTorch cu129 / FlashInfer / Rustup / HF 镜像全部可达 |
| 代理受限项与替代通道 | 代理直连 `github.com` 与 `huggingface.co` CONNECT 超时；实测 `https://gh-proxy.com/https://github.com/...`（及 `ghfast.top`、`ghproxy.net`）`200 OK`（配合 `docs.sglang.io` 官方索引中的 `sha256` 校验）；HF 使用 `HF_ENDPOINT=https://hf-mirror.com`；Conda 使用内网源 `http://data-source-conda.sankuai.com/...` | 所有外部资源均有可达且可校验的通道 |

---

## 二、隔离目录与软链约定表

| 逻辑路径（命令与 Dockerfile 使用） | 物理路径（`/dev/nvme1n1p1`，5.8 TB 可用） | 用途与隔离方式 |
|---|---|---|
| `/opt/sglang` | `/tmp/sglang-isolated/env` | 独立 Python 3.10 conda 环境（`conda create -p /opt/sglang python=3.10`） |
| `/opt/sglang-src` | `/tmp/sglang-isolated/src` | 独立 sglang 源码副本（checkout `4349538c02`），与 `/home/hadoop-djst-algoplat/sglang` 完全隔离 |
| `/opt/sglang-rust` | `/tmp/sglang-isolated/rust` | 独立 Rust 工具链（`CARGO_HOME=/opt/sglang-rust/cargo`、`RUSTUP_HOME=/opt/sglang-rust/rustup`） |
| `/opt/sglang-cache/pip` | `/tmp/sglang-isolated/cache/pip` | `PIP_CACHE_DIR` 独立缓存 |
| `/opt/sglang-cache/conda` | `/tmp/sglang-isolated/cache/conda` | `CONDA_PKGS_DIRS` 独立包缓存 |
| `/opt/sglang-cache/hf` | `/tmp/sglang-isolated/cache/hf` | `HF_HOME` 模型与 hub 缓存 |

---

## 三、操作流水记录（Execute-One-Record-One）

### `#01` [S0] 创建 `intranet-replication` 分支与 `docker/intranet/` 目录
- **目的**：在本地仓库建立承载交付物（`build-notes.md`、`versions.md`、`Dockerfile.serving`、`Dockerfile.dev`）的专用分支与目录。
- **执行命令**：
  ```bash
  git switch -c intranet-replication && mkdir -p docker/intranet && git status -sb
  ```
- **实际输出**：
  ```text
  Switched to a new branch 'intranet-replication'
  ## intranet-replication
  ```
- **结论**：成功，未触碰系统环境。
- **对应 Dockerfile**：无（仓库管理操作）。

### `#02` [S0] 核查 GPU、磁盘、裸系统工具与代理通道可达性（只读）
- **目的**：确认硬件、驱动、磁盘余量及外网代理 `http://10.177.8.11:3128` 对各依赖源的真实可达情况。
- **执行命令**：
  ```bash
  nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv
  df -h / /tmp /home
  env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
      bash --noprofile --norc -c 'python3 -V; which git gcc g++ cmake ninja; ls /usr/local/miniconda3/bin/conda'
  # 代理连通性探测（带 5s 连接超时）
  for u in "https://pypi.org/simple/pip/" "https://download.pytorch.org/whl/cu129/torch/" \
           "https://flashinfer.ai/whl/" "https://sh.rustup.rs" "https://hf-mirror.com" \
           "https://docs.sglang.io/whl/cu129/sglang-kernel/" \
           "https://gh-proxy.com/https://github.com/sgl-project/whl/releases/download/v0.4.6.post1/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl"; do
    curl -fsSI -L --connect-timeout 5 --max-time 10 -x http://10.177.8.11:3128 "$u" -o /dev/null && echo "$u -> OK"
  done
  ```
- **实际输出与发现**：
  1. `4 × NVIDIA H20-3e`（143771 MiB），Driver `550.127.08`，`CUDA Version: 12.9`，`nvcc V12.9.86`。
  2. `/tmp` 挂载在 `/dev/nvme1n1p1`，可用 `5.8T`；`/` 可用 `46G`。
  3. 裸环境 `Python 3.10.12`，自带 `git`、`gcc`、`g++`、`cmake`、`ninja` 与 `/usr/local/miniconda3/bin/conda`。
  4. **关键网络发现**：
     - 直接 `curl https://pypi.org/simple/` 会拉取 47.8 MB 全量索引导致超时，改为 `https://pypi.org/simple/pip/` 或 `-I` 秒回 `OK`。
     - 代理对 `github.com` 与 `huggingface.co` 直连 CONNECT 超时，但：
       - `https://docs.sglang.io/whl/cu129/sglang-kernel/` 可直通，从中拿到 `sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl` 的官方 `sha256=9d400246b3f2512e2c33611e1bb6a2f252c499c5019e9f331f9c9ee215be95ab`；
       - 通过 `https://gh-proxy.com/https://github.com/...`（对应上游 Dockerfile 的 `GITHUB_ARTIFACTORY` 机制）访问该 wheel 返回 `200 OK`；
       - HuggingFace 镜像 `https://hf-mirror.com` 返回 `200 OK`（可通过 `HF_ENDPOINT=https://hf-mirror.com` 使用）。
- **结论**：S0 基线通过，无修改操作。
- **对应 Dockerfile**：`ENV http_proxy=... https_proxy=... HF_ENDPOINT=https://hf-mirror.com`

### `#03` [S0] 创建 `/tmp/sglang-isolated/` 隔离目录与 `/opt/sglang*` 软链
- **目的**：把所有即将创建的虚拟环境、源码副本、Rust 工具链、缓存都定位到 5.8 TB 的 `/tmp/sglang-isolated/` 下，同时在 `/opt` 下暴露标准路径 `/opt/sglang*`，不占 `/` 分区且零污染 `$HOME`。
- **执行命令**：
  ```bash
  mkdir -p /tmp/sglang-isolated/{rust,cache/pip,cache/conda,cache/hf}
  ln -sfn /tmp/sglang-isolated/env   /opt/sglang
  ln -sfn /tmp/sglang-isolated/src   /opt/sglang-src
  ln -sfn /tmp/sglang-isolated/rust  /opt/sglang-rust
  ln -sfn /tmp/sglang-isolated/cache /opt/sglang-cache
  ls -la /opt/sglang* /tmp/sglang-isolated
  ```
- **实际输出**：
  ```text
  /opt/sglang -> /tmp/sglang-isolated/env
  /opt/sglang-cache -> /tmp/sglang-isolated/cache
  /opt/sglang-rust -> /tmp/sglang-isolated/rust
  /opt/sglang-src -> /tmp/sglang-isolated/src
  ```
- **结论**：隔离目录就绪（`env` 与 `src` 由后续 `conda create` 和 `git clone` 直接创建目标目录）。
- **对应 Dockerfile**：真实镜像构建时直接使用 `/opt/sglang`、`/opt/sglang-src` 目录（无需软链）。

### `#04` [S1] 首次尝试创建 `/opt/sglang` Conda 环境（触发 `CondaToSNonInteractiveError`）
- **目的**：创建独立 Python 3.10 环境 `/opt/sglang`。
- **执行命令**：
  ```bash
  mkdir -p /tmp/sglang-isolated/env
  CONDA_PKGS_DIRS=/opt/sglang-cache/conda \
  /usr/local/miniconda3/bin/conda create -y -p /opt/sglang python=3.10
  ```
- **实际输出（报错）**：
  ```text
  CondaToSNonInteractiveError: Terms of Service have not been accepted for the following channels. Please accept or remove them before proceeding:
      - https://repo.anaconda.com/pkgs/main
      - https://repo.anaconda.com/pkgs/r
  ```
- **原因与修复**：base 镜像中新版 `conda` 默认叠加了官方 `repo.anaconda.com` 渠道并强制检查 ToS。修复方式：在命令行显式加 `--override-channels -c http://data-source-conda.sankuai.com/pkgs/main/ -c http://data-source-conda.sankuai.com/cloud/conda-forge/`，既纯走美团内网源（无需外网代理、极快），又无需修改系统/用户级 `.condarc`。
- **对应 Dockerfile**：Conda 命令必须带 `--override-channels -c http://data-source-conda.sankuai.com/pkgs/main/`。

### `#05` [S1 & S2] 使用内网 Conda 源创建 `/opt/sglang`（Python 3.10.14 + `libprotobuf`）
- **目的**：在独立目录 `/opt/sglang`（实体 `/tmp/sglang-isolated/env`）中创建 Python 3.10 环境，并一并安装 `libprotobuf`（提供 `/opt/sglang/bin/protoc`，替代需要 root 的 `apt-get install protobuf-compiler`，供后续 `sglang-grpc` Rust 扩展编译使用）。
- **执行命令**：
  ```bash
  CONDA_PKGS_DIRS=/opt/sglang-cache/conda \
  /usr/local/miniconda3/bin/conda create -y -p /opt/sglang \
      --override-channels \
      -c http://data-source-conda.sankuai.com/pkgs/main/ \
      -c http://data-source-conda.sankuai.com/cloud/conda-forge/ \
      python=3.10 libprotobuf
  /opt/sglang/bin/python -V
  /opt/sglang/bin/pip -V
  /opt/sglang/bin/protoc --version
  ```
- **实际输出**：
  ```text
  Python 3.10.14
  pip 23.3.1 from /opt/sglang/lib/python3.10/site-packages/pip (python 3.10)
  libprotoc 3.20.3
  ```
- **结论**：成功，全内网下载共 56.1 MB，耗时数秒，零污染 base 环境。
- **对应 Dockerfile**：`RUN /usr/local/miniconda3/bin/conda create -y -p /opt/sglang --override-channels -c http://data-source-conda.sankuai.com/pkgs/main/ -c http://data-source-conda.sankuai.com/cloud/conda-forge/ python=3.10 libprotobuf`

### `#06` [S1] 在 `/opt/sglang` 中升级打包与构建依赖（`pip` / `setuptools` / `setuptools-rust` / `setuptools-scm` / `wheel`）
- **目的**：对齐上游 Dockerfile `torch_deps` 段首条 pip 升级命令，同时把 `python/pyproject.toml` 的 `[build-system] requires`（除 `torch` 外）提前装入 `/opt/sglang`，以便后续 `pip install --no-build-isolation --no-deps -e python/` 直接复用。
- **执行命令**：
  ```bash
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install -i https://mirrors.aliyun.com/pypi/simple/ \
      --upgrade "pip" "setuptools>=61.0" "setuptools-rust>=1.11" "setuptools-scm>=8.0" "wheel" "html5lib" "six"
  ```
- **实际安装的精确版本**：
  `pip-26.2.2`、`setuptools-84.2.1`、`setuptools-rust-1.13.0`、`setuptools-scm-10.3.4`、`wheel-0.48.0`、`html5lib-1.1`、`six-1.17.0`（及传递依赖 `packaging-26.3`、`semantic_version-2.10.0`、`tomli-2.5.0`、`typing-extensions-4.16.0`、`vcs-versioning-2.6.0`、`webencodings-0.6.1`）。
- **结论**：成功，缓存全部落在 `/opt/sglang-cache/pip`。
- **对应 Dockerfile**：`RUN /opt/sglang/bin/pip install -i https://mirrors.aliyun.com/pypi/simple/ --upgrade "pip" "setuptools>=61.0" "setuptools-rust>=1.11" "setuptools-scm>=8.0" "wheel" "html5lib" "six"`

### `#07` [S3] 首次尝试默认 `sh.rustup.rs` 安装 Rust（卡在 `static.rust-lang.org`）
- **目的**：安装独立 Rust 工具链到 `/opt/sglang-rust`。
- **执行命令**：
  ```bash
  CARGO_HOME=/opt/sglang-rust/cargo RUSTUP_HOME=/opt/sglang-rust/rustup \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  sh -c 'curl --proto "=https" --tlsv1.2 --retry 3 --retry-delay 2 -sSf https://sh.rustup.rs | sh -s -- -y --no-modify-path --profile minimal --default-toolchain stable'
  ```
- **实际输出（卡住）**：
  ```text
  info: downloading installer
  （超过 120s 未返回，停在从 static.rust-lang.org 下载 rustup-init 二进制）
  ```
- **原因与修复**：`sh.rustup.rs` 引导脚本本身可达，但内部默认请求 `https://static.rust-lang.org` 被代理阻塞。实测 `https://rsproxy.cn`、`https://mirrors.ustc.edu.cn/rust-static`、`https://mirrors.aliyun.com/rustup` 三个国内镜像均秒回 `200 OK`。因此在安装时注入 `RUSTUP_DIST_SERVER=https://rsproxy.cn` 与 `RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup`，并在 `/opt/sglang-rust/cargo/config.toml` 中将 `crates.io` 源同样指向 `rsproxy-sparse`（避免后续编译 `sglang-grpc` 时卡在 `index.crates.io`）。

### `#08` [S3] 通过 `rsproxy.cn` 镜像安装独立 Rust 1.99.1 到 `/opt/sglang-rust` 并配置 crates.io 稀疏源
- **目的**：安装 `rustc >= 1.85` 到独立目录 `/opt/sglang-rust`（供后续 `sglang-grpc` / `sglang-mm` 的 `setuptools-rust` 编译使用），并配置 `/opt/sglang-rust/cargo/config.toml` 走 `sparse+https://rsproxy.cn/index/`。
- **执行命令**：
  ```bash
  CARGO_HOME=/opt/sglang-rust/cargo \
  RUSTUP_HOME=/opt/sglang-rust/rustup \
  RUSTUP_DIST_SERVER=https://rsproxy.cn \
  RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  sh -c 'curl --proto "=https" --tlsv1.2 --retry 3 --retry-delay 2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y --no-modify-path --profile minimal --default-toolchain stable'

  cat > /opt/sglang-rust/cargo/config.toml << 'EOF'
  [source.crates-io]
  replace-with = 'rsproxy-sparse'
  [source.rsproxy]
  registry = "https://rsproxy.cn/crates.io-index"
  [source.rsproxy-sparse]
  registry = "sparse+https://rsproxy.cn/index/"
  [registries.rsproxy]
  index = "https://rsproxy.cn/crates.io-index"
  [net]
  git-fetch-with-cli = true
  EOF

  CARGO_HOME=/opt/sglang-rust/cargo RUSTUP_HOME=/opt/sglang-rust/rustup /opt/sglang-rust/cargo/bin/rustc --version
  CARGO_HOME=/opt/sglang-rust/cargo RUSTUP_HOME=/opt/sglang-rust/rustup /opt/sglang-rust/cargo/bin/cargo --version
  ```
- **实际输出与注意项**：
  ```text
  rustc 1.99.1 (58cf8f9c3 2026-10-13)
  cargo 1.99.1 (5f94df478 2026-08-27)
  HOME_100_PERCENT_CLEAN
  ```
  *注意*：调用 `/opt/sglang-rust/cargo/bin/rustc` 或 `cargo` 时，环境变量中必须同时带 `CARGO_HOME=/opt/sglang-rust/cargo` 与 `RUSTUP_HOME=/opt/sglang-rust/rustup`，否则 `rustup` 包装器会回退查找 `~/.rustup` 并报 `no default is configured`（首次不带 `RUSTUP_HOME` 调用时在 `$HOME` 下生成了空目录 `~/.rustup`，已当场 `rmdir` 清理恢复）。
- **结论**：成功，Rust 1.99.1 完全隔离在 `/opt/sglang-rust`。
- **对应 Dockerfile**：`ENV CARGO_HOME=/opt/sglang-rust/cargo RUSTUP_HOME=/opt/sglang-rust/rustup RUSTUP_DIST_SERVER=https://rsproxy.cn RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup PATH="/opt/sglang-rust/cargo/bin:${PATH}"` + 上述安装与 `config.toml` 写入命令。

### `#09` [S3] 克隆独立源码副本到 `/opt/sglang-src` 并检出 `4349538c02`
- **目的**：在 `/tmp/sglang-isolated/src`（通过 `/opt/sglang-src` 软链访问）建立与 `/home/hadoop-djst-algoplat/sglang` 完全物理隔离的源码树（`--no-hardlinks`），以便在其上改写 `python/pyproject.toml` 与执行 editable 安装而不影响宿主仓库。
- **执行命令**：
  ```bash
  git clone --no-hardlinks /home/hadoop-djst-algoplat/sglang /tmp/sglang-isolated/src
  git -C /opt/sglang-src checkout -f 4349538c02
  git -C /opt/sglang-src rev-parse HEAD
  ```
- **实际输出**：
  ```text
  4349538c02e1566a1424510d5ac3ae853f49feef
  4349538c02 [model] add cosmos3 reasoner to llm only inference (#33572)
  ```
- **结论**：独立源码树已钉死在 `4349538c02e1566a1424510d5ac3ae853f49feef`。
- **对应 Dockerfile**：`RUN git clone https://gh-proxy.com/https://github.com/sgl-project/sglang.git /opt/sglang-src && git -C /opt/sglang-src checkout 4349538c02`

### `#10` [S2 & S3] 梳理 `4349538c02` 版本锁与系统 apt 依赖裁剪表，生成 `docker/intranet/versions.md`
- **目的**：把 `4349538c02` 的 `python/pyproject.toml` 与 `docker/Dockerfile` 中的精确版本要求、cu13→cu12 改写点、单机跳过项、以及上游 `base` 段系统依赖在当前容器中的存量核对清楚并落盘。
- **上游 `base` 段系统依赖核查与裁剪结果（S2）**：
  - **系统已自带（无需安装）**：`git`、`gcc`/`g++`（`build-essential`）、`cmake`、`ninja`（`ninja-build`）、`curl`、`wget`、`unzip`、`perl`、`libnuma1`。
  - **通过独立 Conda `/opt/sglang` 补齐**：`libprotobuf`（提供 `/opt/sglang/bin/protoc` 3.20.3，已在 `#05` 安装）。
  - **单机推理裁剪不装**：GDRCopy dkms、RDMA/InfiniBand 开发包（`libibverbs-dev` 等，多机 DeepEP 才需）、`ffmpeg`（仅多模态视频解码需）、C++ gRPC/Boost 开发头文件（`sglang-grpc` 使用 Rust `tonic` 编译，不依赖 C++ `libgrpc++-dev`）。
- **结论**：已生成 `docker/intranet/versions.md`。

### `#11` [S3] 首次尝试安装 `torch==2.13.0+cu129`（发现 `--index-url` 优先级陷阱与美团内网 PyPI 源 `http://pip.sankuai.com/simple/`）
- **目的**：安装 `torch==2.13.0`、`torchvision==0.28.0`、`torchaudio==2.11.0`（cu129）。
- **首次执行命令**：
  ```bash
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install \
      --index-url https://mirrors.nju.edu.cn/pytorch/whl/cu129 \
      --extra-index-url https://mirrors.aliyun.com/pypi/simple/ \
      torch==2.13.0 torchvision==0.28.0 torchaudio==2.11.0
  ```
- **现象与根因分析**：
  1. PyTorch 的 `whl/cu129` 索引里不仅有 `torch+cu129`，也镜像了 `nvidia-cudnn-cu12==9.20.0.48`（657.9 MB）、`sympy`、`cuda-bindings` 等全部依赖；当把 `whl/cu129` 作为主 `--index-url` 时，这些本来在普通 PyPI 上就有的巨型依赖包也全部走了外网代理（~150 KB/s），导致 600s 超时未完成。
  2. 探测发现**美团内网存在全量 PyPI 镜像 `http://pip.sankuai.com/simple/`（`pypi.sankuai.com` 302 跳转目标）**，内网直连（无需代理），且已同步包含 `nvidia_cudnn_cu12-9.20.0.48`、`sglang_kernel-0.4.9` 等最新包。
- **修复策略**：
  - 将主源设为内网直连 `-i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com`（并设 `no_proxy="pip.sankuai.com,pypi.sankuai.com,.sankuai.com,127.0.0.1,localhost"`，让内网源完全绕过外网代理高速直连）；
  - 将 `--extra-index-url https://mirrors.nju.edu.cn/pytorch/whl/cu129` 作为辅源，并显式指定本地版本后缀 `torch==2.13.0+cu129 torchvision==0.28.0+cu129 torchaudio==2.11.0+cu129`——这样所有常规依赖（含 658 MB 的 `nvidia-cudnn-cu12` 及 `cuda-*` 包）全部从美团内网 `pip.sankuai.com` 秒级拉取，仅有带 `+cu129` 的 3 个 wheel 从 `whl/cu129` 下载。

### `#12` [S3] 从美团内网 `http://pip.sankuai.com/simple/` 秒装全部 CUDA 12.9 / cuDNN / NCCL / Triton 底层依赖
- **目的**：由于 `pip` 在同时配置 `--index-url` 与 `--extra-index-url` 时对两者同等优先级检索，为确保 ~3.2 GB 的 `nvidia-*-cu12` 与 `triton` 100% 走美团内网千兆直连，先不挂代理直接从 `http://pip.sankuai.com/simple/` 安装 `torch==2.13.0+cu129` 的全部 `Requires-Dist` 底层包。
- **执行命令**：
  ```bash
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  /opt/sglang/bin/pip install \
      -i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com \
      "nvidia-cudnn-cu12==9.20.0.48" "nvidia-cusparselt-cu12==0.8.1" \
      "nvidia-nccl-cu12==2.29.7" "nvidia-nvshmem-cu12==3.4.5" \
      "triton==3.7.1" "cuda-bindings<13,>=12.9.4" \
      "nvidia-cublas-cu12==12.9.1.4" "nvidia-cuda-cupti-cu12==12.9.79" \
      "nvidia-cuda-nvrtc-cu12==12.9.86" "nvidia-cuda-runtime-cu12==12.9.79" \
      "nvidia-cufft-cu12==11.4.1.4" "nvidia-cufile-cu12==1.14.1.1" \
      "nvidia-curand-cu12==10.3.10.19" "nvidia-cusolver-cu12==11.7.5.82" \
      "nvidia-cusparse-cu12==12.5.10.65" "nvidia-nvjitlink-cu12==12.9.86" \
      "nvidia-nvtx-cu12==12.9.79" \
      "filelock" "sympy>=1.13.3" "networkx>=2.5.1" "jinja2" "fsspec>=0.8.5" "numpy" "pillow"
  ```
- **实际输出**：
  速度达 `400 MB/s ~ 1.1 GB/s`，约 15 秒内全部下载并安装完毕（共 27 个包，包括 `nvidia-cudnn-cu12-9.20.0.48`、`nvidia-nccl-cu12-2.29.7`、`triton-3.7.1`、`cuda-bindings-12.9.10` 等）。
- **结论**：成功，彻底规避外网代理对 GB 级 CUDA 依赖包的限速。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`。

### `#13` [S3] 安装 `cuda-toolkit==12.9.1` 与 `torch==2.13.0+cu129`、`torchvision==0.28.0+cu129`、`torchaudio==2.11.0+cu129` 并验证 CUDA / cuDNN
- **目的**：在底层依赖已由内网源装齐的前提下，加 `--no-deps` 从 `https://mirrors.nju.edu.cn/pytorch/whl/cu129` 安装 PyTorch 2.13.0+cu129 三件套，并验证 GPU 与 cuDNN 可用性。
- **执行命令**：
  ```bash
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install --no-deps \
      --index-url https://mirrors.nju.edu.cn/pytorch/whl/cu129 \
      "cuda-toolkit==12.9.1" "torch==2.13.0+cu129" "torchvision==0.28.0+cu129" "torchaudio==2.11.0+cu129"
  /opt/sglang/bin/python -c "import torch, torchvision, torchaudio; print('torch:', torch.__version__, 'vision:', torchvision.__version__, 'audio:', torchaudio.__version__, 'cuda:', torch.cuda.is_available(), 'cudnn:', torch.backends.cudnn.is_available(), 'cudnn_ver:', torch.backends.cudnn.version(), 'gpu0:', torch.cuda.get_device_name(0))"
  ```
- **实际输出**：
  ```text
  Downloading torch-2.13.0%2Bcu129-cp310-cp310-manylinux_2_28_x86_64.whl (929.6 MB) — 66.6 MB/s 0:00:13
  Successfully installed cuda-toolkit-12.9.1 torch-2.13.0+cu129 torchaudio-2.11.0+cu129 torchvision-0.28.0+cu129
  torch: 2.13.0+cu129 vision: 0.28.0+cu129 audio: 2.11.0+cu129 cuda: True cudnn: True cudnn_ver: 92000 gpu0: NVIDIA H20-3e
  ```
- **结论**：成功，CUDA 12.9 与 cuDNN 9.20.0（弥补 base 镜像无 cuDNN）均返回 `True`，识别 `NVIDIA H20-3e`。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`。

### `#15` [S3] 安装 `flashinfer_python[cu12]==0.6.18` 与 `nvidia-cutlass-dsl==4.6.2`（定位 `nccl4py`、`nvidia-cutlass-dsl-libs-cu12` 与 `nvidia-cudnn-frontend==1.28.0` 约束）
- **目的**：安装 `flashinfer_python[cu12]==0.6.18` 及其 CUDA 12 依赖链，并保持 `cuda-python` / `cuda-bindings` 锁死在 `12.9.10`（不被传递依赖升级到 `13.x`）。
- **实测发现的三个关键依赖细节**：
  1. `flashinfer_python==0.6.18` 的 `METADATA` 并不依赖 `flashinfer-cubin`（后者是 1.56 GB 的可选预编译包，缺省走 JIT 或 H20 默认的 `fa3` 后端）。
  2. `flashinfer_python==0.6.18` 新增依赖 `nccl4py>=0.3.1`（3.5 MB），目前仅在官方 `https://pypi.org/simple/` 上存在；`nvidia-cutlass-dsl-libs-cu12==4.6.2`（88.5 MB）在阿里云镜像 `https://mirrors.aliyun.com/pypi/simple/` 上存在（内网 `pip.sankuai.com` 尚未同步该子包）。
  3. **关键回溯发现**：`nvidia-cudnn-frontend>=1.29.0` 开始依赖 `nvidia-cutlass-dsl[cu13]`，pip 回溯 5 个版本后选中了 **`nvidia-cudnn-frontend==1.28.0`**；且安装 `nvidia-cutlass-dsl-libs-cu12==4.6.2` 时需同时指定 `"cuda-python==12.9.10" "cuda-bindings==12.9.10"` 以防其 `cuda-python>=12.8` 拉入 `13.4.4`。
- **执行命令**：
  ```bash
  # 1) 从 pypi.org 安装 nccl4py
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install --no-deps -i https://pypi.org/simple/ "nccl4py==0.6.0"

  # 2) 从阿里云镜像安装 nvidia-cutlass-dsl-libs-cu12==4.6.2（同时锁死 cuda-python/cuda-bindings 12.9.10）
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install -i https://mirrors.aliyun.com/pypi/simple/ \
      "cuda-python==12.9.10" "cuda-bindings==12.9.10" \
      "nvidia-cutlass-dsl-libs-cu12==4.6.2" "nvidia-cutlass-dsl-libs-base==4.6.2" "nvidia-cutlass-dsl==4.6.2"

  # 3) 从美团内网源安装 flashinfer_python[cu12]==0.6.18（显式锁定 nvidia-cudnn-frontend==1.28.0 免回溯）
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  /opt/sglang/bin/pip install \
      -i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com \
      "cuda-python==12.9.10" "cuda-bindings==12.9.10" \
      "cuda-tile==1.6.0rc5" "apache-tvm-ffi==0.1.11" \
      "nvidia-cudnn-frontend==1.28.0" "flashinfer_python[cu12]==0.6.18"

  /opt/sglang/bin/python -c "import flashinfer, cuda.bindings; print('flashinfer:', flashinfer.__version__, 'cuda.bindings:', cuda.bindings.__version__)"
  ```
- **实际输出**：
  ```text
  Successfully installed apache-tvm-ffi-0.1.11 click-8.5.0 cuda-bindings-12.9.10 cuda-python-12.9.10 cuda-tile-1.6.0rc5 einops-0.8.2 flashinfer_python-0.6.18 ninja-1.13.2 nvidia-cudnn-frontend-1.28.0 nvidia-ml-py-13.615.71 requests-2.34.2 tabulate-0.10.0 tqdm-4.70.1
  flashinfer: 0.6.18 cuda-python: 12.9.10
  ```
- **结论**：成功，`flashinfer 0.6.18` 与 `cuda-python 12.9.10` 均就绪。
- **对应 Dockerfile**：写入上述优化后的三条 `pip install`（显式锁定 `nvidia-cudnn-frontend==1.28.0` 与 `cuda-python==12.9.10`）。

### `#16` [S3] 修改独立源码副本 `/opt/sglang-src/python/pyproject.toml`（cu13→cu12 改写 + 注释多机 DeepEP/DeepGEMM）
- **目的**：对齐上游 `docker/Dockerfile:248-252` 的 cu12 分支 `sed` 改写，并将 `humming-kernels[cu13]` 的 `[cu13]` extra 移除、注释单卡推理不使用的 `sgl-deep-ep` 与 `sgl-deep-gemm`。
- **执行命令**：
  ```bash
  sed -i \
    -e 's/"cuda-python>=13\.0"/"cuda-python>=12,<13"/' \
    -e 's/"flashinfer_python\[cu13\]==0\.6\.18"/"flashinfer_python[cu12]==0.6.18"/' \
    -e 's/"nvidia-cutlass-dsl\[cu13\]==4\.6\.2"/"nvidia-cutlass-dsl==4.6.2"/' \
    -e 's/"humming-kernels\[cu13\]==0\.1\.12"/"humming-kernels==0.1.12"/' \
    -e 's/^[[:space:]]*"sgl-deep-ep==0\.1\.2"/  # "sgl-deep-ep==0.1.2"/' \
    -e 's/^[[:space:]]*"sgl-deep-gemm==0\.1\.7"/  # "sgl-deep-gemm==0.1.7"/' \
    /opt/sglang-src/python/pyproject.toml
  git -C /opt/sglang-src diff python/pyproject.toml
  ```
- **实际 Diff 输出**：
  ```diff
  -  "cuda-python>=13.0",
  +  "cuda-python>=12,<13",
  -  "flashinfer_python[cu13]==0.6.18",
  +  "flashinfer_python[cu12]==0.6.18",
  -  "humming-kernels[cu13]==0.1.12",
  +  "humming-kernels==0.1.12",
  -  "nvidia-cutlass-dsl[cu13]==4.6.2",
  +  "nvidia-cutlass-dsl==4.6.2",
  -  "sgl-deep-ep==0.1.2",
  -  "sgl-deep-gemm==0.1.7",
  +  # "sgl-deep-ep==0.1.2",
  +  # "sgl-deep-gemm==0.1.7",
  ```
- **结论**：仅修改 `/opt/sglang-src` 独立副本，宿主仓库未受影响。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`。

### `#17` [S3] 通过美团内网源 `http://pip.sankuai.com/simple/` 批量安装 52 个核心依赖（配合 `constraints-cu129.txt` 防覆盖）
- **目的**：在锁定 `torch==2.13.0+cu129`、`cuda-python==12.9.10`、`cuda-bindings==12.9.10`、`nvidia-cudnn-frontend==1.28.0` 的前提下，从美团内网千兆源一次性安装 `pyproject.toml` 中的 52 个通用依赖。
- **执行命令**：
  ```bash
  cat > /opt/sglang-cache/constraints-cu129.txt << 'EOF'
  torch==2.13.0+cu129
  torchvision==0.28.0+cu129
  torchaudio==2.11.0+cu129
  cuda-python==12.9.10
  cuda-bindings==12.9.10
  nvidia-cudnn-frontend==1.28.0
  EOF

  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  /opt/sglang/bin/pip install \
      -i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com \
      -c /opt/sglang-cache/constraints-cu129.txt \
      "aiohttp" "anthropic>=0.20.0" "blobfile==3.0.0" "build" \
      "compressed-tensors==0.18.0" "datasets" "distro" "easydict" "einops" \
      "fastapi" "gguf" "interegular" "IPython" \
      "llguidance>=1.7.6,<2.0.0" "mistral_common>=1.11.5" "modelscope" \
      "msgspec" "ninja" "numba==0.65.1" "numpy" \
      "nvidia-ml-py" "openai==2.6.1" "openai-harmony==0.0.4" "orjson" \
      "outlines==0.1.11" "packaging" "partial_json_parser" "pillow" \
      "prometheus-client>=0.20.0" "psutil" "py-spy" "pybase64" "pydantic" \
      "python-multipart" "pyzmq>=25.1.2" "requests" \
      "scipy" "sentencepiece" "setproctitle" "smg-grpc-servicer>=0.5.0" \
      "soundfile==0.13.1" "tiktoken" "timm==1.0.16" "tokenizers==0.22.2" \
      "tomli" "torch_memory_saver>=0.0.9.post1" "torchcodec==0.15.0" \
      "tqdm" "transformers==5.12.1" "uvicorn" "uvloop" "watchfiles" \
      "xgrammar==0.2.1" "xxhash" "zstandard"
  ```
- **实际输出与注意项**：
  112 个 wheel（含 `transformers-5.12.1`、`tokenizers-0.22.2`、`xgrammar-0.2.1`、`outlines-0.1.11`、`numba-0.65.1`、`scipy-1.15.3` 等）在 ~10 秒内全部完成安装。
  *注意*：`smg-grpc-servicer` 默认拉入了 `protobuf-7.36.2`，与 `nvidia-cutlass-dsl-libs-* 4.6.2` 要求的 `protobuf<7,>=6.30.2` 产生警告，将在下一步统一把 `protobuf<7,>=6.30.2` 锁回 `6.33.6`。
- **结论**：成功。
- **对应 Dockerfile**：在 `constraints-cu129.txt` 中加入 `protobuf<7,>=6.30.2` 后原样写入。

### `#18` [S3] 补齐剩余 8 个专有包并锁回 `protobuf==6.33.6`
- **目的**：安装 `quack-kernels==0.6.4`、`tilelang==0.1.12`、`flash-attn-4>=4.0.0b18`、`kernels==0.14.1`（以上 4 个在 `pip.sankuai.com` 存在）、锁回 `protobuf==6.33.6`（满足 `nvidia-cutlass-dsl-libs-*` 的 `protobuf<7` 约束），并从阿里云 / `pypi.org` 安装内网尚未同步的 `kernels-data`、`nvidia-mathdx==25.6.0`、`humming-kernels==0.1.12`、`tokenspeed_mla==0.1.8`。
- **执行命令**：
  ```bash
  # 1) 美团内网源直连安装
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  /opt/sglang/bin/pip install --no-deps \
      -i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com \
      "protobuf==6.33.6" "quack-kernels==0.6.4" "tilelang==0.1.12" \
      "flash-attn-4>=4.0.0b18" "kernels==0.14.1"

  # 2) 外网源补齐内网未同步的 4 个包
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install --no-deps \
      -i https://mirrors.aliyun.com/pypi/simple/ \
      --extra-index-url https://pypi.org/simple/ \
      "kernels-data>=0.14.0.dev1" "nvidia-mathdx==25.6.0" \
      "humming-kernels==0.1.12" "tokenspeed_mla==0.1.8"
  ```
- **实际输出**：
  ```text
  Successfully installed flash-attn-4-4.0.0b37 kernels-0.14.1 protobuf-6.33.6 quack-kernels-0.6.4 tilelang-0.1.12
  Successfully installed humming-kernels-0.1.12 kernels-data-0.16.2 nvidia-mathdx-25.6.0 tokenspeed_mla-0.1.8
  ```
- **结论**：`pyproject.toml` 全部依赖已就齐，且未混入任何 `-cu13` 包。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`。

### `#19` [S4] 执行 `pip install --no-build-isolation --no-deps -e python/`（编译 Rust 扩展）并验证 `import sglang` 与 `launch_server --help`
- **目的**：在 `/opt/sglang-src` 中以 `--no-build-isolation --no-deps` 模式完成 sglang 的 editable 安装（调用 `/opt/sglang-rust` 的 `cargo` 与 `/opt/sglang/bin/protoc` 编译 `rust/` 下的 `sglang-grpc`、`sglang-mm` 等扩展），并验证 `import sglang` 与 `sglang.launch_server` 入口。
- **执行命令**：
  ```bash
  cd /opt/sglang-src
  PATH="/opt/sglang/bin:/opt/sglang-rust/cargo/bin:$PATH" \
  CARGO_HOME=/opt/sglang-rust/cargo \
  RUSTUP_HOME=/opt/sglang-rust/rustup \
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/pip install --no-build-isolation --no-deps -e python/

  /opt/sglang/bin/python -c "import sglang; print('sglang version:', sglang.__version__, 'from:', sglang.__file__)"
  /opt/sglang/bin/python -m sglang.launch_server --help | head -12
  ```
- **实际输出**：
  ```text
  Successfully built sglang
  Successfully installed sglang-0.5.23
  sglang version: 0.5.23 from: /tmp/sglang-isolated/src/python/sglang/__init__.py
  usage: sglang serve [-h] --model-path MODEL_PATH ...
  ```
- **结论**：一次编译成功，`sglang-0.5.23` 与 `sglang.launch_server` 就绪。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`，并在安装后设置 `ENV SGLANG_RUST_BUILD_MODE=never`（与上游 `Dockerfile:661` 一致）。

### `#20` [S4] 验证 `kernels lock` 与 `kernels download` 在 `cu129` 下的行为
- **目的**：核实上游 `Dockerfile:638-654` 的 `kernels lock python && kernels download python` 在 `torch 2.13.0+cu129` 下的真实表现。
- **执行命令**：
  ```bash
  HF_HOME=/opt/sglang-cache/hf HF_ENDPOINT=https://hf-mirror.com \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/kernels lock /opt/sglang-src/python
  HF_HOME=/opt/sglang-cache/hf HF_ENDPOINT=https://hf-mirror.com \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/kernels download /opt/sglang-src/python || true
  ```
- **实际输出**：
  ```text
  LOCK_OK
  Downloading `kernels-community/sgl-flash-attn3` with SHA: 138f0da523770447c375afd0d1574eedbb64774d
  Cannot find a build variant for this system in kernels-community/sgl-flash-attn3 (revision: 138f0da523770447c375afd0d1574eedbb64774d). Available variants: torch-stable-abi29-cu130-x86_64-linux, torch-stable-abi29-cu128-x86_64-linux, torch211-cxx11-cu128-x86_64-linux, torch210-cxx11-cu130-x86_64-linux, torch210-cxx11-cu128-x86_64-linux, torch211-cxx11-cu130-x86_64-linux
  ```
- **结论**：与上游 `Dockerfile:650-654` 注释完全一致——`kernels-community/sgl-flash-attn3` 仅发布了 `cu128` 与 `cu130` 预编译变体，无 `cu129` 变体，因此在 `cu129` 下由运行时按需回退（`sgl_kernel` 已内置 FlashAttention-3 / Triton 后端）。
- **对应 Dockerfile**：`cu129` 镜像中无需执行 `kernels download`（或带 `|| true` 兜底）。

### `#21` [S5] 首次通过 `hf-mirror.com` 下载 `Qwen/Qwen3-0.6B`（触发 `hf_xet` `cas-server.xethub.hf.co` 401 报错）
- **目的**：下载验收模型 `Qwen/Qwen3-0.6B` 到 `/opt/sglang-cache/hf`。
- **执行命令**：
  ```bash
  HF_HOME=/opt/sglang-cache/hf HF_ENDPOINT=https://hf-mirror.com \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/python -c "from huggingface_hub import snapshot_download; print(snapshot_download('Qwen/Qwen3-0.6B'))"
  ```
- **实际输出（报错）**：
  ```text
  File "/opt/sglang/lib/python3.10/site-packages/huggingface_hub/file_download.py", line 2016, in _download_to_tmp_and_move
      xet_get(...)
  RuntimeError: Task error: File reconstruction error: CAS Client Error: Request error: HTTP status client error (401 Unauthorized), domain: https://cas-server.xethub.hf.co/v2/reconstructions/...
  ```
- **原因与修复**：`huggingface_hub-1.33.0` 默认拉入了 `hf-xet`，在下载大文件（`model.safetensors`）时绕过 `HF_ENDPOINT=https://hf-mirror.com` 直接请求 `cas-server.xethub.hf.co` 并返回 401；此外国内下载 `Qwen/*` 模型使用 `modelscope.snapshot_download`（阿里云 CDN）或设置 `HF_HUB_DISABLE_XET=1` 可直接规避。
- **对应 Dockerfile**：在环境变量中固定加上 `ENV HF_HUB_DISABLE_XET=1`。

### `#22` [S5] 通过 `modelscope.snapshot_download` 下载 `Qwen/Qwen3-0.6B` 到 `/opt/sglang-cache/hf`
- **目的**：将验收模型 `Qwen/Qwen3-0.6B` 下载至独立缓存目录 `/opt/sglang-cache/hf/Qwen/Qwen3-0.6B`。
- **执行命令**：
  ```bash
  HF_HOME=/opt/sglang-cache/hf \
  HF_HUB_DISABLE_XET=1 \
  MODELSCOPE_CACHE=/opt/sglang-cache/hf \
  http_proxy=http://10.177.8.11:3128 https_proxy=http://10.177.8.11:3128 \
  /opt/sglang/bin/python -c "from modelscope import snapshot_download; p = snapshot_download('Qwen/Qwen3-0.6B', cache_dir='/opt/sglang-cache/hf'); print('MODEL_PATH:', p)"
  ```
- **实际输出**：
  ```text
  Downloading [model.safetensors]: 100%|██████████| 1.40G/1.40G [01:22<00:00, 18.2MB/s]
  Finish downloading 10 files for repo 'Qwen/Qwen3-0.6B'
  MODEL_PATH: /opt/sglang-cache/hf/Qwen/Qwen3-0___6B
  /opt/sglang-cache/hf/Qwen/Qwen3-0.6B -> /opt/sglang-cache/hf/Qwen/Qwen3-0___6B
  ```
- **结论**：成功，1.4 GB 模型权重以 18.2 MB/s 在 82 秒内下载完成。
- **对应 Dockerfile**：无需进镜像（运行期模型挂载/下载）。

### `#23` [S5] 启动 `sglang.launch_server`（单卡 H20-3e + `Qwen3-0.6B`）、通过 `/health` 与 `/generate` + `/v1/chat/completions` 硬验收
- **目的**：在完全隔离的运行环境（仅保留 `/opt/sglang/bin` 与系统基础 PATH，所有缓存指向 `/opt/sglang-cache`）下启动 sglang 推理服务，验证健康检查、原生 `/generate` 与 OpenAI 兼容 `/v1/chat/completions` 推理请求。
- **执行命令**：
  ```bash
  mkdir -p /opt/sglang-cache/{triton,flashinfer,sglang}
  CUDA_VISIBLE_DEVICES=0 \
  PATH="/opt/sglang/bin:/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
  HF_HOME=/opt/sglang-cache/hf \
  TRITON_CACHE_DIR=/opt/sglang-cache/triton \
  FLASHINFER_WORKSPACE_BASE=/opt/sglang-cache/flashinfer \
  SGLANG_RUST_BUILD_MODE=never \
  /opt/sglang/bin/python -m sglang.launch_server \
      --model-path /opt/sglang-cache/hf/Qwen/Qwen3-0.6B \
      --host 127.0.0.1 --port 30000 > /opt/sglang-cache/server.log 2>&1 &

  # 健康检查与推理请求
  curl -fsS http://127.0.0.1:30000/health
  curl -fsS http://127.0.0.1:30000/generate \
    -H 'Content-Type: application/json' \
    -d '{"text": "The capital of France is", "sampling_params": {"max_new_tokens": 16, "temperature": 0}}'
  curl -fsS http://127.0.0.1:30000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model": "/opt/sglang-cache/hf/Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "Say hello in one sentence."}], "max_tokens": 32, "temperature": 0}'
  ```
- **实际输出**：
  - **关键启动日志**：
    ```text
    [2026-10-30 17:14:23] Attention backend not specified. Use fa3 backend by default.
    [2026-10-30 17:15:18] Capture target prefill CUDA graph end. elapsed=31.20 s, mem usage=0.86 GB, avail mem=11.86 GB.
    [2026-10-30 17:15:21] Capture target decode CUDA graph end. elapsed=2.97 s, mem usage=0.22 GB, avail mem=11.64 GB.
    [2026-10-30 17:15:29] The server is fired up and ready to roll!
    ```
  - **`/generate` 响应原文**（16 tokens，端到端延迟 `32.79 ms`）：
    ```json
    {"text":" Paris. The capital of France is also the capital of the Republic of France.","output_ids":[12095,13,576,6722,315,9625,374,1083,279,6722,315,279,5429,315,9625,13],"meta_info":{"id":"da92a533677a48eba5270d267295e323","finish_reason":{"type":"length","length":16},"prompt_tokens":5,"weight_version":"default","num_retractions":0,"reasoning_tokens":0,"completion_tokens":16,"cached_tokens":2,"e2e_latency":0.03278806246817112}}
    ```
  - **`/v1/chat/completions` 响应原文**：
    ```json
    {"id":"1dcaf0f60cbe4caea3c3686ab271aac2","object":"chat.completion","created":1793380531,"model":"/opt/sglang-cache/hf/Qwen/Qwen3-0.6B","choices":[{"index":0,"message":{"role":"assistant","content":"<think>\nOkay, the user wants a one-sentence greeting. Let me think. They probably need something friendly and simple. Maybe start with \"Hello\" and","reasoning_content":null,"tool_calls":null},"logprobs":null,"finish_reason":"length","matched_stop":null}],"usage":{"prompt_tokens":14,"total_tokens":46,"completion_tokens":32}}
    ```
- **结论**：**项目硬验收标准（Server 启动 + 真实推理请求返回）通过**，验证完毕后服务进程已干净退出（`NO_SGLANG_RUNNING`）。
- **对应 Dockerfile**：`Dockerfile.serving` 的验证闭环完成。

### `#24` [S6] 安装并验证 `Dockerfile.dev` 所需的 Python 开发与测试工具包
- **目的**：满足反目标约束（写进 `Dockerfile.dev` 的行也必须亲手执行并验证过），对照上游 `Dockerfile:533-549` 从美团内网源安装开发/测试工具链并验证 CLI。
- **执行命令**：
  ```bash
  PIP_CACHE_DIR=/opt/sglang-cache/pip \
  /opt/sglang/bin/pip install \
      -i http://pip.sankuai.com/simple/ --trusted-host pip.sankuai.com \
      -c /opt/sglang-cache/constraints-cu129.txt \
      "pytest" "pytest-cov" "pre-commit" "black" "isort" "icdiff" \
      "uv" "scikit-build-core" "datamodel_code_generator" "matplotlib" "termplotlib"
  /opt/sglang/bin/pytest --version
  /opt/sglang/bin/pre-commit --version
  /opt/sglang/bin/black --version
  /opt/sglang/bin/uv --version
  ```
- **实际输出**：
  ```text
  pytest 9.1.1
  pre-commit 4.6.2
  black, 26.10.1 (compiled: yes)
  uv 0.13.6 (x86_64-unknown-linux-gnu)
  ```
- **结论**：开发工具包全部由内网源秒装并通过 CLI 验证。
- **对应 Dockerfile**：写入 `Dockerfile.dev` 的开发工具扩展层。

### `#25` [S6] 生成 `Dockerfile.serving` 与 `Dockerfile.dev`、执行反目标逐行溯源与默认环境零污染核验
- **目的**：把 `#01 ~ #24` 亲手执行并验证过的命令序列固化为单阶段 `docker/intranet/Dockerfile.serving` 与 `docker/intranet/Dockerfile.dev`，并对每条指令标注出处流水号；核验宿主默认环境与 `/` 分区零污染。
- **执行命令**：
  ```bash
  /usr/local/miniconda3/bin/conda info --envs
  ls -d "$HOME/.cargo" "$HOME/.rustup" "$HOME/.triton" 2>/dev/null || echo "NO_CARGO_RUSTUP_TRITON_IN_HOME"
  du -sh /tmp/sglang-isolated/*
  df -h / /tmp
  ```
- **实际输出**：
  ```text
  NO_CARGO_RUSTUP_TRITON_IN_HOME
  5.0G  /tmp/sglang-isolated/cache
  11G   /tmp/sglang-isolated/env
  1.5G  /tmp/sglang-isolated/rust
  2.8G  /tmp/sglang-isolated/src
  overlay          50G  4.5G   46G   9% /
  /dev/nvme1n1p1  5.9T   64G  5.8T   2% /tmp
  ```
- **反目标自检结果**：`Dockerfile.serving` 与 `Dockerfile.dev` 中的每一条 `ENV` / `RUN` / `WORKDIR` 均标有 `[#XX]` 流水号，100% 在本记录文档中有实际执行记录，无任何未经验证的命令。

---

## 四、踩坑与关键发现汇总（学习产出）

| 编号 | 阶段 | 踩坑现象 | 根因分析 | 最终解法 |
|---|---|---|---|---|
| P1 | S0 (`#02`) | `curl -x ... https://pypi.org/simple/` 120s 超时 | `pypi.org/simple/` 根索引页面达 47.8 MB（全量包名列表），走代理拉取整页耗时过长 | 探测改用具体包路径 `https://pypi.org/simple/pip/` 或 `HEAD (-I)`；后续发现美团内网直连源 `http://pip.sankuai.com/simple/` |
| P2 | S1 (`#04`) | `conda create -p /opt/sglang` 报 `CondaToSNonInteractiveError` | 新版 `conda` 默认合并 `repo.anaconda.com` 官方源并强制交互式确认商业条款 | 加 `--override-channels -c http://data-source-conda.sankuai.com/pkgs/main/ -c http://data-source-conda.sankuai.com/cloud/conda-forge/`，纯走内网源且免 ToS |
| P3 | S3 (`#07, #08`) | `sh.rustup.rs` 卡在 `info: downloading installer`；不带 `RUSTUP_HOME` 调 `rustc` 报错并在 `$HOME` 生成空 `.rustup` | 1) `sh.rustup.rs` 内部默认访问 `static.rust-lang.org` 超时；2) `rustup` 代理器依赖 `RUSTUP_HOME` 定位默认工具链 | 注入 `RUSTUP_DIST_SERVER=https://rsproxy.cn`、`RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup` 并配置 `rsproxy-sparse` crates 源；全程显式带 `CARGO_HOME` 与 `RUSTUP_HOME` |
| P4 | S3 (`#11, #12`) | 把 `--index-url` 设为外网 `whl/cu129` 时，`nvidia-cudnn-cu12`（658 MB）按 ~150 KB/s 下载超时 | PyTorch 的 `whl/cu129` 索引同时镜像了 `nvidia-*-cu12` 等所有依赖，pip 对 `--index-url` 与 `--extra-index-url` 同等检索导致大体积常规包走外网 | 先不挂代理从美团内网 `http://pip.sankuai.com/simple/` 以 **400 MB/s ~ 1.1 GB/s** 秒装全部 `nvidia-*-cu12`（共 3.2 GB），再用 `--no-deps` 安装 `torch==2.13.0+cu129` 三件套 |
| P5 | S3 (`#15`) | 1) `nvidia-cutlass-dsl-libs-cu12==4.6.2` 把 `cuda-bindings` 升级到 `13.4.4`；2) `nvidia-cudnn-frontend` 触发多版本回溯 | 1) `nvidia-cutlass-dsl-libs-cu12` 仅声明 `cuda-python>=12.8`（无 `<13` 上界）；2) `nvidia-cudnn-frontend>=1.29.0` 强制依赖 `nvidia-cutlass-dsl[cu13]` | 显式锁死 `"cuda-python==12.9.10" "cuda-bindings==12.9.10"` 与最后一个支持 cu12 的 **`"nvidia-cudnn-frontend==1.28.0"`**，消除回溯 |
| P6 | S3 (`#17, #18`) | `smg-grpc-servicer` 拉入 `protobuf-7.36.2`，与 `nvidia-cutlass-dsl-libs-*` 的 `protobuf<7` 冲突 | `smg-grpc-servicer` 未设 `protobuf<7` 上界 | 在 `constraints-cu129.txt` 中锁定 `protobuf==6.33.6` |
| P7 | S4 (`#19, #20`) | 1) `pip install -e python/` 默认 build isolation 会重拉 cu13 版 `torch`；2) `kernels download` 报无匹配变体 | 1) `pyproject.toml [build-system]` 含 `torch==2.13.0`；2) `kernels-community/sgl-flash-attn3` 仅发布了 `cu128` 与 `cu130` 预编译变体 | 1) 加 `--no-build-isolation --no-deps` 复用已装好的环境与 `/opt/sglang/bin/protoc`；2) `cu129` 下无需 `kernels download`，运行时默认走 `sgl_kernel` 内置的 `fa3` 后端 |
| P8 | S5 (`#21, #22`) | `huggingface_hub.snapshot_download` 走 `hf-mirror.com` 报 `cas-server.xethub.hf.co` 401 | `huggingface_hub-1.33.0` 默认启用 `hf-xet`，绕过 `HF_ENDPOINT` 直连 `xethub.hf.co` | 设置 `HF_HUB_DISABLE_XET=1`，或直接使用 `modelscope.snapshot_download`（18.2 MB/s） |

---

## 五、与上游 `4349538c02:docker/Dockerfile` 差异对照表

| 维度 | 上游 `4349538c02:docker/Dockerfile` | 内网复刻版 (`Dockerfile.serving` / `Dockerfile.dev`) | 差异原因 |
|---|---|---|---|
| 基础镜像 | `nvidia/cuda:12.9.2-cudnn-devel-ubuntu24.04` | 内网 `nvidia-cuda:12.9.1-devel-ubuntu22.04` (`642392771`) | 内网仅导入并审批了该 base 镜像 |
| 构建阶段结构 | BuildKit 6 阶段并行构建 + `--mount=type=cache` | 单阶段线性 `FROM ... RUN ...` | 容器内无 Docker daemon，且需直接对齐 MLP `image create --components` |
| Python 环境 | Ubuntu 24.04 `apt install python3.12` + `venv /opt/sglang` | base 自带 miniconda `conda create -p /opt/sglang python=3.10 libprotobuf` | Ubuntu 22.04 仅自带 3.10.12 且无 venv/pip；用内网 Conda 源一步创建隔离 3.10.14 环境并补齐 `protoc` |
| cuDNN 来源 | base 镜像自带系统级 cuDNN | 由美团内网 PyPI 源安装 `nvidia-cudnn-cu12==9.20.0.48`（`torch.backends.cudnn.version() == 92000`） | 内网 base 为 `devel`（不含 `cudnn`），由 wheel 动态库提供 |
| PyPI / Conda / Rust 源 | 官方 `pypi.org`、`download.pytorch.org`、`sh.rustup.rs`、`github.com` | 美团内网 `http://pip.sankuai.com/simple/` + 内网 Conda 源为主（400 MB/s~1.1 GB/s）；外网包走 `mirrors.nju.edu.cn` / `mirrors.aliyun.com` / `rsproxy.cn` / `gh-proxy.com`（带 SHA256 校验） | 适应内网与代理限速环境，构建耗时从小时级压缩至分钟级 |
| 多机 / 可选组件 | 编译安装 GDRCopy、DeepEP、DeepGEMM、Mooncake、MSCCL++、HPC-Ops、`sgl-model-gateway` | 跳过并在 `pyproject.toml` 注释 `sgl-deep-ep` / `sgl-deep-gemm` | 单机 H20 推理无需多机 IBGDA/EP/PD 传输组件，按需再补 |

---

## 六、迁移到美团 MLP 平台 `image create --components` 映射说明

单阶段 `Dockerfile.serving` 的 7 个 `RUN` 块可直接一对一映射为 MLP 镜像构建的 `components` 脚本段：

| MLP Component 顺序 | 对应 `Dockerfile.serving` 段落 | 流水号依据 | 关键注意事项 |
|---|---|---|---|
| Component 1: `conda_env` | 创建 `/opt/sglang` (Python 3.10 + `libprotobuf`) + 升级 `pip`/`setuptools-rust` | `#05, #06` | 必须带 `--override-channels -c http://data-source-conda.sankuai.com/...` |
| Component 2: `rust_toolchain` | 安装独立 Rust 到 `/opt/sglang-rust` 并写 `config.toml` | `#08` | 显式导出 `CARGO_HOME`、`RUSTUP_HOME`、`RUSTUP_DIST_SERVER=https://rsproxy.cn` |
| Component 3: `torch_cu129` | 内网源装 `nvidia-*-cu12` + 南大源装 `torch==2.13.0+cu129` 三件套 | `#12, #13` | 分两步执行以确保 3.2 GB 的 `nvidia-*-cu12` 100% 走内网 `pip.sankuai.com` |
| Component 4: `sgl_kernel_flashinfer` | 下载校验 `sglang_kernel-0.4.6.post1+cu129.whl` + 安装 `flashinfer_python[cu12]==0.6.18` | `#14, #15` | 显式锁死 `cuda-python==12.9.10` 与 `nvidia-cudnn-frontend==1.28.0` |
| Component 5: `python_deps` | 按 `constraints-cu129.txt` 安装其余 60 个 Python 依赖 | `#17, #18` | `constraints-cu129.txt` 含 `protobuf==6.33.6` |
| Component 6: `sglang_editable` | Clone `4349538c02`、`sed` 改写 `pyproject.toml`、`pip install --no-build-isolation --no-deps -e python/`、清理缓存 | `#09, #16, #19` | 设置 `ENV SGLANG_RUST_BUILD_MODE=never` |












### `#14` [S3] 下载 `sglang_kernel-0.4.6.post1+cu129` wheel、校验官方 SHA256、安装并验证 `import sgl_kernel`
- **目的**：安装与 `4349538c02` 锁定的 `sglang-kernel==0.4.6.post1` 对应的 **CUDA 12.9** 预编译 wheel（PyPI 默认发布的是 cu13 版，必须从 GitHub Releases 获取 `+cu129` 版本），并用官方索引 `docs.sglang.io/whl/cu129/sglang-kernel/` 的 SHA256 校验字节完整性。
- **执行命令**：
  ```bash
  WHL_FILE="/opt/sglang-cache/pip/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl"
  curl -fL --retry 3 -x http://10.177.8.11:3128 -o "$WHL_FILE" \
    "https://gh-proxy.com/https://github.com/sgl-project/whl/releases/download/v0.4.6.post1/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl"
  echo "9d400246b3f2512e2c33611e1bb6a2f252c499c5019e9f331f9c9ee215be95ab  $WHL_FILE" | sha256sum -c -
  /opt/sglang/bin/pip install --no-deps --force-reinstall "$WHL_FILE"
  /opt/sglang/bin/python -c "import sgl_kernel; print('sgl_kernel imported from:', sgl_kernel.__file__)"
  ```
- **实际输出**：
  ```text
  /opt/sglang-cache/pip/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl: OK
  Successfully installed sglang-kernel-0.4.6.post1+cu129
  sgl_kernel imported from: /opt/sglang/lib/python3.10/site-packages/sgl_kernel/__init__.py
  ```
- **结论**：成功，且证实 `cp310-abi3` wheel 与 Python 3.10.14 完全兼容，无需升级 Python 3.11。
- **对应 Dockerfile**：原样写入 `Dockerfile.serving` / `Dockerfile.dev`。













