# ComfyUI MOSS Prompt Enhancer

**English** | [简体中文](README_ZH.md)

Generate MOSS-Image prompts inside ComfyUI with one node that can use either a local GGUF model or a third-party OpenAI-compatible LLM API for prompt expansion. The node adapts the [OpenMOSS prompt-enhancer skill](https://github.com/OpenMOSS/MOSS-Image/tree/main/skills/moss-image-prompt-enhancer), preserve requested Chinese text, and compile spatial layouts with the upstream compiler.

It generates prompts, not images. It can work from text alone or accept a connected image and reverse-engineer that image into a reusable MOSS prompt. With `backend=local` it uses a vision GGUF and `mmproj`; with `backend=api` it uses a vision-capable API model. Connect its output to your image-generation workflow.

## Features

- **Caption:** descriptive prose for photographs, illustrations, characters, and scenes.
- **Layout:** a structured composition with text and object placement, compiled to MOSS coordinates.
- **Auto:** lets the model choose Caption or Layout.
- **Prompt-reverse task:** set `task=prompt_reverse` and connect an `IMAGE` to reverse-engineer it into a high-detail prompt using the MOSS skill rules. By default it covers art direction and rendering, every character's features, wardrobe, materials, setting, lighting, palette, and composition. `request` may be empty or add optional emphasis.
- **Task modes:** `auto` reverses when an image is connected and enhances otherwise; `prompt_enhance` is text expansion or revision; `prompt_reverse` explicitly performs image-to-prompt reverse-engineering.
- **Automatic vision projector discovery:** image mode uses `mmproj-F16.gguf` from the selected model's directory, or an explicit `mmproj_name`.
- English or Chinese descriptions while preserving the original language of visible text.
- Edit an existing prompt through `previous_prompt`; with an image, use it to correct an older draft.
- Recursive GGUF discovery under `ComfyUI/models/LLM`, excluding `mmproj` files from the main-model list.
- In-node output preview and three `STRING` outputs: `prompt`, `draft_json`, and `info`.
- Schema-guided generation, validation with one repair attempt for invalid drafts, and process cleanup on completion, error, or cancellation for the local backend.
- **One node, two backends:** `backend=local` uses llama.cpp; `backend=api` calls a third-party OpenAI-compatible LLM. Both share the same MOSS skill, schema validation, and one-pass repair flow.
- Local inference stays on loopback with no automatic model download. The API backend uses only the Python standard library.

## Requirements

- An existing ComfyUI installation with Python 3.10 or newer.
- **Local backend:** a GGUF instruction model with a chat template, a compatible **llama-server**, and enough RAM/VRAM for the selected model and context size.
- **Local image mode:** a vision-capable main model and a compatible `mmproj` projector.
- **API backend:** an OpenAI-compatible endpoint and model ID. No GGUF weights or llama-server are required. The endpoint must support Chat Completions; image mode also requires a vision-capable model.

Tested with Windows portable ComfyUI, Python 3.13, llama.cpp **b10436**, and `Qwen3.8-27B-UD-Q4_K_M.gguf` for the text flow. Image message construction, projector arguments, and image encoding are covered by offline tests; real image inference speed and accuracy depend on the selected vision model, VRAM, and runtime build. Thinking is disabled by this adapter.

Model weights and llama.cpp binaries are **not included**.

## Installation

### ComfyUI Manager

Search for **MOSS Prompt Enhancer Local** in ComfyUI Manager, or install Registry ID `comfyui-moss-prompt-enhancer-local`.

[Comfy Registry page](https://registry.comfy.org/nodes/comfyui-moss-prompt-enhancer-local)

### 1. Install the node

From your ComfyUI directory:

```bash
git clone https://github.com/slmonker/ComfyUI-MOSS-PromptEnhancer.git custom_nodes/ComfyUI-MOSS-PromptEnhancer
```

Alternatively, download the repository ZIP and extract it into `ComfyUI/custom_nodes/ComfyUI-MOSS-PromptEnhancer`.

### 2. Add models

Text mode needs only the main GGUF. Image reverse-engineering needs the compatible visual projector in the same directory as the main model; `auto` prefers `mmproj-F16.gguf` there:

```text
ComfyUI/
  models/
    LLM/
      Qwen3.8/
        Qwen3.8-27B-UD-Q4_K_M.gguf
        mmproj-F16.gguf
  custom_nodes/
    ComfyUI-MOSS-PromptEnhancer/
```

Main models may live in subdirectories beneath `models/LLM`. Projectors appear in a separate `mmproj_name` selector and are never mixed into the main-model list. All model paths must remain within `models/LLM`; paths escaping that directory are rejected.

### 3. Configure llama-server

Choose one method:

- If the sibling `ComfyUI_Qwen_H3_Prompt` node already has a working `runtime_config.json`, leave `llama_server_path` empty to reuse its executable.
- Otherwise, obtain a suitable runtime from [llama.cpp releases](https://github.com/ggml-org/llama.cpp/releases) and enter the full local executable path in `llama_server_path`. Keep the accompanying libraries next to the executable. Example: `C:/tools/llama.cpp/llama-server.exe`.

With `backend=local`, the node starts its own server process; you do not need to launch a separate server or enter an API URL. With `backend=api`, it connects to `api_base_url` instead. The node does not modify the sibling plugin.

### 4. Restart ComfyUI

Search under **MOSS-Image / Prompt** for **MOSS Prompt Enhancer**. A Python change requires restarting the ComfyUI service, not only refreshing the browser. If you use `backend=api`, skip the GGUF model and llama-server steps.

## Quick start

### Text to prompt

1. Add the node or load an [Caption workflow](example_workflows/MOSS-Prompt-caption.json).
2. Select an installed main model.
3. Enter a scene request, select `caption`, and set `language` to `English` or `中文`.
4. Run the workflow. The node can run independently because it is an output node.
5. Connect `prompt` to the text input of your text-encoding node.

Example request:

```text
清晨的早点摊，招牌清楚写着“老街早餐”。自然纪实摄影，英文描述，中文招牌原样保留。
```

### Image to prompt

1. Add **Load Image** and connect its `IMAGE` output to the node's `image` input, or load the [image-reverse workflow](example_workflows/MOSS-Prompt-image-reverse.json).
2. Set `task` to `prompt_reverse`. With `auto`, connecting an image selects this task automatically.
3. Select a vision-capable main model. Keep `mmproj_name=auto`, or choose a compatible projector explicitly.
4. Leave `request` empty, or use it for optional guidance such as "focus on clothing materials", "preserve the sign text", or "write the result as Chinese photographic prose".
5. Run with `caption` or `auto`. Use `layout` when text/object placement matters, and keep `width` and `height` as multiples of 16.
6. The node uses its prebuilt reverse-task system prompt, analyzes the image, and emits a prompt ready for MOSS-Image.

For an image batch, the first frame is used. The image is resized and encoded in memory as a JPEG data URL with a 1536-pixel maximum edge; nothing is written to disk and the original file is not modified.

## Third-party LLM API for prompt expansion

Set `backend=api` when you want a third-party LLM to expand or revise the prompt instead of running a local GGUF model. For text-only prompt expansion, set `task=prompt_enhance`; no image, GGUF file, or llama-server is required. The node sends `POST {api_base_url}/chat/completions` using the OpenAI Chat Completions format. `api_base_url` normally ends in `/v1`; a complete URL ending in `/chat/completions` is also accepted.

| Provider | Example `api_base_url` | Notes |
| --- | --- | --- |
| OpenAI | `https://api.openai.com/v1` | Set `api_model` to an available chat model. |
| OpenRouter | `https://openrouter.ai/api/v1` | Use the provider's full model ID. |
| DeepSeek | `https://api.deepseek.com` or `https://api.deepseek.com/v1` | Use a chat model such as `deepseek-chat`. |
| SiliconFlow | `https://api.siliconflow.cn/v1` | Use the model ID shown in its console. |
| Ollama | `http://127.0.0.1:11434/v1` | No llama-server is needed; the key may stay empty. |
| LM Studio | `http://127.0.0.1:1234/v1` | Start its local server first; the key may stay empty. |

Example third-party text expansion: set `backend=api`, `api_base_url=https://api.deepseek.com/v1`, `api_model=deepseek-chat`, `task=prompt_enhance`, and connect the `prompt` output to your text encoder. For SiliconFlow, set `api_base_url=https://api.siliconflow.cn/v1` and use the model ID shown in its console.

Recommended authentication: set an environment variable before launching ComfyUI, then keep `api_key` empty and set `api_key_env` to that variable name.

```powershell
# Windows PowerShell
$env:MOSS_API_KEY = "your-key"
```

```bash
# Linux / macOS
export MOSS_API_KEY="your-key"
```

`api_key` is also available for quick testing, but its value is stored in the workflow JSON and may be exposed when the workflow is shared. Prefer the environment variable.

Compatibility switches:

- `api_json_mode=true` sends `response_format={"type":"json_object"}`. Turn it off if a provider rejects this field; the existing JSON validation and repair still run.
- `api_max_tokens_field=max_tokens` is the common compatible setting. Newer OpenAI reasoning models may require `max_completion_tokens`.
- `api_send_seed=true` sends `seed`. Turn it off for providers that reject unknown parameters.
- Image mode sends the resized image as a data URL. Connect an image only to a model that supports vision input.
- API errors include the HTTP status and response body. The key is never included in logs or `info`.

## Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `request` | Example scene | Scene, composition, visible wording, or revision instructions. Optional guidance when an image is connected. |
| `model_name` | Detected GGUF | Main model path relative to `models/LLM`. |
| `output_mode` | `caption` | `caption`, `layout`, or `auto`. |
| `language` | `English` | Language of descriptive prose; requested visible text stays in its original language. |
| `width`, `height` | `1024`, `1024` | Framing context for Caption; actual canvas dimensions for Layout. |
| `seed` | `42` | Sampling seed. |
| `max_tokens` | `4096` | Output token limit, not a target length. |
| `context_size` | `16384` | Space for skill instructions, request, previous prompt, and output. |
| `temperature` | `0.6` | Sampling temperature. |
| `device` | `auto` | GPU placement with llama.cpp auto-fit, or `cpu`. |
| `timeout_seconds` | `600` | Timeout for model startup and for each generation request. |
| `task` | `auto` | `prompt_reverse` for image-to-prompt; `prompt_enhance` for text expansion; `auto` selects reverse with an image, enhance without one. |
| `image` | Not connected | Optional `IMAGE` input for reverse-engineering. |
| `previous_prompt` | Empty | Existing prompt to revise; with an image, used to correct a draft against the reference. |
| `llama_server_path` | Empty | Optional executable path; otherwise use sibling runtime discovery. |
| `mmproj_name` | `auto` | Vision projector for image mode. `auto` prefers `mmproj-F16.gguf` beside the main model. |
| `backend` | `local` | `local` for GGUF/llama.cpp, `api` for a third-party OpenAI-compatible LLM. |
| `api_base_url` | `https://api.openai.com/v1` | API backend base URL, usually ending in `/v1`. |
| `api_model` | Empty | API model ID, such as `deepseek-chat` or a provider-specific model name. |
| `api_key` | Empty | Direct API key; stored in workflow JSON, so environment variables are preferred. |
| `api_key_env` | `MOSS_API_KEY` | Environment variable read when `api_key` is empty. |
| `api_json_mode` | `true` | Send `response_format=json_object`; disable if the provider rejects it. |
| `api_max_tokens_field` | `max_tokens` | Use `max_completion_tokens` for providers/models that require it. |
| `api_send_seed` | `true` | Send `seed`; disable for providers that reject it. |

With `backend=api`, `model_name`, `context_size`, `device`, `llama_server_path`, and `mmproj_name` are ignored. All prompt, canvas, task, image, and previous-prompt inputs remain shared.

**Dimension rules:** Caption accepts positive dimensions even if they are not multiples of 16; it does not round them. Layout and Auto require positive multiples of 16. Use the same dimensions in the image-generation workflow. If an upstream resolution selector uses `multiple=8`, switch it to `16` for Layout/Auto.

## Outputs

| Output | Content |
| --- | --- |
| `prompt` | Plain text in Caption mode, or one-line compiled Layout JSON. Send this to the image model. |
| `draft_json` | Editable source draft. Layout boxes are absolute pixel `xyxy` coordinates. |
| `info` | Backend, actual output mode, input type, model, endpoint or projector, image dimensions, elapsed time, token usage, and repair count. |

Layout compilation converts boxes to integer `abs_centralized` / `yxyx` coordinates and removes canvas metadata from the final prompt. The original draft remains available separately.

## Getting more detail

The normal enhancement system message is assembled in [`prompting.py`](prompting.py) from the bundled skill, the Layout reference when applicable, and node-specific output rules.

The prompt-reverse task has its own prebuilt constant, `PROMPT_REVERSE_SYSTEM_PROMPT`, with an `exhaustive` default detail level. It encodes the MOSS skill rules plus a systematic observation checklist: art direction and medium; rendering, line work, brushwork, and material response; each character's face, hair, body, expression, pose, hands, and clothing layers/fabrics; props; spatial relationships; camera and composition; lighting, palette, and atmosphere; and visible text. It describes only visible evidence, does not invent identities or facts, does not guess illegible text, applies Caption/Layout constraints, and forbids meta-language.

For complex multi-character images, raise `max_tokens` to `6144` or `8192` so the detailed description has enough output room.

In text mode, ask for concrete visible details in `request`: appearance, clothing materials, pose, hands, expression, foreground/background relationships, camera position, lighting, color, and exact signage. In image mode, treat `request` as a focus checklist rather than a replacement description. Keep existing details in `previous_prompt` when editing.

## Memory and cancellation

With `backend=local` and `device=auto`, the node asks ComfyUI to unload its cached image models before loading the LLM. It launches a local server on `127.0.0.1` with a temporary access key and closes that owned process after each run, including errors and cancellation. Models are not retained between executions. With `backend=api`, the node does not load or unload local GGUF models.

Image reverse-engineering loads both the main model and the `mmproj`, so it needs more VRAM than text-only generation. CPU mode does not clear ComfyUI's model cache and can be slow with large models. The node does not manage servers started by other plugins.

## Troubleshooting

- **No models listed:** put a supported `.gguf` file under `models/LLM`, then refresh node definitions or restart ComfyUI.
- **Image mode says mmproj is missing:** put the matching `mmproj-F16.gguf` beside the main model, or select a compatible projector in `mmproj_name`.
- **Image mode fails to load:** confirm the main model is vision-capable and that the projector matches the model and llama.cpp build.
- **llama-server not found:** set `llama_server_path`, or install the sibling node's runtime first.
- **Output truncated:** increase `max_tokens`; increase `context_size` as needed to leave room for input and output.
- **Invalid Layout:** the node attempts one repair. Simplify the layout or use a more capable model if validation still fails.
- **Dimension error:** Caption does not require 16-pixel alignment. Layout/Auto do; check your upstream resolution selector.
- **API 401/403:** confirm `api_key_env` names an environment variable visible to the ComfyUI process, or fill `api_key` temporarily.
- **API 404:** check `api_base_url`; it usually ends in `/v1`. A full `/chat/completions` URL is also valid.
- **API rejects response_format:** set `api_json_mode` to false.
- **API rejects max_tokens or seed:** switch `api_max_tokens_field` to `max_completion_tokens`, or set `api_send_seed` to false.
- **API image request fails:** verify the selected API model accepts image content, not only text.

Schema validation checks structure and coordinates. It does not guarantee factual accuracy, exact OCR of visible text, or attractive composition. The node does not generate images or publish content.

## Examples and tests

- [Caption workflow](example_workflows/MOSS-Prompt-caption.json)
- [Layout workflow](example_workflows/MOSS-Prompt-layout.json)
- [MOSS T2I + prompt-enhance workflow](example_workflows/MossT2i-prompt-enhance.json)
- [Image-reverse workflow](example_workflows/MOSS-Prompt-image-reverse.json)
- [API-format prompt](example_workflows/MOSS-Prompt-api.json)
- [Third-party API backend workflow](example_workflows/MOSS-Prompt-external-api.json)
- [Third-party API backend API-format prompt](example_workflows/MOSS-Prompt-external-api.api.json)
- [Compiled Layout example](example_workflows/generated-layout-example.txt)

Run the offline tests from the repository directory:

```bash
python -m unittest discover -s tests -v
```

The 42 tests cover Chinese text preservation, coordinate conversion, invalid drafts, model and mmproj path containment, image message construction, vision-server arguments, interruption cleanup, repair behavior, Caption dimension handling, OpenAI-compatible URL normalization, authentication, payload construction, content-part parsing, HTTP errors, truncation, and the API backend. They use mocks for inference and do not require model weights or an API key.

## Attribution and license

This is a community integration, not an official OpenMOSS ComfyUI release.

The bundled [`skill/`](skill/) instructions and [`vendor/compile_prompt.py`](vendor/compile_prompt.py) come from [OpenMOSS/MOSS-Image](https://github.com/OpenMOSS/MOSS-Image). The compiler is kept unmodified. The adapter replaces agent tool execution with local JSON generation and in-process compilation.

Licensed under [Apache-2.0](LICENSE). See [NOTICE](NOTICE). Third-party model weights and llama.cpp runtimes retain their own licenses.
