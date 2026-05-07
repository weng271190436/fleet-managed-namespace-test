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

---

## Scenario 5: Switch to External rollout strategy

**Goal:** Verify switching from default (RollingUpdate) to External rollout strategy.

```bash
# Create with default rollout
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

# Switch to External
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --rollout-strategy External \
  --cluster-update-strategy test-strategy
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

---

## Scenario 6: Cannot switch back from External to RollingUpdate

**Goal:** Verify that once External is set, you cannot switch back to RollingUpdate.

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
  --rollout-strategy External \
  --cluster-update-strategy test-strategy

# Try switching back to RollingUpdate — should fail
az fleet namespace update \
  -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE \
  --rollout-strategy RollingUpdate
```

**Expected:** Error indicating you can't switch from External back to RollingUpdate.

**Cleanup:**

```bash
az fleet namespace delete -g $GROUP -f $FLEET -n $MANAGED_NAMESPACE --yes
kubectl delete clusterstagedupdatestrategy test-strategy
```

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
