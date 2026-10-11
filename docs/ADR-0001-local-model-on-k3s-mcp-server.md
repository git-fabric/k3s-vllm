# ADR-0001: Serve k3s-Ops with vLLM and use k3s-mcp-server as its tool layer

**Status:** Proposed
**Date:** 2026-10-11
**Deciders:** ry-ops

## Context

k3s-vllm is one Ollama `Modelfile`: `qwen2.5:14b` with a ~25 KB system prompt that teaches the model the k3s / Kubernetes API and has it emit tool calls as fenced JSON. The repo's name promises vLLM, but nothing in it runs on vLLM yet.

A review of the current state found:

- **The prompt doesn't fit its context window.** The system prompt is roughly 6–7k tokens against `num_ctx 8192`, leaving ~1.5k tokens for the conversation and tool results. One `GET /api/v1/pods` on a real cluster overflows it.
- **Factual errors in the prompt.**
  - The k3s admin kubeconfig authenticates with a client certificate, not a bearer token, so `grep 'token:' /etc/rancher/k3s/k3s.yaml` returns nothing.
  - ServiceLB pods are prefixed `svclb-`, not `svc-`.
  - k3s renews certificates at startup only when they are within 90 days of expiry, not on every start.
  - Spegel is embedded but off unless k3s runs with `--embedded-registry`.
- **The prompt contradicts itself.**
  - It requires `k8s_apply dry_run: true` before any write, yet its restart and scale workflows send PATCH/PUT through `k8s_request` with no dry run.
  - It says credentials never enter model context, then shows how to read them from `k3s.yaml` and the server token file.
  - It warns never to give automation cluster-admin, while its preferred auth is the admin credential.
  - Every example sets `tls_verify: false`, so the model will copy it everywhere.
- **The prompt reimplements a tool layer in prose.** About half of it is API reference and a home-grown JSON tool-call format. The model makes raw API calls with whatever credential it is given. [k3s-mcp-server](https://github.com/ry-ops/k3s-mcp-server) already provides typed k3s tools whose reach is limited by the kubeconfig's RBAC, and it never reads Secrets.
- **Personal context is in a public prompt.** References to AIANA event ingestion, Redis, alert PRs and "your stackforge StatefulSets" are specific to one homelab.
- **The lab cluster can't host the model.** Its nodes have 2 CPUs and 2–3 GB RAM each, with no GPU. vLLM effectively needs a GPU, and a 14B model at 4-bit needs ~10 GB of VRAM.

The goal is for k3s-vllm to stand on its own: answer routine k3s questions and run routine operations without Claude, and escalate to Claude only when needed (see [fabric-sdk](https://github.com/git-fabric/sdk)).

## Decision

k3s-vllm becomes **a locally served model plus agent that uses k3s-mcp-server for every cluster action**. It does not replace k3s-mcp-server and does not keep its own route into the cluster.

1. **Serving:** vLLM, OpenAI-compatible API, with native tool calling (`--enable-auto-tool-choice --tool-call-parser hermes`). Ollama stays supported for laptop development; the agent picks the backend from `.env` only.
2. **Tools:** the agent connects to k3s-mcp-server over MCP. The prompt's API reference, JSON tool-call format and `{{K3S_TOKEN}}` handling are removed. Credentials and RBAC live in the MCP server's kubeconfig.
3. **Prompt:** cut to operating guidance — k3s-specific knowledge that tools don't convey (bundled components, HelmChartConfig over editing manifests, local-path node affinity), the reasoning checklist and the output formats. Target under 2k tokens. Fix the errors listed above and remove the personal references.
4. **Model:** move from `qwen2.5:14b` to the Qwen3 family. Size (4B, 8B or 14B) is chosen by the eval set, not up front.
5. **Hardware:** a GPU passed through to a Proxmox VM, joined to the cluster as a dedicated, tainted worker for vLLM. Until that exists, vLLM runs outside the cluster and the agent points at it by URL.

## Options Considered

### Option A: Keep the standalone Modelfile, fix the prompt

| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Cost | Free on existing hardware |
| Scalability | Ollama only; single user |
| Team familiarity | High — it exists today |

**Pros:** Smallest change. No new dependencies.
**Cons:** Still a prompt, not a project. The model still makes raw API calls with an admin-level credential. The prompt stays large because it has to teach the API. No path to vLLM's native tool calling.

### Option B: Replace k3s-mcp-server with k3s-vllm

| Dimension | Assessment |
|-----------|------------|
| Complexity | High |
| Cost | GPU required for every user |
| Scalability | Tied to model serving capacity |
| Team familiarity | Low — would rebuild what the MCP server already does |

**Pros:** One project instead of two.
**Cons:** Throws away the RBAC boundary and Secret handling that k3s-mcp-server already enforces. Every Claude Desktop / Claude Code user of the MCP server would then need a GPU. Mixes the model and the tool layer, which change for different reasons.

### Option C: Local model + agent on top of k3s-mcp-server (chosen)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium |
| Cost | Needs a GPU for vLLM; small models may run on modest hardware |
| Scalability | vLLM batches concurrent requests; MCP server scales separately |
| Team familiarity | High — both pieces already exist |

**Pros:** Safety comes from RBAC, not prompt wording. The prompt shrinks enough to fix the context-window overflow. Native tool calling. The same MCP server serves Claude and the local model. Each project can be released on its own schedule.
**Cons:** k3s-vllm gains a runtime dependency on k3s-mcp-server. A GPU is needed for the in-cluster story.

## Trade-off Analysis

Option A is cheapest but leaves the core problem: the model holds a powerful credential and is kept in line only by prompt text. Option B centralises everything in the component that is hardest to run (the model) and removes the boundary that makes the current setup safe. Option C keeps each piece doing one job. The dependency it adds is on a project under the same owner, and MCP is a stable interface, so the coupling is low.

"Standing on its own" is defined as: runs with no Claude involvement, passes the eval set at an agreed threshold, and deploys with one command. Option C is the only one that reaches all three without weakening safety.

## Consequences

- **Easier:** swapping models or backends (`.env` only); auditing what the model can do (read the kubeconfig's RBAC); keeping the prompt correct, since it no longer documents the API.
- **Harder:** local development needs both the MCP server and a model backend running; the in-cluster deployment needs GPU hardware.
- **Revisit:** model size after the first eval run; whether a LoRA fine-tune on recorded tool-call traces beats prompting; whether k3s-mcp-server needs tools this agent relies on that it doesn't have yet.

## Action Items

1. [ ] Rename the README's framing from "an Ollama model" to "a local model and agent for k3s, using k3s-mcp-server".
2. [ ] Fix the factual errors and contradictions listed in Context; remove personal references.
3. [ ] Remove the API reference and JSON tool-call format from the prompt; bring it under 2k tokens.
4. [ ] Add an agent that connects to k3s-mcp-server over MCP and to an OpenAI-compatible endpoint from `.env`.
5. [ ] Build an eval set: 30–50 common questions against the lab cluster, each with expected tool calls and answer checks. Record the pass rate per model.
6. [ ] Run the eval across Qwen3 sizes on Ollama; pick the smallest model that meets the threshold.
7. [ ] Stand up vLLM with the hermes tool-call parser — outside the cluster first, then on a GPU-passthrough Proxmox worker.
8. [ ] Package vLLM, the agent and k3s-mcp-server as one Helm chart or manifest set with resource limits and a least-privilege kubeconfig.
9. [ ] Document the escalation path to Claude through fabric-sdk.
