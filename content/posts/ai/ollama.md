+++
draft = false
date = 2024-10-15T14:22:02+08:00
title = "AI LLM 利器 Ollama 架构和对话处理流程解析"
description = "AI LLM 利器 Ollama 架构和对话处理流程解析"
slug = ""
authors = []
tags = ["AI", "LLM"]
categories = ["AI"]
externalLink = ""
series = []
disableComments = true
+++

## Ollama 概述

`Ollama` 是一个快速运行 `LLM`（Large Language Models，大语言模型）的简便工具。通过 `Ollama`，用户无需复杂的环境配置，即可轻松与大语言模型对话互动。

本文将解析 `Ollama` 的整体架构，并详细讲解用户在与 `Ollama` 进行对话时的具体处理流程。


### Ollama 整体架构

![](https://raw.githubusercontent.com/RifeWang/images/master/ai/ollama-cs.png)

`Ollama` 使用了经典的 CS（Client-Server）架构，其中：
- Client 通过命令行的方式与用户交互。
- Server 可以通过命令行、桌面应用（基于 Electron 框架）、Docker 其中一种方式启动。无论启动方式如何，最终都调用同一个可执行文件。
- Client 与 Server 之间使用 HTTP 进行通信。

`Ollama Server` 有两个核心部分：
- `ollama-http-server`：负责与客户端进行交互。
- `llama.cpp`：作为 LLM 推理引擎，负责加载并运行大语言模型，处理推理请求并返回结果。
- `ollama-http-server` 与 `llama.cpp` 之间也是通过 HTTP 进行交互。

说明：`llama.cpp` 是一个独立的开源项目，具备跨平台和硬件友好性，可以在没有 GPU、甚至是树莓派等设备上运行。


### Ollama 存储结构

`Ollama` 本地存储默认使用的文件夹路径为 `$HOME/.ollama`，文件结构如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/ai/ollama-dir.png)

文件可分为三类：
- 日志文件：包括记录了用户对话输入的 `history` 文件，以及 `logs/server.log` 服务端日志文件。
- 密钥文件：id_ed25519 私钥和 id_ed25519.pub 公钥。
- 模型文件：包括 `blobs` 原始数据文件，以及 `manifests` 元数据文件。

元数据文件，例如图中的 `models/manifests/registry.ollama.ai/library/llama3.2/latest` 文件内容为：

![](https://raw.githubusercontent.com/RifeWang/images/master/ai/ollama-manifest-file.png)

如上图所示，`manifests` 文件是 JSON 格式，文件内容借鉴了云原生和容器领域中的 OCI spec 规范，`manifests` 中的 digest 字段与 `blobs` 相对应。


### Ollama 对话处理流程

用户与 `Ollama` 进行对话的大致流程如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/ai/ollama.drawio.png)

1. 用户通过 CLI 命令行执行 `ollama run llama3.2` 开启对话（`llama3.2` 是一种开源的大语言模型，你也可以使用其它 LLM）。
2. 准备阶段：
    - CLI 客户端向 `ollama-http-server` 发起 HTTP 请求，获取模型信息，后者会尝试读取本地的 `manifests` 元数据文件，如果不存在，则响应 404 not found。
    - 当模型不存在时，CLI 客户端会向 `ollama-http-server` 发起拉取模型的请求，后者会去远程存储仓库下载模型到本地。
    - CLI 再次请求获取模型信息。
3. 交互式对话阶段：
    - CLI 先向 `ollama-http-server` 发起一个空消息的 `/api/generate` 请求，server 会先在内部进行一些 channel（go 语言中的通道）处理。
    - 如果模型信息中包含有 messages，则打印出来。用户可以基于当前使用的模型和 session 对话记录保存为一个新的模型，而对话记录就会被保存为 messages。
    - 正式进入对话：CLI 调用 `/api/chat` 接口请求 `ollama-http-server`，而 `ollama-http-server` 需要依赖 `llama.cpp` 引擎加载模型并执行推理（`llama.cpp` 也是以 HTTP server 的方式提供服务）。此时，`ollama-http-server` 会先向 `llama.cpp` 发起 `/health` 请求，确认后者的健康状况，然后再发起 `/completion` 请求，得到对话响应，并最终返回给 CLI 显示出来。

通过上述步骤，`Ollama` 完成了用户与大语言模型的交互对话。


## 总结

`Ollama` 通过集成 `llama.cpp` 推理引擎，并进一步封装，将复杂的 `LLM` 技术变得触手可及，为开发者和技术人员提供了一个高效且灵活的工具，很好地助力了各种应用场景下的大语言模型推理与交互。


(关注我，无广告，专注技术，不煽动情绪，也欢迎与我交流)

---

参考资料：

- *https://github.com/ollama/ollama*
- *https://github.com/ggerganov/llama.cpp*