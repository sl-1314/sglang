# sglang 镜像内网复刻 · 设计文档

日期：2026-10-30
状态：待评审

## 意图与动机

通过在内网环境完整复刻 sglang 官方镜像的构建过程，学习现代 GPU 推理框架的镜像构建工程：依赖分层、版本锁定、wheel 与源码编译的取舍、以及约束条件下的适配。产出物同时具备实用价值（开发/服务两个可用的镜像构建方案）。

## 背景事实（已核实）

- 上游 Dockerfile：`sgl-project/sglang` 仓库 `docker/Dockerfile`，多阶段 BuildKit 构建（base → torch_deps → 并行 hpc_ops/flashinfer/devtools/gateway → framework → runtime）
- 上游 base：`nvidia/cuda:12.9.2-cudnn-devel-ubuntu24.04`（Python 3.12 原生、含 cuDNN）
- 目标 base：内网 `nvidia/cuda:12.9.1-devel-ubuntu22.04`（imageId 642392771，Approved/Released、isBase、9.48GB、X86、从未被使用过）
  - 差异：Ubuntu 22.04（Python 仅 3.10）、无 cuDNN、CUDA 12.9.1（vs 12.9.2）
- 最新 main（16a3bf3d9d，2026-10-30）已删除 cu12 构建路径（Dockerfile 仅支持 CUDA 13.0.3，torch==2.14.1、requires-python>=3.11，全 cu13 依赖链）；**最后含 cu12 支持的提交是 4349538c02（2026-09-04）**，其 pyproject：`torch==2.13.0`、`requires-python >= 3.10`，落后当前 main 约 3503 个提交
- 构建环境：base 镜像起的容器，可配置代理访问外网；容器内无 docker daemon

## 目标

1. 在构建环境中把 sglang（main @ 钉死的 commit）跑起来：server 启动、完成一次推理请求——硬验收标准
2. 交付一份 Dockerfile：从实际执行并验证过的命令序列演化而来，单阶段（FROM 内网 base），每步可追溯
3. 交付该跑通的环境本身
4. 过程记录：每个阶段为什么这么做、踩了什么坑、与上游的差异及原因（学习产出）

## 非目标

- 不追求与官方镜像 bit-for-bit 一致（约束不同，记录差异即可）
- 不在本项目内做镜像导入平台/发布（后续单独立项）
- 不做多 CUDA 版本泛化（只做 cu129 一条路径）
- 不维护长期 CI/自动重建

## 反目标（做了就算失败）

- Dockerfile 里出现没有亲手跑过、不知道为什么存在的行
- 为了「跑通」引入与上游口径不一致的静默 hack 而不记录

## 约束

- base 固定为内网 `nvidia-cuda:12.9.1-devel-ubuntu22.04`，不换
- 单阶段 Dockerfile（无 BuildKit、无多阶段、无 cache mount）——正好与 mlp 平台 `image create --components` 的能力对齐，未来可直接迁移
- 外网访问走代理；所有网络操作要考虑代理可用性
- 两个方向：开发镜像（framework 对应物，含 dev 工具链）与服务镜像（runtime 对应物，精简）——共享依赖步骤，末段分叉

## 过程设计（阶段化执行，每段先跑通再记录）

每阶段在容器里手工执行 → 用验证命令确认 → 写进 Dockerfile 草稿 + 记录理由 → 进入下一段。失败则在该段内修复，不带病推进。

| 阶段 | 内容 | 验证方式 |
|---|---|---|
| S0 | 代理连通性、apt 源、基础工具 | curl 外网、apt install 成功 |
| S1 | Python ≥3.10（干净环境实测：base 系统 python 3.10.12 达标但无 pip/venv；用 base addendum 的 miniconda `conda create -p /opt/sglang python=3.10` 做环境隔离） | `/opt/sglang/bin/python -V`、`import torch` |
| S2 | venv `/opt/sglang` + 系统依赖（RDMA/OpenMPI/构建工具等，按上游 base 段裁剪） | apt 完整性、关键 .so 存在 |
| S3 | 核心依赖：Rust 工具链、sgl-kernel cu129 wheel、torch 2.13.0+cu129、flashinfer-cubin（+按需 DeepEP/DeepGEMM） | 逐包 `python -c "import ..."` |
| S4 | sglang 源码（clone main @ 固定 sha）+ `pip install --no-deps -e .` + kernels cubin | `python -c "import sglang"` |
| S5 | server 验证：起服务、发一次推理请求 | 健康检查 + 请求返回 |
| S6 | 分叉：开发镜像变体（补 dev 工具）与服务镜像变体（裁剪清单） | 各自启动冒烟 |

## 已定决策（使用者拍板，2026-10-30）

- **基线 × base**：保 base 12.9.1-devel-ubuntu22.04 → sglang 钉 **4349538c02**（2026-09-04，最后含 cu12 路径；torch 2.13.0 / Python ≥3.10 与 cu129 wheel 全配套）。放弃追最新 main（其依赖链已全量转 cu13）；内网无 12.9.2-cudnn-ubuntu24.04 官方导入，故不换 base
- **Python**：系统 python 3.10.12（干净环境实测，≥3.10 达标，但缺 pip/venv；早期"codelab 容器测得 3.6.8"是登录环境被 conda 污染的误读）。仍选 base addendum 的 miniconda `conda create -p /opt/sglang python=3.10`——为隔离与 pip 完整性，非版本必需
- **S0 收窄**：git/patch/make/gcc 等 base 自带（干净环境核实），S0 只剩代理连通性与磁盘/GPU 核查
- **验证纪律（2026-10-30 教训）**：镜像内容验证必须用非登录 shell（`env -i` + `bash --noprofile --norc`）或裸 `docker run`——平台组件（codelab）注入的 PATH/conda/家目录会伪装成镜像内容
- **验收模型**：由助手挑选 HF 小模型（数 GB 级，走代理下载）

## 裁剪决策（助手自决，依据记录进过程文档）

- 跳过 DeepEP/DeepGEMM/MSCCL++/mooncake：单卡 H20 推理验收非必需（多机 EP/传输引擎组件），后续需要再补
- 跳过 Rust gateway（sgl-model-gateway 二进制）：与推理验收无关；但 **Python 侧装 sglang 依赖需要 Rust 工具链**（sglang-grpc 的 setuptools-rust 扩展），Rust 必装——评审补漏项
- `kernels lock/download`（sgl-flash-attn3 cubin 预热）依赖 HF 网络，保留但失败可回退运行时 JIT（上游同款兜底）
- hpc-ops：跳过（opt-in 后端内核，非必需）
- cuDNN 缺失由 pip torch 自带 nvidia-cudnn-cu12 顶，S5 报错再补
- apt 包名需按 Ubuntu 22.04 核对（24.04 的 t64 后缀包、部分版本号包在 22.04 不存在）——评审补漏项


## 交付物清单

1. `Dockerfile.dev`、`Dockerfile.serving`（单阶段，FROM 内网 base）
2. 过程文档 `build-notes.md`：阶段记录、坑、与上游差异表
3. 跑通的环境（容器/环境本身）
4. 后续迁移到 mlp 平台构建的提示（components 格式对应关系）

## 环境等待项

构建环境由使用者在开工时提供。需要确认的属性：GPU 型号（预期 H20/sm90）、代理地址与用法、容器内可用磁盘空间（镜像层+pip 缓存预计 >30GB）、是否可长时间占用。
