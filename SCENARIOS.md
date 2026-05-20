# Fleet Managed Namespace Test Scenarios

Test scenarios for `az fleet namespace` commands. Each scenario is independent — delete the namespace between scenarios to start clean.

## Prerequisites

Complete the setup steps in [README.md](README.md) up to and including the RBAC role assignments. Ensure you have:

```bash
export GROUP="$USER-testgroup"
export FLEET="$USER-testfleet"
export MANAGED_NAMESPACE="test-ns"
```

---

## Scenario 1: Basic create with member clusters

**Goal:** Verify namespace is created with PickFixed placement and propagated to specified clusters.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --delete-policy Delete --adoption-policy Never
```

**Verify:**

```bash
# ARM resource exists
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE -o table

# CRP is PickFixed with both clusters
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.policy"

# Namespace exists on hub
kubectl get ns $MANAGED_NAMESPACE

# Namespace propagated to members (may take a minute)
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE

az aks get-credentials -g $GROUP -n contoso-prd-02-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE
```

**Cleanup:**

```bash
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- ARM resource created with `PickFixed`, both clusters listed
- Namespace `test-ns` Active on hub (29s), member-01 (33s), member-02 (38s)
- Delete succeeded

---

## Scenario 2: Create with all optional properties

**Goal:** Verify all optional properties (labels, annotations, quota, network policy) are set correctly.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --labels "team=platform env=staging" \
  --annotations "owner=team-alpha contact=oncall@example.com" \
  --cpu-requests 500m --cpu-limits 2000m \
  --memory-requests 256Mi --memory-limits 1Gi \
  --ingress-policy DenyAll --egress-policy AllowSameNamespace \
  --delete-policy Keep --adoption-policy Always
```

**Verify:**

```bash
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.managedNamespaceProperties"

# Check ResourceQuota on member
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get resourcequota -n $MANAGED_NAMESPACE
kubectl get networkpolicy -n $MANAGED_NAMESPACE
```

**Cleanup:**

```bash
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- All properties set correctly: labels (team=platform, env=staging), annotations (owner, contact), quota (500m/2000m CPU, 256Mi/1Gi memory), network policy (DenyAll ingress, AllowSameNamespace egress), adoptionPolicy=Always, deletePolicy=Keep
- ResourceQuota created on member: `requests.cpu: 0/500m, requests.memory: 0/256Mi, limits.cpu: 0/2, limits.memory: 0/1Gi`
- NetworkPolicy `default` created on member
- Delete succeeded

---

## Scenario 3: Create without member clusters (should fail or create hub-only)

**Goal:** Verify behavior when no member clusters are specified and no rollout strategy is set.

```bash
# No member clusters, no rollout strategy — should create hub-only namespace (no CRP)
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --delete-policy Delete --adoption-policy Never
```

**Verify:**

```bash
# Namespace exists on hub
kubectl get ns $MANAGED_NAMESPACE

# No CRP should exist for this namespace
kubectl get clusterresourceplacement | grep $MANAGED_NAMESPACE

# Now add members via update — should create PickFixed CRP
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm

az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.policy"
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- Create without `--member-cluster-names` correctly blocked with: `--member-cluster-names is required for creating a managed namespace.`
- No ARM resource, namespace, or CRP was created
- Note: This validation is in the preview wheel (v2.1.0). The behavior may change based on PM decision (see README for options).

---

## Scenario 4: Update member cluster list

**Goal:** Verify adding and removing member clusters via update.

```bash
# Create with one member
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never

# Add second member
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm
```

**Verify:**

```bash
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.policy.clusterNames"
# Should show both clusters
```

```bash
# Remove first member (only keep second)
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-02-fm

az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.policy.clusterNames"
# Should show only contoso-prd-02-fm
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- Created with one member (`contoso-prd-01-fm`), added second via update → both listed
- Removed first member via update → only `contoso-prd-02-fm` listed
- Add/remove member cluster list works correctly

---

## Scenario 5: Switch to External rollout strategy

**Goal:** Verify switching from default (RollingUpdate) to External rollout strategy using `--rollout-update-strategy`.

```bash
# Create with default rollout (RollingUpdate is inferred when --rollout-update-strategy is omitted)
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --delete-policy Delete --adoption-policy Never

# Create the strategy on the hub
kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: test-strategy
spec:
  stages:
    - name: dev
      labelSelector:
        matchLabels:
          environment: team-alpha-development
      afterStageTasks:
        - type: TimedWait
          waitTime: 30s
    - name: prod
      labelSelector:
        matchLabels:
          environment: team-alpha-production
EOF

# Switch to External (inferred from --rollout-update-strategy)
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --rollout-update-strategy test-strategy
```

**Verify:**

```bash
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.rolloutStrategy"
# Should show type=External, clusterUpdateStrategy.name=test-strategy
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
kubectl delete clusterstagedupdatestrategy test-strategy
```

**Result: PASS** (tested 2026-05-20)
- Created with default rollout (RollingUpdate), switched to External with `--rollout-update-strategy test-strategy`
- Verified: `type=External`, `clusterUpdateStrategy.name=test-strategy`
- Note: `--rollout-update-strategy` is a preview argument; `--rollout-strategy` has been removed (rollout type is now inferred)

---

## Scenario 6: External strategy is preserved across non-rollout updates

**Goal:** Verify that once External is set, updating other fields (tags, member clusters, labels) preserves the External rollout strategy.

```bash
# Create and switch to External (use steps from Scenario 5)
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never

kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: test-strategy
spec:
  stages:
    - name: all
      labelSelector:
        matchLabels:
          environment: team-alpha-development
EOF

az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --rollout-update-strategy test-strategy

# Update tags — should preserve External
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --tags env=test

# Update member clusters — should preserve External
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm
```

**Verify:**

```bash
# After each update, External strategy should be preserved
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.rolloutStrategy"
# Should show type=External, clusterUpdateStrategy.name=test-strategy
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
kubectl delete clusterstagedupdatestrategy test-strategy
```

**Result: PASS** (tested 2026-05-20)
- External strategy preserved after `--tags` update
- External strategy preserved after `--member-cluster-names` update
- Note: The CLI no longer exposes `--rollout-strategy RollingUpdate` directly. Rollout type is inferred: omitting `--rollout-update-strategy` preserves the existing strategy on update; providing it sets External.

---

## Scenario 7: Update only labels and quota (no propagation change)

**Goal:** Verify partial updates don't affect propagation policy or member clusters.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --labels "team=original" \
  --cpu-requests 100m \
  --delete-policy Delete --adoption-policy Never

# Update only labels and quota
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --labels "team=updated env=production" \
  --cpu-requests 500m --cpu-limits 1000m
```

**Verify:**

```bash
# Labels updated
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.managedNamespaceProperties.labels"

# Quota updated
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.managedNamespaceProperties.defaultResourceQuota"

# Member clusters unchanged
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.policy.clusterNames"
# Should still be both clusters
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- Labels updated: `team=updated`, `env=production` (was `team=original`)
- Quota updated: `cpuRequest=500m`, `cpuLimit=1000m`
- Member clusters unchanged: both `contoso-prd-01-fm` and `contoso-prd-02-fm` still listed

---

## Scenario 8: Delete with Keep policy

**Goal:** Verify that deleting a namespace with Keep policy leaves the Kubernetes namespace on clusters.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Keep --adoption-policy Never

# Wait for propagation
sleep 30

# Delete the managed namespace
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Verify:**

```bash
# ARM resource gone
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE
# Should return ResourceNotFound

# Kubernetes namespace still exists on member
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE
# Should still exist

# Manual cleanup
kubectl delete ns $MANAGED_NAMESPACE
```

**Result: PASS** (tested 2026-05-07)
- ARM resource deleted (ResourceNotFound)
- Kubernetes namespace `test-ns` still Active on member cluster (Keep policy preserved it)
- Note: When reusing the same namespace name after Keep delete, use `--adoption-policy Always` or wait for full cleanup

---

## Scenario 9: Delete with Delete policy

**Goal:** Verify that deleting a namespace with Delete policy removes the Kubernetes namespace from clusters.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never

# Wait for propagation
sleep 30

# Delete the managed namespace
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Verify:**

```bash
# Kubernetes namespace removed from member (may take a minute)
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE
# Should return NotFound
```

**Result: PASS** (tested 2026-05-07)
- Namespace `test-ns-9` removed from member after delete (NotFound)
- Delete policy correctly cleaned up K8s namespace on member cluster

---

## Scenario 10: Get namespace-scoped credentials

**Goal:** Verify `az fleet namespace get-credentials` sets the default namespace in kubeconfig.

```bash
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never

# Get hub credentials scoped to namespace
az fleet namespace get-credentials \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE

# Get member credentials scoped to namespace
az fleet namespace get-credentials \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member contoso-prd-01-fm
```

**Verify:**

```bash
kubectl config view --minify -o jsonpath='{.contexts[0].context.namespace}'
# Should output: test-ns
```

**Cleanup:**

```bash
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- Hub credentials: default namespace set to `test-ns-10`
- Member credentials: default namespace set to `test-ns-10`
- Both `kubectl config view` confirmed correct namespace

---

## Scenario 11: List and show namespaces

**Goal:** Verify list and show commands work correctly with multiple namespaces.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-ns-1 \
  --member-cluster-names contoso-prd-01-fm --delete-policy Delete --adoption-policy Never
az fleet namespace create -g $GROUP -f $FLEET -n test-ns-2 \
  --member-cluster-names contoso-prd-02-fm --delete-policy Delete --adoption-policy Never

# List all
az fleet namespace list -g $GROUP -f $FLEET -o table

# Show specific
az fleet namespace show -g $GROUP -f $FLEET -n test-ns-1 -o table
az fleet namespace show -g $GROUP -f $FLEET -n test-ns-2 -o table
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n test-ns-1 --yes
az fleet namespace delete -g $GROUP -f $FLEET -n test-ns-2 --yes
```

**Result: PASS** (tested 2026-05-07)
- List returned all 3 namespaces (test-ns-10, test-ns-11a, test-ns-11b) in table format
- Show returned correct details for each individual namespace

---

## Scenario 12: Re-create namespace after Keep-delete using adoption policy

**Goal:** Verify that after deleting a namespace with Keep policy, you can re-create it with `--adoption-policy Always` to adopt the existing K8s namespace.

```bash
# Step 1: Create with Keep policy
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Keep --adoption-policy Never

# Wait for propagation
sleep 30

# Step 2: Delete (K8s namespace preserved)
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Verify namespace still exists on hub and member:**

```bash
# Hub
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE

# Member
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE
```

```bash
# Step 3: Re-create with adoption-policy Always (should succeed)
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Always
```

**Verify:**

```bash
# ARM resource exists again
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE -o table

# Namespace still Active on member
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns $MANAGED_NAMESPACE
```

```bash
# Step 4: Also verify that adoption-policy Never would fail
# (Don't run this after step 3 — only for reference)
# az fleet namespace create -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
#   --member-cluster-names contoso-prd-01-fm \
#   --delete-policy Delete --adoption-policy Never
# Expected error: AdoptionNotPossible
```

**Cleanup:**

```bash
az fleet get-credentials -g $GROUP -n $FLEET --overwrite-existing
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-07)
- Created with Keep policy, deleted — K8s namespace preserved on hub and member
- Re-created with `--adoption-policy Always` — succeeded, ARM resource recreated
- Namespace remained Active on member throughout (age 71s at verification)
- Confirms the workaround for the Keep-delete + re-create flow

---

## Scenario 13: Create with External rollout strategy directly

**Goal:** Verify creating a namespace with External rollout strategy at creation time (not via update).

```bash
kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: s13-strategy
spec:
  stages:
    - name: dev
      labelSelector:
        matchLabels:
          environment: team-alpha-development
      afterStageTasks:
        - type: TimedWait
          waitTime: 30s
    - name: prod
      labelSelector:
        matchLabels:
          environment: team-alpha-production
EOF

az fleet namespace create \
  -g $GROUP -f $FLEET -n test-s13 \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --rollout-update-strategy s13-strategy \
  --delete-policy Delete --adoption-policy Never
```

**Verify:**

```bash
az fleet namespace show -g $GROUP -f $FLEET -n test-s13 \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.rolloutStrategy"
# Should show type=External, clusterUpdateStrategy.name=s13-strategy
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n test-s13 --yes
kubectl delete clusterstagedupdatestrategy s13-strategy
```

**Result: PASS** (tested 2026-05-07, re-tested 2026-05-20 with `--rollout-update-strategy`)
- Created with PickFixed + External rollout + strategy reference in a single create command
- Verified: `placementType=PickFixed`, `rolloutStrategy.type=External`, `clusterUpdateStrategy.name=s13-strategy`

---

## Scenario 14: Switch between update strategies on an External namespace

**Goal:** Verify switching from one `ClusterStagedUpdateStrategy` to another on an existing External namespace.

```bash
# Step 1: Create strategies on the hub
kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: strategy-a
spec:
  stages:
    - name: dev
      labelSelector:
        matchLabels:
          environment: team-alpha-development
EOF

kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: strategy-b
spec:
  stages:
    - name: prod
      labelSelector:
        matchLabels:
          environment: team-alpha-production
EOF

# Step 2: Create namespace with strategy-a
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --rollout-update-strategy strategy-a \
  --delete-policy Delete --adoption-policy Never

# Step 3: Switch to strategy-b
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --rollout-update-strategy strategy-b
```

**Verify:**

```bash
# After step 2
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.rolloutStrategy"
# Should show type=External, clusterUpdateStrategy.name=strategy-a

# After step 3
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy.placementProfile.defaultClusterResourcePlacement.rolloutStrategy"
# Should show type=External, clusterUpdateStrategy.name=strategy-b
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
kubectl delete clusterstagedupdatestrategy strategy-a strategy-b
```

**Result: PASS** (tested 2026-05-20)
- Created with `strategy-a`: `type=External`, `clusterUpdateStrategy.name=strategy-a`
- Switched to `strategy-b`: `type=External`, `clusterUpdateStrategy.name=strategy-b`
- Note: The old Scenario 14 bug (strategy name silently ignored) is fixed — `--rollout-update-strategy` now sets both the strategy reference and the External type in one parameter

---

## Scenario 15: adoption-policy IfIdentical

**Goal:** Verify that `--adoption-policy IfIdentical` adopts a namespace only when labels/annotations match.

```bash
# Create with labels, delete with Keep, re-create with IfIdentical and same labels
az fleet namespace create -g $GROUP -f $FLEET -n test-s15 \
  --member-cluster-names contoso-prd-01-fm \
  --labels "team=alpha" \
  --delete-policy Keep --adoption-policy Never

az fleet namespace delete -g $GROUP -f $FLEET -n test-s15 --yes

az fleet namespace create -g $GROUP -f $FLEET -n test-s15 \
  --member-cluster-names contoso-prd-01-fm \
  --labels "team=alpha" \
  --delete-policy Delete --adoption-policy IfIdentical
```

**Result: FAIL (possible bug or expected behavior)** (tested 2026-05-07)
- `IfIdentical` rejected the adoption: `AdoptionNotPossible: resources are not identical`
- Even though user-specified labels match (`team=alpha`), the existing K8s namespace has additional system labels (e.g., `fleet.azure.com/managed-by: arm`) from the previous managed state
- The server considers the resources "not identical" due to these extra labels
- **Question for PM:** Is this expected? Should `IfIdentical` only compare user-specified labels, or all labels including system-managed ones?

---

## Scenario 16: Invalid inputs

**Goal:** Verify error handling for various invalid inputs.

### Non-existent member cluster name

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s16 \
  --member-cluster-names fake-cluster-name \
  --delete-policy Delete --adoption-policy Never
```

**Result: UNEXPECTED** — Server accepted `fake-cluster-name` without validation. The namespace was created with a PickFixed CRP referencing a non-existent cluster. This may be by design (eventual consistency) or a validation gap.

### Non-existent strategy name

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s16b \
  --member-cluster-names contoso-prd-01-fm \
  --rollout-strategy External \
  --cluster-update-strategy nonexistent-strategy \
  --delete-policy Delete --adoption-policy Never
```

**Result: PASS** — Server correctly rejected: `cluster staged update strategy does not exist on the hub cluster`

### Invalid namespace name

```bash
az fleet namespace create -g $GROUP -f $FLEET -n "INVALID_NAME!" \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never
```

**Result: PASS** — Server correctly rejected: `namespace name must start/end with a lowercase letter or digit, and contain only lowercase letters, digits, or hyphens`

---

## Scenario 17: az fleet namespace wait

**Goal:** Verify `az fleet namespace wait` works as a workaround for the LRO polling bug.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s17 \
  --member-cluster-names contoso-prd-01-fm \
  --delete-policy Delete --adoption-policy Never

az fleet namespace wait -g $GROUP -f $FLEET -n test-s17 --created

az fleet namespace show -g $GROUP -f $FLEET -n test-s17 --query "properties.provisioningState"
```

**Result: PASS** (tested 2026-05-07)
- `create` returned in ~2s with `provisioningState: Creating` (LRO bug)
- `wait --created` returned in <1s (provisioning completed fast)
- `show` confirmed `provisioningState: Succeeded`
- `wait` is a valid workaround for the LRO bug when scripts need to ensure completion

---

## Scenario 18: Remove all members via update

**Goal:** Verify behavior when updating with empty `--member-cluster-names`.

```bash
az fleet namespace update \
  -g $GROUP -f $FLEET -n test-s17 \
  --member-cluster-names
```

**Result: NO-OP** (tested 2026-05-07, re-tested 2026-05-20)
- `--member-cluster-names` with no values results in an empty list `[]`
- `_build_propagation_policy` sees empty list as falsy, returns `None`
- PATCH sends no propagation policy change — existing cluster list unchanged
- **Root cause:** PATCH omits `propagationPolicy` when `None`; server interprets omission as "don't change"
- **Workaround:** Use `az fleet namespace create` (PUT) with `--adoption-policy Always` and no `--member-cluster-names` to overwrite the resource and clear the propagation policy (see Scenario 18b)

---

## Scenario 18b: Remove all members via PUT (create --adoption-policy Always)

**Goal:** Verify that re-creating a namespace without `--member-cluster-names` removes the propagation policy (hub-only).

```bash
# Step 1: Create with members
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm \
  --delete-policy Delete --adoption-policy Never

# Step 2: Re-create (PUT) without members — clears propagation policy
az fleet namespace create \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --delete-policy Delete --adoption-policy Always
```

**Verify:**

```bash
az fleet namespace show -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --query "properties.propagationPolicy"
# Should be null
```

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
```

**Result: PASS** (tested 2026-05-20)
- Created with two members (`contoso-prd-01-fm`, `contoso-prd-02-fm`)
- Re-created via PUT with `--adoption-policy Always` and no `--member-cluster-names`
- `propagationPolicy` is now `null` — members removed, namespace is hub-only
- Note: PUT replaces the full resource, so `propagationPolicy=None` means "no policy". PATCH omits `None` fields, so it can't clear the policy.

---

## Scenario 19: Create with reserved/system namespace name

**Goal:** Verify server rejects reserved namespace names like `kube-system`, `default`.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n kube-system \
  --member-cluster-names contoso-prd-01-fm --delete-policy Delete --adoption-policy Never

az fleet namespace create -g $GROUP -f $FLEET -n default \
  --member-cluster-names contoso-prd-01-fm --delete-policy Delete --adoption-policy Never
```

**Result: PASS** (tested 2026-05-07)
- Both rejected: `namespace name must not match a reserved pattern`
- Reserved patterns: `default`, `kube-*`, `azure-arc*`, `fleet-*`, `*-system`, `cert-manager`

---

## Scenario 20: Update adoption and delete policy

**Goal:** Verify adoption and delete policies can be changed after creation.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s20 \
  --member-cluster-names contoso-prd-01-fm --delete-policy Keep --adoption-policy Never

az fleet namespace update -g $GROUP -f $FLEET -n test-s20 \
  --delete-policy Delete --adoption-policy Always

az fleet namespace show -g $GROUP -f $FLEET -n test-s20 \
  --query "{adoptionPolicy: properties.adoptionPolicy, deletePolicy: properties.deletePolicy}"
```

**Result: PASS** (tested 2026-05-07)
- Changed from `Keep/Never` to `Delete/Always` successfully
- Verified via show: `adoptionPolicy=Always`, `deletePolicy=Delete`

---

## Scenario 21: Create with annotations only, no labels

**Goal:** Verify annotations propagate to member clusters independently of labels.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s21 \
  --member-cluster-names contoso-prd-01-fm \
  --annotations "owner=team-alpha contact=oncall" \
  --delete-policy Delete --adoption-policy Never
```

**Verify:**

```bash
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns test-s21 -o jsonpath='{.metadata.annotations}'
```

**Result: PASS** (tested 2026-05-07)
- Annotations `owner=team-alpha` and `contact=oncall` propagated to member cluster namespace
- Fleet system annotations also present (`kubernetes-fleet.io/spec-hash`, etc.)

---

## Scenario 22: get-credentials with invalid member name

**Goal:** Verify clear error when using a non-existent member name with get-credentials.

```bash
az fleet namespace get-credentials \
  -g $GROUP -f $FLEET -n test-s21 --member fake-member
```

**Result: PASS** (tested 2026-05-07)
- Clear error: `Error getting credentials for fleet member 'fake-member': ResourceNotFound`

---

## Scenario 23: Two namespaces on same member cluster

**Goal:** Verify multiple managed namespaces can coexist on the same member cluster.

```bash
az fleet namespace create -g $GROUP -f $FLEET -n test-s23a \
  --member-cluster-names contoso-prd-01-fm --delete-policy Delete --adoption-policy Never

az fleet namespace create -g $GROUP -f $FLEET -n test-s23b \
  --member-cluster-names contoso-prd-01-fm --delete-policy Delete --adoption-policy Never
```

**Verify:**

```bash
az aks get-credentials -g $GROUP -n contoso-prd-01-fm --overwrite-existing
kubectl get ns test-s23a test-s23b
```

**Result: PASS** (tested 2026-05-07)
- Both namespaces Active on the same member cluster
- `test-s23a` (19s) and `test-s23b` (18s) coexist without conflict
