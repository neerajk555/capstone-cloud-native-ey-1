# Capstone: Local Cloud-Native Pipeline (Docker, Kubernetes, Helm, Observability, GitOps)

## Goal

Build a small but genuinely multi-service application, then take it through the full path real teams
use to run software in production - containerize it, secure it with Kubernetes-native primitives,
package it with Helm, observe it with Prometheus and Grafana, and deploy it continuously via GitOps with
ArgoCD - **entirely on your own machine**, no cloud account, no cost.

This capstone deliberately covers **local, tool-equivalent versions of most of this course's modules**,
skipping only the two areas that inherently require a cloud provider (AWS-specific services, and
Agentic AI / Amazon Bedrock).

## What You'll Build

A small **asynchronous task-processing system**: an `api` service accepts work over HTTP and hands it
off to a `worker` service via Redis, instead of processing it inline - a real, if small-scale, example of
the synchronous-vs-asynchronous communication patterns and database-per-service thinking this course
covers conceptually.

```
                     ┌─────────────────────────────────────────────┐
                     │              Kubernetes (minikube)            │
                     │                                                │
   git push  ───────▶│   ArgoCD  ──watches──▶  Helm Release           │
                     │                              │                 │
                     │        ┌─────────────────────┼──────────┐      │
                     │        ▼                      ▼          ▼      │
                     │      api (x2)  ──async──▶  redis  ◀──  worker  │
                     │        │                                        │
                     │        └──scraped by──▶ Prometheus ──▶ Grafana │
                     │                                                │
                     │   Namespace + RBAC + NetworkPolicy + Secret    │
                     └─────────────────────────────────────────────┘
```

## Time Budget (~3 hours of active building, plus a self-assessment step)

| Task | File | What | Time |
|---|---|---|---|
| 1 | `01-environment-setup.md` | Verify tools, start minikube | 10 min |
| 2 | `02-build-microservices.md` | Build api + worker + Redis, test locally with Docker Compose | 35 min |
| 3 | `03-kubernetes-security-primitives.md` | Namespace, Secret, RBAC, NetworkPolicy - raw YAML | 30 min |
| 4 | `04-package-with-helm.md` | Package everything as one Helm chart, deploy manually | 35 min |
| 5 | `05-observability-prometheus-grafana.md` | Expose metrics, install Prometheus + Grafana, build a dashboard | 30 min |
| 6 | `06-gitops-with-argocd.md` | Push to GitHub, install ArgoCD, hand off deployment to it | 25 min |
| 7 | `07-resilience-and-scaling.md` | Prove GitOps + self-healing, autoscaling | 20 min |
| **Tasks 1-7 total** | | | **185 min (~3 hours)** |
| 8 | `08-evaluation-rubric.md` | Self-assess against the grading rubric, done AFTER building | 15 min |

Task 8 is deliberately not counted in the "3 hours" - it's a wrap-up self-assessment against a finished,
running system, not part of the build itself.

## Which Course Modules This Maps To (Local Equivalents)

| Module | Topic | Covered Here Via |
|---|---|---|
| 1 | Cloud Native fundamentals | The whole capstone's structure demonstrates this end to end |
| 2 | Microservices, sync vs async communication | `api` + `worker` communicating asynchronously through Redis, not a direct call |
| 3 | Docker, Dockerfiles, Compose | Task 2 |
| 4 | Kubernetes, ConfigMaps/Secrets, storage, namespaces, HPA, Helm | Tasks 3, 4, 7 |
| 5 | Design patterns (circuit breaker, async processing) | Tasks 2, 7 |
| 8 | CI/CD, GitOps with ArgoCD | Tasks 6, 7 |
| 9 | Observability: metrics, dashboards, health probes | Task 5 |
| 10 | Security: RBAC, Network Policies, Secrets | Task 3 |
| 11 | Data management, caching | Redis as both queue and cache throughout |

**Not covered** (need a cloud provider or Agentic AI, out of scope for this local capstone): Modules 6-7
(API Gateway, App Mesh - AWS-specific managed services), Module 11's RDS/DynamoDB/SQS/SNS (used Redis
locally instead), Modules 12-14 (Agentic AI / Bedrock).

## Prerequisites

- Ubuntu VM with Docker, `kubectl`, `minikube`, `helm`, and `git` already installed
- VS Code installed
- A free GitHub account, with Git configured (`git config --global user.name/user.email`)
- Node.js 18+ (for running things locally outside Docker during testing)

## Files in This Project

```
00-README.md                              This file
01-environment-setup.md
02-build-microservices.md
03-kubernetes-security-primitives.md
04-package-with-helm.md
05-observability-prometheus-grafana.md
06-gitops-with-argocd.md
07-resilience-and-scaling.md
08-evaluation-rubric.md
solutions/                                 Reference solutions - see each task file for what to check against
  02-build-microservices/
  03-kubernetes-security-primitives/
  04-package-with-helm/
  05-observability/
  06-gitops-with-argocd/
  07-resilience-and-scaling/
```

**Do not open the `solutions/` folder until you've attempted a task yourself** - each task file tells
you exactly which solution files correspond to it, once you're ready to check your work.

Start with `01-environment-setup.md`.
