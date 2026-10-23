# LlamaPi 本地 Agent 底座搭建实战

## 什么是 LlamaPi？

LlamaPi（读作"拉马派"）是大模型部署与管理工具，面向具备大模型推理能力的边缘设备，帮助用户在本地部署和管理开源大模型，并通过 OpenAI 兼容 API 为第三方应用提供推理能力。

LlamaPi 旨在降低边缘设备上的大模型部署门槛，缩短从模型获取、硬件适配到模型运行的流程，为离线、私有的 AI 应用提供轻量化推理底座。

在边缘算力设备上运行大模型通常涉及硬件识别、推理平台选择、模型格式匹配、运行库配置和服务部署等多个环节。LlamaPi 将这些环节整合为统一的命令行和服务接口，让用户可以用较短的操作流程完成模型下载、运行、管理和应用接入。

LlamaPi 不是单个模型或单一推理引擎，而是一套连接设备、模型、端侧推理平台和上层应用的部署与管理工具。

## 核心能力

- **本地部署开源大模型**：在设备上下载、加载并运行兼容的对话模型和 Embedding 模型。
- **统一模型管理**：查找模型、查看运行状态、调整实例、配置自动加载，以及卸载和删除模型。
- **适配多种推理平台**：根据当前设备识别可用推理平台及兼容模型，支持多芯片协同模型。
- **提供 OpenAI 兼容 API**：当前支持 Chat Completions、Embeddings 和 Models 接口。
- **提供 LlamaPi 扩展接口**：支持模型实例管理和硬件平台查询。
- **图形界面客户端**：提供 Windows 客户端，可在图形界面中完成模型下载、部署和对话。

## 软件组成

LlamaPi 由三个主要组件组成：

| 组件 | 作用 |
| --- | --- |
| `llamapi-cli` | 用于发现、下载、运行和管理模型的命令行工具；命令入口为 `llamapi` |
| `llamapi-server` | 负责检测推理平台、加载模型、管理模型实例并提供 HTTP API 的服务组件 |
| `llamapi-modelstore` | 负责远程模型查找、下载和本地模型文件管理的服务组件 |

`llamapi-cli` 为交互操作和常见任务提供便捷入口，`llamapi-server` 提供持续运行的模型推理服务，`llamapi-modelstore` 负责模型文件的下载与本地管理。三者配合完成从模型获取到应用接入的完整流程。

其中 `llamapi-modelstore` 是模型侧的关键一层：`llamapi list --online` 的远程模型列表由它获取并缓存，`llamapi pull` 的下载、断点续传、本地模型文件的增删也由它负责。这意味着**模型文件的整个生命周期与推理服务解耦** —— 推理服务重启不影响模型库，反过来清理模型也不依赖推理服务在运行。

此外，LlamaPi 提供运行在 DirectAI 平台上的 Windows 客户端，可以在图形界面中完成模型的下载、部署和对话，见下文「客户端的使用」章节。

## 整体架构

架构图如下：

```
  应用层   OpenCode · 其他 OpenAI 兼容应用 · DirectAI 客户端
                     │  OpenAI 兼容 API  :9265
  ───────────────────┼───────────────────────────────────────
  兼容层   llamapi-server            llamapi-modelstore
           推理平台检测 · 实例管理     远程模型查找 · 下载 · 断点续传
           请求队列 · 持久化配置       本地模型文件管理   :9266
  ───────────────────┼───────────────────────────────────────
  硬件层   RK3588（rkllm） · RK1828 ×4（rknn3） · bge-m3（rknn2）
```

| 分层 | 组件 | 说明 |
| --- | --- | --- |
| 应用层 | OpenCode / OpenClaw / Hermes Agent / 其他 OpenAI 兼容应用 | 通过 OpenAI 兼容 API 把推理请求发给 LlamaPi |
| 兼容层 | LlamaPi 服务组（`llamapi-server` + `llamapi-modelstore`） | `llamapi-server` 完成模型管理与服务调度，`llamapi-modelstore` 负责模型的查找、下载与本地文件管理 |
| 硬件层 | RK3588 / RK1828 / RK1820 等 | NPU 硬件加速，按平台加载对应格式的模型 |

上层各类第三方应用通过 OpenAI 兼容 API 将推理请求发送给 LlamaPi；LlamaPi 对内统一完成模型管理、服务调度，把推理任务交付给设备的 NPU 完成加速计算，推理结果再原路返回给上层应用。

## 模型部署

### 软硬件环境

本实例使用的硬件配置如下：

| 部件 | 型号 | 说明 |
| --- | --- | --- |
| 平台 | Firefly RK3588 | 系统调度、编解码、I/O 主控 |
| 加速卡 | RK1828 × 4 | M.2 / PCIe 接入，片上堆叠 DRAM **5 GB**，INT8 峰值 **20 TOPS**，内存带宽 **1 TB/s** |
| 互联 | PCIe 级联 | 4 卡算力无缝拼接，性能近似线性叠加 |
| 功耗 | 单卡 ≈ 10 W | 4 卡满载推理 ≈ 40 W，风冷即可满足散热 |

系统环境是 **Firefly Ubuntu 22.04**。

### 部署流程总览

整个流程只有 7 步：

```text
Step 0  装包        sudo apt install llamapi
Step 1  硬件自检    llamapi platform          ← 确认 4 卡被识别
Step 2  查可用模型  llamapi list --online
Step 3  拉模型      llamapi pull qwen3.8:27b  ← 16.8 GB，支持断点续传
Step 4  部署        llamapi load qwen3.8:27b
Step 5  持久化      llamapi enable ... --now  ← 重启自动加载
Step 6  验证        curl /health && curl /v1/models
```

### 安装 LlamaPi

设备已配置 Firefly APT 软件源时，apt 安装：

```bash
sudo apt install llamapi
```

也可以从 [Firefly 官网下载页](https://community.t-firefly.com/doc/download/428) 获取 deb 安装包，将其复制到设备上，再进入文件所在目录安装：

```bash
sudo dpkg -i ./firefly-llamapi-*.deb
llamapi --version
# ✦ Firefly AI · LlamaPi v0.3.2
```

安装会同时部署三个组件及其 systemd 服务（`llamapi-server.service`、`llamapi-modelstore.service`）。

### 硬件自检

```bash
llamapi platform
```

```text
SoC:                    RK3588
Coprocessor:            RK1828 ×4
Supported platforms:    rknn3, rkllm, rknn2
Unsupported platforms:  -
Service:                not running  (http://127.0.0.1:9265/v1)
```

`Coprocessor` 显示 `RK1828 ×4`，说明 4 张加速卡全部被识别，`rknn3` 平台可用。

字段规则值得留意：同型号芯片数量大于 1 时显示为 `型号 ×数量`；未检测到协处理器时显示 `null`（单 RK3588 板卡的正常输出）；服务未运行时硬件字段显示 `unknown`，因为平台信息本身来自 `GET /v1/platforms`。

### 查可用模型

```bash
llamapi list --online
```

```text
MODEL           TYPE       PLATFORM              SIZE       LOCAL
bge-m3          embedding  rknn2/rk3588          1.1 GB     ● installed
qwen3.5:0.8b    chat       rkllm/rk3588          1.5 GB     ● installed
qwen3.5:9b      chat       rknn3/rk1828          -          ○ not installed
qwen3.5:27b     chat       rknn3/rk1828          -          ○ not installed
qwen3.8:27b     chat       rknn3/rk1828          16.8 GB    ○ not installed
gemma4:12b      chat       rknn3/rk1828          -          ○ not installed
gemma4:31b      chat       rknn3/rk1828          -          ○ not installed
```

`PLATFORM` 列是选型的关键：看到 `rknn3/rk1828` 就知道这个变体是为加速卡准备的，而 `rkllm/rk3588` 是主控 NPU 版本（只到 4B 量级）；`rknn2` 则是 Embedding 模型平台。

补充两点行为：

- 这个远程列表由 `llamapi-modelstore` 获取并缓存，**首次查询需要等待元数据拉取完成，可能耗时数十秒**，属于正常现象。之后走缓存会快很多。
- `llamapi list --all` 不按当前硬件过滤，可用来区分"模型不存在"和"当前硬件不支持"。

### 拉取 27B 模型

```bash
llamapi pull qwen3.8:27b --platform rknn3/rk1828
```

```text
✦ Firefly AI · LlamaPi
✔ Download complete: qwen3.8:27b(rknn3/rk1828)
  Platform  rknn3/rk1828
  Size      16.8 GB
```

拉下来的**不是原始 safetensors，而是已经为 RK1828 适配好的量化变体** —— 这一步替掉了传统流程里最耗时的"模型转换 + 精度调试"，也避免了转换后精度损失。默认存储路径为 `/var/lib/llamapi/models`。

下载由 `llamapi-modelstore` 负责，**支持断点续传** —— 16.8 GB 的模型拉到一半中断，重新执行 `pull` 会从断点继续，不必重头再来。如果某个下载源不可达，可以显式切换：

```bash
llamapi pull qwen3.8:27b --platform rknn3/rk1828 --source modelscope
```

下载源的默认值（`auto`）可在 `/etc/llamapi-modelstore/config.toml` 中改为 `modelscope` 或 `huggingface` 长期固定。

### 部署

```bash
llamapi load qwen3.8:27b
llamapi ps
```

```text
ID                          MODEL        TYPE   PLATFORM        STATUS      INSTANCES   ENABLE
qwen3.8:27b@rknn3-rk1828    qwen3.8:27b  chat   rknn3/rk1828    ● active    1           no
```

分片、加载、平台绑定全部由 `llamapi-server` 自动完成，用户侧不需要接触任何 RKNN3 的部署细节。

### 持久化

```bash
llamapi enable qwen3.8:27b --now
llamapi ps
```

```text
ID                          MODEL        TYPE   PLATFORM        STATUS      INSTANCES   ENABLE
qwen3.8:27b@rknn3-rk1828    qwen3.8:27b  chat   rknn3/rk1828    ● active    1           yes(1)
```

`ENABLE=yes(1)` 意味着设备下次上电会自动加载模型并拉起服务。这条命令实际操作的是 `/etc/llamapi-server/config.toml` 里的 `[[models]]` 条目，建议始终用 `llamapi enable` / `disable` 维护，避免手写配置时把 `model_path` 或运行时 ID 写错。

### 验证服务

```bash
curl -s http://127.0.0.1:9265/health
# ok

curl -s http://127.0.0.1:9265/v1/models
```

```json
{
  "object": "list",
  "data": [
    {
      "id": "qwen3.8:27b@rknn3-rk1828",
      "object": "model",
      "owned_by": "llamapi/rknn3",
      "platform": "rknn3",
      "instance_count": 1,
      "model_kind": "chat"
    }
  ]
}
```

注意 `/health` 返回 `ok` **只说明 HTTP 服务在跑，不代表模型已加载**。要确认模型可用，必须看 `/v1/models` 里是否有目标 ID 且 `instance_count ≥ 1`。

### 交互式对话与多模态附件

命令行也可以直接对话，便于快速验证：

```bash
llamapi run qwen3.5:4b --stats
```

`--stats` 会在回答后输出 token 数、首 token 延迟和生成速度。进入交互模式后，`/system` 设系统提示词、`/set` 调生成参数、`"""` 进入多行输入、`/exit` 退出。

对于具有多模态能力的模型，可以在对话中用 `@` 引入附件（图片、音频），无需额外接口。

### 服务与端口一览

| 服务 | systemd unit | 端口 | 配置文件 | 健康检查 |
| --- | --- | --- | --- | --- |
| 推理服务 | `llamapi-server.service` | `9265` | `/etc/llamapi-server/config.toml` | `curl :9265/health` |
| 模型管理 | `llamapi-modelstore.service` | `9266` | `/etc/llamapi-modelstore/config.toml` | `curl :9266/health` |

```bash
# 查看两个服务的状态与日志
systemctl status llamapi-server
systemctl status llamapi-modelstore
journalctl -u llamapi-server -b
```

## 客户端的使用

除命令行外，LlamaPi 提供 Windows 客户端。客户端可以完成入门教程中的所有操作：下载、加载、运行和部署模型，并在客户端中与模型对话。

### DirectAI 客户端

LlamaPi 客户端应用需要运行在 DirectAI 平台上，DirectAI 安装包从 [Firefly 官网下载页](https://community.t-firefly.com/doc/download/429)获取。参考 [DirectAI 文档](https://community.t-firefly.com/docs/ai/applications/DirectAI/directai/introduction) 安装完成后打开客户端，**DirectAI 会自动扫描局域网内的设备**，点击要使用的设备进行连接。

### 与模型对话

在`新对话`页面选择模型进行对话，LlamaPi 会自动下载和加载模型。模型准备就绪后即开始推理并返回结果；对话过程中可以自定义模型的参数设置。

### 管理本地模型

在 LlamaPi 的`模型库`页面中浏览当前硬件可用的模型，选择需要的模型下载，也可以删除模型。

### 加载和部署模型

下载完成后，在`部署配置`页面中点击`新建部署`，使用已经下载好的模型编辑部署配置 —— 可以自定义模型部署的 ID 和模型实例数，也可以设置模型推理的生成参数。编辑好的部署配置可以按需加载和卸载，也可以配置为设备启动时自动加载。

回到`新对话`页面，即可选择已经加载的模型组进行对话。

## 接入开源编程 Agent

本实例选择 OpenCode 进行接入，对于接入其他开源编程 Agent（Aider、Cline、Hermes Agent 等）是类似的 —— 只要该应用支持自定义 OpenAI Base URL。

### 接入配置

在项目根目录创建 `opencode.json`：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "llamapi/qwen3.8:27b@rknn3-rk1828",
  "provider": {
    "llamapi": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LlamaPi (4x RK1828)",
      "options": {
        "baseURL": "http://127.0.0.1:9265/v1",
        "apiKey": "llamapi-local"
      },
      "models": {
        "qwen3.8:27b@rknn3-rk1828": {
          "name": "Qwen3.8-27B · LlamaPi 4x RK1828",
          "limit": {
            "context": 32768,
            "output": 8192
          }
        }
      }
    }
  }
}
```

其中：

- `options.baseURL` 指向 LlamaPi 的 OpenAI 兼容端点，**跨设备访问时换成设备 IP**（`http://<设备IP>:9265/v1`）。
- 模型名必须是 `llamapi ps` 输出的**完整运行时 ID**（`qwen3.8:27b@rknn3-rk1828`），不能只填 `qwen3.8:27b`。
- `apiKey` 填任意非空值即可 —— LlamaPi **当前不提供 API 鉴权**，这个字段只用于满足客户端的必填校验。
- `limit.context` 决定 Agent 能用多长的上下文。**这个值直接决定 Agent 能处理多大的项目**：32K 够做单模块任务，跨文件重构建议按硬件余量往上调（同时受 `max_context_len` 配置约束）。

### 连通性验证

编程 Agent 的命脉是 function calling。LlamaPi 的 chat 接口支持 OpenAI 风格的 `tools` 定义，并且 `finish_reason` 会返回 `tool_calls`：

```bash
curl -s http://127.0.0.1:9265/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3.8:27b@rknn3-rk1828",
    "messages": [
      {"role": "user", "content": "读取 /tmp/hello.py 的内容"}
    ],
    "tools": [{
      "type": "function",
      "function": {
        "name": "read_file",
        "description": "读取指定路径的文本文件",
        "parameters": {
          "type": "object",
          "properties": {"path": {"type": "string"}},
          "required": ["path"]
        }
      }
    }],
    "stream": false
  }'
```

```json
{
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "",
      "tool_calls": [{
        "id": "call_0",
        "type": "function",
        "function": {
          "name": "read_file",
          "arguments": "{\"path\": \"/tmp/hello.py\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

`finish_reason: "tool_calls"` + 结构完整的 `tool_calls`，意味着 **agentic loop 的返回通道是通的**：Agent 发工具定义 → 模型生成调用 → Agent 执行 → 用 `role: "tool"` 回灌结果 → 继续下一轮。这是编程 Agent 的工作循环。

一个小细节：`tool_choice` 字段会被服务端解析但当前不生效。OpenCode 这类框架不依赖它做硬性约束，所以不影响使用；但如果是自研客户端想做强制工具调用，需要靠 prompt 而非 `tool_choice` 来约束。

## 参考资料

- [https://community.t-firefly.com/docs/ai/applications/LlamaPi/llamapi/introduction](https://community.t-firefly.com/docs/ai/applications/LlamaPi/llamapi/introduction)
- [https://huggingface.co/Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)
- [https://opencode.ai/v2/docs](https://opencode.ai/v2/docs)
