# llama.cpp — Quick Start Guide

Run a local LLM server and interact with it via a web browser or `curl`.
The model loads **once** — change any parameter per request without restarting.

![LLAMA.CPP Flow](images/flow.png)

---

## Step 1 — Install llama.cpp

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

# Build with CUDA (NVIDIA GPU)
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

The compiled binary will be at:

```
llama.cpp/build/bin/llama-server
```

> **No GPU?** Replace `-DGGML_CUDA=ON` with `-DGGML_METAL=ON` (macOS) or omit it for CPU-only.

---

## Step 2 — Create a Preset File (.ini)

The preset file tells the server which model to load and sets default parameters.

File: `Qwen3.5-27B-FP16.ini`

```ini
version = 1

[*]
model           = /home/indows-11/my_code/model/Qwen3.5 benchmark/Qwen3.5-27B/test on vayucloud/Qwen3.5-27B-BF16-merge.gguf
mmproj          = /home/indows-11/my_code/model/Qwen3.5 benchmark/Qwen3.5-27B/mmproj-F32.gguf
c               = 12000    # context size (tokens)
n-gpu-layers    = 99       # layers offloaded to GPU (99 = all)
jinja           = true     # required for thinking mode
top-k           = 20
min-p           = 0.0
repeat-penalty  = 1.0
n-predict       = 8192     # max output tokens

[Qwen3.5-27B]              # this name is used in the "model" field of every request
temp             = 0.7
top-p            = 0.95
presence-penalty = 0.0
load-on-startup  = true
```

> Parameters set here are just **defaults** — you can override any of them per request.

---

## Step 3 — Start the Server

```bash
LLAMA_SERVER=/home/indows-11/my_code/llama.cpp/build/bin/llama-server
PRESET="/home/indows-11/my_code/model/Qwen3.5 benchmark/Qwen3.5-27B/llamacpp with curl (web interface)/Qwen3.5-27B-FP16.ini"

$LLAMA_SERVER \
  --models-preset "$PRESET" \
  --host 127.0.0.1 \
  --port 8081 \
  --models-max 1 \
  -np 1
```

| Flag | What it does |
|------|-------------|
| `--models-preset` | Path to your `.ini` file |
| `--models-max 1` | Keep only 1 model loaded (saves VRAM) |
| `-np 1` | 1 parallel request slot |
| `--host 127.0.0.1` | Local access only — change to `0.0.0.0` to allow network access |

When the server is ready you will see:

```
llama server listening at http://127.0.0.1:8081
```

The model is now loaded in VRAM and ready to accept requests.

---

## Step 4 — Use the Web Interface

Open your browser and go to:

```
http://127.0.0.1:8081
```

You will see a chat UI. Type a message and press Enter — no extra setup needed.

![Web Interface](images/index.png)

The sidebar on the left lets you switch presets and adjust parameters live.
Use the `☰` button to collapse the sidebar, and `🌙 / ☀️` to toggle dark/light mode.

### Using a custom Web UI

If you have a custom `index.html`, tell the server where to find it:

```bash
LLAMA_SERVER=/home/indows-11/my_code/llama.cpp/build/bin/llama-server
PRESET="/home/indows-11/my_code/model/Qwen3.5 benchmark/Qwen3.5-27B/llamacpp with curl (web interface)/Qwen3.5-27B-FP16.ini"
GUI="/home/indows-11/my_code/model/Qwen3.5 benchmark/Qwen3.5-27B/llamacpp with curl (web interface)/gui"

$LLAMA_SERVER \
  --models-preset "$PRESET" \
  --host 127.0.0.1 \
  --port 8081 \
  --models-max 1 \
  -np 1 \
  --path "$GUI"
```

---

## Step 5 — Chat via curl (Terminal)

### Basic chat

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "messages": [
      {"role": "user", "content": "Hello! What is a transformer model?"}
    ]
  }'
```

### With a system prompt

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant that replies concisely."},
      {"role": "user",   "content": "Explain REST API in simple terms."}
    ]
  }'
```

### Multi-turn conversation (keep chat history)

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "messages": [
      {"role": "user",      "content": "What is a list comprehension in Python?"},
      {"role": "assistant", "content": "It is a concise way to create lists, e.g. [x*2 for x in range(5)]."},
      {"role": "user",      "content": "Show me a more advanced example."}
    ]
  }'
```

### Adjust temperature without reloading

```bash
# Precise / deterministic (good for code, math)
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "temperature": 0.2,
    "messages": [
      {"role": "user", "content": "Write a Python quicksort function."}
    ]
  }'

# Creative / varied (good for writing, brainstorming)
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "temperature": 1.0,
    "messages": [
      {"role": "user", "content": "Write a short story about a robot learning to cook."}
    ]
  }'
```

### Stream tokens as they are generated

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  --no-buffer \
  -d '{
    "model": "Qwen3.5-27B",
    "stream": true,
    "messages": [
      {"role": "user", "content": "Write a Python linked list class."}
    ]
  }'
```

### Toggle thinking mode (Qwen3.5 / models that support it)

```bash
# Thinking ON — model reasons step by step before answering
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "temperature": 0.6,
    "chat_template_kwargs": {"enable_thinking": true},
    "messages": [
      {"role": "user", "content": "What are the trade-offs between SQL and NoSQL databases?"}
    ]
  }'

# Thinking OFF — direct answer, faster
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.5-27B",
    "temperature": 0.7,
    "chat_template_kwargs": {"enable_thinking": false},
    "messages": [
      {"role": "user", "content": "Summarize the trade-offs in two sentences."}
    ]
  }'
```

When thinking is ON, the response contains a separate `reasoning_content` field:

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "SQL is better for structured data with relationships...",
      "reasoning_content": "<think>Let me consider scalability, consistency, and schema flexibility...</think>"
    }
  }]
}
```

---

## Step 6 — OCR via curl (Extract text from an image)

> Requires `mmproj` to be set in the preset file (vision model).

Two styles are shown for each example — choose whichever fits your workflow:

| Style | When to use |
|-------|-------------|
| **Variable** — declare `IMAGE` and `IMAGE_B64` first | Scripts, reuse the same file multiple times |
| **Inline** — embed `$(base64 ...)` directly in the command | Quick one-off commands, no temp variable needed |

> **How inline works:** bash evaluates `$(...)` inside double-quoted strings, so `$(base64 -w 0 /path/to/file.png)` is substituted in place before curl runs.

---

### Extract all text from an image

**Variable style**

```bash
IMAGE=/path/to/document.png   # path to the image you want to process

IMAGE_B64=$(base64 -w 0 "$IMAGE")

curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,${IMAGE_B64}\"}},
        {\"type\": \"text\",      \"text\": \"Please extract all text from this image exactly as it appears.\"}
      ]
    }]
  }"
```

**Inline style** (same result, no variable declaration)

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,$(base64 -w 0 /path/to/document.png)\"}},
        {\"type\": \"text\",      \"text\": \"Please extract all text from this image exactly as it appears.\"}
      ]
    }]
  }"
```

---

### Extract text from a receipt / form

**Variable style**

```bash
IMAGE=/path/to/receipt.jpg   # path to the image you want to process

IMAGE_B64=$(base64 -w 0 "$IMAGE")

curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,${IMAGE_B64}\"}},
        {\"type\": \"text\",      \"text\": \"Extract all text from this receipt. List each line separately.\"}
      ]
    }]
  }"
```

**Inline style**

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,$(base64 -w 0 /path/to/receipt.jpg)\"}},
        {\"type\": \"text\",      \"text\": \"Extract all text from this receipt. List each line separately.\"}
      ]
    }]
  }"
```

---

### Extract text and output structured JSON

**Variable style**

```bash
IMAGE=/path/to/invoice.png   # path to the image you want to process

IMAGE_B64=$(base64 -w 0 "$IMAGE")

curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,${IMAGE_B64}\"}},
        {\"type\": \"text\",      \"text\": \"Extract the invoice data and return it as JSON with fields: date, total, items[].\"}
      ]
    }]
  }"
```

**Inline style**

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.1,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,$(base64 -w 0 /path/to/invoice.png)\"}},
        {\"type\": \"text\",      \"text\": \"Extract the invoice data and return it as JSON with fields: date, total, items[].\"}
      ]
    }]
  }"
```

---

### Describe an image / screenshot

**Variable style**

```bash
IMAGE=/path/to/screenshot.png   # path to the image you want to process

IMAGE_B64=$(base64 -w 0 "$IMAGE")

curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.5,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,${IMAGE_B64}\"}},
        {\"type\": \"text\",      \"text\": \"Describe what you see in this image.\"}
      ]
    }]
  }"
```

**Inline style**

```bash
curl http://127.0.0.1:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"Qwen3.5-27B\",
    \"temperature\": 0.5,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/png;base64,$(base64 -w 0 /path/to/screenshot.png)\"}},
        {\"type\": \"text\",      \"text\": \"Describe what you see in this image.\"}
      ]
    }]
  }"
```

> **Tip:** Use `temperature: 0.1` for OCR tasks to get the most accurate and literal transcription.

---

## Quick Reference

### Parameters you can change per request (no restart needed)

| Parameter | Example value | Effect |
|-----------|--------------|--------|
| `temperature` | `0.1` – `1.0` | Lower = more precise, higher = more creative |
| `top_p` | `0.8` – `0.95` | Nucleus sampling threshold |
| `max_tokens` | `512` | Limit output length |
| `stream` | `true` | Stream output token by token |
| `chat_template_kwargs` | `{"enable_thinking": false}` | Toggle thinking mode |

### Useful endpoints

```bash
# Check if server is running
curl http://127.0.0.1:8081/health

# List loaded models
curl http://127.0.0.1:8081/v1/models | python3 -m json.tool
```

---

## Project File Structure

```
llamacpp with curl (web interface)/
├── Qwen3.5-27B-FP16.ini                  # preset — model path & default params
├── Qwen3.5-27B-BF16-merge.gguf           # model weights (text + vision)
├── mmproj-F32.gguf                        # vision projector (required for OCR/image tasks)
├── images/
│   ├── flow.png                           # architecture flow diagram
│   └── index.png                          # web UI screenshot
└── gui/
    └── index.html                         # custom web UI
```
