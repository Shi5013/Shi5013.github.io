本文详细解释我在内容运行的docker命令，从基础概念开始。

## Introduction
我在公司服务器上运行自己的代码时，由于内网无法通过网络配置环境，所以使用docker，在经过配置后，运行下面这一行命令启动容器，然后运行代码。虽然已经运行很多次，但是并没有详细了解过docker以及这行代码的意思。因此本文做一个笔记。
```bash
sudo docker run --gpus all -it --rm -v /data/sww_code/hmm:/app my_gpu_env:1229 /bin/bash
```
以上命令是一个经典用Docker拉起GPU容器并挂卷的写法。想要深度理解，首先要了解一些Docker的基础概念。
## 一、基础概念
Docker是一个应用容器引擎。可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植容器中，然后发布到任何流行的linux机器上，也可以实现虚拟化。

容器时完全使用沙箱(sandbox)机制，相互之间不会有任何接口。本质上就是给每个容器画一个圈，让它只能在这个圈内活动，圈内发生的事不会影响到圈外，圈与圈之间也互不干扰。在Docker中这个圈由Linux内核提供的6把锁共同完成，包括namespace、cgroups、capabilities/seccomp、AppArmor / SELinux、rootfs + overlayfs和网络隔离，具体内容不做进一步介绍。
### 1.1 容器-Container
轻量化的运行实例，包括应用代码、运行环境和依赖库。基于镜像创建，与其他容器隔离，共享主机操作系统内核
### 1.2 镜像-Image
只读模板，定义了容器的运行环境(如操作系统、软件配置)。通过分层存储(Layer)优化空间和构建速度
### 1.3 Dockerfile
文本文件，描述如何自动构建镜像(例如指定基础镜像、安装软件、复制文件等)
### 1.4 仓库-Registry
存储和分发镜像的平台，如 Docker Hub（官方公共仓库）或私有仓库（如 Harbor）

## 二、命令解析

`docker run`：启动一个容器

`--gpus all`：把宿主机所有GPU设备透传给容器，底层实际调用nvidia-container-runtime

`-it` `-i`保持STDIN打开。就是告诉Docker，即使当前终端没有立即输入，也不要把容器的标准输入(STDIN)关掉。只有需要交互式输入的时候才`-i`，如果容器内的进程不等人敲命令就能完成工作，就不用`-i`。 加了 `-i`  STDIN 一直连着，bash 会等你敲命令，就像在本机开了一个 shell 一样。`-t` 分配一个伪终端。

`-v /data/sww_code/hmm:/app`:把宿主机目录`/data/sww_code/hmm` 挂载到容器内的 `/app`，读写双向同步。
这两个文件夹任何一方的新建，修改，都会在另一方体现，并不存在先写缓存、定时同步这类延迟机制。举例：
- 在容器里 `echo 123 > /app/test.txt`，宿主机 `/data/sww_code/hmm/test.txt` 立即出现。
- 在宿主机 `rm /data/sww_code/hmm/old.log`，容器里 `/app/old.log` 也瞬间消失。

`my_gpu_env:1229`:使用的镜像名+标签。镜像是只读模板，容器时基于该模板启动的正在运行的实例。容器就是镜像＋可写顶层，启动时Docker在镜像层最上面加一层可写层，所有运行时修改都在这层。容器停止/删除后，可写层默认被丢弃，镜像本身保持不变。无论在容器里怎么折腾，镜像本身永远只读，不会被改写。一个镜像可以启动任意多个相互隔离的容器实例，彼此完全隔离。

`/bin/bash`: 容器启动后执行的默认命令：启动一个 bash shell，方便你手动操作。这并不是一个必须写的命令，只是一个普通进程。不写时，Docker会拿镜像里Dockerfile最后一行CMD规定的默认命令来启动；如果镜像里也没写CMD，容器会立即退出。

## 三、一些问题

**每一个容器中都包含一个操作系统吗？**
不是“完整操作系统”，而是“足够让应用以为自己跑在独立 OS 里的最小环境”。每个容器并不包含一个完整操作系统，只包含“让应用跑起来所需的最小用户态环境”。所有容器共享宿主机的 **同一个 Linux 内核**（宿主机内核版本决定容器能用的系统调用）。-在容器里 `ps`、`top`、`ls /` 看上去像一台小电脑，但它其实只是宿主机内核上的一个“被隔离的进程组”。

我的dockerfile：
```
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime

ENV DEBIAN_FRONTEND=noninteractive

WORKDIR /app

RUN pip install --no-cache-dir torchdata==0.7.0 && \
    pip install --no-cache-dir dgl -f https://data.dgl.ai/wheels/cu121/repo.html && \
    pip install --no-cache-dir \
    numpy \
    pydantic \
    scipy \
    tqdm \
    networkx \
    pandas \
    tensorboard \
    osmnx \
    ipywidgets \
    geopandas \
    haversine

# 设置默认启动命令
CMD ["/bin/bash"]
```