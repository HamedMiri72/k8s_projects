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
