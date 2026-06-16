---
layout: post
title: curl访问大模型服务
tags: record
math: true
date: 2026-6-16 14:30 +0800
---

在日常使用过程中，无论内网还是外网，是如何访问大模型服务的？

使用VLLM作为高性能推理引擎，配合New API作为统一的管理和分发网关。

```bash
curl http://10.4.65.6:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-qk35xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxPg8M" \
  -d '{
    "model": "deepseek-v4-flash",
    "messages": [
      {"role": "user", "content": "你好，请用一句话介绍你自己"}
    ]
  }'
```
返回结果：
```json
{
  "id": "chatcmpl-849d5706629ecffb",           // 请求唯一ID
  "model": "deepseek-v4-flash",                // 使用的模型
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "你好！我是DeepSeek，一个由深度求索公司开发的AI助手，致力于用热情、细腻的方式为你提供智能、准确且温暖的帮助。😊"
    },
    "finish_reason": "stop"                    // 正常完成
  }],
  "system_fingerprint": "vllm-0.22.0-tp8-10ad8e40",  // 后端是vLLM
  "usage": {
    "prompt_tokens": 11,                       // 输入消耗11个token
    "completion_tokens": 35,                   // 输出消耗35个token  
    "total_tokens": 46                         // 总计46个token
  }
}
```

用curl访问大模型API和访问普通网页的原理完全一样，都是标准的HTTP请求。

<table>
  <thead>
    <tr>
      <th style="text-align: center">对比项</th>
      <th style="text-align: center">访问普通网页</th>
      <th style="text-align: center">访问大模型 API</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: center"><strong>协议</strong></td>
      <td style="text-align: center">HTTP/HTTPS</td>
      <td style="text-align: center">HTTP/HTTPS（完全一样）</td>
    </tr>
    <tr>
      <td style="text-align: center"><strong>请求方法</strong></td>
      <td style="text-align: center">GET（浏览器输入网址）</td>
      <td style="text-align: center">POST（发送数据给模型）</td>
    </tr>
    <tr>
      <td style="text-align: center"><strong>请求头</strong></td>
      <td style="text-align: center"><code>User-Agent</code>、<code>Accept</code> 等</td>
      <td style="text-align: center"><code>Content-Type</code>、<code>Authorization</code> 等</td>
    </tr>
    <tr>
      <td style="text-align: center"><strong>请求体</strong></td>
      <td style="text-align: center">通常为空</td>
      <td style="text-align: center">JSON 格式的提示词和参数</td>
    </tr>
    <tr>
      <td style="text-align: center"><strong>响应</strong></td>
      <td style="text-align: center">HTML 网页</td>
      <td style="text-align: center">JSON 格式的模型回答</td>
    </tr>
    <tr>
      <td style="text-align: center"><strong>网络传输</strong></td>
      <td style="text-align: center">走 TCP/IP</td>
      <td style="text-align: center">走 TCP/IP（完全一样）</td>
    </tr>
  </tbody>
</table>


大模型API服务本质上是一个运行在服务器上的Web服务。唯一的区别是请求体(Payload)，普通网页GET请求没有请求体，而大模型API是POST请求，需要在`-d`参数中发送JSON格式的指令。


不止是内网中的，访问Deepseek这些平台也是一样的。只不过背后并不是用 New API 这类开源网关搭建的，而是它们**自己研发的、企业级的大规模分布式系统**。两者的目标和技术复杂度完全不在一个量级上。

有时候在填写接口的时候，`http://10.4.65.6:3000/v1/chat/completions`
```text
http://10.4.65.6:3000/v1/chat/completions
  └─┬─┘ └──┬──┘ └┬┘ └──────┬──────┘
  协议    IP地址  端口    API路径
```
其中`/v1/chat/completions`是API路径，v1表示API版本，`chat/completions`表示聊天补全接口。

`.bashrc` 是操作系统的环境变量配置（系统层），`settings.json` 是 Claude Code 的配置文件（应用层），而 CC Switch 是通过接管 `settings.json` 来实现统一管理的。

