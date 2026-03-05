---
layout: post
title: "My AI Infrastructure Journey"
date: 2026-03-03
tags: [mlops, infrastructure, career, journey]
excerpt: "Why I'm building AI infrastructure from scratch — and what that means coming from a telecom and cloud background in Nairobi."
---

Every engineer has a moment when the work stops being a job and starts being a mission.

Mine came when I realized that the gap killing AI adoption in African startups isn't a shortage of models — it's a shortage of **infrastructure**. The systems that make models reliable. The pipelines that keep them running at 3am. The observability that tells you when something breaks before your users do.

That's what I'm building toward. And this blog is where I document the journey.

---

## Why AI Infrastructure?

I came from telecommunications. Routing protocols, VPN tunnels, SD-WAN deployments, firewall architectures. The work demanded a very specific kind of thinking: **systems thinking**.

Every packet has a path. Every path has a failure mode. Every failure mode has a mitigation. You design for the steady state and you design for when the steady state fails.

When I first encountered MLOps seriously — the idea of treating model deployment with the same rigor as network infrastructure — something clicked. The vocabulary is different but the discipline is identical:

- A model endpoint is a service with an SLA
- Inference latency is a packet that needs to arrive on time
- Model drift is link degradation — silent, insidious, deadly if unmonitored
- A retraining pipeline is a failover mechanism

I didn't need to un-learn telecom to learn MLOps. I needed to translate.

---

## What I'm Building

I've laid out five active workstreams for myself:

**1. Production ML API**
FastAPI + Docker + GitHub Actions CI/CD. The goal is a model endpoint that is version-controlled, containerized, tested, and auto-deployed on push. Not a tutorial project — a system I'd stake my name on.

**2. GPU Experiment Lab**
MLflow + DVC for tracking experiments and versioning data. When I run an experiment, every artifact, every metric, every config should be reproducible by anyone with access to the repo.

**3. Kubernetes ML Scaling**
Deploying inference workloads to Kubernetes. Helm charts, horizontal pod autoscalers, resource quotas for GPU nodes. The kind of setup that handles 10x traffic spikes without a human in the loop.

**4. AI Observability Stack**
Prometheus scraping custom ML metrics. Grafana dashboards showing model latency distributions, throughput, error rates. Alerts that fire before the customer calls.

**5. AI Infrastructure Capstone**
The capstone ties it all together — a reference architecture for a complete MLOps platform that I can point to and say: *this is how I think*.

---

## My North Star

There's a version of AI in Africa that I want to help build — one where startups can ship reliable ML products without needing a staff of 20 infrastructure engineers from San Francisco.

That means building open, documented, reproducible patterns. Writing them down. Sharing them publicly.

This blog is part of that commitment.

---

## Next Post

I'll walk through setting up a FastAPI ML API from scratch — Dockerfile, CI pipeline, health endpoints, and all the boring-but-critical infrastructure decisions that tutorials always skip.

Stay tuned.

---

*— Tiberius, Nairobi*
