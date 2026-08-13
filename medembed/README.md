# MedEmbed on AKS (CPU, ClusterIP)

Serves [`abhinand/MedEmbed-base-v0.1`](https://huggingface.co/abhinand/MedEmbed-base-v0.1) with [Hugging Face Text Embeddings Inference (TEI)](https://github.com/huggingface/text-embeddings-inference) on CPU. The API is reachable inside the cluster via ClusterIP, and can be exposed through an Istio VirtualService route.

**Present this deployment:** open [`presentation.html`](presentation.html) in a browser (arrow keys / buttons to navigate), or use [`PRESENTATION.md`](PRESENTATION.md) with [Marp](https://marp.app/).

Sibling services in this repo:

- [`bge-small/`](../bge-small/) — `BAAI/bge-small-en-v1.5` (TEI embeddings)
- [`vexyl-stt/`](../vexyl-stt/) — VEXYL-STT + IndicConformer 600M (WebSocket + REST STT)

## Prerequisites

- `kubectl` configured for your AKS cluster
- Nodes with enough free capacity for **2–4 CPU** and **4–8Gi** memory
- Egress from the cluster to Hugging Face on first start (model download)

## Deploy

```bash
kubectl apply -f medembed/namespace.yaml -f medembed/deployment.yaml -f medembed/service.yaml
kubectl -n medembed rollout status deploy/medembed
```

First startup can take several minutes while TEI downloads model weights. ONNX 404 warnings on CPU are expected for this model; TEI falls back to Candle + safetensors and should reach `Ready`.

Do **not** apply `virtualservice-route.yaml` directly — it is a route snippet for your existing VirtualService.

## In-cluster endpoint

```
http://medembed.medembed.svc.cluster.local
```

## Istio VirtualService route

Paste the HTTP route from [`virtualservice-route.yaml`](virtualservice-route.yaml) into your existing VirtualService `http:` list:

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

Through the mesh gateway, clients call (keep the trailing slash on the prefix path):

- `POST /medembed/embed`
- `POST /medembed/v1/embeddings`
- `GET /medembed/health`

Istio rewrites `/medembed/` → `/`, so TEI receives `/embed`, `/v1/embeddings`, and `/health`.

## Smoke test

```bash
kubectl -n medembed run curl --rm -it --image=curlimages/curl -- \
  curl -s http://medembed.medembed.svc.cluster.local/embed \
  -H 'Content-Type: application/json' \
  -d '{"inputs":"patient with chest pain and elevated troponin"}'
```

OpenAI-compatible embeddings:

```bash
kubectl -n medembed run curl --rm -it --image=curlimages/curl -- \
  curl -s http://medembed.medembed.svc.cluster.local/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"input":"patient with chest pain and elevated troponin","model":"abhinand/MedEmbed-base-v0.1"}'
```

## Client usage

- `POST /embed` with `{"inputs":"..."}` or a list of strings
- `POST /v1/embeddings` for OpenAI-style clients
- `GET /health` for readiness

## Notes

- Model cache uses `emptyDir` at `/data`; weights are re-downloaded if the pod is recreated. Switch to a PVC later if cold starts matter.
- If the pod stays Pending or CrashLooping on first boot, check egress to `huggingface.co` or bake the model into a private image.
