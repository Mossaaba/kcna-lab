# k8s — Local Learning Repo

## Purpose

Hands-on Kubernetes learning for local development on M4 Mac. Dan knows the concepts — this repo is about building reps.

## Learning Rules

- Dan writes manifests, not just reads them
- When something breaks, Dan diagnoses first — paste the error, explain what you think it means, then get confirmation or redirection
- After each phase, Dan should be able to explain it in plain language
- Annotate configs with comments on non-obvious lines — the why matters as much as the what

## AWS Translation Layer

Dan's background is ECS/AWS. Always map new K8s concepts to their AWS equivalents:
- Pod → ECS Task
- Deployment → ECS Service
- Namespace → loose equivalent of ECS Cluster or AWS account isolation
- ConfigMap → Parameter Store
- Secret → Secrets Manager (local/plaintext for now)
- Ingress → ALB
- ServiceAccount + RBAC → IAM Role + least-privilege (Dan knows this well — apply same instincts)

## Workflow

- Follow the global research → plan → annotate → implement protocol
- One phase at a time. Deliverable must work before moving on.
- `plan.md` tracks the full learning plan and current phase progress

## Stack

- Tool: kind (Kubernetes in Docker)
- Platform: MacBook M4 Pro
- Docker must be running before any cluster operations
