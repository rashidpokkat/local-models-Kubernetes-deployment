# VEXYL-STT on AKS (CPU, ClusterIP)

Self-hosted speech-to-text using [VEXYL-STT](https://github.com/vexyl-ai/vexyl-stt) wrapping [`ai4bharat/indic-conformer-600m-multilingual`](https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual) (600M). Sibling to [`medembed/`](../medembed/) and [`bge-small/`](../bge-small/).

Provides **WebSocket streaming** and **REST batch** transcription on one port.

## Prerequisites

- Hugging Face access **approved** for the gated model: [ai4bharat/indic-conformer-600m-multilingual](https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual)
- HF token with read access
- Azure Container Registry (or other registry) to host the image
- AKS nodes with enough free capacity for **4–8 CPU** and **8–16Gi** memory
- `kubectl` configured for your AKS cluster

## Build and push image

The upstream Dockerfile bakes model weights at **build** time (`HF_TOKEN`).

```bash
git clone https://github.com/vexyl-ai/vexyl-stt.git
cd vexyl-stt

export HF_TOKEN=hf_your_token
export ACR=YOUR_ACR   # without .azurecr.io

docker build --build-arg HF_TOKEN=$HF_TOKEN -t ${ACR}.azurecr.io/vexyl-stt:1.0 .
az acr login --name $ACR
docker push ${ACR}.azurecr.io/vexyl-stt:1.0
```

Edit [`deployment.yaml`](deployment.yaml) and replace `YOUR_ACR.azurecr.io/vexyl-stt:1.0` with your real image.

Ensure the AKS cluster can pull from that registry (attach ACR or imagePullSecrets).

## Deploy

```bash
kubectl apply -f vexyl-stt/namespace.yaml -f vexyl-stt/deployment.yaml -f vexyl-stt/service.yaml
kubectl -n vexyl-stt rollout status deploy/vexyl-stt
```

Startup can take several minutes while the process warms up.

Do **not** apply `virtualservice-route.yaml` directly — paste it into your existing VirtualService.

## In-cluster endpoint

```
http://vexyl-stt.vexyl-stt.svc.cluster.local
```

WebSocket (in-cluster): `ws://vexyl-stt.vexyl-stt.svc.cluster.local/`

## Istio VirtualService route

Paste the HTTP route from [`virtualservice-route.yaml`](virtualservice-route.yaml) into your existing VirtualService `http:` list:

```yaml
- match:
    - uri:
        prefix: /vexyl-stt/
  rewrite:
    uri: /
  timeout: 3600s
  route:
    - destination:
        host: vexyl-stt.vexyl-stt.svc.cluster.local
        port:
          number: 80
```

Through the mesh gateway (keep the trailing slash on the prefix path):

- `GET /vexyl-stt/health`
- `POST /vexyl-stt/batch/transcribe`
- `GET /vexyl-stt/batch/status/{job_id}`
- `GET /vexyl-stt/batch/result/{job_id}`
- WebSocket: `wss://<gateway-host>/vexyl-stt/`

Istio HTTP routes support WebSocket upgrade by default. The `timeout: 3600s` helps long streaming sessions.

## Smoke test

Health:

```bash
kubectl -n vexyl-stt run curl --rm -it --image=curlimages/curl -- \
  curl -s http://vexyl-stt.vexyl-stt.svc.cluster.local/health
```

Expect JSON with `"status":"ok"` and `"model":"indic-conformer-600m-multilingual"`.

Batch (from a pod that has your audio file, or copy a WAV into the curl pod):

```bash
curl -s -X POST http://vexyl-stt.vexyl-stt.svc.cluster.local/batch/transcribe \
  -F "file=@recording.wav" \
  -F "language_code=hi-IN"
```

Then poll `/batch/status/{job_id}` / `/batch/result/{job_id}`.

## Optional API key

Set `VEXYL_STT_API_KEY` on the Deployment (via Secret) to require `X-API-Key` on clients. `/health` stays open. Left unset by default for ClusterIP-only access.

## Notes

- CPU image from upstream Dockerfile; set `VEXYL_STT_DEVICE=cuda` only if you build a GPU image and use GPU nodes.
- Decode mode defaults to `ctc` (faster). Use `rnnt` for higher accuracy if needed.
- Supported language codes include `hi-IN`, `ta-IN`, `te-IN`, `ml-IN`, and others documented upstream.
