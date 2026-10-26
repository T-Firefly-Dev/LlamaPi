# Building a Local Agent Base with LlamaPi

> **Verified version: v0.3.2**.
> 
> For the latest version and issue feedback, ask in the [LlamaPi category on the Firefly forum](https://forum.t-firefly.com/c/eco-software/llamapi).

## What Is LlamaPi?

LlamaPi is a large-model deployment and management tool. It is designed for edge devices with large-model inference capabilities, helping users deploy and manage open-source models locally and provide inference to third-party applications through OpenAI-compatible APIs.

LlamaPi lowers the barrier to deploying large models on edge devices. It shortens the workflow from obtaining a model and adapting it to the hardware to running the model, providing a lightweight foundation for offline and private AI applications.

Running a large model on an edge-computing device typically involves hardware detection, inference-platform selection, model-format matching, runtime configuration, and service deployment. LlamaPi brings these steps together behind a consistent command-line interface and service API, so users can download, run, manage, and integrate models with fewer manual steps.

LlamaPi is not a model or a single inference engine. It is a deployment and management tool that connects devices, models, on-device inference platforms, and applications.

## Core Capabilities

- **Deploy open-source models locally**: download, load, and run compatible chat and Embedding models on the device.
- **Manage models consistently**: discover models, inspect runtime state, resize instances, configure automatic loading, unload models, and remove local files.
- **Support multiple inference platforms**: detect the inference platforms available on the current device and find compatible models, including models that span multiple chips.
- **Provide OpenAI-compatible APIs**: currently supports the Chat Completions, Embeddings, and Models APIs.
- **Provide LlamaPi extensions**: manage model instances and query hardware platforms through additional endpoints.
- **Graphical client**: a Windows client that can download, deploy, and chat with models through a graphical interface.

## Components

LlamaPi has three main components:

| Component | Purpose |
| --- | --- |
| `llamapi-cli` | Command-line component for discovering, downloading, running, and managing models; invoked with the `llamapi` command |
| `llamapi-server` | Service component that detects inference platforms, loads models, manages model instances, and provides HTTP APIs |
| `llamapi-modelstore` | Service component responsible for remote model discovery, downloads, and local model file management |

`llamapi-cli` provides convenient entry points for interactive operations and common tasks, `llamapi-server` hosts the persistent model inference service, and `llamapi-modelstore` handles model downloads and local file management. Together they cover the path from obtaining a model to connecting an application.

`llamapi-modelstore` is the key layer on the model side: the remote model list shown by `llamapi list --online` is fetched and cached by it, and the downloads, resumable transfers, and add/remove operations on local model files performed by `llamapi pull` are its responsibility as well. This means **the entire lifecycle of model files is decoupled from the inference service** — restarting the inference service does not affect the model library, and cleaning up models does not require the inference service to be running.

In addition, LlamaPi provides a Windows client that runs on the DirectAI platform and can download and deploy models and chat with them through a graphical interface. See [Using the Client](#using-the-client) below.

## Overall Architecture

The architecture is shown below:

```text
  Application    OpenCode · other OpenAI-compatible apps · DirectAI client
                       │   OpenAI-compatible API   :9265
  ─────────────────────┼───────────────────────────────────────────────────
  Compatibility    llamapi-server                  llamapi-modelstore
                   platform detection ·            remote model discovery ·
                   instance management ·           download · resume ·
                   request queue ·                 local model file management
                   persistent configuration                                 :9266
  ─────────────────────┼───────────────────────────────────────────────────
  Hardware         RK3588 (rkllm) · RK1828 ×4 (rknn3) · bge-m3 (rknn2)
```

| Layer | Component | Description |
| --- | --- | --- |
| Application | OpenCode / OpenClaw / Hermes Agent / other OpenAI-compatible applications | Send inference requests to LlamaPi through the OpenAI-compatible API |
| Compatibility | LlamaPi service group (`llamapi-server` + `llamapi-modelstore`) | `llamapi-server` handles model management and service scheduling, while `llamapi-modelstore` handles model discovery, download, and local file management |
| Hardware | RK3588 / RK1828 / RK1820 and others | NPU hardware acceleration; loads the model format matching each platform |

Third-party applications send inference requests to LlamaPi through the OpenAI-compatible API. Internally, LlamaPi handles model management and service scheduling in a unified way, hands the inference task to the device's NPU for accelerated computation, and returns the result to the application along the same path.

## Model Deployment

### Hardware and Software Environment

The hardware configuration used in this example is as follows:

| Item | Model | Description |
| --- | --- | --- |
| Board | Firefly RK3588 | System scheduling, codec, and I/O host |
| Accelerator | RK1828 × 4 | M.2 / PCIe; on-package stacked DRAM **5 GB**, INT8 peak **20 TOPS**, memory bandwidth **1 TB/s** |
| Interconnect | PCIe cascade | Four cards joined into a single compute pool; performance scales near-linearly |
| Power | ≈ 10 W per card | ≈ 40 W at full load on four cards; air cooling is sufficient |

The system environment is **Firefly Ubuntu 22.04**.

### Deployment Workflow

The whole process takes only 7 steps:

```text
Step 0  Install       sudo apt install llamapi
Step 1  Self-check    llamapi platform          ← confirm all 4 cards are detected
Step 2  List models   llamapi list --online
Step 3  Pull model    llamapi pull qwen3.8:27b  ← 16.8 GB, resumable
Step 4  Deploy        llamapi load qwen3.8:27b
Step 5  Persist       llamapi enable ... --now  ← auto-load on reboot
Step 6  Verify        curl /health && curl /v1/models
```

### Install LlamaPi

If the Firefly APT repository is configured on the device, install LlamaPi with apt:

```bash
sudo apt install llamapi
```

You can also download the deb package from the [Firefly website](https://community.t-firefly.com/doc/download/428), copy it to the device, open its directory, and install it locally:

```bash
sudo dpkg -i ./firefly-llamapi-*.deb
llamapi --version
# ✦ Firefly AI · LlamaPi v0.3.2
```

Installation deploys all three components together with their systemd services (`llamapi-server.service` and `llamapi-modelstore.service`).

### Hardware Self-Check

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

`Coprocessor` showing `RK1828 ×4` means all four accelerator cards are detected and the `rknn3` platform is available.

The field rules are worth noting: when more than one chip of the same model is present, the value is displayed as `<model> ×<count>`; when no coprocessor is detected it shows `null`, which is the normal output on a single RK3588 board; and when the service is not running, the hardware fields show `unknown`, because the platform information itself comes from `GET /v1/platforms`.

### List Available Models

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

`PLATFORM` is the key column for selection: `rknn3/rk1828` indicates that the variant is built for the accelerator cards, `rkllm/rk3588` is the host-NPU variant (which only reaches the 4B class), and `rknn2` is the Embedding platform.

Two additional behaviors are worth knowing:

- This remote list is fetched and cached by `llamapi-modelstore`. **The first query has to wait for metadata retrieval and may take tens of seconds**, which is normal. Later queries hit the cache and are much faster.
- `llamapi list --all` does not filter by the current hardware, which helps distinguish "the model does not exist" from "the current hardware does not support it".

### Pull the 27B Model

```bash
llamapi pull qwen3.8:27b --platform rknn3/rk1828
```

```text
✦ Firefly AI · LlamaPi
✔ Download complete: qwen3.8:27b(rknn3/rk1828)
  Platform  rknn3/rk1828
  Size      16.8 GB
```

What you get is **not the original safetensors, but a quantized variant already adapted for the RK1828** — this step replaces the most time-consuming part of the traditional workflow, "model conversion + accuracy tuning", and avoids the accuracy loss that conversion can introduce. The default storage path is `/var/lib/llamapi/models`.

Downloads are handled by `llamapi-modelstore` and **support resumable transfer** — if a 16.8 GB download is interrupted halfway, re-running `pull` continues from the breakpoint instead of starting over. If a download source is unreachable, switch it explicitly:

```bash
llamapi pull qwen3.8:27b --platform rknn3/rk1828 --source modelscope
```

The default source (`auto`) can be pinned to `modelscope` or `huggingface` in `/etc/llamapi-modelstore/config.toml`.

### Deploy

```bash
llamapi load qwen3.8:27b
llamapi ps
```

```text
ID                          MODEL        TYPE   PLATFORM        STATUS      INSTANCES   ENABLE
qwen3.8:27b@rknn3-rk1828    qwen3.8:27b  chat   rknn3/rk1828    ● active    1           no
```

Sharding, loading, and platform binding are all handled automatically by `llamapi-server`; users never need to touch any RKNN3 deployment detail.

### Persist the Deployment

```bash
llamapi enable qwen3.8:27b --now
llamapi ps
```

```text
ID                          MODEL        TYPE   PLATFORM        STATUS      INSTANCES   ENABLE
qwen3.8:27b@rknn3-rk1828    qwen3.8:27b  chat   rknn3/rk1828    ● active    1           yes(1)
```

`ENABLE=yes(1)` means the model is loaded automatically the next time the device powers on and the service starts. This command operates on the `[[models]]` entry in `/etc/llamapi-server/config.toml`; always maintain it through `llamapi enable` / `disable` to avoid mistyping `model_path` or the runtime ID when editing the configuration by hand.

### Verify the Service

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

Note that `/health` returning `ok` **only means the HTTP service is running; it does not mean a model has been loaded**. To confirm that a model is usable, check that `/v1/models` contains the target ID with `instance_count ≥ 1`.

### Interactive Chat and Multimodal Attachments

The command line can also chat directly, which is convenient for a quick check:

```bash
llamapi run qwen3.5:4b --stats
```

`--stats` prints the token count, time to first token, and generation speed after the answer. In interactive mode, `/system` sets the system prompt, `/set` adjusts generation parameters, `"""` starts multiline input, and `/exit` quits.

For models with multimodal capabilities, use `@` to attach files (images, audio) in the conversation — no separate endpoint is needed.

### Services and Ports

| Service | systemd unit | Port | Configuration file | Health check |
| --- | --- | --- | --- | --- |
| Inference service | `llamapi-server.service` | `9265` | `/etc/llamapi-server/config.toml` | `curl :9265/health` |
| Model management | `llamapi-modelstore.service` | `9266` | `/etc/llamapi-modelstore/config.toml` | `curl :9266/health` |

```bash
# Check the status and logs of both services
systemctl status llamapi-server
systemctl status llamapi-modelstore
journalctl -u llamapi-server -b
```

## Using the Client

Besides the command line, LlamaPi provides a Windows client. The client can perform all operations covered in this guide: downloading, loading, running, and deploying models, and chatting with them in the client.

### DirectAI Client

The LlamaPi client app runs on the DirectAI platform. Download the DirectAI installer from the [Firefly website](https://community.t-firefly.com/doc/download/429), install it by following the [DirectAI documentation](https://community.t-firefly.com/docs/ai/applications/DirectAI/directai/introduction), and then open the client. **DirectAI automatically scans the LAN for devices**; click the device you want to use to connect.

### Chat with a Model

On the `New Chat` page, select a model to chat with. LlamaPi downloads and loads the model automatically. Once the model is ready, inference runs and the result is returned; the model's parameter settings can be customized during the conversation.

### Manage Local Models

On the LlamaPi `Model Library` page, browse the models available on the current hardware, download the ones you need, and delete the ones you no longer need.

### Load and Deploy Models

After a model is downloaded, open the `Deployment Config` page and click `New Deployment` to create a deployment configuration from a downloaded model — you can customize the deployment ID and the number of model instances, and configure the model's generation parameters. A saved deployment configuration can be loaded and unloaded on demand, or configured to load automatically when the device starts.

Return to the `New Chat` page and select a loaded model group to chat with.

## Connect an Open-Source Coding Agent

This example uses OpenCode. Connecting other open-source coding agents (Aider, Cline, Hermes Agent, and others) is similar — any application that supports a custom OpenAI Base URL will do.

### Configuration

Create `opencode.json` in the project root:

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

Where:

- `options.baseURL` points to the LlamaPi OpenAI-compatible endpoint. **For access from another device, replace it with the device IP** (`http://<device-ip>:9265/v1`).
- The model name must be the **complete runtime ID** printed by `llamapi ps` (`qwen3.8:27b@rknn3-rk1828`); entering only `qwen3.8:27b` will not work.
- `apiKey` can be any non-empty value — LlamaPi **currently does not provide API authentication**; the field only satisfies the client's required-field validation.
- `limit.context` determines how much context the agent can use. **This value directly determines how large a project the agent can handle**: 32K is enough for single-module tasks, and cross-file refactoring should raise it according to the hardware headroom (it is also bounded by the `max_context_len` configuration).

### Connectivity Check

Function calling is the lifeline of a coding agent. LlamaPi's chat API supports OpenAI-style `tools` definitions, and `finish_reason` returns `tool_calls`:

```bash
curl -s http://127.0.0.1:9265/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3.8:27b@rknn3-rk1828",
    "messages": [
      {"role": "user", "content": "Read the contents of /tmp/hello.py"}
    ],
    "tools": [{
      "type": "function",
      "function": {
        "name": "read_file",
        "description": "Read a text file at the given path",
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

`finish_reason: "tool_calls"` together with a well-formed `tool_calls` array means **the return channel of the agentic loop works**: the agent sends tool definitions → the model generates a call → the agent executes it → the result is fed back with `role: "tool"` → the next round begins. This is the working loop of a coding agent.

One small detail: the `tool_choice` field is parsed by the server but is not currently effective. Frameworks such as OpenCode do not rely on it for hard constraints, so this does not affect usage; but a custom client that wants to force a tool call must constrain it through the prompt rather than through `tool_choice`.

## Forum and Feedback

If you hit an issue during deployment, or have a request about model support or features, post in the [LlamaPi category on the Firefly forum](https://forum.t-firefly.com/c/eco-software/llamapi).

## References

- [https://community.t-firefly.com/en/docs/ai/applications/LlamaPi/llamapi/introduction](https://community.t-firefly.com/en/docs/ai/applications/LlamaPi/llamapi/introduction)
- [https://huggingface.co/Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)
- [https://opencode.ai/v2/docs](https://opencode.ai/v2/docs)
