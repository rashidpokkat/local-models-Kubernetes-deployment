---
marp: true
title: MedEmbed on AKS
description: How we deployed MedEmbed-base medical embeddings on Azure Kubernetes Service
paginate: true
theme: default
style: |
  section {
    font-family: "IBM Plex Sans", "Segoe UI", sans-serif;
  }
  h1, h2 {
    font-family: "IBM Plex Serif", Georgia, serif;
    color: #0b3d2e;
  }
  code, pre {
    font-family: "IBM Plex Mono", ui-monospace, monospace;
  }
  section.lead h1 {
    font-size: 2.4em;
  }
  .muted { color: #5a6b63; font-size: 0.85em; }
  table { font-size: 0.85em; }
---

<!-- _class: lead -->

# MedEmbed

## How we deployed medical embeddings on AKS

Self-hosted clinical text embeddings with Hugging Face TEI · CPU · ClusterIP + Istio

`abhinand/MedEmbed-base-v0.1`

---

## What we deployed

A medical-domain embedding service for clinical / biomedical text similarity and retrieval.

| Piece | Choice |
|---|---|
| Model | `abhinand/MedEmbed-base-v0.1` |
| Runtime | Hugging Face **Text Embeddings Inference (TEI)** |
| Image | `ghcr.io/huggingface/text-embeddings-inference:cpu-1.9` |
| Compute | **CPU** (no GPU) |
| Platform | **AKS** |
| Exposure | ClusterIP + optional Istio VirtualService |

---

## Why this shape

- **Self-hosted** — keep clinical text inside the cluster
- **TEI** — production embedding server (native + OpenAI-compatible APIs)
- **CPU first** — no GPU node pool required for this model size
- **Same pattern** as sibling services (`bge-small`, `vexyl-stt`)

---

## Architecture

```
Client / mesh gateway
        │
        ▼
 Istio VirtualService
   /medembed/*  ──rewrite──►  /
        │
        ▼
 Service  medembed.medembed.svc.cluster.local:80
        │
        ▼
 Deployment  medembed (TEI container)
        │
        ▼
 emptyDir /data  (HF model cache)
```

In-cluster DNS: `http://medembed.medembed.svc.cluster.local`

---

## Manifests we ship

```
medembed/
├── namespace.yaml              # medembed namespace
├── deployment.yaml            # TEI + probes + resources
├── service.yaml               # ClusterIP :80
├── virtualservice-route.yaml  # Istio route snippet (paste-in)
└── README.md
```

Apply order: **Namespace → Deployment → Service**, then paste the Istio route into the existing VirtualService.

---

## Namespace & Service

**Namespace** isolates the workload:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: medembed
```

**Service** is ClusterIP only — no public LoadBalancer:

```yaml
type: ClusterIP
ports:
  - name: http
    port: 80
    targetPort: http
```

Traffic enters via the mesh gateway, not a dedicated public IP.

---

## Deployment highlights

| Setting | Value |
|---|---|
| Replicas | `1` |
| Model arg | `--model-id abhinand/MedEmbed-base-v0.1` |
| Listen port | `80` |
| Cache path | `/data` (`emptyDir`) |
| Thread envs | `OMP` / `MKL` / `RAYON` = `4` |
| Requests | **2 CPU · 4Gi** |
| Limits | **4 CPU · 8Gi** |

Probes hit `GET /health` with a long **startup** window so first model download can finish.

---

## Deploy steps

```bash
kubectl apply \
  -f medembed/namespace.yaml \
  -f medembed/deployment.yaml \
  -f medembed/service.yaml

kubectl -n medembed rollout status deploy/medembed
```

**First boot** can take several minutes while TEI downloads weights from Hugging Face.

ONNX 404 warnings on CPU are expected — TEI falls back to Candle + safetensors and becomes Ready.

---

## Exposing via Istio

Do **not** `kubectl apply` the route file alone — paste into the existing VirtualService `http:` list:

```yaml
- match:
    - uri:
        prefix: /medembed/
  rewrite:
    uri: /
  route:
    - destination:
        host: medembed.medembed.svc.cluster.local
        port:
          number: 80
```

Istio rewrites `/medembed/` → `/` so TEI sees its native paths.

---

## APIs clients use

Through the mesh (keep the trailing slash on the prefix):

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/medembed/embed` | TEI native embeddings |
| `POST` | `/medembed/v1/embeddings` | OpenAI-compatible |
| `GET` | `/medembed/health` | Readiness / health |

Native body: `{"inputs":"..."}`  
OpenAI-style: `{"input":"...","model":"abhinand/MedEmbed-base-v0.1"}`

---

## Smoke test (in-cluster)

```bash
kubectl -n medembed run curl --rm -it --image=curlimages/curl -- \
  curl -s http://medembed.medembed.svc.cluster.local/embed \
  -H 'Content-Type: application/json' \
  -d '{"inputs":"patient with chest pain and elevated troponin"}'
```

Confirms DNS, Service, and TEI are healthy before relying on the gateway path.

---

## Prerequisites & ops notes

**Need**
- `kubectl` pointed at AKS
- Node capacity for **2–4 CPU** and **4–8Gi**
- Egress to Hugging Face on first start

**Watch**
- Cold starts re-download weights (`emptyDir` is ephemeral)
- Pending / CrashLoop often = egress or capacity
- Prefer a PVC (or bake the model into an image) if restart latency matters

---

## Sibling pattern

Same AKS packaging style across the repo:

| Service | Model / stack | Role |
|---|---|---|
| **medembed** | MedEmbed-base + TEI | Medical embeddings |
| **bge-small** | BGE-small-en + TEI | General embeddings |
| **vexyl-stt** | IndicConformer + VEXYL | Speech-to-text |

Reusable recipe: Namespace + Deployment + ClusterIP + Istio prefix rewrite.

---

## Takeaways

1. **MedEmbed runs as TEI on AKS CPU** — no GPU required for this footprint
2. **ClusterIP + Istio** keeps exposure behind the existing mesh gateway
3. **Probes tolerate cold start** while HF weights download
4. **Same layout as sibling ML services** — easy to operate and extend

---

<!-- _class: lead -->

# Questions?

Repo path: `medembed/`  
In-cluster: `medembed.medembed.svc.cluster.local`  
Gateway: `/medembed/`
