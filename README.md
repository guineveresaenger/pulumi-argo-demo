# pulumi-argo-demo

A minimal demo of using **ArgoCD** to drive the **Pulumi Kubernetes Operator** (PKO) via GitOps,
built to show off the operator's deletion-safety behavior.

ArgoCD has no native Pulumi support. What works is: ArgoCD syncs a `Stack` custom resource into
the cluster, and PKO does the actual infrastructure work. **ArgoCD manages the operator's CRs;
the operator manages the cloud resources.**

```
  git repo (this repo, app/)
        │  ArgoCD watches + syncs
        ▼
  Stack CR  ──►  PKO operator  ──►  workspace pod runs `pulumi up`  ──►  cloud resource
  Program CR                                                              (random pet, later an S3 bucket)
```

## Layout

| Path | What | Applied by |
|---|---|---|
| `app/` | the `Stack` + `Program` CRs ArgoCD syncs | **ArgoCD** |
| `argocd/application.yaml` | the ArgoCD `Application` pointing at `app/` | you (once) |
| `cluster-prereqs/` | namespace, RBAC, token secret — the things the workspace pod needs | **you, manually** |

Why the split: `cluster-prereqs/` are deliberately **out-of-band** (not managed by ArgoCD). When you
delete the ArgoCD Application during the teardown demo, only the `Stack` + `Program` get pruned —
the namespace, ServiceAccount, and token survive. That isolates the teardown to a clean
"the source got deleted out from under the Stack" scenario.

## Prerequisites (already done on the demo cluster)

- A Kubernetes cluster (`kind` is fine).
- **ArgoCD** installed in namespace `argocd`.
- **PKO v2.7.0** installed in namespace `pulumi-kubernetes-operator`.

## Step 1 — Cluster prerequisites (manual)

These give the PKO **workspace pod** (the throwaway pod that runs `pulumi up`) what it needs.

**1a. Namespace** — created out-of-band so it survives an Application delete:

```bash
kubectl apply -f cluster-prereqs/namespace.yaml
```

**1b. ServiceAccount + ClusterRoleBinding** — the workspace pod's agent authenticates the
operator's gRPC calls using Kubernetes TokenReview; `system:auth-delegator` grants permission to
perform those reviews:

```bash
kubectl apply -f cluster-prereqs/rbac.yaml
```

**1c. Pulumi Cloud token** — injected into the workspace as `PULUMI_ACCESS_TOKEN` (see the Stack's
`envRefs`) so it can reach the Pulumi Cloud backend. This is the one real secret, which is why it
is **not** in git. Create it imperatively:

```bash
kubectl create secret generic pulumi-api-secret \
  --namespace demo \
  --from-literal=accessToken="$(pulumi login --help >/dev/null; cat ~/.pulumi/credentials.json | python3 -c 'import json,sys;print(json.load(sys.stdin)["accessTokens"]["https://api.pulumi.com"])')"
```

…or just create a fresh access token at <https://app.pulumi.com/account/tokens> and:

```bash
kubectl create secret generic pulumi-api-secret -n demo --from-literal=accessToken=pul-XXXXXXXX
```

(`cluster-prereqs/pulumi-token-secret.example.yaml` is a manifest template if you prefer apply.)

## Step 2 — Point ArgoCD at this repo

```bash
kubectl apply -f argocd/application.yaml
```

The `Application` has `automated` sync on, so ArgoCD pulls `app/`, creates the `Stack` + `Program`,
and PKO runs `pulumi up`. Watch it:

```bash
# Argo's view
kubectl -n argocd get applications.argoproj.io pulumi-argo-demo

# the operator's view
kubectl -n demo get stack,program
kubectl -n demo get updates.auto.pulumi.com     # an "up" Update appears
kubectl -n demo get workspace                    # the workspace pod doing the work

# the result: stack outputs surface on the Stack status
kubectl -n demo get stack pulumi-argo-demo -o jsonpath='{.status.outputs.petName}'; echo
```

### The ArgoCD UI

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  — user: admin
# password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## The demo — two acts

### Act 1 — GitOps round-trip (the happy path)

Edit `app/program.yaml`, commit, push. ArgoCD notices and syncs; PKO runs `pulumi up`; the change
reconciles. This is the everyday, safe path — it works on stock v2.7.0.

> Note: ArgoCD flips to **Synced** the instant the CRs match git, while PKO is still running Pulumi
> underneath. That gap is exactly why custom **health checks**
> ([PKO issue #1007](https://github.com/pulumi/pulumi-kubernetes-operator/issues/1007)) are worth
> adding — so Argo shows *Progressing → Healthy* instead of a premature green.

### Act 2 — Teardown (the dangerous path)

Delete the Application (the **Delete** button in the Argo UI, or):

```bash
kubectl delete application pulumi-argo-demo -n argocd
```

ArgoCD cascade-deletes the `Stack` **and** `Program` together. The Stack's finalizer
(`destroyOnFinalize: true`) now needs to run `pulumi destroy` — but to do that it must bootstrap a
workspace from the source, which was just deleted.

- On **stock v2.7.0**: the Stack hangs `Stalled / SourceUnavailable`; the cloud resource is
  **orphaned** in Pulumi state (with the S3 variant, the bucket lingers in AWS).
- On a **fixed build** (destroy-from-state): PKO bootstraps a *stub* workspace that runs destroy
  against the backend state alone — no source needed — and the resource is cleaned up.

This is [PKO issue #1222](https://github.com/pulumi/pulumi-kubernetes-operator/issues/1222) /
[#752](https://github.com/pulumi/pulumi-kubernetes-operator/issues/752).

## Swapping in a real AWS bucket

`app/program.yaml` uses a `random:RandomPet` so the pipeline works with **zero cloud credentials**.
To make the orphan visceral (a real bucket sitting in AWS), swap the resource for an
`aws:s3:Bucket` and add AWS credentials as another `envRef`. See `DEMO.md` (added later) for the
exact diff.
