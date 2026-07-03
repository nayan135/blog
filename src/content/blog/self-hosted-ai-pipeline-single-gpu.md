---
title: 'What I learned running a self-hosted AI pipeline on a single RTX 3060'
description: 'No cluster, no massive cloud budget. Just one cheap GPU, Ollama, Whisper, and a lot of systemd service tuning. Here is what broke and how I fixed it.'
pubDate: '2026-07-03'
---
A few months ago, I decided to pull my old RTX 3060 (the 12GB VRAM variant) out of a dusty PC and set it up as a dedicated homelab server here in Kathmandu. Power cuts are less frequent now, but bandwidth is still expensive, and relying on OpenAI APIs for local automation scripting felt dumb. My goal was simple: get Whisper transcriptions and a llama3-based rag pipeline running on the same machine without running out of memory.

I thought it would be a weekend project of running docker-compose up and walking away. It wasn't. Here is what actually happens when you try to squeeze multiple models onto a single consumer-grade desktop card.

### The VRAM math is unforgiving

You cannot trust the default settings of modern LLM runners if you want to share hardware. 

Initially, I started Ollama for text generation and a custom Python script using the faster-whisper library for audio transcribing. By default, Ollama likes to keep models loaded in memory for 5 minutes after your last request. While Ollama held 6.5GB of VRAM for llama3:8b, my Whisper script would trigger. Whisper-large-v3 needs about 4.5GB of VRAM to initialize. Add the overhead of the Linux desktop running on the same card, and the system immediately hit the out-of-memory (OOM) threshold.

The screen flashed, the Nvidia driver reset, and both processes crashed.

To fix this, I had to force serial execution and aggressive unloading. I configured Ollama with the environment variable `OLLAMA_NUM_PARALLEL=1` and set `OLLAMA_KEEP_ALIVE=15s` in its systemd service file. This forces the LLM to release its grip on the VRAM almost immediately after generating a response, freeing up the space for Whisper to run.

### Python subprocesses are memory hogs

My initial pipeline script was a monolithic Python daemon. It listened to a local queue, pulled audio files, ran Whisper, then passed the text to the LLM. 

PyTorch leaks memory. Or rather, PyTorch's internal CUDA allocator does not like releasing memory back to the system immediately. Even after calling `del model` and running `gc.collect()` followed by `torch.cuda.empty_cache()`, the Nvidia system management interface (`nvidia-smi`) still showed gigabytes of locked memory.

The solution was ugly but incredibly reliable: process isolation. 

Instead of importing Whisper and PyTorch into my main long-running Python process, I offloaded them to separate CLI helper scripts. The main daemon calls them using Python's `subprocess.run()`. When the Whisper subprocess finishes transcribing, the entire OS process terminates, and the Linux kernel forcefully reclaims every single byte of VRAM. It adds about two seconds of startup overhead per call, but the pipeline has now run for three weeks without a single crash.

Here is how I structure the calls inside the main loop now:

```python
import subprocess
import json

def transcribe_audio(file_path):
    # We call a separate script so memory is freed instantly on exit
    result = subprocess.run(
        ["python3", "transcribe_worker.py", file_path],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        raise Exception(f"Transcription failed: {result.stderr}")
    return json.loads(result.stdout)["text"]
```

### Thermal throttling in a closed cupboard

I put the server in a small wooden cupboard to keep the fan noise down. That was a mistake.

The RTX 3060 is a quiet card, but running Whisper-large on a ten-minute audio file locks the GPU cores at 100% load. Within four minutes of continuous transcription, the GPU temperature hit 86°C, and the Nvidia driver started throttling the clock speed down to 900 MHz to prevent physical damage. Transcriptions that should have taken thirty seconds started taking four minutes.

I had to do two things:
1. Buy a cheap 120mm USB cabinet fan and cut a hole in the back of the cupboard.
2. Use `nvidia-smi` to manually set a power limit. 

By running `sudo nvidia-smi -pl 130` (limiting the card to 130 watts instead of its default 170W max), I lost about 5% in raw compute speed but dropped the peak operating temperature by almost 12°C. The card runs cool, never throttles, and the performance is entirely predictable now.

### The local setup wins

Despite the initial headaches, having an offline pipeline is incredible. I do not have to worry about API keys, monthly subscription limits, or sending personal notes to third-party servers. The latency for a standard 8b model is around 45 tokens per second on this setup, which is faster than I can read anyway.