# AKS + Node Auto Provisioning — Spot-First, On-Demand Fallback, Auto Scale-Down

Goals:
1. Schedule workloads on **Spot** capacity first.
2. **Fall back to On-Demand** automatically when spot can't be provisioned or is reclaimed.
3. **Scale nodes down** automatically when empty or underutilized (consolidation).

Instance types:
- **Spot pool** (`spot`, weight 99): `Standard_E2ds_v4`, `Standard_E2ds_v5`, `Standard_E2ds_v6`
- **On-demand pool** (`ondemand`, weight 10): `Standard_E2ds_v5`

```
azuredeploy.json                  (OPTIONAL) ARM template: new VNet + AKS with NAP
01-nodepool-spot.yaml             Karpenter NodePool 'spot'     (weight 99)
02-nodepool-ondemand.yaml         Karpenter NodePool 'ondemand' (weight 10)
03-deployment-app.yaml            Sample workload + PodDisruptionBudget (one file, '---' separated)
```

---

## Part 0 — (OPTIONAL) Deploy a new AKS cluster with the ARM template

**Skip this part if you already have an AKS cluster with NAP enabled.** Jump to Part 1.

The template creates a **new VNet** + AKS cluster with **NAP enabled** (CNI Overlay + Cilium,
AAD/Azure RBAC, workload identity, OIDC), plus a user-assigned identity granted **Network
Contributor** on the VNet (required so the cluster can place nodes in a bring-your-own VNet).

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FMariuszFerdyn%2Faks-spot-first-Node-Auto-Provisioning%2Fmain%2Fazuredeploy.json)

> The `spot`/`ondemand` NodePools are **Karpenter CRDs**, not ARM resources — applied with
> `kubectl` after the cluster exists.

### 0a. Resource group (the region is taken from the RG)
```bash
export RG=aks-nap
az group create -n "$RG" -l westeurope     # region decided HERE; template inherits it
```

### 0b. Pre-flight — the one check that prevents most failures
Restricted subscriptions allow only certain VM sizes, and not every SKU offers zones. Verify
your system SKU is allowed AND see its zones in your region BEFORE deploying:
```bash
az vm list-skus -l "$(az group show -n $RG --query location -o tsv)" \
  --resource-type virtualMachines \
  --query "[?name=='Standard_D2s_v5'].{name:name, zones:locationInfo[0].zones, restrictions:restrictions[].reasonCode}" \
  -o table
```
- Not listed / `NotAvailableForSubscription` -> pick another allowed size (set `systemNodeVmSize`).
- `zones` empty -> leave `availabilityZones: []` (non-zonal). Only set `["1","2","3"]` if all three appear.

The template defaults to `Standard_D2s_v5`, **non-zonal** (`availabilityZones: []`), which is the
combination that works in restricted / zone-limited regions.

### 0c. Deploy
```bash
# validate first (creates nothing)
az deployment group what-if -g "$RG" \
  --template-file azuredeploy.json

az deployment group create -g "$RG" \
  --template-file azuredeploy.json
```

### 0d. Connect
```bash
az aks get-credentials -g "$RG" -n aks-nap --overwrite-existing
kubectl get nodes        # the 2 system 'agentpool' nodes
```

### ARM deploy troubleshooting
| Error | Cause | Fix |
|---|---|---|
| `AvailabilityZoneNotSupported ... supported zones ... are '2'` | SKU offers only some zones in this region/sub | Set `availabilityZones` to the supported set, or `[]` for non-zonal |
| `AvailabilityZoneNotSupported ... are ''` | SKU has no zones here for this subscription | `availabilityZones: []` (non-zonal) |
| `... VM size ... is not allowed in your subscription` | Restricted subscription SKU allowlist | Pick a size from the listed allowed sizes (check 0b) |
| `RoleDefinitionDoesNotExist` | Wrong role GUID | Network Contributor = `4d97b98b-1d4f-4787-a291-c67834d212e7` (already set) |
| `PrincipalNotFound` on the role assignment | Identity not yet replicated in Entra ID | Re-run the same deployment (idempotent) |

---

## Part 1 — Apply the NodePools + workload
```bash
kubectl apply -f 01-nodepool-spot.yaml
kubectl apply -f 02-nodepool-ondemand.yaml
kubectl apply -f 03-deployment-app.yaml      # Deployment + PDB in one file
# (NodePool/workload files only -- do NOT 'kubectl apply -f azuredeploy.json')

kubectl get nodepools -o custom-columns=NAME:.metadata.name,WEIGHT:.spec.weight
# spot 99 / ondemand 10  (default & system-surge are the built-in NAP pools)
```

How it behaves:
- `weight` makes the **spot** pool tried first; **ondemand** is the fallback.
- Pods carry the **spot toleration** (mandatory -- AKS taints spot nodes) and a **required**
  affinity on `karpenter.sh/capacity-type` so they run only on NAP nodes (never the agentpool).
- `WhenEmptyOrUnderutilized` consolidation scales nodes down when idle.

Before applying, edit `03`: set your container `image:` and tag.

---

## Part 2 — Checks & verification

### 2a. Spot (Low-priority) quota in your region  <- #1 cause of "no spot node"
```bash
az vm list-usage -l "$REGION" -o table | grep -i -E "spot|low"
# Columns: Name CurrentValue Limit.  Limit 0 => spot can never provision.
# Each E2ds node = 2 vCPU, so Limit must be >= 2 per spot node. Quota is PER REGION.
```

### 2b. Watch provisioning (spot first)
```bash
kubectl get pods -l app=app -o wide
sleep 15                       # Wait 15 seconds
kubectl rollout status deploy/app
kubectl get nodes -L karpenter.sh/capacity-type,karpenter.sh/nodepool
kubectl get nodeclaims -w      # TYPE Standard_E2ds_*, CAPACITY spot expected first
```

### 2c. Test on-demand fallback (safe)
```bash
kubectl patch nodepool spot --type merge -p '{"spec":{"limits":{"cpu":"0"}}}'   # block spot
kubectl scale deployment app --replicas=8
sleep 35                         # Wait 35 seconds
kubectl get nodeclaims -w        # expect capacity-type=on-demand from 'ondemand'
kubectl patch nodepool spot --type merge -p '{"spec":{"limits":{"cpu":"100"}}}' # revert
kubectl scale deployment app --replicas=3
sleep 135                        # Wait 135 seconds
kubectl get nodeclaims -w        # shoud return to previous state
```

---

## Workload troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Nodeclaims `on-demand`, never `spot` | Spot quota too low / unavailable, or zone spread forcing more nodes than quota | Check 2a; remove `topologySpreadConstraints`; widen spot SKUs |
| Pods stay on `aks-agentpool-*` | Deployment lacks the required `karpenter.sh/capacity-type` affinity | Re-apply `03` |
| `ImagePullBackOff ... not found` | Image tag doesn't exist | Set a real tag in `03` |
| `ImagePullBackOff ... i/o timeout / 403` | Cluster egress to registry blocked | Import to ACR, attach ACR, reference `<acr>.azurecr.io/...` |
| Node won't scale down | PDB at 100%/==replicas, `do-not-disrupt`, naked pods, strict kube-system PDB | `kubectl get events --field-selector reason=Unconsolidatable -A` |

### Consolidation blockers (keep off nodes you want reclaimed)
PDB at `minAvailable: 100%` or == replicas; `karpenter.sh/do-not-disrupt: "true"`; naked pods
(no controller); strict PDBs on `kube-system` pods (DaemonSets are ignored); hard `required`
anti-affinity / topology spread. The `10%` budget rounds up, so it never freezes scale-down.

---

## Notes
- **Zone-redundancy** depends on the SKU + subscription + region. Where zones exist, set
  `availabilityZones: ["1","2","3"]`; where they don't (restricted subs / capacity), non-zonal
  is the only option. Verify with the check in 0b.
- **Spot capacity** for the `spot`/`ondemand` pools follows the same regional reality as the
  system SKU. If a region is zone/SKU-starved, consider a region (e.g. westeurope) that offers
  more -- quota and capacity are both per-region.
- Interruption-sensitive/stateful workloads: separate Deployment with a `required` affinity on
  `karpenter.sh/capacity-type: on-demand` (don't run them on spot).

---

## Contribution rules
- Keep this README updated whenever implementation changes are made.
- Validate JSON and YAML files before committing.

```bash
# JSON
python -m json.tool azuredeploy.json >/dev/null

# YAML (syntax check)
ruby -e 'require "psych"; ARGV.each { |f| Psych.parse_stream(File.read(f)) }' \
  01-nodepool-spot.yaml 02-nodepool-ondemand.yaml 03-deployment-app.yaml
```
