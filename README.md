# Kubernetes Deployments — Managing ReplicaSets the Right Way

Follow-up to the [ReplicaSet lab](../k8s-replicaset-lab). That lab covered how a ReplicaSet keeps a fixed number of Pods running. This one covers what sits on top of it in any real environment: the `Deployment` object, and why you'd almost never manage a ReplicaSet directly in production.

## Where Deployment fits in

A ReplicaSet answers one question well: "are the right number of Pods running?" It doesn't know anything about updating the application, rolling back a bad release, or coordinating a change without downtime. That's the gap `Deployment` fills.

The hierarchy is straightforward:

```
Deployment → ReplicaSet → Pods
```

You interact with the Deployment. The Deployment creates and manages a ReplicaSet on your behalf. The ReplicaSet does what it always did — keep the Pod count correct. On top of that, Deployment adds:

- **Declarative updates** — describe the desired state in YAML, Kubernetes reconciles the cluster to match it
- **Rolling updates** — new versions roll out gradually, with zero downtime
- **Rollback** — revert to a previous ReplicaSet/revision if a rollout goes wrong
- **Scaling** — up or down, manually or via HPA

In short: ReplicaSet handles *replication*, Deployment handles the *lifecycle* around that replication.

## The lab

**Goal:** create a Deployment named `nginx-deployment`, running `nginx:latest`, with 3 replicas — using both the declarative and imperative approaches, so the tradeoffs are obvious side by side.

### Declarative approach (the one you'd actually use)

`deployment-definition.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

This is structurally almost identical to a bare ReplicaSet manifest — same `replicas`, `selector`, `template`. The difference is entirely in `kind: Deployment`, which is what gives you the extra controller behavior on top.

Apply it:

```bash
kubectl create -f deployment-definition.yaml
```

This is the approach worth defaulting to. The config lives in a file, which means it's version-controlled, reviewable in a PR, and reproducible — none of which is true if you only ever type commands directly at the cluster.

### Imperative approach (fine for a quick check, not for anything real)

Same result, no YAML file:

```bash
kubectl create deployment nginx-deployment --image=nginx:latest --replicas=3 --port=80
```

Useful for a fast sanity check or throwaway testing. But nothing here gets persisted anywhere reviewable — if you tear down your shell history, the exact configuration that created this Deployment is gone. Not something to build a production workflow around.

### Verifying it

```bash
kubectl get deployments
kubectl get pods
```

`get deployments` confirms the Deployment itself reports the right desired/current/ready counts. `get pods` shows the three Pods it's ultimately responsible for — though notice they're not owned by the Deployment directly. Run this to see the actual chain:

```bash
kubectl get replicaset
```

You'll see a ReplicaSet with a name like `nginx-deployment-<hash>` — that's the object the Deployment actually created and delegates Pod management to. This is the hierarchy from the diagram above, made visible.

### Cleanup

```bash
kubectl delete deployment nginx-deployment
```

One command tears down the whole chain — Deployment, its ReplicaSet, and all three Pods. This works because of Kubernetes' owner-reference-based garbage collection: child objects carry a reference back to their owner, so deleting the top of the chain cascades down automatically.

## The actual point of this lab

The YAML for a Deployment and a bare ReplicaSet look almost the same, which can make it feel like Deployment is just a rename. It isn't — it's a controller sitting a level higher, and that becomes obvious the moment you need to change something rather than just keep it running: push a new image version, roll it out without downtime, and have a way back if it breaks. A ReplicaSet alone gives you none of that; it'll happily keep 3 broken Pods running forever if that's what the template says.

## Notes / worth trying next

- `kubectl set image deployment/nginx-deployment nginx=nginx:1.25` and watch `kubectl rollout status deployment/nginx-deployment` — this is where the rolling update behavior actually shows itself, which a bare ReplicaSet can't do
- `kubectl rollout undo deployment/nginx-deployment` to see the rollback in action after a bad update
- `kubectl rollout history deployment/nginx-deployment` to see how revisions are tracked
- Compare this directly against the [ReplicaSet lab](../k8s-replicaset-lab) — same self-healing behavior underneath (delete a Pod, it comes back), but now with an update/rollback layer on top
