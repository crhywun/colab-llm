# colab-llm

用 Google Colab 的 GPU 跑大模型，通过 Cloudflare 隧道得到一个固定的 OpenAI 兼容地址，在本地任意客户端里调用。

```
浏览器里的 Colab GPU [推理服务 ◀─ 保活代理 ◀─ cloudflared] ──隧道──▶ Cloudflare (llm.example.com) ◀── 本地客户端
```

## 选哪个 notebook

| Notebook | 模型 | 推理引擎 | 运行时 | 上下文 | 速度（实测） | 并发 |
| --- | --- | --- | --- | --- | --- | --- |
| `02_serve.ipynb` | Qwen3.8-27B | llama.cpp | A100 / H100 / G4 | 256K | — | 1 |
| `03_serve_flash_next.ipynb` | [Qwen3.8-Flash-Next 无审查 IQ3_S](https://huggingface.co/SC117/Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF)（180B MoE） | llama.cpp | **G4** | 1M | 单人约 100 tok/s；4 人同时约 57 tok/s/人 | 4（可调） |
| `04_serve_glm_strata.ipynb` | [GLM-5.3-Flash Maya-S-v2](https://huggingface.co/peasantsmith/GLM-5.3-Flash-Maya-GGUF) + [去审查 LoRA](https://huggingface.co/GCSA-AiLab/GLM-5.3-Flash-Abliterated-LoRA-GGUF)（320B MoE） | [Strata-GLM](https://github.com/Indras-Mirror/Strata-GLM) | **G4** | 512K | 单人约 45 tok/s | **1** |

- 多人同时用选 03；只有自己用、想试更大的模型可以选 04（每个问题都会先思考很久）。
- 02 需要先用 `01_download_to_drive.ipynb`（CPU 运行时即可）把模型下载到 Drive；03、04 每次直接从 Hugging Face 下载到机器上（几分钟）。
- G4 = RTX PRO 6000 Blackwell（96GB 显存、176GB 内存），需要 Colab Pro / Pro+。

## 安装（一次性）

1. **Cloudflare 固定域名**（可选，需要一个托管在 Cloudflare 的域名）。在 Mac 上运行：
   ```bash
   brew install cloudflared && cloudflared tunnel login
   ./scripts/setup_tunnel.sh llm.example.com
   ```
   它会创建隧道，把 `secrets.json`（隧道 token、域名、API key）写到 Google Drive 的 `My Drive/models/`，并在本地生成 `.env`。
   没装 Drive 桌面版：给脚本传一个本地目录作为第二个参数，再手动把 `secrets.json` 上传到 Drive 的 `models/` 文件夹。
   **没有域名就跳过这一步**：notebook 会改用临时的 `trycloudflare.com` 地址，并打印要填的 key。
2. 把要用的 notebook 上传到 Google Drive（或在 Colab 里「文件 → 上传笔记本」）。

## 每次使用

1. 在 Colab 打开 notebook，「代码执行程序 → 更改运行时类型」选 **G4**（02 也可以选 A100）。
2. 从上到下依次运行每个 cell。第 1 步会要求授权 Drive。
   - 第一次运行会编译推理引擎（03 约 10 分钟，04 约 25 分钟），编好存到 Drive 的 `models/bin/`，以后直接复用。
3. 第 5 步打印出地址后，在客户端里填：
   - Base URL：`https://llm.example.com/v1`
   - API Key：`.env` 里的 `COLAB_LLM_KEY`
   - 模型名：第 5 步打印的 `COLAB_LLM_MODEL`（03 是 `qwen3.8-flash-next`，04 是 `glm-5.3-flash`）
4. 最后运行监控 cell 并让它一直跑着，否则 Colab 会把空闲的运行时收回。
5. 用完关机：先停掉监控 cell，再运行看日志那个 cell 里的 `runtime.unassign()`，否则会继续消耗计算单元。

命令行对话：`python3 chat.py`（读取 `.env`）。其他客户端（Magpie、Cherry Studio、Chatbox、Open WebUI 等）选「OpenAI 兼容」。

## 别人使用时需要改什么

notebook 里没有写死任何个人信息，域名、隧道 token 和 API key 都从 Drive 的 `secrets.json`（或 Colab 左侧钥匙图标的 Secrets）读取。所以通常**什么都不用改**，只需要：

| 要做的事 | 说明 |
| --- | --- |
| 准备自己的 `secrets.json` | 运行 `scripts/setup_tunnel.sh 你的子域名`；或者不配置，用临时地址 |
| 或者用 Colab Secrets | 在 Colab 左侧钥匙图标里添加 `CF_TUNNEL_TOKEN`、`CF_HOSTNAME`、`LLM_API_KEY`，优先级高于 `secrets.json` |
| 运行时选 G4 | 03、04 需要 96GB 显存；没有 G4 只能用 02 |
| （可选）调参数 | 03 第 3 步：`LONG_CONTEXT`（1M / 256K）、`PARALLEL`（并发数）；04 第 3 步：`CTX`、`SLOTS` |

Drive 里用到的路径（不存在会自动创建）：`My Drive/models/secrets.json`、`My Drive/models/bin/`（编译结果缓存）。

## 注意

- 03、04 用的是去除了拒答的社区模型，输出不经过安全过滤，使用时自行遵守法律和模型许可。
- 长请求请用流式输出：非流式请求在约 100–140 秒时可能被 Cloudflare 断开（保活代理已尽量避免）。
- Colab 运行时最长约 24 小时，也可能因为 GPU 被抢占而提前收回，需要重新运行 notebook。

## 安全

- `.env` 和 `secrets.json` 已在 `.gitignore` 中，不要提交。
- `secrets.json` 里的隧道 token 能控制你的子域名指向哪里，不要分享存放它的 Drive 文件夹。泄露后用 `cloudflared tunnel delete colab-llm` 删除重建。
