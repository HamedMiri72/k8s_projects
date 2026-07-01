# Fullstack App — Two-Tier Kubernetes Demo

A complete, config-driven two-tier application running on Kubernetes, built with only
public images (no app code to build). It exercises everything: Deployments, Services,
labels/selectors, Endpoints, in-cluster DNS, readiness probes, rolling updates, Namespaces,
Ingress, ConfigMaps (env-var **and** volume-mount styles), and Secrets.

A single request to `/api` travels the entire stack, so this repo doubles as an
interview walkthrough.

---

## The application

- **Frontend:** `nginx:alpine` — receives external traffic, serves `/`, and proxies `/api`
  to the backend. Its nginx config comes from a ConfigMap (mounted as a file).
- **Backend:** `hashicorp/http-echo` — returns a text response. That text is built from a
  ConfigMap value (greeting) and a Secret value (API token), injected as environment variables.

Both images pull straight from Docker Hub — nothing to build.

## Architecture — the full request path

```
                       EXTERNAL USER
                            |
                            v
                    Ingress (fullstack.local)
                            |  (Host header match)
                            v
                    Frontend Service (ClusterIP)
                            |
                     Endpoints -> nginx Pods (x2)
                            |         ^
                            |         '-- nginx config mounted from ConfigMap (frontend-config)
                            |
                 nginx proxies /api  --(CoreDNS resolves "backend")-->
                            |
                            v
                    Backend Service (ClusterIP)
                            |
                     Endpoints -> http-echo Pods (x3)
                                      ^
                                      |-- GREETING  env from ConfigMap (backend-config)
                                      '-- API_TOKEN env from Secret   (backend-secret)
```

Everything runs inside the `fullstack` **Namespace**, isolated from other projects on the cluster.

## Files

```
fullstack-app/
├── namespace.yaml                    # the fullstack namespace
├── backend/
│   ├── backend-configmap.yaml        # GREETING (non-secret config)
│   ├── backend-secret.yaml           # API_TOKEN (base64, NOT encrypted)
│   ├── backend-deployment.yaml       # http-echo x3, env from configmap+secret, readiness probe
│   └── backend-service.yaml          # ClusterIP, port 80 -> targetPort 8080
├── frontend/
│   ├── frontend-configmap.yaml       # nginx default.conf that proxies /api -> backend
│   ├── frontend-deployment.yaml      # nginx x2, mounts configmap as file, readiness probe
│   └── frontend-service.yaml         # ClusterIP, port 80 -> targetPort 80
└── ingress.yaml                      # external traffic (fullstack.local) -> frontend
```

---

## Concepts demonstrated

**Namespace** — a logical partition inside one cluster. Isolates this project's objects from
others. Set it as the default so you don't type `-n fullstack` every time.

**Deployment / ReplicaSet / Pods** — you declare desired replicas; the Deployment creates a
ReplicaSet, which keeps that many Pods alive and self-heals when one dies.

**Labels & selectors** — a Service has no hard link to Pods. It selects every Pod carrying a
matching label (`app: backend`). Scale up and new Pods are included automatically.

**Endpoints** — the live list of *Ready* Pod IPs matching a Service's selector. Empty Endpoints
= selector matches nothing OR no Pod is Ready. Highest-signal thing to check when traffic fails.

**Service (ClusterIP)** — a stable virtual IP/name in front of churning Pods. `port` is what the
Service listens on; `targetPort` is the container's port (backend: 80 -> 8080).

**CoreDNS** — resolves Service names to ClusterIPs. nginx reaches the backend at
`backend.fullstack.svc.cluster.local` (form: `<service>.<namespace>.svc.cluster.local`).

**Readiness probe** — decides if a Pod is ready to receive traffic. Until it passes, the Pod is
`Running` but excluded from Endpoints. (Different from a *liveness* probe, which restarts a broken
container. Readiness gates traffic; liveness gates restarts.)

**ConfigMap** — non-secret key-value config, injected two ways in this project:
- *As env vars* (backend): `GREETING` from `backend-config` becomes an env var.
- *As a mounted file* (frontend): `default.conf` from `frontend-config` becomes
  `/etc/nginx/conf.d/default.conf`, driving nginx behavior.

**Secret** — same mechanism as ConfigMap but for sensitive data. **Only base64-encoded by
default, NOT encrypted** — anyone with read access can decode it instantly. Real protection needs
encryption-at-rest, RBAC, and/or external managers (Vault). Also: don't commit real Secret values
to git (they live in history forever) — use sealed-secrets / external managers in production.

**Ingress = resource + controller** — the Ingress *resource* is routing rules; the *controller*
(ingress-nginx) is the software that executes them. No controller = nothing happens. An Ingress can
only route to Services in its **own namespace**, and routes by the request's `Host` header.

---

## Run it from scratch

Assumes a running cluster (Docker Desktop Kubernetes here) and the ingress-nginx controller
already installed. (To install the controller if needed:
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml`)

```bash
# --- Namespace ---
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=fullstack   # default to this ns
kubectl get ns

# --- Backend: config + secret first, so the deployment can consume them ---
kubectl apply -f backend/backend-configmap.yaml
kubectl describe configmap backend-config

kubectl apply -f backend/backend-secret.yaml
kubectl get secret backend-secret -o yaml                      # values are base64
kubectl get secret backend-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d   # decode = plain text!

# --- Backend deployment + service ---
kubectl apply -f backend/backend-deployment.yaml
kubectl get pods -w                                            # wait for 3x 1/1, then Ctrl+C
kubectl apply -f backend/backend-service.yaml
kubectl get endpoints backend                                  # 3 IPs = selector matched

# --- Frontend: nginx config, deployment (mounts it), service ---
kubectl apply -f frontend/frontend-configmap.yaml
kubectl describe configmap frontend-config
kubectl apply -f frontend/frontend-deployment.yaml
kubectl get pods -w                                            # wait for 2x 1/1, then Ctrl+C
kubectl apply -f frontend/frontend-service.yaml
kubectl get endpoints frontend                                 # 2 IPs

# --- Ingress ---
kubectl apply -f ingress.yaml
kubectl get ingress
```

## Verify each layer

```bash
# Backend directly (http-echo has NO shell — curl instead of exec).
# The response text carries the injected ConfigMap greeting + Secret token:
kubectl port-forward deploy/backend 8080:8080
curl http://localhost:8080/
# -> Hello from the backend API (config-driven) | token=super-secret-token-12345

# Confirm the env wiring without entering the container:
kubectl get pod -l app=backend -o jsonpath='{.items[0].spec.containers[0].env}' | python3 -m json.tool

# Frontend has a shell — confirm the mounted nginx config file:
kubectl exec -it deploy/frontend -- cat /etc/nginx/conf.d/default.conf

# Frontend internally (both paths):
kubectl port-forward svc/frontend 8081:80
curl http://localhost:8081/          # Frontend is up...
curl http://localhost:8081/api       # proxied -> backend greeting + token

# The whole app from "outside", through the Ingress:
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
curl -H "Host: fullstack.local" http://localhost:8080/       # frontend
curl -H "Host: fullstack.local" http://localhost:8080/api    # frontend -> backend, full chain
```

---

## Handy commands & debugging cheatsheet

```bash
# Inspecting
kubectl get pods --show-labels                 # see labels that Services select on
kubectl get pods -w                            # watch state changes live
kubectl get endpoints <svc>                    # live Ready-Pod IPs behind a service
kubectl describe pod <pod>                      # full detail + Events (start debugging here)
kubectl get <resource> -o yaml                  # full manifest as stored
kubectl get pod -l app=backend -o jsonpath='{...}'   # extract specific fields

# Namespaces
kubectl config set-context --current --namespace=fullstack   # set default ns
kubectl config set-context --current --namespace=default     # switch back
kubectl get all -n fullstack                    # everything in a namespace

# Reaching things (Docker Desktop-friendly)
kubectl port-forward deploy/<name> <local>:<pod>     # tunnel to a Deployment's pod
kubectl port-forward svc/<name> <local>:<svcport>    # tunnel to a Service
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80  # reach ingress

# Shell-less (distroless) images — http-echo has no sh/bash:
kubectl debug -it deploy/backend --image=busybox --target=backend -- sh   # ephemeral debug container

# Secrets
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d   # decode (proves base64 != encryption)

# Rollouts
kubectl rollout status deployment <name>       # track a rolling update
kubectl rollout undo deployment <name>         # roll back to previous ReplicaSet
```

### Debug order for "a Pod can't reach a Service"

1. `kubectl get svc <name>` — exists, has a ClusterIP?
2. `kubectl get endpoints <name>` — **empty = selector != labels, or no Ready Pods.** Catches most cases.
3. `kubectl get pods --show-labels` — do Pod labels match the selector?
4. `kubectl describe pod <pod>` — Running *and* Ready? (failing readiness pulls a Pod from Endpoints)
5. `kubectl exec / curl` — is CoreDNS resolving the name? (from a Pod that has a shell)
6. For Ingress: is a controller installed & Ready? Is `ingressClassName` set? Does the `Host` header match a rule? Is the Ingress in the *same namespace* as the target Service?

---

## Key caveat points from this project

- **Config is decoupled from the image** — same image runs anywhere; the environment supplies config.
- **Two ConfigMap injection styles** — env vars (static at Pod start) vs mounted files (can update live, good for config files).
- **Secrets are base64, not encrypted** — decode proves it; secure with encryption-at-rest + RBAC + Vault.
- **Ingress needs a controller** — the resource alone does nothing.
- **Shell-less images** — use `curl` through a port-forward, `-o jsonpath` on the spec, or `kubectl debug` to inspect.
- **Docker Desktop networking** — `port-forward` reliably bypasses the VM port-mapping quirk on macOS.

