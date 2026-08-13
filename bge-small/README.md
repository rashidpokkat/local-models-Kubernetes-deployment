# BGE-small on AKS (CPU, ClusterIP)

Serves [`BAAI/bge-small-en-v1.5`](https://huggingface.co/BAAI/bge-small-en-v1.5) with [Hugging Face Text Embeddings Inference (TEI)](https://github.com/huggingface/text-embeddings-inference) on CPU. Sibling to [`medembed/`](../medembed/) and [`vexyl-stt/`](../vexyl-stt/).

## Prerequisites

- `kubectl` configured for your AKS cluster
- Nodes with enough free capacity for **1–2 CPU** and **2–4Gi** memory
- Egress from the cluster to Hugging Face on first start (model download)

## Deploy

```bash
kubectl apply -f bge-small/namespace.yaml -f bge-small/deployment.yaml -f bge-small/service.yaml
kubectl -n bge-small rollout status deploy/bge-small
```

First startup can take a few minutes while TEI downloads model weights.

Do **not** apply `virtualservice-route.yaml` directly — it is a route snippet for your existing VirtualService.

## In-cluster endpoint

```
http://bge-small.bge-small.svc.cluster.local
```

## Istio VirtualService route

Paste the HTTP route from [`virtualservice-route.yaml`](virtualservice-route.yaml) into your existing VirtualService `http:` list:

```yaml
- match:
    - uri:
        prefix: /bge-small/
  rewrite:
    uri: /
  route:
    - destination:
        host: bge-small.bge-small.svc.cluster.local
        port:
          number: 80
```

Through the mesh gateway (keep the trailing slash on the prefix path):

- `POST /bge-small/embed`
- `POST /bge-small/v1/embeddings`
- `GET /bge-small/health`
- `GET /bge-small/info`

## Smoke test

```bash
kubectl -n bge-small run curl --rm -it --image=curlimages/curl -- \
  curl -s http://bge-small.bge-small.svc.cluster.local/info

kubectl -n bge-small run curl --rm -it --image=curlimages/curl -- \
  curl -s http://bge-small.bge-small.svc.cluster.local/embed \
  -H 'Content-Type: application/json' \
  -d '{"inputs":"hello world"}'
```

Confirm `/info` reports `model_id` as `BAAI/bge-small-en-v1.5`.

## Client usage

- `POST /embed` with `{"inputs":"..."}` or a list of strings
- `POST /v1/embeddings` for OpenAI-style clients
- `GET /health` for readiness
- `GET /info` for the loaded model id

## Notes

- Model cache uses `emptyDir` at `/data`; weights are re-downloaded if the pod is recreated.
- If the pod fails on first boot, check egress to `huggingface.co`.
