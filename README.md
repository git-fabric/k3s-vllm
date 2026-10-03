<p align="center"><img src="docs/banner.svg" alt="k3s-vllm: k3s-Ops: Ollama model specialized for k3s / Kubernetes API" width="100%"></p>

# k3s-vllm

**k3s-Ops**: an Ollama model specialized for the **k3s / Kubernetes API**, part of git-fabric's **fabric-llm** layer.

It answers questions about its domain locally, so the fabric only escalates to Claude when it has to. See [fabric-sdk](https://github.com/git-fabric/sdk) for how requests are routed.

| | |
|---|---|
| Base model | `qwen2.5:14b` |
| Context window | 8,192 tokens |
| Temperature | 0.15 |

## Use it

```bash
ollama create k3s-ops -f Modelfile
ollama run k3s-ops
```

## What's inside

A single [`Modelfile`](Modelfile): the base model, its sampling parameters, and a system prompt that teaches the model the k3s / Kubernetes API.

<!-- org-footer -->
---

<p align="center"><sub>Part of <a href="https://github.com/git-fabric">git-fabric</a> · composable fabric apps for Git-native infrastructure · built by <a href="https://github.com/ry-ops">ry-ops</a></sub></p>
