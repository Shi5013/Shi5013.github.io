---
layout: post
title: 在内网中部署图像生成
tags: record
math: true
date: 2026-3-09 9:13 +0800
---

本文记录在内网中实现图像生成功能的的过程。

## 一、图像生成引擎选择
根据检索结果，有三个东西可以实现在openwebui中集成图像生成功能：AUTOMATIC1111，comfyui，OpenAI DALL·E。其中OpenAI DALL·E需要联网，无法在内网中使用。根据leader安排，部门使用的图像生成模型是Z-Image Turbo。官方推荐和社区实践都倾向于使用ComfyUI。具体原因：

1. [官方](https://z-image.ai/blog/z-image-github-guide-to-perfect-text)提供了现成的 ComfyUI 工作流文件，可以直接导入使用；
2. 网上有大量针对 ComfyUI + Z-Image Turbo 的部署教程和问题解决方案；
3. ComfyUI 的节点式工作流能充分发挥 Z-Image Turbo 快速生成（8步即可出图）的优势，并且方便未来进行更复杂的图像编辑。

deepseek一直建议我将openwebui升级到最新版本，我的版本目前是0.6.30，目前是支持图像生成功能的。而查阅openwebui的release，在[0.8.1](https://github.com/open-webui/open-webui/releases/tag/v0.8.1)版本更新了数据库：

> Database Migrations: This release includes database schema changes; we strongly recommend backing up your database and all associated data before upgrading in production environments. If you are running a multi-worker, multi-server, or load-balanced deployment, all instances must be updated simultaneously, rolling updates are not supported and will cause application failures due to schema incompatibility.

所以暂时先不更新openwebui。

## 二、内网部署ComfyUI
寻求ComfyUI的docker部署方式，在github找到一个[仓库](https://github.com/YanWenKun/ComfyUI-Docker)。其中有slim版本和megepak版本。第一次遇到这种情况，我也不知道要部署哪一个版本，deepseek给的建议是先部署轻量级的slim版本，运行没有问题再部署完整版的megapak。听信deepseek的建议是一个不幸的选择，因为slim版本中并没有cuda，因此当我download，save，load在内网之后，在启动容器时有一个命令`--runtime nvidia`我用的docker-compose，命令是`runtime:nvidia`。不幸从此开始。

### 2.1 runtime:nvidia
启动容器时在这一行出现报错：
```
failed to create shim task:OCI runtime create failed:unabled to retrieve OCI runtime error :exec:"nvidia":executable file not found in$PATH
```

指示Docker引擎在宿主机系统中找不到名为nvidia的容器运行时。这意味着Docker还没有配置好支持GPU的能力。于是开始检索哪里的cuda出了问题，首先是宿主机，查看宿主机的cuda和cudnn，这都是没有问题的。那也就是在docker中出了问题。

#### 运行时runtime
docker运行时是负责真正启动和运行容器的底层组件。运行时是一个多层次的概念，指的是让容器能够真正运行起来的一系列软件组件。简单的说，从输入`docker run`命令到容器内程序开始执行，这背后经过的整个技术栈，都属于运行时范畴。

运行时分为两层结构：
1. 高层运行时：主要负责容器镜像的管理和容器生命周期的控制。它负责拉取镜像、管理容器的存储和网络、监控容器状态等。本身并不直接运行容器进程，而是调用下层组件来完成实际工作
2. 底层运行时：与操作系统内核交互，负责启动和运行容器进程的组件。

而`runtime:nvidia`，时在这个技术栈上进行了一次关键的替换或者增强。它不是一个底层的运行时，而是一个基于默认`runc`的包装器。工作原理：
1. 劫持流程：当通过`runtime:nvidia`启动容器时，Docker会调用`nvidia-container-runtime`而不是直接调用`runc`；
2. 检查需求：`nvidia-container-runtime`首先会检查容器的配置，特别是环境变量，以判断这个容器是否需要访问GPU；
3. 动态注入：如果需要 GPU，它会在调用真正的 `runc` 之前，将一个名为 `nvidia-container-runtime-hook` 的钩子（hook）程序注入到容器的生命周期中
4. 准备环境：钩子程序通过`libnvidia-container`挂载GPU设备，挂载驱动库；
5. 启动容器

这种机制的好处在于，你无需在容器镜像中打包庞大的 NVIDIA 驱动，只需安装精简的 CUDA 工具包即可，实现了驱动与应用的解耦。

下面说回报错的问题，这个slim版本的镜像中，没有找到相应的cuda，但是最开始并没有意识到这个问题，尝试了很多办法想把宿主机GPU挂载到容器上，浪费了一天时间。后来直接使用megapak版本的镜像，就解决了这个问题。

## 三、模型问题
根据ComfyUI教程编写docker-compose.yml文件，这里有一个yaml和yml的疑惑。实际上，YML 和 YAML 没有本质区别，它们是同一种数据序列化格式。可以简单理解为：YAML 是官方名称（标准叫法），YML 是文件扩展名（缩写）。早期的 Docker Compose 文件默认名为 docker-compose.yml，因此这个后缀在容器领域非常普遍（现在也支持 .yaml）

所以这里我也使用docker-compose.yml。其中关于文件夹的挂载：

```yaml
./storage:/root 
./storage-models/models:/root/ComfyUI/models 
```
在model中，需要放三个safetensors，分别为，text_encoder，diffusion，vae:

- Text encoder file: [qwen_3_4b.safetensors](https://huggingface.co/Comfy-Org/z_image_turbo/blob/main/split_files/text_encoders/qwen_3_4b.safetensors)(goes in ComfyUI/models/text_encoders/).
- diffusion model file: [z_image_turbo_bf16.safetensors](https://huggingface.co/Comfy-Org/z_image_turbo/blob/main/split_files/diffusion_models/z_image_turbo_bf16.safetensors)(goes in ComfyUI/models/diffusion_models/).
- VAE: [ae.safetensors](https://huggingface.co/Comfy-Org/z_image_turbo/blob/main/split_files/vae/ae.safetensors) the Flux 1 VAE if you don’t have it already (goes in ComfyUI/models/vae/)

然而我用内网中已经下载好的文件，搭建流程后生成的图像是纯黑的，查看docekr logs的输出的后50行，其中有一个：
`Model AutoencodingEngine prepared for dynamic VRAM loading. 159MB Staged.0 patches attached`
这说明vae有问题，deepseek说是因为使用的数值不同的原因。

### 3.1 大模型数值精度

| 精度格式 | 类型   | 位宽 | 典型应用阶段           | 核心特点                                                   | 内存占用(相对) |
| :-------: | :-----: | :---: | :---------------------: | :---------------------------------------------------------: | :-------------: |
| FP32     | 浮点   | 32位 | 训练(主权重更新)       | 精度极高，数值稳定，但计算慢、占内存大                     | 基准 (1x)      |
| BF16     | 浮点   | 16位 | 训练(前向/后向计算)    | 动态范围与FP32相同，训练稳定，是现代AI硬件的首选           | 约 1/2         |
| FP16     | 浮点   | 16位 | 训练/推理              | 精度比BF16高，但动态范围窄，容易溢出                       | 约 1/2         |
| FP8      | 浮点   | 8位  | 训练(前沿)/推理        | 有E4M3(高精度)和E5M2(广范围)两种变体，正成为新标准         | 约 1/4         |
| INT8     | 整数   | 8位  | 推理(生产主力)         | 技术成熟，在保证模型质量的同时，能显著加速推理             | 约 1/4         |
| INT4     | 整数   | 4位  | 推理(极致压缩)         | 压缩率极高，适合在手机等边缘设备部署，但需精良算法保质量   | 约 1/8         |

在实际应用中，大模型并不会只使用一种精度，而是会采用更灵活的策略：
1. 混合精度训练：在计算时主要使用FP16或BF16来加快速度、节省显存，同时保留一份FP32的“主权重”来累加更新，确保训练的稳定性和最终效果；
2. 将训练好的高精度模型“压缩”成低精度（如INT8、INT4）用于推理的过程；
3. 分而治之：模型的不同部分对精度敏感度不同。例如，对 outliers（离群值）敏感的嵌入层和输出层，在量化时往往需要保留更高精度，而中间层则可以大胆使用低比特，这就是所谓的“保护边缘”策略。

#### bf和fp的区别
FP16（IEEE标准）：它用5位表示指数，10位表示尾数。这使它能够精确表示较精细的小数，但能表示的数值最大不超过 65504，最小能表示的规范化正数也有限。在深度学习训练中，梯度很容易就“溢出”（变成无穷大）或“下溢”（变成0），这是它的主要痛点。
BF16（Google发明的Brain Float）：它直接“克隆”了FP32的指数位（8位），只是截断了尾数位（从FP32的23位减到7位）。这意味着它的动态范围和FP32完全一样（最大约 3.4e38，最小约 1.18e-38），几乎不会溢出。代价就是精度大幅降低，尾数只有7位，在表示小数时可能会粗糙一些。

在训练阶段bf16完胜，在推理阶段，bf16逐渐称为主流。

| 精度格式 | 符号位 | 指数位 | 尾数位 | 公式表示 (近似)           | 核心特点         |
| :------- | :----- | :----- | :----- | :------------------------- | :--------------- |
| FP16     | 1位    | 5位    | 10位   | `(-1)^S * 2^(E-15) * 1.M`  | 精度高，范围窄   |
| BF16     | 1位    | 8位    | 7位    | `(-1)^S * 2^(E-127) * 1.M` | 范围广，精度低   |