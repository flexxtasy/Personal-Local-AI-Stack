# Local AI Stack

A self-hosted, fully local AI assistant stack — large language models running on my own hardware, exposed only on the loopback interface and driven by an agent framework. **Nothing leaves the machine.** Built to learn GPU inference, VRAM/RAM budgeting, Linux service management, and the operational side of running LLMs privately.

## Overview

This repo documents how I run open-weight LLMs locally for daily use and for authorized cybersecurity lab work — the hardware, the inference engines, the trade-offs I measured, and the operational tooling (systemd services, model switching, monitoring). It's a companion to my [Homelab](https://github.com/flexxtasy/Homelab) and [TSTC Local AI Project](https://github.com/flexxtasy/TSTC-Local-AI-Project) repos.

## Why local

- **Privacy / data sovereignty** — prompts, code, and documents never leave the host; no third-party API sees them. Even web search is routed through my own [SearXNG](https://github.com/searxng/searxng) instance so queries aren't logged by a search company.
- **No per-token cost** — once the hardware is there, inference is free and unmetered.
- **Control** — choice of model, quantization, context length, and tooling, with the whole thing reproducible from config.

## Hardware

| Component | Detail |
|---|---|
| CPU | AMD Ryzen 7 8700F (8-core, AVX-512) |
| GPU 0 | NVIDIA RTX 5070 Ti, 16 GB (CUDA — runs the model, also drives the display) |
| GPU 1 | AMD Radeon RX 9060 XT, 16 GB (Vulkan — now dedicated to GPU password cracking, `hashcat`) |
| RAM | 64 GB DDR5 |
| OS | NixOS (declarative config) + Hyprland |

## Current model

| Model | Quant / size | Engine | Context | Notes |
|---|---|---|---|---|
| **Qwen3.8-Flash-Next** (~125B MoE) | IQ3_XXS (GSQ-RCO) | [Strata](https://github.com/Niko1221/Strata) | **256K** | vision-capable; refusal-direction control vector applied (for unrestricted Q&A in authorized security labs) |

Running a **125B-parameter mixture-of-experts model at 256K context on a single 16 GB GPU** is the headline trade-off this stack solves — see below. A `model-switch` script starts/stops the engine and repoints the agent's default model, and frees the GPUs for gaming or cracking on demand.

> Earlier iterations of this stack ran a 35B dense model split across both GPUs (see *Dual-GPU hybrid inference* below) and a 2-bit build of the same Flash-Next MoE. Those are kept here as documented engineering, even though the current daily driver is the single 256K IQ3_XXS build.

## What I built & learned

### A 125B MoE at 256K context on one 16 GB card
The current daily model is a ~125B-parameter mixture-of-experts quant that, naively, needs far more than 16 GB of VRAM. It runs by **keeping the expert weights resident in system RAM (~50 GB of the 64 GB), pinning a hot cache of experts in VRAM, and streaming the 256K-token KV cache from RAM** instead of holding it all on the GPU. The result is full 256K context with vision, at ~85 tokens/sec decode and ~2,100 tokens/sec prompt reading, on a card that "shouldn't" fit the model. The main lesson was learning where each tensor actually has to live (resident experts vs. streamed KV vs. GPU working set) and tuning the splits against real `free`/`nvidia-smi` numbers rather than guessing.

### Dual-GPU CUDA + Vulkan hybrid inference *(previous daily model)*
An earlier daily model ran **split across two different-vendor GPUs at once** — an NVIDIA card over CUDA and an AMD card over Vulkan — in a single llama.cpp process. Getting both backends loaded together and tuning the layer split gave a measured **~+10% decode throughput** over Vulkan-only (~78 vs ~70 tok/s). The display runs on the NVIDIA card, so the split was tuned to leave it headroom to stay smooth under load.

**Lesson learned (the hard way):** a GPU inference binary built against one toolchain broke after an OS update changed the system graphics libraries (glibc/Mesa mismatch → the Vulkan backend failed to load → OOM crash loop). Fix: run a prebuilt binary through the system's dynamic loader so it matches the running system's libraries, instead of a self-built one pinned to a different set. Takeaway: GPU apps must match the host's graphics stack, or be isolated from it.

### Vision on a VRAM budget
Enabled image input (screenshots, diagrams) by placing the vision projector on the GPU and capping image tokens, while keeping the full long context — a deliberate VRAM trade-off (CPU-side image encoding was ~100× slower; naive GPU encoding OOM'd). Result: image understanding in a couple of seconds per image without giving up context length.

### Operations
- **systemd user services** for the model, enabled with lingering so it survives logout / starts on boot; `KillMode=control-group` so stopping the service also reaps its child engine instead of orphaning it on the GPU.
- **`model-switch`** CLI: `strata | stop | status` — starts the model, or frees the GPU + ~50 GB RAM for gaming/cracking, and keeps the agent config in sync.
- **Loopback-only** API endpoints — never bound to the network without an API key.
- Driven by an **agent framework** (Discord bot + CLI) with tool use gated behind an approvals step, and its own per-message context kept lean so each new chat is cheap.

## Measured results (informal, on this hardware)

| Setup | Decode | Prompt read | Notes |
|---|---:|---:|---|
| Flash-Next IQ3_XXS, 256K, single 16 GB GPU | ~85 tok/s | ~2,100 tok/s | ~0.3 s to first token; vision ~2 s/image; ~50 GB RAM resident |
| *(prev)* 35B dense, dual-GPU CUDA+Vulkan | ~78 tok/s | — | +10% over Vulkan-only |

## Skills demonstrated

- GPU inference (CUDA + Vulkan), multi-GPU layer splitting, and fitting a 125B MoE on 16 GB via RAM-resident experts + streamed KV cache
- VRAM/RAM budgeting measured against real system metrics
- Linux service management (systemd units, lingering, process-group lifecycle)
- Declarative/reproducible config (NixOS), dependency/toolchain debugging
- Running and benchmarking open-weight LLMs; quantization trade-offs
- Operating a local agent safely (loopback-only, API keys, approval-gated tools, self-hosted private search)

---

*Sanitized: no real hostnames, IP addresses, keys, or private network details are committed.*
