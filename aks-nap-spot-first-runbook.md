# AKS Node Auto-Provisioning — Spot-First, On-Demand Fallback, Auto Scale-Down

This config does exactly three things:

1. **Schedule workloads on Spot capacity first.**
2. **Fall back to On-Demand** automatically when spot can't be provisioned or is reclaimed.
3. **Scale nodes down** automatically when they become empty or underutilized (consolidation).

No HorizontalPodAutoscaler — capacity is driven purely by your Deployment's replica count and
Karpenter's pod-driven provisioning.

How it works:
- **`weight`** orders the NodePools (highest first): `application` (spot, 99) is always tried
  before `application2` (on-demand, 10). On-demand is used only when spot can't be provisioned.
- Pods must **tolerate the spot taint** AKS applies to spot nodes
  (`kubernetes.azure.com/scalesetpriority=spot:NoSchedule`) — without it they skip spot entirely.
- **`consolidationPolicy: WhenEmptyOrUnderutilized`** removes empty nodes and repacks
  underutilized ones; this is goal #3.

---

## Files

| File | Purpose |
|------|---------|
| `01-nodepool-application-spot.yaml`      | Spot pool, weight 99 (preferred) |
| `02-nodepool-application2-ondemand.yaml` | On-demand pool, weight 10 (fallback) |
| `03-deployment-app.yaml`                 | Workload: spot toleration + preferred-spot affinity + requests |
| `04-pdb-app.yaml`                        | PodDisruptionBudget (protects availability during disruption) |

Apply:
```bash
cd aks-spot-first
kubectl apply -f .          # or file-by-file in numbered order
```
Before applying: replace the container **image** in `03`, and tune the **limits** in `01`/`02`.

---

## Values used

| Setting | Spot (`application`) | On-Demand (`application2`) |
|---|---|---|
| `weight` | 99 | 10 |
| `consolidationPolicy` | WhenEmptyOrUnderutilized | WhenEmptyOrUnderutilized |
| `consolidateAfter` | 30m | 30m |
| `limits.cpu` / `memory` | 100 / 200Gi | 100 / 200Gi |
| `expireAfter` | Never | Never |
| `budgets` | 10% | 10% |

`consolidateAfter: 30m` is the wait before a scale-down candidate is acted on. Lower it
(e.g. `1m` on spot) for faster, cheaper scale-down at the cost of more churn.

---

## Making sure nodes scale DOWN (goal #3)

Consolidation runs automatically. The only reason a node *won't* be removed is one of these
blockers — keep them off nodes you expect to be reclaimed:

1. **An unsatisfiable PodDisruptionBudget** (`minAvailable: 100%` or == replica count). The
   included PDB uses 50%, which is safe.
2. **`karpenter.sh/do-not-disrupt: "true"`** annotation on a pod/node — pins the node.
3. **Naked pods** with no controller (Deployment/ReplicaSet/StatefulSet/Job) — never evicted.
4. **`kube-system` / static pods** with strict PDBs. DaemonSet pods are ignored and don't block.
5. **Hard (`required`) anti-affinity or topology spread** — reduces repacking. The Deployment
   here uses a soft spread (`ScheduleAnyway`) on purpose.

The `10%` budget rounds **up**, so it always allows at least one disruption — it won't freeze
scale-down on a small cluster.

Diagnose a node that lingers:
```bash
kubectl get events --field-selector reason=Unconsolidatable -A
kubectl describe node <node> | sed -n '/Events/,$p'
```
Typical messages: `pdb <ns>/<name> prevents pod evictions`, `can't replace with a lower-priced node`.

---

## Verification

```bash
# Pools and weights
kubectl get nodepools -o custom-columns=NAME:.metadata.name,WEIGHT:.spec.weight

# Watch nodes appear / disappear
kubectl get nodeclaims -w
kubectl get nodes -L karpenter.sh/capacity-type,karpenter.sh/nodepool,topology.kubernetes.io/zone

# Confirm pods landed on SPOT under normal conditions
kubectl get pods -l app=app -o wide
```

### Test fallback (safe)
```bash
kubectl patch nodepool application --type merge -p '{"spec":{"limits":{"cpu":"0"}}}'
kubectl scale deployment app --replicas=12      # expect on-demand claims to appear
kubectl get nodeclaims -w
kubectl patch nodepool application --type merge -p '{"spec":{"limits":{"cpu":"100"}}}'
```

### Test scale-down
```bash
kubectl scale deployment app --replicas=12      # forces new nodes
kubectl scale deployment app --replicas=3       # frees them
kubectl get nodes -w                             # within consolidateAfter, empty nodes drain & vanish
```

---

## Behavior notes

- After a spot reclaim, evicted pods reschedule onto **spot again first** (weight 99); they
  don't "stick" to on-demand. The PDB + `terminationGracePeriodSeconds` protect availability.
- Interruption-sensitive / stateful workloads should NOT use the preferred-spot affinity — give
  them a separate Deployment with a `required` affinity on `karpenter.sh/capacity-type: on-demand`.
- Optional: check the AKSNodeClass `default` isn't pinned to Azure Linux 2.0 (support ended;
  node images scheduled for removal) — pin `imageFamily` to Ubuntu or AzureLinux3 if needed.
