---
layout: post
title: "How I Architected My AI Workstation Storage So Docker Never Crashes My OS Again"
date: 2026-03-04
tags: [docker, infrastructure, storage, mlops, devops, gpu]
excerpt: "An 8GB PyTorch image sitting on your system disk is a disaster waiting to happen. Here's how I separated OS, applications, and Docker storage across three disks — and why every AI infrastructure engineer should do this before their first training run."
---

Here's a failure mode that kills AI workstations silently:

You pull a CUDA base image. Then a PyTorch image. Then your custom model container. Three pulls later, your system disk is at 95% capacity — and the next `apt update` or Docker build fails with a cryptic error that takes an hour to trace back to `/` being full.

I've seen this happen. I decided not to let it happen to me.

This post documents how I redesigned my workstation's storage architecture before running any serious AI workloads — separating the OS, applications, and Docker data across three dedicated disks. It's a pattern that belongs in every AI infrastructure setup, from a personal workstation to a production training server.

---

## The Problem: Everything on One Disk

The default Docker setup puts everything in `/var/lib/docker` — which sits on your system disk. For general software development, that's fine. For AI workloads, it's a structural problem.

<div style="margin: 28px 0;">
  <iframe 
    src="/blackAI/assets/diagrams/storage-exploded-view.html" 
    width="100%" 
    height="580" 
    frameborder="0" 
    scrolling="no"
    style="border: 1px solid #1a2a3a; border-radius: 3px; display: block;">
  </iframe>
  <p style="font-size: 0.72rem; color: #4a6070; margin-top: 8px; font-family: monospace;">
    ↑ Interactive — click "Simulate Crash" to see failure isolation in action
  </p>
</div>

When that disk fills, your OS stops functioning. Logs stop writing. Package managers fail. Running containers crash. It's not a graceful degradation — it's a hard stop.

---

## The Target Architecture

Three disks. Three responsibilities. Clean separation.

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   sdb → /        │  │   sda → /apps    │  │   sdc → /data    │
│                  │  │                  │  │                  │
│  OS              │  │  Project repos   │  │  /data/docker    │
│  System services │  │  Virtual envs    │  │    images        │
│  Docker engine   │  │  ML code         │  │    layers        │
│  (binary only)   │  │  Experiments     │  │    volumes       │
│                  │  │                  │  │  Future:         │
│  Protected.      │  │  Portable.       │  │    model weights │
│  Never fills.    │  │  Replaceable.    │  │    training data │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

The OS disk only holds the OS. Docker images and layers — which can be gigabytes each — live on the data disk. Projects and code live on the apps disk. Each concern is isolated.

---

## Phase 1 — Verify the Disk Layout

Before touching anything, confirm reality matches intent.

```bash
lsblk
```

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 931.5G  0 disk /apps
sdb      8:16   0 119.2G  0 disk /
sdc      8:32   0 931.5G  0 disk /data
```

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb        119G   28G   85G  25% /
/dev/sda        932G  1.2G  931G   1% /apps
/dev/sdc        932G   12G  920G   2% /data
```

Architecture confirmed. Now move Docker off the system disk before those numbers change.

---

## Phase 2 — Stop Docker Completely

Most guides say `sudo systemctl stop docker`. That's not enough.

```bash
sudo systemctl stop docker
sudo systemctl stop docker.socket
```

**Why both?** Docker uses socket activation — `docker.socket` listens for connections and auto-starts `docker.service` when needed. If you only stop the service, the socket restarts it the moment anything touches `/var/run/docker.sock`. You need both stopped before moving data.

```bash
# Verify nothing is running
sudo systemctl status docker
sudo systemctl status docker.socket
```

Both should show `inactive (dead)` before proceeding.

---

## Phase 3 — Move the Docker Data Directory

```bash
sudo mv /var/lib/docker /data/docker
```

This moves everything: image layers, container filesystems, volumes, build cache, plugin data. On a fast disk, this is quick. On spinning rust with 8GB+ of images, give it a few minutes.

```bash
# Verify the move completed
ls -la /data/docker/
du -sh /data/docker/
```

---

## Phase 4 — Update Docker Configuration

This is where most people make a mistake. Docker's config file is JSON — and JSON does not allow two root objects. If you have an existing `daemon.json` (for NVIDIA runtime, for example), you must **merge** the new setting in, not append a second block.

**The wrong way** (breaks Docker completely):
```json
{
  "data-root": "/data/docker"
}
{
  "runtimes": {
    "nvidia": { ... }
  }
}
```

**The correct way** (one root object, all settings merged):
```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "data-root": "/data/docker",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "args": []
    }
  }
}
```

This single file now handles both: storage location and NVIDIA GPU runtime. One source of truth.

> **Config principle for MLOps:** Every system you manage will have configuration files that multiple concerns share. Always treat config as code — merge carefully, validate before applying, never duplicate root structures.

---

## Phase 5 — Restart and Verify

```bash
sudo systemctl reset-failed docker    # clear any previous failure state
sudo systemctl start docker
sudo systemctl status docker          # should show: active (running)
```

The critical verification:

```bash
docker info | grep "Docker Root Dir"
```

```
Docker Root Dir: /data/docker
```

Then confirm images survived the move:

```bash
docker images
```

```
REPOSITORY    TAG                        IMAGE ID       CREATED        SIZE
pytorch-api   latest                     a3f8c291b2d1   2 days ago     8.34GB
nvidia/cuda   12.1.1-base-ubuntu22.04    b7d4e8f12c90   3 weeks ago    3.2GB
hello-world   latest                     9c7a54a9a43c   8 months ago   13.3kB
```

All images intact. The move was transparent to Docker — it just reads from the new path.

---

## Phase 6 — Understanding What's Actually Storing Your Data

While verifying, I noticed something worth understanding:

```bash
docker info | grep Storage
```

```
Storage Driver: io.containerd.snapshotter.v1
```

Not `overlay2`. This is the modern containerd snapshotter — used by Docker 24+ on Ubuntu 24.04. If you run:

```bash
find /data/docker -name overlay2
```

You'll find nothing. That's expected. The storage layout under containerd snapshotter looks different from the classic overlay2 filesystem, but functions identically from an operational perspective.

The takeaway: when debugging Docker storage, always check `docker info` first. Don't assume the storage driver.

---

## The Complete Playbook

```bash
# ── 1. Verify disk layout ────────────────────────────────────
lsblk
df -h

# ── 2. Stop Docker completely ────────────────────────────────
sudo systemctl stop docker
sudo systemctl stop docker.socket

# ── 3. Move Docker data directory ────────────────────────────
sudo mv /var/lib/docker /data/docker

# ── 4. Update daemon.json (merge, don't append) ──────────────
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "data-root": "/data/docker",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "args": []
    }
  }
}
EOF

# ── 5. Restart and verify ────────────────────────────────────
sudo systemctl reset-failed docker
sudo systemctl start docker
docker info | grep "Docker Root Dir"
docker images
```

---

## Final Architecture: Verified

| Disk | Mount | Purpose | Docker? |
|---|---|---|---|
| sdb | `/` | OS + system services | Engine binary only |
| sda | `/apps` | Projects, code, envs | No |
| sdc | `/data` | Images, layers, volumes | `/data/docker` ✅ |

---

## Why This Matters at Production Scale

The same pattern applies at every scale — from a workstation to a GPU cluster node:

```
Production Node Storage Architecture
┌────────────────────────────────────────────────────────┐
│  Boot Volume (50GB)                                    │
│  OS, system services, monitoring agents                │
├────────────────────────────────────────────────────────┤
│  Application Volume (200GB)                            │
│  Container runtime, code, configs                      │
├────────────────────────────────────────────────────────┤
│  Data Volume (2TB+, fast NVMe)                         │
│  Container images, model weights, training datasets    │
│  Logs, experiment artifacts, database files            │
└────────────────────────────────────────────────────────┘
```

The principle scales. A cloud GPU instance (AWS `p3.2xlarge`, GCP `a2-highgpu`) always separates boot volume from data volume for exactly this reason. Learning that discipline on a workstation means it's already natural when you're configuring production nodes.

Specific production benefits this architecture enables:

- **Disk-specific backups** — snapshot `/data` for model artifacts without touching the OS
- **Independent scaling** — expand the data volume without rebuilding the system
- **Failure isolation** — a full data disk doesn't take down the OS
- **Clean wipes** — rebuild the OS disk without losing training data or images

---

## What's Next

With storage architecture clean, the next step is populating `/data/docker` with purpose-built images:

- **Production ML API** — FastAPI + model weights, containerized, stored on `/data`
- **GPU Experiment Lab** — MLflow tracking server + DVC pipeline, artifacts on `/data`
- **Observability Stack** — Prometheus + Grafana, metrics stored on `/data`

The foundation is solid. Now we build.

---

*Building AI infrastructure from Nairobi. Follow along on [GitHub](https://github.com/Tremblant).*
