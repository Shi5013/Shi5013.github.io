---
layout: post
title: Base image
tags: record
math: false
date: 2026-08-07 16:30 +0800
---

# Docker 基础镜像构建与依赖排障记录

## 1. 目标与最终结果

目标是在可联网但没有 NVIDIA GPU 的电脑上构建一个通用机器学习基础镜像，再将镜像部署到内网 GPU 服务器。多个项目共同使用 Torch、CUDA Runtime、vLLM、Transformers、NumPy 等依赖，避免每个项目重复下载和安装。

最终形成两层镜像：

| 层级 | 镜像 | 用途 |
| --- | --- | --- |
| 通用基础层 | `ml-base:vllm0.26-cu129` | 提供 vLLM、Torch、CUDA 12.9 Runtime、科学计算库和常用 API SDK |
| Tiermem 项目层 | `tiermem:0807` | 在基础层上增加 Qdrant、Sentence Transformers、FastEmbed、Tantivy 等依赖 |

相关文件：

- `Dockerfile`：构建通用基础镜像。
- `requirements-base.txt`：基础镜像需要额外安装的包。
- `constraints-vllm.txt`：保护 vLLM 官方镜像中的核心依赖版本。
- `Dockerfile.tiermem`：构建 Tiermem 项目镜像。
- `requirements-tiermem.txt`：Tiermem 缺少的直接依赖及固定版本。
- `constraints-tiermem.txt`：Tiermem 新增依赖的传递依赖版本。

Docker 镜像仓库名必须使用小写，因此使用 `tiermem:0807`，不能使用 `Tiermem:0807`。运行后的容器名称可以使用 `Tiermem-0807`。

## 2. 基础镜像与 CUDA 兼容性

### 环境信息

- 构建机：没有 NVIDIA GPU。
- 内网宿主机：Ubuntu 22.04.5 LTS。
- NVIDIA 驱动：595.58.03。
- `nvidia-smi` 显示最高支持 CUDA 13.2。
- 基础镜像：`vllm/vllm-openai:v0.26.0-cu129-ubuntu2404`。
- 固定镜像摘要：

```text
sha256:f21f5e1987142d4a7c77a4fe41726ab2910153c5253df61e883771258db59440
```

### 结论

构建镜像时不需要 GPU。镜像内携带 CUDA 12.9 用户态 Runtime，真正运行 GPU 任务时才需要宿主机驱动和 NVIDIA Container Toolkit。

宿主机使用 Ubuntu 22.04、容器使用 Ubuntu 24.04 通常没有问题，因为容器使用宿主机内核，但拥有自己的用户态文件系统。595 系列驱动支持该 CUDA 12.9 Runtime。

基础镜像采用 tag 和 digest 同时固定：

```dockerfile
ARG VLLM_BASE_IMAGE=vllm/vllm-openai:v0.26.0-cu129-ubuntu2404@sha256:f21f5e1987142d4a7c77a4fe41726ab2910153c5253df61e883771258db59440
FROM ${VLLM_BASE_IMAGE}
```

这样可以防止上游镜像标签被更新后，构建结果在不知情的情况下变化。

## 3. Docker 镜像下载失败

### 典型现象

```text
lease does not exist: not found
```

或：

```text
short read: expected ... bytes but got ...: unexpected EOF
```

以及拉取大镜像层时：

```text
failed to copy ... image-mirror.r2.daocloud.vip ... EOF
```

### 原因

这些报错虽然可能显示在 `WORKDIR` 等简单指令上，但通常不是 Dockerfile 指令本身有问题，而是基础镜像层没有完整写入本地内容存储。常见诱因包括：

- 镜像站连接中断，大镜像层只下载了一部分。
- BuildKit 中留下损坏或失效的 lease/cache 记录。
- 多个镜像站质量不一致，同一次拉取可能被转发到不稳定节点。
- vLLM 镜像包含数 GB 的大层，网络短暂中断更容易暴露问题。

### 处理方法

先单独拉取并验证基础镜像，不要一开始就反复执行完整构建：

```bash
docker pull 'vllm/vllm-openai:v0.26.0-cu129-ubuntu2404@sha256:f21f5e1987142d4a7c77a4fe41726ab2910153c5253df61e883771258db59440'
```

成功标志：

```text
Status: Downloaded newer image
Digest: sha256:f21f5e1987142d4a7c77a4fe41726ab2910153c5253df61e883771258db59440
```

基础镜像完整存在本地后，后续 `FROM` 会复用本地内容寻址层，不会重新下载数 GB 数据。Docker 仍可能访问仓库检查 manifest，但镜像层不需要重新获取。

如果确认本地 BuildKit 缓存已经损坏，可以清理构建缓存后重试：

```bash
docker builder prune
```

该命令会删除未使用的构建缓存，应先用下面的命令检查空间，再确认是否需要执行：

```bash
docker system df
df -h /var/lib/docker
docker buildx ls
```

不要因为一次 EOF 就直接删除全部镜像；已经成功拉取的大镜像层很有价值。

## 4. Docker Hub、镜像站与代理的关系

### 当前镜像站配置

`/etc/docker/daemon.json` 中配置了多个 registry mirror，包括 DaoCloud、南京大学、dockerproxy、百度等。

拉取日志中出现：

```text
image-mirror.r2.daocloud.vip
```

说明该次 blob 下载经过了 DaoCloud 镜像服务，而不是直接从 Docker Hub blob 存储下载。

### 终端代理不等于 Docker daemon 代理

终端中存在：

```text
http_proxy=http://127.0.0.1:7890
https_proxy=http://127.0.0.1:7890
all_proxy=socks5://127.0.0.1:7891
```

这些变量会影响当前 Shell 中启动的部分程序，但 Docker 镜像拉取主要由 Docker daemon 执行。daemon 不会自动继承当前终端后来设置的代理变量。

另外，在普通构建容器中，`127.0.0.1` 指向容器自身，并不指向宿主机代理。只有 Docker daemon 本身访问宿主机回环地址，或者构建明确使用 `--network=host` 时，才能按相应网络语义访问宿主机服务。

### 实际网络测试结果

通过 HTTP 代理访问 PyPI 时 TLS 握手失败：

```text
HTTP/1.1 200 Connection established
curl: (35) SSL routines::unexpected eof while reading
```

使用 SOCKS 代理长时间无响应。绕过代理后，官方 PyPI 和清华 PyPI 镜像都可以正常访问：

```bash
curl -I --noproxy '*' https://pypi.org/simple/openai/
curl -I --noproxy '*' https://pypi.tuna.tsinghua.edu.cn/simple/openai/
```

因此本次构建采用：

- 清空代理变量。
- `--network=host`。
- Python 包索引使用清华镜像。

```bash
env -u http_proxy -u https_proxy -u all_proxy \
    -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
docker buildx build \
    --network=host \
    --build-arg PYTHON_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    ...
```

镜像站可以降低直接访问 Docker Hub 的需求，但它仍是第三方网络服务，并不代表完全不需要可靠网络。对于超大镜像，最稳定的方法是成功拉取一次后保留本地镜像，并在内网使用 `docker save`/`docker load` 或自建 Registry。

## 5. Dockerfile frontend 下载失败

### 现象

```text
failed to fetch anonymous token
TLS handshake timeout
```

或：

```text
failed to resolve source metadata for docker.io/docker/dockerfile:1
```

报错位置是：

```dockerfile
# syntax=docker/dockerfile:1
```

### 原因与解决方法

这行指令要求 BuildKit 解析并获取 `docker/dockerfile:1` frontend。网络不稳定时，即使业务基础镜像已经在本地，也可能因为这个额外访问而失败。

当前 Docker/BuildKit 已支持所需的 cache mount 语法，因此删除 `# syntax=docker/dockerfile:1`，直接使用内置 frontend，避免额外访问 Docker Hub。

## 6. PyPI TLS 握手失败

### 现象

```text
Failed to fetch: https://pypi.org/simple/openai/
tls handshake eof
```

### 原因

宿主机测试证明直连 PyPI 正常，而通过 `127.0.0.1:7890` 的代理连接在 CONNECT 成功后中断 TLS。`-k`、强制 TLS 1.2 等操作都无效，说明不是证书验证问题，而是代理链路本身中断。

### 解决方法

不要继续关闭 TLS 校验。清除代理并使用可直连的包索引：

```dockerfile
ARG PYTHON_INDEX_URL=https://pypi.org/simple

RUN UV_HTTP_RETRIES=10 \
    UV_HTTP_TIMEOUT=120 \
    UV_CONCURRENT_DOWNLOADS=1 \
    uv pip install \
      --system \
      --default-index "${PYTHON_INDEX_URL}" \
      --requirement /path/to/requirements.txt
```

构建时切换到清华镜像：

```bash
--build-arg PYTHON_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple
```

增加重试和超时可以缓解临时抖动，但不能修复错误的代理链路。

## 7. vLLM 与 NumPy 版本冲突

### 现象

第一次固定 `numpy==2.5.1` 时：

```text
vllm>=0.26.0 -> numba==0.65.0 -> numpy<2.5
```

改为 `numpy==2.4.6` 后仍然冲突：

```text
vllm>=0.26.0 -> mistral-common>=1.11.5 -> numpy<2.4
```

### 原因

只满足某个直接依赖的版本范围不代表整个依赖图兼容。vLLM 会通过 Numba、Mistral Common 等传递依赖对 NumPy 提出更严格的上限。

### 解决方法

不要分别追逐“最新版本”。以 vLLM 官方镜像已经验证的 Torch/CUDA/vLLM 组合为核心，再让解析器选择满足整个依赖图的版本。最终采用：

```text
vllm==0.26.0+cu129
torch==2.11.0+cu129
numpy==2.2.6
numba==0.65.0
transformers==5.14.1
```

将这些关键版本写入 `constraints-vllm.txt`，安装额外包时始终传入：

```bash
uv pip install \
  --constraint /opt/base-image/constraints-vllm.txt \
  --requirement requirements.txt
```

项目依赖不要再次要求安装已有的 Torch、vLLM 和 NumPy，只安装基础镜像中缺失的包。

## 8. 如何获得一套兼容且可复现的版本

宽泛的最低版本约束适合描述项目需求，例如 `numpy>=1.24`，但不适合直接作为长期稳定镜像的最终锁定结果，因为以后重新构建可能解析出完全不同的新版本。

采用两阶段方法：

1. 在真实基础镜像内，用用户给出的最低版本范围执行 `uv pip install --dry-run`。
2. 检查解析结果不会升级核心栈，然后把选中的直接和传递依赖固定下来。

示例：

```bash
docker run --rm --network host \
  -e HTTP_PROXY= -e HTTPS_PROXY= -e ALL_PROXY= -e NO_PROXY='*' \
  -v "$PWD/requirements-tiermem.in:/tmp/requirements-tiermem.in:ro" \
  ml-base:vllm0.26-cu129 \
  uv pip install \
    --dry-run \
    --system \
    --default-index https://pypi.tuna.tsinghua.edu.cn/simple \
    --constraint /opt/base-image/constraints-vllm.txt \
    --requirement /tmp/requirements-tiermem.in
```

Tiermem 最终只需要新增 7 个直接依赖：

```text
qdrant-client==1.19.0
sentence-transformers==5.7.0
fastembed==0.8.0
tantivy==0.26.0
tenacity==9.1.4
pytz==2026.3.post1
json-repair==0.62.0
```

其余用户要求的包已经由基础镜像满足，因此不会重复安装。

## 9. NIXL 元数据不匹配

### 现象

```text
The package nixl requires nixl-cu12==1.3.1, but 1.3.2 is installed
```

### 原因

该不匹配来自上游 vLLM 官方镜像本身：`nixl` 的包元数据要求 1.3.1，但镜像实际携带 1.3.2。它不是本次新增 Python 包造成的。

### 处理方法

不根据单条元数据告警贸然把 `nixl-cu12` 降级，因为降级可能破坏上游镜像编译和验证过的运行组合。本次处理为：

- 保留官方镜像中的 `nixl-cu12==1.3.2`。
- 不把全局 `uv pip check` 作为构建成功的唯一判据。
- 精确验证 Torch、vLLM、NumPy 等核心版本未被改变。
- 对项目需要的模块执行实际导入测试。

如果将来 vLLM 官方镜像修正该元数据，应重新解析并删除这项临时处理。

## 10. 分层构建如何避免重复下载

项目镜像直接使用本地基础镜像：

```dockerfile
ARG BASE_IMAGE=ml-base:vllm0.26-cu129
FROM ${BASE_IMAGE}
```

Docker 使用内容寻址层。只要基础镜像仍保存在本机、`FROM` 指向相同镜像，而且没有主动清理镜像层，就会直接复用基础层。

Tiermem 构建命令：

```bash
env -u http_proxy -u https_proxy -u all_proxy \
    -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
docker buildx build \
    --builder default \
    --network=host \
    --build-arg BASE_IMAGE=ml-base:vllm0.26-cu129 \
    --build-arg PYTHON_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    --load \
    --progress=plain \
    -f Dockerfile.tiermem \
    -t tiermem:0807 \
    .
```

Dockerfile 中还使用 BuildKit 的 uv 缓存：

```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv,sharing=locked \
    uv pip install ...
```

该缓存不会写入最终镜像层，但在本机构建缓存仍存在时，可复用已经下载的 Python wheel。

为了保持缓存命中率：

- 先复制依赖文件并安装，再复制频繁变化的项目代码。
- 不要无意义地修改依赖文件。
- 不要频繁执行 `docker builder prune` 或 `docker system prune`。
- 为项目准备 `.dockerignore`，排除 `.git`、模型、数据集、虚拟环境和缓存。

## 11. 构建和运行验证

Tiermem 镜像已经实际构建成功，新增 19 个包（7 个直接依赖及其传递依赖），并完成全部目标模块的 CPU 导入测试：

```text
Tiermem dependency imports: OK
```

在没有 GPU 的构建机上进入容器：

```bash
docker run --rm -it \
  --name Tiermem-0807 \
  --entrypoint /bin/bash \
  tiermem:0807
```

基础检查：

```bash
python3 -c "import torch, vllm; print(torch.__version__); print(vllm.__version__); print(torch.cuda.is_available())"
```

构建机没有 GPU 时，`torch.cuda.is_available()` 返回 `False` 是正常现象，不表示 CUDA 版 Torch 安装失败。

在安装了 NVIDIA Container Toolkit 的 GPU 宿主机上：

```bash
docker run --rm -it \
  --gpus all \
  --name Tiermem-0807 \
  tiermem:0807
```

然后检查：

```bash
nvidia-smi
python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

## 12. 传输到内网

如果内网不能连接 Docker Hub，可以在联网机器导出：

```bash
docker save tiermem:0807 | gzip > tiermem_0807.tar.gz
```

传入内网后加载：

```bash
gzip -dc tiermem_0807.tar.gz | docker load
```

`tiermem:0807` 已经包含它依赖的基础镜像层，因此只需导出最终镜像。Docker 会按层去重；如果内网宿主机已经有相同基础层，加载时不会额外保存一份完全相同的数据。

如果需要长期给多个内网节点分发镜像，建议部署内网 Docker Registry；客户端从内网 Registry 拉取时同样按层复用，只传输缺失层。

## 13. 推荐的后续维护规则

1. 不使用 `latest` 作为生产基础镜像，固定 tag 和 digest。
2. 把 vLLM、Torch、CUDA、NumPy 视为一个整体升级，不单独追逐最新版本。
3. 新项目只声明基础镜像中缺少的依赖。
4. 先在基础镜像中执行 dry-run，再保存解析结果。
5. 依赖文件或基础镜像发生变化时重新执行导入测试和 GPU 冒烟测试。
6. 给新版本使用新标签，例如 `tiermem:0808`，不要覆盖已经验证的 `tiermem:0807`。
7. 保存 Dockerfile、requirements、constraints 和基础镜像 digest，确保环境可以复现。
