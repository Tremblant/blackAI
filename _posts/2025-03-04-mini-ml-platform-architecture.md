---
layout: post
title: "I Built a Mini ML Platform on My Workstation — Here's the Architecture"
date: 2025-03-04
tags: [mlops, kubernetes, docker, infrastructure, architecture, fastapi]
excerpt: "Most AI engineers train models. Fewer think about where models live, how they're served, and what keeps the whole system from collapsing under load. This is how I designed a three-layer ML platform from scratch — on a single workstation — using the same patterns production clusters use."
---

There's a gap between *training a model* and *running an ML platform*.

Training a model is a notebook. An ML platform is a system — with storage strategy, containerized serving, orchestration, and a repeatable workflow that works for the first project and the fiftieth.

I built a minimal version of that platform on my workstation. Not as a toy. As a template — one that mirrors how serious AI infrastructure is designed at production scale, and that I can evolve as each layer matures.

This post documents the architecture, the reasoning behind each decision, and the full workflow I'll use for every AI project going forward.

---

## The Core Principle: Separation of Concerns

The worst thing you can do in AI infrastructure is let everything live in one place. Models mixed with code. Logs mixed with datasets. Docker images on the OS disk. It feels fine on day one and falls apart on day thirty.

My workstation has three disks. Each one has exactly one responsibility.

```
╔══════════════════════════════════════════════════════════════════╗
║                  AI WORKSTATION ARCHITECTURE                    ║
╠══════════════════╦═══════════════════╦════════════════════════════╣
║   sdb  →  /      ║   sda  →  /apps   ║      sdc  →  /data         ║
║                  ║                   ║                            ║
║  SYSTEM LAYER    ║  APPLICATION      ║  DATA LAYER                ║
║                  ║  LAYER            ║                            ║
║  • OS            ║  • Source code    ║  • Trained models          ║
║  • Docker Engine ║  • Dockerfiles    ║  • Datasets                ║
║  • Minikube      ║  • K8s manifests  ║  • Logs                    ║
║  • kubectl       ║  • ML notebooks   ║  • Docker images           ║
║                  ║  • Experiments    ║  • Persistent volumes      ║
║                  ║                   ║                            ║
║  Runs workloads. ║  Defines them.    ║  Stores artifacts.         ║
║  Never fills.    ║  Clean deploys.   ║  Never mixes with code.    ║
╚══════════════════╩═══════════════════╩════════════════════════════╝
```

No layer stores what belongs to another. That discipline is what keeps the platform stable as projects grow.

---

## Layer 1 — The System Layer (`/`)

```
/ (sdb — System Disk)
├── Docker Engine        ← container runtime only
├── Minikube             ← local Kubernetes cluster
├── kubectl              ← cluster control plane client
└── OS services          ← kernel, networking, systemd
```

This disk runs the platform. It never stores anything the platform produces.

**Installed in this layer:**

```bash
# Docker (container runtime)
sudo apt install docker.io

# kubectl (official binary method — not snap)
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start local Kubernetes cluster
minikube start --driver=docker
```

**What Minikube gives us:**

```
minikube start
     │
     ▼
Local Kubernetes Cluster
     │
     ├── API Server      ← receives kubectl commands
     ├── Scheduler       ← decides which node runs each pod
     ├── etcd            ← cluster state storage
     └── kubelet         ← runs containers on the node
```

A single-node cluster that behaves identically to a cloud-managed Kubernetes cluster for development and validation. The same YAML that deploys here deploys to EKS or GKE.

---

## Layer 2 — The Application Layer (`/apps`)

```
/apps/projects/ai-starter/
├── labs/                    ← exploratory notebooks
│   └── exploration.ipynb
├── notebooks/               ← structured experiments
│   └── train.ipynb
├── scripts/
│   └── api/                 ← production API code
│       ├── app.py
│       ├── Dockerfile
│       └── requirements.txt
├── deployment/              ← Kubernetes manifests
│   ├── deployment.yaml
│   └── service.yaml
└── requirements.txt         ← project dependencies
```

This layer defines behaviour. It holds code, configuration, and infrastructure definitions — nothing that changes at runtime.

**The key rule:** models are never saved here. A notebook trains a model and writes it to `/data/models/<project>/`. The application layer only contains the code that *uses* the model, not the model itself.

This separation makes deployments clean: you can rebuild or replace the application layer without touching any trained artifact.

---

## Layer 3 — The Data Layer (`/data`)

```
/data/
├── datasets/
│   └── ai-starter/          ← raw and processed data
├── models/
│   └── ai-starter/
│       └── containerize-pytorch/
│           └── model.pt     ← trained model artifact
├── logs/
│   └── ai-starter/          ← inference logs, training logs
└── docker/                  ← Docker images & layers
```

This is the persistence layer. Everything the system produces lands here.

The analogy to cloud infrastructure is direct:

| Local | Cloud Equivalent |
|---|---|
| `/data/models/` | S3 model bucket / GCS bucket |
| `/data/datasets/` | S3 data bucket / BigQuery |
| `/data/logs/` | CloudWatch / Cloud Logging |
| `/data/docker/` | ECR / Artifact Registry |

Building this habit locally means the mental model for cloud storage is already correct when you get there.

---

## The Full ML Platform: Architecture Diagram

Here is how all three layers interact when a request hits the deployed API:

```
                        ┌─────────────────┐
                        │   User / Client  │
                        └────────┬────────┘
                                 │ HTTP Request
                                 ▼
                   ┌─────────────────────────┐
                   │  Kubernetes Service      │
                   │  (NodePort :30080)       │
                   └────────────┬────────────┘
                                │ routes to pod
                   ┌────────────▼────────────┐
                   │  Kubernetes Deployment   │
                   │  replicas: 1–N pods      │
                   └────────────┬────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
        ┌──────────┐     ┌──────────┐      ┌──────────┐
        │  Pod 1   │     │  Pod 2   │      │  Pod N   │
        │ FastAPI  │     │ FastAPI  │      │ FastAPI  │
        │Container │     │Container │      │Container │
        └────┬─────┘     └────┬─────┘     └────┬─────┘
             │                │                 │
             └────────────────┼─────────────────┘
                              │ volumeMount
                              ▼
                  ┌─────────────────────┐
                  │  /app/models/       │
                  │  (mounted from)     │
                  └──────────┬──────────┘
                             │ hostPath / PVC
                             ▼
                  ┌─────────────────────┐
                  │  /data/models/      │
                  │  ai-starter/        │
                  │  model.pt           │  ← sdc (Data Layer)
                  └─────────────────────┘
```

Every pod reads the same model file. When you scale from 1 to 4 replicas, there's no model duplication — all pods mount the same source. This is exactly how production inference clusters operate.

---

## The Repeatable Project Workflow

Every new AI project follows this sequence. No exceptions.

```
PHASE 1 — INITIALIZE
─────────────────────────────────────────────────────────
/apps/projects/<project-name>/
  notebooks/   scripts/api/   deployment/   requirements.txt

/data/datasets/<project-name>/
/data/models/<project-name>/
/data/logs/<project-name>/

PHASE 2 — DEVELOP
─────────────────────────────────────────────────────────
Work in:  /apps/projects/<project>/notebooks/
Save to:  /data/models/<project>/<version>/model.pt
Rule:     Model artifacts NEVER saved inside /apps

PHASE 3 — BUILD API
─────────────────────────────────────────────────────────
/apps/projects/<project>/scripts/api/app.py

  MODEL_PATH = os.getenv("MODEL_PATH", "/data/models/...")
              ↑
  Environment variable makes the path injectable.
  Same code runs locally and in Kubernetes.

PHASE 4 — CONTAINERIZE
─────────────────────────────────────────────────────────
docker build -t <project>-api:latest \
  -f scripts/api/Dockerfile .
              ↑
  Image contains code only. Model mounts at runtime.

PHASE 5 — DEPLOY
─────────────────────────────────────────────────────────
/apps/projects/<project>/deployment/deployment.yaml

  volumeMounts:
    - name: model-storage
      mountPath: /app/models

  volumes:
    - name: model-storage
      hostPath:
        path: /data/models/<project>

  env:
    - name: MODEL_PATH
      value: "/app/models/model.pt"

PHASE 6 — SCALE
─────────────────────────────────────────────────────────
kubectl scale deployment <name> --replicas=4

  4 pods. 1 model file. Shared storage.
  This is real inference scaling.
```

---

## Mini ML Platform Blueprint

```
┌─────────────────────────────────────────────────────────────────┐
│                     MINI ML PLATFORM                            │
├────────────────┬────────────────────────┬───────────────────────┤
│  DEVELOPMENT   │      SERVING           │     STORAGE           │
│                │                        │                       │
│  Jupyter       │  FastAPI (app.py)      │  /data/models/        │
│  Notebooks     │      ↓                 │  /data/datasets/      │
│      ↓         │  Docker Container      │  /data/logs/          │
│  Train model   │      ↓                 │                       │
│      ↓         │  Kubernetes Pod        │  Mounted into pods    │
│  Save .pt      │      ↓                 │  via hostPath / PVC   │
│  → /data       │  K8s Service           │                       │
│                │  (NodePort)            │  Future:              │
│                │      ↓                 │  PVC → cloud storage  │
│                │  kubectl scale         │  MLflow registry      │
│                │  replicas: 1→N         │  Private ECR          │
├────────────────┴────────────────────────┴───────────────────────┤
│                   INFRASTRUCTURE LAYER                          │
│          Docker Engine  •  Minikube  •  kubectl                 │
│                   sdb (System Disk)                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## What This Platform Currently Proves

| Capability | Implementation | Production Equivalent |
|---|---|---|
| Model artifact storage | `/data/models/` | S3 / GCS model bucket |
| Containerized serving | FastAPI + Docker | ECS / Cloud Run |
| Orchestration | Kubernetes + Minikube | EKS / GKE |
| Model injection | K8s volumeMount | PVC / EFS mount |
| Horizontal scaling | `kubectl scale` | HPA on inference cluster |
| IaC | deployment.yaml + service.yaml | Helm charts / Terraform |
| Disk separation | 3-disk architecture | Boot vol + data vol |

Each row is a transferable skill. The local implementation and the cloud implementation are structurally identical — only the scale changes.

---

## The Evolution Path

This platform is a foundation, not a finish line. Here's what each layer evolves into:

```
NOW                              NEXT                        PRODUCTION
─────────────────────────────────────────────────────────────────────────
hostPath volume       →    PersistentVolumeClaim    →    EFS / S3 mount
manual docker build   →    GitHub Actions CI/CD     →    ECR push on merge
kubectl scale manual  →    HPA (CPU/memory)         →    KEDA (request-based)
no model registry     →    MLflow local             →    MLflow on S3 backend
stdout logs           →    Loki + Promtail          →    CloudWatch / Datadog
no metrics            →    Prometheus scraping      →    Grafana dashboards
```

Every item on the left is already working. Every item in the middle is the next build. Every item on the right is where this is heading.

---

## The Bigger Picture

What I've built is not just a local setup. It's a mental model for how AI infrastructure works:

- **Code and data are separate concerns** — always, at every scale
- **Models are artifacts** — they live in storage, they're mounted into containers, they're never baked into images
- **Kubernetes is the orchestration layer** — whether it's Minikube locally or EKS in production, the YAML is the same
- **Every layer has one responsibility** — the system runs, the application defines, the data persists

Most AI engineers reach the model. Fewer reach the platform. That's the gap I'm building toward closing.

---

*Building AI infrastructure from Nairobi. Follow the full build on [GitHub](https://github.com/Tremblant).*
