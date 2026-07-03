---
title: 'Running Open WebUI and Ollama on Windows: What Actually Broke'
description: 'Notes from running Ollama and Open WebUI on Windows, including WSL2 quirks, Docker networking issues, and GPU passthrough problems.'
pubDate: '2026-07-03'
---
I wanted a local ChatGPT-style setup for messing with models without paying per token or sending anything to an API. Ollama for serving models, Open WebUI for the chat interface. Sounds simple. It is simple, technically, but Windows adds a layer of friction that the docs don't really prepare you for. Here's what actually happened.

## Native Ollama vs WSL2

First decision: run Ollama natively on Windows or inside WSL2. Ollama ships a native Windows installer now, so I started there. It works, but GPU detection was inconsistent — my RTX card would get picked up on some runs and silently fall back to CPU on others, with no clear log message telling me why. Restarting the Ollama service fixed it half the time.

I eventually moved Ollama into WSL2 (Ubuntu) instead, since NVIDIA's CUDA support in WSL2 is more mature and better documented than the native Windows path. The catch: you need the WSL-specific NVIDIA driver setup, not the regular Windows GeForce driver assuming it'll just work. If you install the standard driver and skip the WSL CUDA toolkit steps, `nvidia-smi` inside WSL will just not find your GPU and give you a vague error.

Once that was sorted, Ollama in WSL2 was noticeably more stable for GPU offloading than the native Windows build was at the time.

## Open WebUI and Docker Desktop networking

Open WebUI is meant to be run via Docker, pointing at your Ollama instance. This is where Windows networking got annoying. If Ollama is running inside WSL2 and Open WebUI is running in a Docker container (also technically inside WSL2, via Docker Desktop's backend), you'd think `localhost` would just work between them. It doesn't, reliably.

The fix that worked for me: use `host.docker.internal` as the Ollama base URL inside the Open WebUI container config, instead of `localhost` or `127.0.0.1`. Docker Desktop on Windows resolves that to the host, which is where Ollama's API is actually listening. If you're running Ollama as a native Windows service instead of in WSL2, this matters even more since your container and your Ollama process are on genuinely different network stacks.

Also: make sure Ollama is bound to `0.0.0.0` and not just `127.0.0.1`, otherwise nothing outside its own process can reach it, container or not. Set `OLLAMA_HOST=0.0.0.0` as an environment variable before starting the service.

## Firewall prompts and antivirus interference

Windows Defender flagged the Ollama binary on first run, which is expected for any unsigned-ish executable making network calls, but it's easy to click through the firewall prompt too fast and accidentally block it instead of allow it. If Open WebUI can't reach Ollama at all and the URL config looks right, check Windows Defender Firewall's allowed apps list before you assume it's a Docker problem.

## Disk space adds up fast

Models are big and Windows doesn't make it obvious where they're going. Ollama's model storage lives under your user profile by default (`%USERPROFILE%\.ollama` if native, or the WSL filesystem's home directory if you went that route). Pulling a handful of 7B and 13B models will eat 40-50GB without you noticing, especially since Windows doesn't surface WSL2's virtual disk usage in normal Explorer disk-usage views. You have to check it from inside WSL or run `wsl --manage <distro> --set-sparse` type maintenance occasionally, since the WSL virtual disk file doesn't shrink automatically even after you delete models.

## What I'd tell someone starting this today

Run Ollama inside WSL2, not native Windows, if you have an NVIDIA GPU and care about performance. Set `OLLAMA_HOST=0.0.0.0`. Point Open WebUI at `host.docker.internal` instead of `localhost`. Check the Windows firewall before you assume it's a config issue. And keep an eye on your WSL virtual disk size, because it grows quietly and Windows won't warn you.

None of this is hard once you know it, but the failure modes are silent and unhelpful. Everything just... doesn't connect, with no error pointing at the actual cause. If you're setting this up fresh, save yourself the debugging loop and start with WSL2 from the beginning.