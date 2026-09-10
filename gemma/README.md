# Gemma SLM on AKS (GPU, ClusterIP)

Hosts Google’s small language model [`google/gemma-2-2b-it`](https://huggingface.co/google/gemma-2-2b-it) with [vLLM](https://docs.vllm.ai/) (OpenAI-compatible server) on a GPU node. The API is reachable inside the cluster via ClusterIP, and can be exposed through an Istio VirtualService route.

This is a **generative SLM** (chat / completion), not an embedding model. Sibling services in this repo:

- [`medembed/`](../medembed/) — medical embeddings (TEI, CPU)
- [`bge-small/`](../bge-small/) — general embeddings (TEI, CPU)
- [`vexyl-stt/`](../vexyl-stt/) — speech-to-text

## Prerequisites

- Hugging Face access **approved** for the gated model: [google/gemma-2-2b-it](https://huggingface.co/google/gemma-2-2b-it) (accept the license on the model page)
- HF token with read access
- AKS cluster with a **GPU node pool** (1× NVIDIA GPU with ~12–16Gi+ VRAM is enough for 2B; e.g. T4 / A10 / L4 class)
- NVIDIA device plugin installed on the GPU pool
- `kubectl` configured for your AKS cluster
- Egress from the cluster to Hugging Face on first start (model download)

## Create the HF token secret

```bash
kubectl apply -f gemma/namespace.yaml
kubectl -n gemma create secret generic hf-token \
  --from-literal=token=hf_your_token_here \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Deploy

```bash
kubectl apply -f gemma/namespace.yaml -f gemma/deployment.yaml -f gemma/service.yaml
kubectl -n gemma rollout status deploy/gemma
```

First startup can take several minutes while vLLM downloads weights and loads the model onto the GPU. If the pod stays `Pending`, check GPU scheduling (`nvidia.com/gpu`), node pool capacity, and any GPU taints (uncomment `nodeSelector` / `tolerations` in [`deployment.yaml`](deployment.yaml) to match your pool).

Do **not** apply `virtualservice-route.yaml` directly — it is a route snippet for your existing VirtualService.

## In-cluster endpoint

```
http://gemma.gemma.svc.cluster.local
```

## Istio VirtualService route

Paste the HTTP route from [`virtualservice-route.yaml`](virtualservice-route.yaml) into your existing VirtualService `http:` list:

```yaml
- match:
    - uri:
        prefix: /gemma/
  rewrite:
    uri: /
  timeout: 300s
  route:
    - destination:
        host: gemma.gemma.svc.cluster.local
        port:
          number: 80
```

Through the mesh gateway (keep the trailing slash on the prefix path):

- `POST /gemma/v1/chat/completions`
- `POST /gemma/v1/completions`
- `GET /gemma/v1/models`
- `GET /gemma/health`

Istio rewrites `/gemma/` → `/`, so vLLM receives the native OpenAI-style paths.

## Smoke test

Health / models:

```bash
kubectl -n gemma run curl --rm -it --image=curlimages/curl -- \
  curl -s http://gemma.gemma.svc.cluster.local/health

kubectl -n gemma run curl --rm -it --image=curlimages/curl -- \
  curl -s http://gemma.gemma.svc.cluster.local/v1/models
```

Chat completion:

```bash
kubectl -n gemma run curl --rm -it --image=curlimages/curl -- \
  curl -s http://gemma.gemma.svc.cluster.local/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"google/gemma-2-2b-it","messages":[{"role":"user","content":"Explain what an SLM is in one sentence."}],"max_tokens":64}'
```

## Client usage

Point any OpenAI-compatible client at the in-cluster or gateway base URL:

```text
base_url = "http://gemma.gemma.svc.cluster.local/v1"
# or via mesh: "https://<gateway-host>/gemma/v1"
model    = "google/gemma-2-2b-it"
```

## Scaling up the model

Default is **Gemma 2 2B Instruct** (fits one modest GPU). To try a larger instruct checkpoint, edit `--model` in [`deployment.yaml`](deployment.yaml) and raise GPU / memory accordingly, for example:

| Model | Typical GPU need |
|---|---|
| `google/gemma-2-2b-it` (default) | 1× GPU, ~12–16Gi VRAM |
| `google/gemma-2-9b-it` | 1× larger GPU (often 24Gi+) |
| `google/gemma-2-27b-it` | Multi-GPU / tensor parallel (`--tensor-parallel-size`) |

Accept the license for the larger model on Hugging Face before deploying.

## Notes

- Model cache uses `emptyDir` at `/data`; weights re-download if the pod is recreated. Switch to a PVC if cold starts matter.
- `/dev/shm` is a memory-backed `emptyDir` — helpful for vLLM worker shared memory.
- If the pod CrashLoops on first boot, check: HF license accepted, `hf-token` secret present, egress to `huggingface.co`, and GPU driver / device plugin health.
- Prefer GPU for usable latency. CPU-only Gemma hosting is usually too slow for interactive chat.
