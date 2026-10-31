# sglang (`4349538c02`) 依赖版本锁与 CUDA 12.9 适配清单 (`versions.md`)

> 数据源：`/opt/sglang-src/python/pyproject.toml` 與 `/opt/sglang-src/docker/Dockerfile` @ `4349538c02e1566a1424510d5ac3ae853f49feef`（2026-09-04）

---

## 一、核心 GPU / C++ / Rust 扩展组件（独立分步安装并逐一验证）

| 组件 / 包名 | `4349538c02` 锁定版本 | 安装源 / Wheel 地址 | CUDA 12.9 / Python 3.10 适配说明 | 出处 |
|---|---|---|---|---|
| `torch` | `2.13.0+cu129` | `--index-url https://download.pytorch.org/whl/cu129` | 自带 `nvidia-cudnn-cu12` 等 CUDA 12 动态库，弥补 base 镜像无 cuDNN | `pyproject.toml:94`, `Dockerfile:258` |
| `torchvision` | `0.28.0+cu129` | `--index-url https://download.pytorch.org/whl/cu129` | 与 `torch==2.13.0+cu129` 配套 | `Dockerfile:258` |
| `torchaudio` | `2.11.0+cu129` | `--index-url https://download.pytorch.org/whl/cu129` | 与 `torch==2.13.0+cu129` 配套 | `pyproject.toml:96`, `Dockerfile:258` |
| `sglang-kernel` (`import sgl_kernel`) | `0.4.6.post1+cu129` | `https://gh-proxy.com/https://github.com/sgl-project/whl/releases/download/v0.4.6.post1/sglang_kernel-0.4.6.post1+cu129-cp310-abi3-manylinux2014_x86_64.whl` | **不能从 PyPI 直装**（PyPI 默认是 cu13 版）；使用 `cp310-abi3` 预编译 wheel 并校验官方 `sha256=9d400246b3f2512e2c33611e1bb6a2f252c499c5019e9f33611e1bb6a2f252c499c5019e9f331f9c9ee215be95ab` | `Dockerfile:6,207` |
| `flashinfer_python` | `0.6.18` (`[cu12]`) | PyPI (`mirrors.aliyun.com` / `pypi.org`) | `pyproject.toml:48` 原为 `flashinfer_python[cu13]==0.6.18`，按 `Dockerfile:250` 改写为 `flashinfer_python[cu12]==0.6.18` | `pyproject.toml:48`, `Dockerfile:17,250` |
| `flashinfer-cubin` | `0.6.18` | `--index-url https://flashinfer.ai/whl` | CUDA 版本无关的预编译 cubin 包 | `Dockerfile:318` |
| `cuda-python` | `>=12,<13` | PyPI | `pyproject.toml:38` 原为 `cuda-python>=13.0`，按 `Dockerfile:249,571` 改写为 `cuda-python>=12,<13` | `pyproject.toml:38`, `Dockerfile:249,571` |
| `nvidia-cutlass-dsl` | `4.6.2` | PyPI | `pyproject.toml:61` 原为 `nvidia-cutlass-dsl[cu13]==4.6.2`，按 `Dockerfile:251` 去掉 `[cu13]` | `pyproject.toml:61`, `Dockerfile:251` |
| `humming-kernels` | `0.1.12` | PyPI | `pyproject.toml:50` 原为 `humming-kernels[cu13]==0.1.12`；其 `[cu13]` extra 会拉入 `cuda-Bindings==13.*`，在 cu12 下去掉 `[cu13]` extra 仅装本体 `humming-kernels==0.1.12` | `pyproject.toml:50` |

---

## 二、按 Spec 决策跳过的多机 / 可选组件（单卡/单机 H20 推理非必需）

| 组件 | 原锁定版本 | 跳过原因与处理方式 | 出处 |
|---|---|---|---|
| `sgl-deep-ep` | `0.1.2` | 多机 Expert Parallelism 通信库（依赖 IBGDA/NVSHMEM）；在 `/opt/sglang-src/python/pyproject.toml` 中注释掉 | `pyproject.toml:84`, `Spec:69` |
| `sgl-deep-gemm` | `0.1.7` | DeepSeek FP8 专用 GEMM 可选包（按需 JIT）；在 `/opt/sglang-src/python/pyproject.toml` 中注释掉 | `pyproject.toml:85`, `Spec:69` |
| `mooncake-transfer-engine` | `0.3.13` | PD 分离 KV 传输引擎，单实例推理不调用，不安装 | `Dockerfile:18,505`, `Spec:69` |
| `mscclpp` | `sglang-v0.9.1` | 微软集合通信可选后端，单卡不调用，不安装 | `Dockerfile:19,515`, `Spec:69` |
| `hpc-ops` | `ab1a4027` | 混元可选后端算子，默认不启用，不安装 | `Dockerfile:285`, `Spec:72` |
| `sgl-model-gateway` |源码构建 | 独立 Rust 路由网关二进制，与 `sglang.launch_server` 无关，不构建 | `Dockerfile:386`, `Spec:70` |

---

## 三、`pyproject.toml` 其余标准依赖锁定清单（由 `pip install` 统一安装）

```text
aiohttp
anthropic>=0.20.0
apache-tvm-ffi==0.1.11
blobfile==3.0.0
build
compressed-tensors==0.18.0
cuda-python>=12,<13
cuda-tile==1.6.0rc5
datasets
distro
easydict
einops
fastapi
flash-attn-4>=4.0.0b18
flashinfer_python[cu12]==0.6.18
gguf
humming-kernels==0.1.12
interegular
IPython
kernels>=0.14.1,<0.15
llguidance>=1.7.6,<2.0.0
mistral_common>=1.11.5
modelscope
msgspec
ninja
numba==0.65.1
numpy
nvidia-cutlass-dsl==4.6.2
nvidia-mathdx==25.6.0
nvidia-ml-py
openai==2.6.1
openai-harmony==0.0.4
orjson
outlines==0.1.11
packaging
partial_json_parser
pillow
prometheus-client>=0.20.0
psutil
py-spy
pybase64
pydantic
python-multipart
pyzmq>=25.1.2
quack-kernels==0.6.4
requests
scipy
sentencepiece
setproctitle
smg-grpc-servicer>=0.5.0
soundfile==0.13.1
tiktoken
tilelang==0.1.12
timm==1.0.16
tokenizers==0.22.2
tokenspeed_mla==0.1.8
tomli ; python_version < '3.11'
torch==2.13.0
torch_memory_saver>=0.0.9.post1
torchaudio==2.11.0
torchcodec==0.15.0
torchvision
tqdm
transformers==5.12.1
uvicorn
uvloop
watchfiles
xgrammar==0.2.1
xxhash
zstandard
```
