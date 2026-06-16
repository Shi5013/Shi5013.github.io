---
layout: post
title: SSH 密钥、公钥私钥、known_hosts 是什么
tags: record
math: false
date: 2026-6-16 14:30 +0800
---

从一次 `git clone` 的报错说起，搞清楚 SSH 密钥体系的三个核心问题：公钥私钥是什么、known_hosts 干什么用、以及同一个密钥能否用于多个服务。

## 公钥和私钥

SSH 使用**非对称加密**，也就有一对密钥：

| | 私钥 | 公钥 |
|---|---|---|
| 在哪里 | 只在你电脑上 `~/.ssh/id_rsa` | 上传到要访问的服务端 |
| 能不能给别人 | **绝对不能** | 可以公开 |
| 作用 | 证明**你就是你** | 验证**你确实是私钥持有者** |

类比：私钥是钥匙，公钥是锁。你把锁（公钥）装在服务端，只有你的钥匙（私钥）能打开。

## known_hosts 是什么

私钥证明"你是谁"，known_hosts 则反过来证明**对方是谁**。

你第一次 SSH 连一台新机器时，会弹出：

```
The authenticity of host 'github.com' can't be established.
ED25519 key fingerprint is SHA256:+DiY3w...
Are you sure you want to continue connecting (yes/no)?
```

输入 `yes` 后，对方指纹被写进 `~/.ssh/known_hosts`。以后每次连接，SSH 都会对比指纹是否和上次一致：

- **一致** → 继续连接
- **不一致** → 报警，拒绝连接

这是在防御**中间人攻击**——没有 known_hosts，攻击者可以伪装成 GitHub 截获你的代码和密钥交互。

`~/.ssh/` 目录下常见文件：

| 文件 | 作用 |
|---|---|
| `id_rsa` | 私钥，证明"我是我" |
| `id_rsa.pub` | 公钥，服务端验证"你确实是你" |
| `known_hosts` | 服务端指纹，验证"你真的是某某服务器，不是假的" |
| `config` | 连接规则，给不同主机指定不同参数 |

## 可以共用同一对密钥

一台电脑可以有多对密钥，不同服务也可以用同一把：

```
~/.ssh/
├── id_rsa          # GitHub
├── id_rsa.pub
├── id_ed25519      # 公司 GitLab
├── id_ed25519.pub
└── config          # 给不同服务指定用哪把密钥
```

config 里可以这样区分：

```
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/id_rsa

Host gitlab.company.com
  HostName gitlab.company.com
  IdentityFile ~/.ssh/id_ed25519
```

如果不写 `IdentityFile`，SSH 默认用 `~/.ssh/id_rsa`，于是所有服务共用一把钥匙。

## 各种服务都在用这套

| 服务 | 公钥放哪里 |
|---|---|
| GitHub / GitLab | 网页 Settings → SSH Keys |
| 云服务器 / NAS | `~/.ssh/authorized_keys` 文件 |
| VS Code Remote | 远端机器的 `authorized_keys` |
| scp / rsync | 同上，只要对方是 SSH 协议 |

不管对面是什么服务，流程都一样：你持私钥，对方存公钥。

## 一次实战：clone 报错的排查过程

```bash
git clone git@github.com:Shi5013/Shi5013.github.io.git .
```

三连报错：

1. **`Host key verification failed`** — `known_hosts` 没有 GitHub 的指纹，手动添加：
   ```bash
   ssh-keyscan github.com >> ~/.ssh/known_hosts
   ```

2. **`Permission denied (publickey)`** — 公钥没上传到 GitHub，在 [github.com/settings/keys](https://github.com/settings/keys) 添加上即可

3. **`目标路径 '.' 已存在，并且不是一个空目录`** — 当前目录有 `.claude` 文件夹，git 拒绝覆盖非空目录。解决：clone 到临时目录再移入
