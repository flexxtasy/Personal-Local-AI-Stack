# Local AI Stack

A self-hosted, fully local AI assistant stack — large language models running on my own hardware, exposed only on the loopback interface and driven by an agent framework. **Nothing leaves the machine.** Built to learn GPU inference, multi-GPU scheduling, Linux service management, and the operational side of running LLMs privately.

## Overview

This repo documents how I run open-weight LLMs locally for daily use and for authorized cybersecurity lab work — the hardware, the inference engines, the trade-offs I measured, and the operational tooling (systemd services, model switching, monitoring). It's a companion to my [Homelab](https://github.com/flexxtasy/Homelab) and [TSTC Local AI Project](https://github.com/flexxtasy/TSTC-Local-AI-Project) repos.

## Why local

- **Privacy / data sovereignty** — prompts, code, and documents never leave the host; no third-party API sees them.
- **No per-token cost** — once the hardware is there, inference is free and unmetered.
- **Control** — choice of model, quantization, context length, and tooling, with the whole thing reproducible from config.

## Hardware

| Component | Detail |
|---|---|
| CPU | AMD Ryzen 7 8700F |
| GPU 0 | NVIDIA RTX 5070 Ti (CUDA, also drives the display) |
| GPU 1 | AMD Radeon RX 9060 XT (Vulkan) |
| RAM | 48 GB (expansion planned) |
| OS | NixOS (declarative config) + Hyprland |

## Models

| Name | Model | Engine | Role |
|---|---|---|---|
| **empero** | Qwen3.8-35B-A3B-Distill (Q5_K_M) | llama.cpp | daily driver — dual-GPU, **vision-capable**, ~225K context |
| **strata** | Qwen3.8-Flash-Next 125B (Q2_0) | [Strata](https://github.com/Niko1221/Strata) | on-demand heavyweight for harder reasoning tasks |

Only one model runs at a time (they share the GPUs); a `model-switch` script stops the others, starts the chosen one, and repoints the agent's default model at it.

## What I built & learned

### Dual-GPU CUDA + Vulkan hybrid inference
The interesting part: the daily model runs **split across two different-vendor GPUs at once** — an NVIDIA card over CUDA and an AMD card over Vulkan — in a single llama.cpp process. Getting both backends loaded together and tuning the layer split between them gave a measured **~+10% decode throughput** over Vulkan-only (~78 vs ~70 tok/s), and prefill improved too. The display runs on the NVIDIA card, so the split is tuned to leave it enough headroom to stay smooth under load.

**Lesson learned (documented the hard way):** a GPU inference binary built against one toolchain broke after an OS update changed the system graphics libraries (glibc/Mesa version mismatch → the AMD Vulkan backend failed to load → out-of-memory crash loop). Fix: run a prebuilt binary through the system's dynamic loader so it matches the running system's libraries, instead of a self-built one pinned to a different library set. Takeaway: GPU apps must match the host's graphics stack, or be isolated from it.

### Vision on a VRAM budget
Enabled image input (screenshots, diagrams) by placing the vision projector on the NVIDIA card and capping image tokens, while keeping the full long context — a deliberate VRAM trade-off (CPU-side image encoding was ~100× slower; naive GPU encoding OOM'd). Result: image understanding at a few seconds per image without giving up context length.

### Operations
- **systemd user services** for each model, enabled with lingering so they survive logout / start on boot; `KillMode=control-group` so stopping a service also reaps its child engine instead of orphaning it on the GPU.
- **`model-switch`** CLI: `empero | strata | stop | status` — swaps the running model and updates the agent config atomically.
- **Loopback-only** API endpoints — never bound to the network without an API key.
- Driven by an **agent framework** with tool use gated behind an approvals step.

## Measured results (informal, on this hardware)

| Model | Decode | Notes |
|---|---:|---|
| empero (35B, hybrid) | ~78 tok/s | +10% over Vulkan-only; vision ~3.4 s/image |
| strata (125B Q2_0) | ~82 tok/s | heavier model, loads on demand (~1–2 min) |

## Skills demonstrated

- GPU inference (CUDA + Vulkan), multi-GPU layer splitting, VRAM budgeting
- Linux service management (systemd units, lingering, process-group lifecycle)
- Declarative/reproducible config (NixOS), dependency/toolchain debugging
- Running and benchmarking open-weight LLMs; quantization trade-offs
- Operating a local agent safely (loopback-only, API keys, approval-gated tools)

---

*Sanitized: no real hostnames, IP addresses, keys, or private network details are committed.*
