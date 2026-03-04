---
layout: post
title: "Dead GPU to Docker-Ready: How I Built a Complete GPU Container Stack from Scratch"
date: 2025-03-04
tags: [gpu, docker, nvidia, cuda, infrastructure, mlops]
excerpt: "nvidia-smi failed. No driver. No container runtime. No GPU. Here's how I diagnosed, fixed, and verified a complete GPU container stack — and what every AI infrastructure engineer needs to understand about the layers underneath."
---

Most AI infrastructure guides start with a working GPU.

This one starts with a broken one.

```bash
$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver
```

That single error blocks everything downstream — Docker GPU passthrough, PyTorch CUDA training, container runtimes — all of it. Before you can build AI infrastructure, the foundation has to work.

This is the story of diagnosing that failure, fixing it layer by layer, and ending up with a clean, verified GPU container stack. Every step is reproducible. Every command is real.

---

## The Stack We're Building

Before touching a terminal, here's what a complete GPU container stack looks like — and where things break:

```
┌─────────────────────────────────────────────────┐
│           Your AI Workload (PyTorch, etc.)      │
├─────────────────────────────────────────────────┤
│              Docker Container                   │
├─────────────────────────────────────────────────┤
│         NVIDIA Container Toolkit (Runtime)      │
├─────────────────────────────────────────────────┤
│              NVIDIA Driver (470)                │
├─────────────────────────────────────────────────┤
│           Linux Kernel + GPU Device Node        │
├─────────────────────────────────────────────────┤
│              Physical GPU (GT 710)              │
└─────────────────────────────────────────────────┘
```

Each layer depends on the one below it. If the driver layer breaks, everything above it fails — silently or loudly. My stack was broken at the driver layer. The fix required working up from the bottom.

---

## Phase 1 — Diagnose the Failure

The first thing to do with `nvidia-smi` failing is not panic and reinstall everything. It's to diagnose *which layer* broke.

```bash
# Check if the GPU is physically visible to the system
lspci | grep -i nvidia
```

If your GPU appears here, the hardware is fine. The problem is software — driver or kernel module.

```bash
# Check if the kernel module loaded
lsmod | grep nvidia
```

Empty output = the NVIDIA kernel module is not loaded = driver missing or mismatched.

**My situation:** GPU visible via `lspci`, kernel module absent. Driver was either never installed or broken after a kernel update.

---

## Phase 2 — Identify Your GPU and the Right Driver

This step is critical and often skipped. Installing the wrong driver wastes an hour.

```bash
ubuntu-drivers devices
```

Output told me everything:

```
== /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
modalias : pci:v000010DEd00001287sv...
vendor   : NVIDIA Corporation
model    : GK208B [GeForce GT 710]
driver   : nvidia-driver-470 - distro non-free recommended
```

**Key information extracted:**

| Property | Value |
|---|---|
| GPU | NVIDIA GeForce GT 710 (GK208B) |
| Architecture | Kepler (legacy) |
| Recommended Driver | nvidia-driver-470 |
| Max CUDA Version | 11.4 |

> **Why this matters for MLOps:** Matching driver to GPU architecture isn't optional. A Kepler GPU running a driver built for Ampere will fail silently or not load at all. Always let `ubuntu-drivers` tell you what's correct — don't guess from a blog post.

---

## Phase 3 — Install the Correct Driver

```bash
sudo apt install nvidia-driver-470
sudo reboot
```

After reboot, the verification:

```bash
nvidia-smi
```

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.256.02   Driver Version: 470.256.02   CUDA Version: 11.4    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC|
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M.|
|===============================+======================+======================|
|   0  GeForce GT 710      Off  | 00000000:01:00.0 Off |                  N/A|
|  40%   40C    P8    N/A /  N/A|    208MiB /  1998MiB |      0%      Default|
+-----------------------------------------------------------------------------+
```

Driver layer: ✅ working.

---

## Phase 4 — NVIDIA Container Toolkit (Where It Gets Interesting)

This is the layer that allows Docker containers to access your GPU. Without it, `--gpus all` does nothing.

### The broken path (what NOT to do)

My first attempt hit a wall:

```bash
# This will likely 404 on Ubuntu 24.04
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor ...
echo "deb ... ubuntu22.04/$(ARCH) /" | sudo tee ...
sudo apt update
# → 404 Not Found
```

**What went wrong:** The repo URL was pinned to Ubuntu 22.04 (`ubuntu22.04`). I'm on Ubuntu 24.04 (`ubuntu24.04`). Outdated blog posts and docs cause this constantly.

### The correct path (verified working)

```bash
# Step 1: Clean up any broken repo entries
sudo rm /etc/apt/sources.list.d/nvidia-container-toolkit.list 2>/dev/null

# Step 2: Add the correct NVIDIA GPG key
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# Step 3: Add the correct repo for your Ubuntu version
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Step 4: Install
sudo apt update
sudo apt install -y nvidia-container-toolkit

# Step 5: Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### The GPU passthrough flow

```
docker run --gpus all my-image
        │
        ▼
  Docker Engine
        │  (--gpus all flag)
        ▼
  NVIDIA Container Runtime
        │  (translates GPU request)
        ▼
  libnvidia-container
        │  (mounts GPU device nodes)
        ▼
  /dev/nvidia0, /dev/nvidiactl
        │
        ▼
  NVIDIA Kernel Driver (470)
        │
        ▼
  Physical GPU
```

Every arrow in that chain has to work for GPU containers to function.

---

## Phase 5 — Verify End-to-End

```bash
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.256.02   Driver Version: 470.256.02   CUDA Version: 11.4    |
|-------------------------------+----------------------+----------------------+
|   0  GeForce GT 710      Off  | 00000000:01:00.0 Off |                  N/A|
+-----------------------------------------------------------------------------+
```

GPU visible inside a container. ✅

---

## The Complete Setup Playbook

Here's the entire process condensed — use this as a reference.

```bash
# ── 1. Diagnose ──────────────────────────────────────────────
lspci | grep -i nvidia          # confirm GPU visible
lsmod | grep nvidia             # confirm kernel module loaded
nvidia-smi                      # confirm driver communication

# ── 2. Identify correct driver ───────────────────────────────
ubuntu-drivers devices          # read recommended driver

# ── 3. Install driver ────────────────────────────────────────
sudo apt install nvidia-driver-470   # use YOUR recommended version
sudo reboot
nvidia-smi                      # verify: driver version + CUDA version

# ── 4. Install NVIDIA Container Toolkit ──────────────────────
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# ── 5. Verify GPU passthrough ────────────────────────────────
docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu22.04 nvidia-smi
```

---

## Final State: Layer Verification

| Layer | Component | Status |
|---|---|---|
| Hardware | NVIDIA GT 710 | ✅ Detected |
| Kernel | GPU device nodes | ✅ Exposed |
| Driver | nvidia-driver-470 | ✅ Running |
| CUDA | Version 11.4 | ✅ Available |
| Container Runtime | NVIDIA CTK | ✅ Configured |
| Docker | GPU passthrough | ✅ Verified |

---

## The Hardware Reality Check

I want to be transparent about something that matters for any AI infrastructure engineer doing this on a budget:

```
GPU:       GT 710 (Kepler architecture, 2014)
VRAM:      ~2GB
Max CUDA:  11.4
Modern PyTorch GPU builds: ❌ (require CUDA 12+)
Heavy AI training: ❌ (VRAM too limited)
Infrastructure validation: ✅ Fully capable
```

The GT 710 cannot run a modern LLM. But that's not what this work is about.

What it *can* validate — and what I did validate — is the entire infrastructure stack: driver installation, container runtime configuration, GPU device passthrough, repository management, kernel module loading. Those skills transfer directly to a cluster with A100s.

An engineer who can debug a broken driver stack on a GT 710 can operate a production GPU cluster. The discipline is identical. The stakes are higher.

---

## Dev Environment Discipline: Two More Installs

After the GPU stack was stable, I added two tools that belong in every ML infrastructure environment:

**`tmux`** — persistent terminal sessions that survive disconnections
```bash
sudo apt install tmux
```
When you're running a training job that takes 4 hours, your SSH session cannot die. `tmux` keeps sessions alive server-side.

**`pre-commit`** — automated code hygiene before every commit
```bash
pip install pre-commit
pre-commit install
```
Every commit runs linters, formatters, and checks automatically. Infrastructure code that skips this degrades fast.

---

## What's Next

This GPU container stack is now the foundation for:

- **GPU Experiment Lab** — running MLflow-tracked PyTorch training jobs inside Docker containers
- **Production ML API** — containerized FastAPI inference with GPU access
- **Observability Stack** — monitoring GPU utilization, memory pressure, and inference latency with Prometheus + Grafana

The infrastructure is ready. Time to build on top of it.

---

*Building AI infrastructure from Nairobi. Follow along on [GitHub](https://github.com/Tremblant).*
