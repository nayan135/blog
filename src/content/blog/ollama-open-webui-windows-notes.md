---
title: 'Running Ollama and open-webui on Windows: the annoying parts nobody mentions'
description: 'Notes from running Ollama and open-webui on a Windows machine, including the Docker networking issue that cost me an hour and the GPU driver mess.'
pubDate: '2026-07-03'
---
I set this up on a Windows 11 box with an RTX 3060 because I wanted a local chat interface for a few models without paying for API calls every time I test something. Took me about three hours total, most of which was fighting things that had nothing to do with the actual LLM part.

**Ollama itself is fine.** Installing it on Windows just works. It drops itself in as a background service and listens on `127.0.0.1:11434`. Pull a model with `ollama pull llama3.1:8b` and it downloads to `C:\Users\<you>\.ollama\models` by default. If your C: drive is small (mine has a 250GB SSD split with a games partition, don't ask), you'll want to move that. There's an `OLLAMA_MODELS` environment variable you can set to point at a different drive. Just remember to restart the Ollama service after changing it, or it'll keep writing to the old path and you'll wonder why your D: drive still shows 40GB free after downloading a 20GB model.

**The Docker networking thing wasted an hour.** I ran open-webui through Docker Desktop with the standard command from their docs:


docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main


Then went into the admin settings and pointed the Ollama API URL at `http://localhost:11434`. Got a connection refused every time. The fix is obvious once you know it: from inside a Docker container on Windows, `localhost` refers to the container, not your host machine. You need `http://host.docker.internal:11434` instead. The `--add-host` flag in that command is what makes `host.docker.internal` resolve at all, so if you copy-pasted the run command without it, nothing will work and the error message won't tell you why.

**GPU acceleration needs the right container tag and drivers.** Ollama will use your GPU automatically on native Windows if you've got a recent NVIDIA driver, no CUDA toolkit install needed, it bundles what it needs. But if you're running open-webui in Docker and expecting Ollama-in-Docker too (instead of the native Windows install), you need the NVIDIA Container Toolkit configured for WSL2, and Docker Desktop's WSL2 backend enabled. I skipped that by just running Ollama natively on Windows and only containerizing open-webui, which talks to it over the network. Simpler, and one less place for GPU passthrough to break.

**Windows Defender flagged the Ollama binary once.** Not every time, just once after an update, quarantined `ollama.exe` with zero explanation in the notification. Had to go into Protection History and manually restore it. If your models suddenly stop responding and the service won't start, check there before you start debugging YAML files that have nothing wrong with them.

**Memory math matters more than people say.** An 8B model at Q4 quantization needs roughly 5-6GB of VRAM to run comfortably. I tried loading a 70B model out of curiosity and Ollama happily started swapping to system RAM, which took a normal chat response from under a second to almost 90 seconds. It didn't crash, it just got unusably slow, no error, no warning. If a model's just quietly taking forever, check `nvidia-smi` before assuming something's broken.

**One more thing worth knowing:** open-webui stores its data (chat history, uploaded docs, settings) in that named Docker volume, not on your Windows filesystem in any obvious spot. If you `docker rm` the container without checking your volumes with `docker volume ls`, your chat history is still there, but it's easy to convince yourself you lost everything because you can't see the files in Explorer. They're sitting inside Docker Desktop's WSL2 VM disk image, not somewhere you'd casually browse to.

End result works well now. Ollama does the model serving, open-webui gives me a decent chat UI with model switching and RAG for a couple PDFs I dumped in, and the whole thing runs on hardware I already own. The setup pain was almost entirely networking and permissions, not the AI part, which tracks with most self-hosted software honestly.