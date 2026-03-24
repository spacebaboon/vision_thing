# vision-thing

Local vision-language model exploration using **Qwen3.5-9B** and **Phi-4-reasoning-vision-15B**, running on a Windows PC (RTX 4080 / WSL2) and driven from a Mac over SSH.

## Architecture

```
Mac (VS Code Remote-SSH / terminal)
         |  SSH
         v
WSL2 (Ubuntu) on Windows PC
  +-- llama-server  port 8080  (one model at a time)
  +-- ~/vlm-compare/
        +-- wsl/        server scripts
        +-- mac/        client + exercise scripts
        +-- models/     GGUF files (~17 GB total)
```

One model runs at a time. Switch with:

```bash
./wsl/start-servers.sh qwen    # Qwen3.5-9B
./wsl/start-servers.sh phi4    # Phi-4-reasoning-vision-15B
./wsl/stop-servers.sh
```

## Models

| Model | Size | Strengths |
|---|---|---|
| `Qwen3.5-9B-UD-Q4_K_XL.gguf` + mmproj | ~7 GB | General vision, fast |
| `Phi-4-reasoning-vision-15B-Q4_K_M.gguf` + mmproj | ~10 GB | Documents, charts, UI, reasoning |

Files go in `~/models/`. Download with:

```bash
./wsl/download-models.sh
```

## Setup

Requires NVIDIA driver ≥ 527 on Windows (no Linux driver needed inside WSL).

```bash
cd ~/vlm-compare
chmod +x wsl/*.sh
./wsl/setup-wsl.sh      # installs CUDA toolkit, builds llama.cpp, sets up SSH
```

Full prerequisites and verification steps are in [work-plan.md](work-plan.md) — Phases 0–1.

## Usage

### Interactive client

```bash
python3 mac/client.py                                    # interactive
python3 mac/client.py --prompt "Describe yourself."     # one-shot text
python3 mac/client.py --image img.png --prompt "..."    # with image
```

### Structured exercises

```bash
python3 mac/exercises.py --list                          # list suites
python3 mac/exercises.py --suite general --image img.jpg
python3 mac/exercises.py --suite document --image doc.png
python3 mac/exercises.py --suite ui --image screenshot.png
```

Exercise suites: `general`, `document`, `reasoning`, `ui`, `creative`

Results are saved to `mac/exercise_log_<model>_<timestamp>.json`.

## Learning plan

Work through [work-plan.md](work-plan.md) in order:

| Phase | Goal |
|---|---|
| 0–1 | WSL2 + CUDA + llama.cpp setup |
| 2 | Download models |
| 3 | Smoke-test both models (text + vision) |
| 4A–E | Structured exercises: visual understanding, documents, reasoning, UI, creative |
| 5 | Prompt engineering experiments |
| 6 | Build a practical pipeline (batch analyser, screenshot-to-code, RAG, or TTRPG assistant) |
| 7 | Explore additional models |
| 8 | Fine-tuning with QLoRA (stretch goal) |

## GPU

```bash
nvidia-smi          # current VRAM usage
nvidia-smi -l 1     # live refresh (useful during inference)
```

The RTX 4080 has 16 GB VRAM. Only one model loads at a time — `start-servers.sh` stops the current model before loading the next.
