# VLM Learning Project — Work Plan

Local vision-language model exploration using Qwen3.5-9B and Phi-4-reasoning-vision-15B,
running on a Windows PC (RTX 4080, WSL2), driven from a Mac via SSH.

**Status key:** `[ ]` not started · `[~]` in progress · `[x]` done  
**CC** = good candidate for Claude Code execution

---

## Project overview

```
Mac (VS Code Remote-SSH / terminal)
         |  SSH
         v
WSL2 (Ubuntu) on Windows PC
  +-- llama-server  port 8080  (one model at a time)
  +-- ~/vlm-compare/
        +-- wsl/        server scripts
        +-- mac/        client + exercise scripts
        +-- models/     GGUF files (~17GB total)
```

One model runs at a time. Switch with:

```
./wsl/start-servers.sh qwen
./wsl/start-servers.sh phi4
```

---

## Phase 0 — Windows & WSL2 prerequisites

These steps happen on the Windows side or in a fresh WSL2 session.
Do these once before anything else.

### 0.1 — NVIDIA driver `[ ]`

Install NVIDIA Game Ready or Studio driver **version 527 or later** on Windows.
This is the only NVIDIA software needed on the Windows side — it provides
CUDA support to WSL2 automatically via `/usr/lib/wsl/lib/`.

Do **not** install a Linux NVIDIA driver inside WSL.

Verify in PowerShell:

```powershell
nvidia-smi
```

### 0.2 — Confirm WSL2 (not WSL1) `[ ]`

```powershell
wsl --set-version Ubuntu 2
wsl --update
```

### 0.3 — Verify GPU visible inside WSL `[ ]`

```bash
wsl nvidia-smi
# Should show RTX 4080 with memory stats
```

### 0.4 — Set up SSH access from Mac `[ ]`

The setup script in Phase 1 enables the SSH server inside WSL.
After that, from your Mac:

```bash
ssh your-username@YOUR_PC_LAN_IP
```

Find your PC's LAN IP with `ipconfig` in Windows — use the IPv4 address
for your WiFi or Ethernet adapter, not the WSL internal IP.

For VS Code: install the **Remote - SSH** extension, then
`Cmd+Shift+P → Remote-SSH: Connect to Host`.

---

## Phase 1 — WSL2 environment setup

**CC: good fit** — all bash, no interaction needed once started.

### 1.1 — Clone / copy project into WSL `[ ]`

Put the `vlm-compare` project folder in your WSL home directory:

```bash
# Option A: copy the zip across and unzip
cp /mnt/c/Users/yourname/Downloads/vlm-compare-ssh-v2.zip ~/
cd ~ && unzip vlm-compare-ssh-v2.zip && mv vlm-compare-ssh vlm-compare

# Option B: clone from a git repo if you've pushed it there
git clone <your-repo-url> ~/vlm-compare
```

### 1.2 — Run setup script `[ ]`

```bash
cd ~/vlm-compare
chmod +x wsl/*.sh
./wsl/setup-wsl.sh
```

This does the following in one go:

- Installs build tools (`cmake`, `git`, `gcc`, etc.)
- Installs CUDA toolkit for WSL2 (`cuda-toolkit-12-4`)
- Clones and builds `llama.cpp` with `-DGGML_CUDA=ON`
- Installs Python packages: `flask requests huggingface_hub hf_transfer openai rich`
- Enables and auto-starts the SSH server via `/etc/wsl.conf`
- Creates `~/models/` directory
- Writes a `.env` file for the other scripts

Expected duration: **5–10 minutes** (mostly llama.cpp compile).

Verify the build succeeded:

```bash
~/llama.cpp/build/bin/llama-server --version
```

### 1.3 — Verify CUDA is being used `[ ]`

```bash
# Should show nvcc and report a CUDA version
nvcc --version

# After building, confirm CUDA support compiled in
~/llama.cpp/build/bin/llama-server --help | grep -i cuda
```

---

## Phase 2 — Download models

**CC: good fit** — long-running download, no interaction needed.

### 2.1 — Download both GGUFs `[ ]`

```bash
./wsl/download-models.sh
```

Downloads four files to `~/models/`:

| File                                         | Size  | Notes                |
| -------------------------------------------- | ----- | -------------------- |
| `Qwen3.5-9B-UD-Q4_K_XL.gguf`                 | ~6 GB | Main Qwen weights    |
| `mmproj-Qwen3.5-9B-f16.gguf`                 | ~1 GB | Qwen vision encoder  |
| `Phi-4-reasoning-vision-15B-Q4_K_M.gguf`     | ~9 GB | Main Phi-4 weights   |
| `mmproj-Phi-4-reasoning-vision-15B-f16.gguf` | ~1 GB | Phi-4 vision encoder |

Total: ~17 GB. The mmproj files are the vision encoders — llama.cpp loads
them separately alongside the main model. Without them, image input won't work.

> **If a download 404s:** GGUF files are community-uploaded and filenames
> occasionally change. Check the repos and update filenames in
> `download-models.sh` and `start-servers.sh`:
>
> - https://huggingface.co/unsloth/Qwen3.5-9B-GGUF
> - https://huggingface.co/bartowski/Phi-4-reasoning-vision-15B-GGUF

### 2.2 — Verify files `[ ]`

```bash
ls -lh ~/models/*.gguf
```

All four files should be present with sensible sizes.

---

## Phase 3 — First run verification

Smoke-test each model before doing any exercises.

### 3.1 — Start Qwen, test text `[ ]`

```bash
./wsl/start-servers.sh qwen
# Wait for "ready" message (~30-60s)

# Quick text smoke test
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"default","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":50}' \
  | python3 -m json.tool | grep content
```

### 3.2 — Test Qwen with an image `[ ]`

```bash
# Encode any image and send it
IMAGE_PATH=~/some-image.png   # put any image here
B64=$(base64 -w 0 "$IMAGE_PATH")

curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"default\",\"messages\":[{\"role\":\"user\",\"content\":[
    {\"type\":\"text\",\"text\":\"What is in this image?\"},
    {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/png;base64,$B64\"}}
  ]}],\"max_tokens\":200}" \
  | python3 -m json.tool | grep content
```

A real description confirms vision is working end-to-end.

### 3.3 — Repeat for Phi-4 `[ ]`

```bash
./wsl/start-servers.sh phi4
# Repeat the curl tests above
```

### 3.4 — Test the Python client `[ ]`

From inside VS Code Remote-SSH terminal (or with SSH tunnel active on Mac):

```bash
python3 mac/client.py --prompt "Describe yourself in one sentence."
```

You should see a coloured panel with the model name in the title and a response.

---

## Phase 4 — Learning exercises

Work through these in order. Each phase builds on the last.
Run exercises on **Qwen first**, then switch to **Phi-4** and repeat
the same prompts to build comparison intuition.

Log files are saved automatically by `exercises.py` with the model ID
in the filename, making it easy to diff later.

### Phase 4A — General visual understanding

**Goal:** establish baselines; understand what these models notice and miss.

```bash
./wsl/start-servers.sh qwen
python3 mac/exercises.py --suite general --image ~/path/to/any-image.jpg
./wsl/start-servers.sh phi4
python3 mac/exercises.py --suite general --image ~/path/to/any-image.jpg
```

| #   | Exercise                | Prompt                                                      | What to look for                                |
| --- | ----------------------- | ----------------------------------------------------------- | ----------------------------------------------- |
| 1   | Basic description `[ ]` | "Describe this image in detail."                            | Depth of observation, what gets missed          |
| 2   | Object counting `[ ]`   | "Count every distinct object or person visible. List them." | Accuracy, hallucination of non-existent objects |
| 3   | Colour and layout `[ ]` | "Describe the colour palette and spatial layout."           | Perceptual precision                            |

**Suggested images:** a busy street photo, a room interior, a group photo.

**Notes / observations:**

```
Qwen:
Phi-4:
```

---

### Phase 4B — Document and chart understanding

**Goal:** test structured information extraction — a high-value real-world capability.
Phi-4 is specifically strong here; use this phase to feel that difference.

```bash
python3 mac/exercises.py --suite document --image ~/path/to/document-or-chart.png
```

| #   | Exercise               | Prompt                                                         | What to look for                        |
| --- | ---------------------- | -------------------------------------------------------------- | --------------------------------------- |
| 4   | Document summary `[ ]` | "Summarise the key information on this page in bullet points." | Accuracy, completeness, hallucination   |
| 5   | Chart extraction `[ ]` | "Extract all data points and labels. Present as a table."      | Numeric accuracy, axis/label reading    |
| 6   | OCR accuracy `[ ]`     | "Transcribe all text visible, preserving layout."              | Character accuracy, layout preservation |

**Suggested images:** a PDF page screenshot, a bar/line chart, a receipt or form.

**Notes / observations:**

```
Qwen:
Phi-4:
```

---

### Phase 4C — Visual reasoning

**Goal:** probe higher-level understanding beyond description — inference, logic, problem-solving.
This is where thinking mode matters. Both models support it; experiment with
prompting that triggers chain-of-thought vs. direct answers.

```bash
python3 mac/exercises.py --suite reasoning --image ~/path/to/image.png
```

| #   | Exercise                 | Prompt                                                                     | What to look for                    |
| --- | ------------------------ | -------------------------------------------------------------------------- | ----------------------------------- |
| 7   | What is wrong here `[ ]` | "Is anything unusual, incorrect, or inconsistent? Explain your reasoning." | Depth of reasoning, false positives |
| 8   | Sequence inference `[ ]` | "If this is a step in a process, what came before and what comes next?"    | Inferential capability              |
| 9   | Maths and logic `[ ]`    | "Solve any mathematical or logical problem visible. Show your working."    | Thinking mode activation, accuracy  |

**Suggested images:** a logic puzzle, a maths worksheet photo, a diagram with a deliberate error.

**Thinking mode tip:** prepend your prompt with "Think step by step before answering." and
compare the output to a direct prompt on the same image.

**Notes / observations:**

```
Qwen:
Phi-4:
```

---

### Phase 4D — UI and screenshot understanding

**Goal:** test GUI grounding — Phi-4 is specifically trained on this and should noticeably
outperform Qwen. This phase is also directly relevant to your frontend work.

```bash
python3 mac/exercises.py --suite ui --image ~/Desktop/screenshot.png
```

| #   | Exercise                   | Prompt                                                                                    | What to look for                                 |
| --- | -------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 10  | UI element inventory `[ ]` | "List every UI element (buttons, inputs, labels, icons). For each: type, text, position." | Completeness, positional accuracy                |
| 11  | Screenshot to HTML `[ ]`   | "Write HTML and CSS that reproduces this UI layout as closely as possible."               | Layout fidelity — you can evaluate this directly |
| 12  | Accessibility audit `[ ]`  | "Review this UI for accessibility issues. What would fail WCAG 2.1 AA?"                   | Domain reasoning quality                         |

**Suggested images:** a web app screenshot, a form, a navigation menu, a mobile UI.

**Notes / observations:**

```
Qwen:
Phi-4:
```

---

### Phase 4E — Creative and calibration

**Goal:** test interpretive reasoning and establish a text-only baseline
(no vision) to separate language capability from visual capability.

```bash
python3 mac/exercises.py --suite creative --image ~/path/to/image.jpg
```

| #   | Exercise                       | Prompt                                                              | What to look for                    |
| --- | ------------------------------ | ------------------------------------------------------------------- | ----------------------------------- |
| 13  | Alt text `[ ]`                 | "Write alt text suitable for a visually impaired user."             | Prioritisation of meaningful detail |
| 14  | Mood and tone `[ ]`            | "Describe the emotional tone as if writing for an art catalogue."   | Creative / interpretive range       |
| 15  | Text only — architecture `[ ]` | "Explain early-fusion vs mid-fusion vision-language architectures." | Pure language reasoning, no image   |

**Notes / observations:**

```
Qwen:
Phi-4:
```

---

## Phase 5 — Prompt engineering experiments

**Goal:** develop systematic intuition for how prompt phrasing affects
visual output. These are freeform experiments — use the interactive client.

```bash
./wsl/start-servers.sh qwen   # or phi4
python3 mac/client.py
```

### 5.1 — Persona framing `[ ]`

Compare three framings on the same image:

- Neutral: "Describe this image."
- Expert: "You are an expert analyst. List every distinct element visible."
- Accessibility: "Describe this image as if explaining it to someone who cannot see it."

**Observation:**

```

```

### 5.2 — Specificity ladder `[ ]`

Start vague, progressively get more specific, note how output changes:

- "What is this?"
- "What type of document is this and what is its purpose?"
- "Extract the five most important pieces of information from this document."

**Observation:**

```

```

### 5.3 — Thinking mode toggle `[ ]`

On the same reasoning-heavy image, compare:

- Direct: "What is wrong with this diagram?"
- Thinking: "Think step by step before answering. What is wrong with this diagram?"
- Forced concise: "In one sentence, what is wrong with this diagram?"

**Observation:**

```

```

### 5.4 — Output format control `[ ]`

Same image, different format instructions:

- "Describe this image."
- "Describe this image. Respond in JSON with keys: objects, colours, layout, mood."
- "Describe this image as a markdown table with columns: element, description, position."

**Observation:**

```

```

---

## Phase 6 — Build a practical pipeline

**Goal:** apply what you have learned to build something genuinely useful.
Choose one or more based on interest.

### Option A — Batch image analyser `[ ]` CC

A script that processes a folder of images and generates a structured report.
Useful for: analysing screenshots, cataloguing photos, processing documents.

```bash
# Sketch: python3 mac/batch_analyse.py ~/folder/ --prompt "Describe this image" --output report.json
```

Key things to build:

- Iterate over image files in a directory
- Call the inference server for each
- Aggregate results into JSON or markdown
- Handle errors gracefully (model timeout, unsupported format)

### Option B — Screenshot-to-code pipeline `[ ]` CC

Given a UI screenshot, generate HTML/CSS, write it to a file, and open
a preview. A natural fit given your frontend background.

Key things to build:

- Accept a screenshot path as input
- Send to Phi-4 (better for UI tasks) with the screenshot-to-HTML prompt
- Write output to a `.html` file
- Optionally open in a browser with `xdg-open` (WSL) or `open` (Mac)

### Option C — Visual RAG (retrieval-augmented generation) `[ ]`

Build a simple image "search" system: describe each image in a folder
using the model, store descriptions in a vector store (ChromaDB), then
query by natural language.

Key things to build:

- Index phase: generate text descriptions for all images, embed with a
  small text embedding model, store in ChromaDB
- Query phase: embed a natural language query, retrieve nearest images,
  display results

### Option D — TTRPG visual assistant `[ ]`

Given a photo of handwritten notes, a scanned map, or a miniature setup,
generate structured campaign content. A fun domain-specific application
that tests several capabilities at once (OCR, spatial reasoning, creative output).

---

## Phase 7 — Explore additional models

**Goal:** expand beyond the initial two models as new releases emerge
or specific capabilities warrant investigation.

### 7.1 — Criteria for adding a new model `[ ]`

Before downloading a new model, check:

- Does it fit in 16GB VRAM at a usable quantization?
- Does llama.cpp support its architecture and mmproj format?
- What specific capability gap does it address vs. Qwen/Phi-4?

### 7.2 — Adding a model to the setup `[ ]`

Update `start-servers.sh` with a new `case` block:

```bash
newmodel)
    MODEL_KEY="newmodel"
    GGUF="$MODEL_DIR/new-model.gguf"
    MMPROJ="$MODEL_DIR/new-model-mmproj.gguf"
    LABEL="New Model Name"
    EXTRA_ARGS="--temp 0.7"
    CTX=32768
    ;;
```

Add its display config to `client.py` and `exercises.py`:

```python
"newmodel": {"label": "New Model Name", "color": "yellow"},
```

### 7.3 — Models to watch `[ ]`

| Model               | Why interesting                         | Status                     |
| ------------------- | --------------------------------------- | -------------------------- |
| Qwen3.5-27B (dense) | Stronger than 9B, may fit with Q3 quant | Monitor VRAM fit           |
| Llama 4 vision      | Meta's next multimodal release          | Watch for GGUF support     |
| Gemma 3 vision      | Google, strong on documents             | Watch for quantized builds |

---

## Phase 8 — Fine-tuning (stretch goal)

**Goal:** apply your prior QLoRA experience to a vision-language model.
This is complex and compute-intensive — treat as a later-stage exploration.

### 8.1 — Prerequisites `[ ]`

- Complete Phases 4–5 first to understand model behaviour before modifying it
- Unsloth supports QLoRA fine-tuning for Qwen3.5 (they provided day-zero access)
- For vision fine-tuning you need image-text pairs in a structured format

### 8.2 — Tooling options `[ ]`

| Tool               | Notes                                                                            |
| ------------------ | -------------------------------------------------------------------------------- |
| Unsloth            | Best RTX 4080 support, minimal VRAM overhead, familiar from your Qwen Coder work |
| LLaMA-Factory      | More flexible, supports more architectures                                       |
| Swift (ModelScope) | Recommended by Qwen team, good multimodal support                                |

### 8.3 — Dataset ideas `[ ]`

- UI screenshot → HTML pairs (generate with Phase 6 Option B, manually curate)
- TTRPG map → description pairs (domain-specific, unique dataset)
- Document → structured extraction pairs

---

## Running notes

Space for observations, decisions, and things to revisit.

### Model comparison notes

```
Qwen3.5-9B impressions:


Phi-4-15B impressions:


Key differences observed:

```

### Setup issues encountered

```

```

### Decisions log

| Date | Decision                            | Rationale                           |
| ---- | ----------------------------------- | ----------------------------------- |
|      | Started with single-model-at-a-time | Simpler; can add simultaneous later |
|      |                                     |                                     |

### Next steps (rolling)

```
- [ ]
- [ ]
- [ ]
```

---

## Quick reference

### Start / stop

```bash
./wsl/start-servers.sh qwen     # load Qwen3.5-9B
./wsl/start-servers.sh phi4     # load Phi-4-15B (stops current model first)
./wsl/stop-servers.sh           # stop everything
```

### Client

```bash
python3 mac/client.py                          # interactive mode
python3 mac/client.py --prompt "..."           # one-shot text
python3 mac/client.py --image img.png --prompt "..."   # one-shot with image
```

### Exercises

```bash
python3 mac/exercises.py --list                          # list suites
python3 mac/exercises.py --suite ui --image img.png      # run a suite
```

### Logs

```bash
tail -f ~/vlm-compare/wsl/logs/qwen.log     # live server output
tail -f ~/vlm-compare/wsl/logs/phi4.log
ls ~/vlm-compare/mac/exercise_log_*.json    # saved exercise results
```

### Check GPU

```bash
nvidia-smi                  # current VRAM usage
nvidia-smi -l 1             # live refresh every second (useful during inference)
```
