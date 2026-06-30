# Kubernetes Services & Networking Demo

A minimal but complete project demonstrating how Pods, Services, labels/selectors,
Endpoints, in-cluster DNS, readiness probes, and rolling updates work together —
and how to debug them when they don't.

Built to *show* the request path in action, not just describe it.

## What's in here

| File | What it is |
|------|------------|
| `backend-deployment.yaml` | A Deployment running 3 replicas of a tiny `http-echo` server (labelled `app=backend`) with a readiness probe. |
| `backend-service.yaml` | A `ClusterIP` Service that selects `app=backend` and exposes the Pods at a stable name. |
| `frontend-pod.yaml` | A single Pod with `curl`, used to make requests from *inside* the cluster. |

## The architecture

```
frontend Pod  --(1)-->  CoreDNS  --(2)-->  Service (ClusterIP)
                                                |
                                          (watches selector)
                                                |
                                          Endpoints list  <-- live IPs of READY Pods matching app=backend
                                                |
                                          kube-proxy  --(3)-->  one of 3 backend Pods
```

When the frontend calls `http://backend`:

1. **CoreDNS** resolves the name `backend` to the Service's stable ClusterIP.
2. The request reaches the **Service** (a virtual IP, not tied to any machine).
3. **kube-proxy** picks one Pod IP from the **Endpoints** list and routes the request there, load-balancing across the replicas.

Four things must be healthy for this to work: DNS, the Service, a non-empty Endpoints list, and Ready Pods.

## Key concepts demonstrated

**Labels and selectors are the glue.** The Service has no hard link to specific Pods.
It defines a selector (`app: backend`) and continuously matches every Pod carrying that
label. Scale the Deployment and new Pods are picked up automatically; kill one and it drops out.

**Endpoints is the live list.** When a Service's selector matches Pods, Kubernetes maintains
an Endpoints object listing the current IPs of *Ready* Pods. This is the highest-signal thing
to check when traffic isn't flowing — an empty Endpoints list means the selector matches nothing,
or no matching Pod is Ready.

**`port` vs `targetPort`.** The Service listens on `port: 80` but forwards to `targetPort: 8080`,
the port the container actually serves on. These are deliberately different to show they're
two separate things.

**DNS means you never hardcode IPs.** The frontend calls `http://backend` — just the name.
CoreDNS turns it into the ClusterIP. Pods can churn and get new IPs; the name stays constant.

**Readiness probes gate traffic.** A Pod can be `Running` but `0/1` (not Ready). A readiness
probe decides whether a Pod is *ready to receive traffic*; until it passes, Kubernetes keeps
the Pod **out of Endpoints** even though it's running. "Running" and "receiving traffic" are
different states. (Distinct from a *liveness* probe, which restarts a broken container —
readiness controls traffic, liveness controls restarts.)

**Rolling updates keep the app available.** Changing the Pod template creates a new ReplicaSet.
The Deployment brings up a new Pod, waits for it to become Ready, and only *then* terminates an
old one — repeating Pod-by-Pod so there's never a gap in available, Ready Pods.

## Running it

```bash
# 1. Backend Deployment — 3 Pods labelled app=backend, with a readiness probe
kubectl apply -f backend-deployment.yaml
kubectl get pods --show-labels          # 3 Pods, all app=backend

# 2. Service in front of them
kubectl apply -f backend-service.yaml
kubectl get svc backend                 # note the assigned CLUSTER-IP
kubectl get endpoints backend           # 3 IP:8080 pairs — proof the selector matched

# 3. Frontend Pod to call from inside the cluster
kubectl apply -f frontend-pod.yaml
kubectl get pod frontend                # wait for 1/1 Running

# 4. Call the backend by name
kubectl exec -it frontend -- curl http://backend
# -> "hello from backend"
```

## Lesson 1: a selector mismatch (the most common Service failure)

A selector that doesn't match the Pod labels — a typo, a renamed label, a bad copy-paste.
The Pods are healthy; the *wiring* is wrong.

Change the Service selector to something that matches nothing (e.g. `app: backend-typo`),
re-apply, and inspect:

```bash
kubectl get svc backend          # still exists, still has a ClusterIP — looks fine!
kubectl get endpoints backend    # <none> — THIS is where the failure shows
kubectl exec -it frontend -- curl -m 5 http://backend   # times out
```

The trap: the Service *looks* healthy from `get svc`. The fault only appears at `get endpoints`.
DNS resolves fine, the Service exists — but with no endpoints behind it, there's nowhere to route.
**Empty Endpoints = no backends = failure.** The Pods were never the problem
(`kubectl get pods --show-labels` shows them still Running and correctly labelled).

Fix by restoring the correct selector (`app: backend`); Endpoints repopulates.

## Lesson 2: readiness probes and the not-Ready window

The readiness probe uses `initialDelaySeconds: 20`, so a freshly-started Pod spends ~20s as
`0/1 Running` — alive but not Ready — before the first probe runs and passes.

Watch a Pod go through it:

```bash
kubectl apply -f backend-deployment.yaml
kubectl get pods -w        # new Pods appear 0/1, sit, then flip to 1/1 after ~20s
```

Prove the Endpoints exclusion — delete a Pod and watch the Ready endpoint count dip and recover:

```bash
kubectl delete pod -l app=backend | head -1
for i in $(seq 1 15); do
  echo "ready endpoints: $(kubectl get endpoints backend -o jsonpath='{.subsets[*].addresses[*].ip}' | wc -w)"
  sleep 2
done
# count reads 3 -> 2 -> ... -> 3 as the replacement spends ~20s not-Ready (excluded), then rejoins
```

The not-Ready Pod is Running but excluded from Endpoints, so it receives zero traffic until
its probe passes. That's the readiness->Endpoints link, on demand.

## Lesson 3: reading a rolling update

Adding the probe changes the Pod template, which triggers a rolling update. In the Pod list you
see two ReplicaSet hashes — the old one (Terminating) and the new one (being created):

- The new Pod comes up `0/1`, waits out the readiness delay, becomes `1/1`.
- **Only then** does an old Pod start Terminating.
- This repeats Pod-by-Pod, so there are always enough Ready Pods to serve traffic.

`Error` states on *Terminating* Pods are cosmetic — the container exits non-zero on shutdown
signal. The new Pods transition cleanly `0/1 -> 1/1`.

```bash
kubectl rollout status deployment backend   # "successfully rolled out"
kubectl rollout undo deployment backend     # roll back to the previous ReplicaSet if needed
```

## Debug order for "a Pod can't reach a Service"

1. `kubectl get svc <name>` — does it exist and have a ClusterIP?
2. `kubectl get endpoints <name>` — **empty means selector != labels, or no Ready Pods.** Catches most cases.
3. `kubectl get pods --show-labels` — do the Pod labels actually match the selector?
4. `kubectl describe pod <pod>` — are the Pods Running *and* Ready? (a failing readiness probe pulls a Pod out of Endpoints)
5. `kubectl exec -it <pod> -- nslookup <service>` — is CoreDNS resolving the name?
6. Check NetworkPolicies — is traffic being blocked?

# Kubernetes Services, Networking & Ingress Demo

A minimal but complete project demonstrating how Pods, Services, labels/selectors,
Endpoints, in-cluster DNS, readiness probes, rolling updates, and Ingress work together —
and how to debug them when they don't.

Built to *show* the request path in action, not just describe it.

## What's in here

| File | What it is |
|------|------------|
| `kind-config.yaml` | A kind cluster config that maps host ports 80/443 into the node and labels it `ingress-ready` (needed for Ingress on kind). |
| `backend-deployment.yaml` | A Deployment running 3 replicas of a tiny `http-echo` server (labelled `app=backend`) with a readiness probe. |
| `backend-service.yaml` | A `ClusterIP` Service that selects `app=backend` and exposes the Pods at a stable name. |
| `backend-ingress.yaml` | An Ingress resource routing external HTTP traffic for `myapp.local` to the backend Service. |
| `frontend-pod.yaml` | A single Pod with `curl`, used to make requests from *inside* the cluster. |

## The full request path

```
EXTERNAL:
  your machine → Ingress Controller → (matches host/path) → Service → Endpoints → Pod

INTERNAL:
  frontend Pod → CoreDNS → Service (ClusterIP) → Endpoints → kube-proxy → Pod
```

Ingress is a layer added *in front of* the Service. The internal Service → Endpoints → Pod
chain still runs underneath; Ingress just adds an external, HTTP-aware entry point.

## Key concepts demonstrated

**Labels and selectors are the glue.** A Service has no hard link to specific Pods — it defines
a selector (`app: backend`) and continuously matches every Pod carrying that label.

**Endpoints is the live list.** Kubernetes maintains an Endpoints object listing the current IPs
of *Ready* Pods matching the selector. Empty Endpoints = selector matches nothing, or no Ready Pod.

**`port` vs `targetPort`.** The Service listens on `port: 80` but forwards to `targetPort: 8080`,
the port the container serves on.

**DNS means you never hardcode IPs.** Callers use the name (`http://backend`); CoreDNS resolves it.

**Readiness probes gate traffic.** A Pod can be `Running` but `0/1` (not Ready). Until its readiness
probe passes, it's kept *out of Endpoints* and receives no traffic. (Distinct from a liveness probe,
which restarts a broken container — readiness controls traffic, liveness controls restarts.)

**Rolling updates keep the app available.** Changing the Pod template makes a new ReplicaSet; the
Deployment brings up a new Ready Pod before terminating an old one, Pod-by-Pod.

**Ingress = resource + controller (two separate things).** The Ingress *resource* is just routing
rules (declared intent). The Ingress *controller* (e.g. ingress-nginx) is the running software that
reads those rules and does the routing. A cluster has no controller by default — you must install one.
**An Ingress resource does nothing without a controller running.**

## Running it

```bash
# 0. Create the cluster WITH ingress port mappings
kind create cluster --name dev --config kind-config.yaml

# 1. Backend Deployment (3 Pods, readiness probe) + Service
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl get endpoints backend           # 3 IP:8080 pairs = selector matched Ready Pods

# 2. Frontend Pod, and call backend internally by name
kubectl apply -f frontend-pod.yaml
kubectl exec -it frontend -- curl http://backend     # -> "hello from backend"

# 3. Install the ingress-nginx controller (the "machine" that runs Ingress rules)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s

# 4. Apply the Ingress rule and reach the app from OUTSIDE the cluster
kubectl apply -f backend-ingress.yaml
curl -H "Host: myapp.local" http://localhost/        # -> "hello from backend"
```

## Lesson 1: a selector mismatch (most common Service failure)

Change the Service selector to something that matches nothing (`app: backend-typo`), re-apply:

```bash
kubectl get svc backend          # still exists, still has a ClusterIP — looks fine!
kubectl get endpoints backend    # <none> — THIS is where the failure shows
kubectl exec -it frontend -- curl -m 5 http://backend   # times out
```

The Service *looks* healthy from `get svc`; the fault only appears at `get endpoints`.
**Empty Endpoints = no backends = failure.** Fix by restoring the correct selector.

## Lesson 2: readiness probes and the not-Ready window

`initialDelaySeconds: 20` makes a fresh Pod spend ~20s as `0/1 Running` — alive but not Ready —
and excluded from Endpoints until its probe passes:

```bash
kubectl get pods -w        # new Pods appear 0/1, sit, then flip to 1/1 after ~20s
kubectl delete pod -l app=backend | head -1
for i in $(seq 1 15); do
  echo "ready endpoints: $(kubectl get endpoints backend -o jsonpath='{.subsets[*].addresses[*].ip}' | wc -w)"
  sleep 2
done
# count reads 3 -> 2 -> ... -> 3 as the replacement spends ~20s not-Ready, then rejoins
```

## Lesson 3: reading a rolling update

Adding the probe changes the Pod template → rolling update. The Pod list shows two ReplicaSet
hashes: the new Pod comes up `0/1`, becomes `1/1`, and only *then* does an old Pod Terminate —
so there's never a gap in Ready Pods. `Error` states on Terminating Pods are cosmetic (container
exits non-zero on shutdown signal).

```bash
kubectl rollout status deployment backend   # "successfully rolled out"
kubectl rollout undo deployment backend     # roll back if needed
```

## Lesson 4: Ingress needs a controller (debugging a missing one)

Symptom: `curl` to the Ingress refuses the connection, and:

```bash
kubectl get ns                      # no "ingress-nginx" namespace
kubectl get pods -n ingress-nginx   # Error: namespace not found
```

Diagnosis: the Ingress *resource* exists, but no *controller* is installed — so nothing listens
on port 80. Fix: install the controller (step 3 above), wait for it Ready, retry.

### Docker Desktop note (macOS)

On Docker Desktop, `curl localhost:80` may fail even with everything healthy, because of the
Docker Desktop VM port-mapping boundary. Reliable fallback — tunnel straight to the controller,
bypassing the VM mapping:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
curl -H "Host: myapp.local" http://localhost:8080/    # -> "hello from backend"
```

`port-forward` is also a great diagnostic: it isolates "is the app broken, or just the ingress plumbing?"

## Debug order for "a Pod can't reach a Service"

1. `kubectl get svc <name>` — exists, has a ClusterIP?
2. `kubectl get endpoints <name>` — **empty = selector != labels, or no Ready Pods.** Catches most cases.
3. `kubectl get pods --show-labels` — do Pod labels match the selector?
4. `kubectl describe pod <pod>` — Running *and* Ready? (failing readiness pulls a Pod from Endpoints)
5. `kubectl exec -it <pod> -- nslookup <service>` — is CoreDNS resolving?
6. Check NetworkPolicies — traffic blocked?

For Ingress specifically, also check: is a controller installed and Ready? Does the Ingress
have `ingressClassName` set? Does the request's `Host` header match a rule?

## Concepts to revisit next

- **NetworkPolicies** — restricting which Pods may talk to which.
- **Liveness probes** — restarting broken containers (vs readiness, which gates traffic).
- **TLS on Ingress** — terminating HTTPS at the ingress layer.