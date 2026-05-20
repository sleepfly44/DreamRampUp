# Mission

## Purpose

DreamRampUp is a personal learning lab for building and operating a production-like Kubernetes environment on a local Mac. The goal is hands-on fluency with the tools, patterns, and failure modes that real platform teams deal with — not to run production workloads.

## What success looks like

- A multi-node RKE2 cluster running locally via Multipass VMs
- Secrets managed end-to-end through HashiCorp Vault and External Secrets Operator — no secrets in manifests or environment files
- A working three-tier app (frontend → backend → Postgres) deployed via Helm and Kubernetes manifests
- Zero-trust networking enforced by Istio: strict mTLS mesh-wide, default deny-all, explicit allow policies per service pair
- Every phase completed with enough understanding to explain and reproduce it without the README

## What this is not

- A production system or production blueprint
- A CI/CD pipeline or GitOps workflow (out of scope for now)
- A multi-tenant or multi-cluster environment

## Audience

Solo learner (the author). The README and specs exist to reinforce understanding and make the environment reproducible after a break.
