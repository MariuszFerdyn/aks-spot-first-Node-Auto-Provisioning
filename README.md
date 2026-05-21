# AKS Node Auto-Provisioning — Spot-First, On-Demand Fallback, Auto Scale-Down

Goals:
1. Schedule workloads on **Spot** capacity first.
2. **Fall back to On-Demand** automatically when spot can't be provisioned or is reclaimed.
3. **Scale nodes down** automatically when empty or underutilized (consolidation).

Instance types used:
- **Spot pool** (`application`, weight 99): `Standard_E2ds_v4`, `Standard_E2ds_v5`, `Standard_E2ds_v6`
- **On-demand pool** (`application2`, weight 10): `Standard_E2ds_v5`

---

## Files
| File | Purpose |
|------|---------|
| `01-nodepool-application-spot.yaml`      | Spot pool, weight 99 (preferred) |
| `02-nodepool-application2-ondemand.yaml` | On-demand pool, weight 10 (fallback) |
| `03-deployment-app.yaml`                 | Workload: spot toleration + required NAP affinity + preferred-spot |
| `04-pdb-app.yaml`                        | PodDisruptionBudget |

---

## Set your variables first
These commands use placeholders — **export them once** (the region especially must be your own):
```bash
export REGION=westus2          # <-- your cluster region
export RG=<your-resource-group>
export CLUSTER=<your-aks-name>
```

---

## STEP 1 — Pre-flight checks (do these BEFORE applying)

### 1a. NAP is enabled and the pools register later
```bash
az aks show -g "$RG" -n "$CLUSTER" --query "nodeProvisioningProfile.mode" -o tsv
# expect: Auto
```

### 1b. Spot (Low-priority) quota > 0 in YOUR region  ← the #1 cause of "no spot"
```bash
az vm list-usage -l "$REGION" -o table | grep -i -E "spot|low"
# Output columns: Name  CurrentValue  Limit
# Example good result:  Total Regional Low-priority vCPUs   0   99
```
- **Limit 0** => spot can NEVER provision. Request an increase: Portal -> Subscriptions ->
  Usage + quotas -> filter "spot"/"low-priority" -> request for `$REGION`.
- Each `E2ds` node = **2 vCPU**, so Limit must be >= 2 per spot node you expect.

### 1c. (optional) Confirm the SKU offers spot in the region
```bash
az vm list-skus -l "$REGION" --size Standard_E2ds --all \
  --query "[].{name:name, restrictions:restrictions[].reasonCode}" -o table
# A 'NotAvailableForSubscription' restriction = that SKU's capacity is blocked for you now.
```

### 1d. Image is reachable
The Deployment uses `mafamafa/phpinfo-ubuntu:202401182132`. If you change it, confirm the
**exact tag exists** (a missing tag gives `ImagePullBackOff … not found`):
```bash
curl -s "https://registry.hub.docker.com/v2/repositories/mafamafa/phpinfo-ubuntu/tags?page_size=100" \
  | jq -r '.results[].name'
```

---

## STEP 2 — Edit before applying
- `03`: set your container `image:` (and tag).
- `01`/`02`: tune `limits` to your quota if needed.

---

## STEP 3 — Apply
```bash
kubectl apply -f .
```

Verify the pools registered with the right weights:
```bash
kubectl get nodepools -o custom-columns=NAME:.metadata.name,WEIGHT:.spec.weight
# application 99 / application2 10 (default & system-surge are the built-in NAP pools)
```

---

## STEP 4 — Verify scheduling (spot first)
```bash
# Karpenter provisions nodes for the pending pods — watch them appear
kubectl get nodeclaims
#   TYPE = Standard_E2ds_*, CAPACITY = spot (expected first)

# Once READY, confirm pods are on NAP nodes (NOT the agentpool)
kubectl get pods -l app=app -o wide
kubectl get nodes -L karpenter.sh/capacity-type,karpenter.sh/nodepool

kubectl rollout status deploy/app          # should reach "successfully rolled out"
```
Expected: a `capacity-type=spot` node from `nodepool=application`, pods Running on it.

If pods sit `Pending` with `didn't match node affinity/selector` for the agentpool nodes,
that's correct — the required affinity is excluding them while the spot node boots. A fresh
node also carries `node.cilium.io/agent-not-ready:NoExecute` until Cilium is ready; the pod
schedules the moment the node goes Ready.

---

## STEP 5 — Test the on-demand fallback (safe)
```bash
kubectl patch nodepool application --type merge -p '{"spec":{"limits":{"cpu":"0"}}}'  # block spot
kubectl scale deployment app --replicas=8
kubectl get nodeclaims                      # expect capacity-type=on-demand from application2
# revert
kubectl patch nodepool application --type merge -p '{"spec":{"limits":{"cpu":"100"}}}'
kubectl scale deployment app --replicas=3
```

---

## STEP 6 — Test scale-down (consolidation)
```bash
kubectl scale deployment app --replicas=0   # sscale to 0 that will drain all nodes
kubectl get nodes                           # get info about nodes (aftre 3 minutes)
kubectl scale deployment app --replicas=8   # forces extra nodes
# wait for them to come up...
kubectl scale deployment app --replicas=3   # frees capacity
kubectl get nodes                           # within consolidateAfter (1m spot/2m on-demand),
                                             # emptied nodes drain and disappear
```

Verify nothing is blocking consolidation:
```bash
kubectl get events --field-selector reason=Unconsolidatable -A
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Nodeclaims are `on-demand`, never `spot` | Spot quota too low, or zone spread forcing more nodes than quota allows, or spot capacity unavailable | Check Step 1b; remove any `topologySpreadConstraints`; widen SKUs/families |
| Pods stay on `aks-agentpool-*` | Deployment lacks the required `karpenter.sh/capacity-type` affinity | Re-apply `03`; verify with `kubectl get deploy app -o jsonpath='{.spec.template.spec.affinity}'` |
| `ImagePullBackOff … not found` | Image tag doesn't exist | Step 1d; set a real tag in `03` |
| `ImagePullBackOff … i/o timeout / 403` | Cluster egress to registry blocked | Import image to ACR, attach ACR, reference `<acr>.azurecr.io/...` |
| One node per zone appears | `topologySpreadConstraints` still in the live Deployment | Remove it, re-apply `03` |
| Node won't scale down | PDB at 100%/==replicas, `karpenter.sh/do-not-disrupt`, naked pods, or strict PDB on a kube-system pod | `kubectl get events --field-selector reason=Unconsolidatable -A` |

### Consolidation blockers (keep these off nodes you want reclaimed)
1. PDB that can't be satisfied (`minAvailable: 100%` or == replica count).
2. `karpenter.sh/do-not-disrupt: "true"` annotation on a pod/node.
3. Naked pods (no Deployment/ReplicaSet/StatefulSet/Job owner).
4. `kube-system`/static pods with strict PDBs (DaemonSets are ignored, they don't block).
5. Hard (`required`) anti-affinity/topology spread.

The `10%` budget rounds **up**, so it always allows >=1 disruption even on small clusters.

---

## Clean up
```bash
kubectl delete -f 03-deployment-app.yaml -f 04-pdb-app.yaml
kubectl delete nodeclaims --all      # remove NAP nodes immediately instead of waiting
```

## Notes
- After a spot reclaim, evicted pods reschedule onto **spot first** again (weight 99); they
  don't stick to on-demand. PDB + `terminationGracePeriodSeconds` protect availability.
- Interruption-sensitive/stateful workloads: separate Deployment with a `required` affinity on
  `karpenter.sh/capacity-type: on-demand` (do not run them on spot).
- Optional: confirm the AKSNodeClass `default` isn't pinned to Azure Linux 2.0 (support ended).
