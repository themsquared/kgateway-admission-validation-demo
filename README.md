# kgateway admission validation demo

kgateway [v2.5.1](https://github.com/kgateway-dev/kgateway/releases/tag/v2.5.1) (shipped 2026-11-06) adds explicit minimum/maximum bounds to several CRD fields across `BackendConfigPolicy`, `Backend`, `ListenerPolicy`, and `GatewayExtension` ([PR #14992](https://github.com/kgateway-dev/kgateway/pull/14992)). Fields that Envoy itself would reject are now rejected by the Kubernetes API server at `kubectl apply`, instead of being accepted and failing later, deeper in the control plane.

This repo is a 10-minute kind cluster that proves it: two manifests the API server now rejects outright, and a working gateway built from the manifests that pass.

## The problem this closes

Before this release, a kgateway CRD's OpenAPI schema mostly checked *type* (is this a string, is this an integer) but not *range*. A `healthCheck.timeout: 0s` or a static backend `port: 70000` would pass CRD admission, get picked up by the controller, and only fail once it reached Envoy's own config validation, deeper in the reconcile path, with none of it visible from `kubectl apply`'s exit code. PR #14992 moves that check to the boundary: the same value now fails at `kubectl apply`, with the specific field and the specific reason in the error.

## What's in this cluster

- A `BackendConfigPolicy` with `healthCheck.timeout: 0s` — **rejected at admission**.
- A `Backend` with a static host `port: 70000` — **rejected at admission** (ports must be 1-65535).
- A working `Gateway` + `HTTPRoute` + `BackendConfigPolicy` (valid `timeout: 2s`) routing to an `httpbin` backend, to show the valid shape actually works end to end.

## Quickstart

Prerequisites: `kind`, `kubectl`, `helm`, Docker running.

```bash
kind create cluster --name kgateway-admission-demo

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.7.0/standard-install.yaml

helm install kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds \
  --version v2.5.1 --namespace kgateway-system --create-namespace

helm install kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.5.1 --namespace kgateway-system --create-namespace

kubectl -n kgateway-system wait --for=condition=Available deployment/kgateway --timeout=120s

kubectl apply -f manifests/00-namespace.yaml
```

### 1. Watch the API server reject an invalid health check timeout

```bash
kubectl apply -f manifests/01-bad-backendconfigpolicy.yaml
```

```
The BackendConfigPolicy "bad-healthcheck" is invalid: spec.healthCheck.timeout: Invalid value: "0s": must be at least 1ms
```

### 2. Watch it reject an out-of-range backend port

```bash
kubectl apply -f manifests/05-bad-backend-port.yaml
```

```
The Backend "bad-static-backend" is invalid: spec.static.hosts[0].port: Invalid value: 70000: port must be between 1 and 65535
```

### 3. Bring up the valid version and prove it routes traffic

```bash
kubectl apply -f manifests/02-httpbin.yaml
kubectl apply -f manifests/03-gateway.yaml
kubectl apply -f manifests/04-good-backendconfigpolicy.yaml

kubectl -n demo wait --for=condition=Programmed gateway/demo-gw --timeout=60s
kubectl -n demo wait --for=condition=Ready pod --all --timeout=90s

kubectl -n demo port-forward svc/demo-gw 18080:80 &
curl -s http://localhost:18080/get -H "Host: demo"
```

```json
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "demo",
    "User-Agent": "curl/8.7.1",
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000",
    "X-Envoy-External-Address": "127.0.0.1"
  },
  "origin": "10.244.0.7",
  "url": "http://demo/get"
}
```

```bash
kubectl -n demo get backendconfigpolicy good-healthcheck -o jsonpath='{.status}'
```

```json
{"ancestors":[{"ancestorRef":{"group":"","kind":"Service","name":"httpbin","namespace":"demo"},"conditions":[{"reason":"Valid","status":"True","type":"Accepted"},{"reason":"Attached","status":"True","type":"Attached"}],"controllerName":"kgateway.dev/kgateway"}]}
```

## Architecture notes

- `kgateway-crds` chart installs the CRDs (including the tightened OpenAPI schema with CEL `x-kubernetes-validations` rules); `kgateway` chart installs the controller that watches Gateway API resources and `gateway.kgateway.dev` CRDs and programs Envoy proxies per-`Gateway`.
- The rejection in this demo happens entirely at the Kubernetes API server, via CRD schema validation (`x-kubernetes-validations` CEL rules and OpenAPI `minimum`/`maximum`) — the kgateway controller never sees the invalid object. That's the whole point of the change: fail fast at the boundary, not somewhere inside a reconcile loop.
- The other PR #14992 bounds not demoed here but in the same release: `BackendConfigPolicy` `connectTimeout`/`healthCheck.interval` of 0s, `dns.refreshRate` of 1ms or less, `maxHeadersCount` 0, `leastRequest.choiceCount` below 2, ring hash sizes outside 1-8388608; `Backend` `aws.ec2.port` above 65535; `ListenerPolicy` local reply `statusCode` below 200, gRPC retry `baseInterval` of 0s; `GatewayExtension` an `extAuth.statusOnError` code Envoy doesn't define, `extProc` message timeouts above 1h. Existing objects with these values are grandfathered in; they just can't be created again.

## Cleanup

```bash
kind delete cluster --name kgateway-admission-demo
```

## Write-up

> 📖 **Read the write-up:** _link added once the post is live_

## Topics

`kgateway` `gateway-api` `kubernetes` `envoy` `ai-gateways` `admission-control`
