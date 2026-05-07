# Fleet Managed Namespace E2E Test Steps

End-to-end test steps for Azure Fleet Manager Managed Namespaces using a preview CLI extension.

## Prerequisites

- Azure CLI 2.85.0+
- Access to subscription `c0f60687-8f09-4186-801b-9dd11d82d2e1`

## Install preview Fleet CLI extension

```bash
az extension remove --name fleet 2>/dev/null
curl -L -o /tmp/fleet-2.1.0-py3-none-any.whl "https://weiwengstorage.blob.core.windows.net/fleet-cli/fleet-2.1.0-py3-none-any.whl?se=2026-06-06T17%3A06Z&sp=r&sv=2026-02-06&sr=b&sig=9L3yHg%2B3cJgMEFCcVdJTQrH54xbz5xJoLRspksIYmuc%3D"
az extension add --source /tmp/fleet-2.1.0-py3-none-any.whl -y
```

## Set up environment

```bash
az account set -s c0f60687-8f09-4186-801b-9dd11d82d2e1

LOCATION="westcentralus"
GROUP="$USER-testgroup"
FLEET="$USER-testfleet"
COUNT=2
MANAGED_NAMESPACE=my-managed-namespace
```

## Create resource group, fleet, and AKS clusters

```bash
az group create -l "$LOCATION" -n "$GROUP"

az fleet create -g "$GROUP" -n "$FLEET" --enable-hub &
for ((i=1; i<=COUNT; i++)); do
  az aks create -g "$GROUP" -n "contoso-prd-0$i-fm" \
    --enable-aad --enable-azure-rbac \
    --node-count 1 --node-vm-size standard_a2_v2_gen2 --no-ssh-key &
done
wait
```

## Join AKS clusters to fleet

```bash
az aks list -g "$GROUP" -o tsv

for i in $(az aks list -g "$GROUP" -o tsv --query '[].id'); do
  az fleet member create -g "$GROUP" -f "$FLEET" -n "${i##*/}" --member-cluster-id "$i" &
done
wait

# Verify all members joined
az fleet member list -g "$GROUP" -f "$FLEET" -o tsv
```

## Assign RBAC roles

```bash
PRINCIPAL_ID=$(az ad signed-in-user show --query "id" --output tsv)
FLEET_SCOPE="/subscriptions/c0f60687-8f09-4186-801b-9dd11d82d2e1/resourceGroups/$GROUP/providers/Microsoft.ContainerService/fleets/$FLEET"

az role assignment create \
  --role "Azure Kubernetes Fleet Manager RBAC Cluster Admin" \
  --assignee "$PRINCIPAL_ID" --scope "$FLEET_SCOPE"

az role assignment create \
  --role "Azure Kubernetes Fleet Manager RBAC Cluster Admin for Member Clusters" \
  --assignee "$PRINCIPAL_ID" --scope "$FLEET_SCOPE"
```

## Label member clusters

```bash
az fleet member update -g $GROUP -f $FLEET -n contoso-prd-01-fm \
  --labels "environment=team-alpha-development"

az fleet member update -g $GROUP -f $FLEET -n contoso-prd-02-fm \
  --labels "environment=team-alpha-production"
```

## Create a managed namespace

```bash
az fleet namespace create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE \
  --annotations "annotation1=value1 annotation2=value2" \
  --labels "team=my-team label2=value2" \
  --cpu-requests 1m \
  --cpu-limits 4m \
  --memory-requests 1Mi \
  --memory-limits 4Mi \
  --ingress-policy AllowAll \
  --egress-policy AllowAll \
  --delete-policy Delete \
  --adoption-policy Never \
  --member-cluster-names contoso-prd-01-fm contoso-prd-02-fm
```

## Get credentials

```bash
az fleet get-credentials --resource-group $GROUP --name $FLEET --overwrite-existing

az aks get-credentials --resource-group $GROUP --name contoso-prd-01-fm --overwrite-existing
az aks get-credentials --resource-group $GROUP --name contoso-prd-02-fm --overwrite-existing
```

## Create a ClusterStagedUpdateStrategy

```bash
kubectl apply -f - <<EOF
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterStagedUpdateStrategy
metadata:
  name: my-update-strategy
spec:
  stages:
    - name: development
      labelSelector:
        matchLabels:
          environment: team-alpha-development
      afterStageTasks:
        - type: TimedWait
          waitTime: 1m
    - name: production
      labelSelector:
        matchLabels:
          environment: team-alpha-production
EOF
```

## Switch to external rollout strategy

```bash
az fleet namespace update \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE \
  --rollout-strategy External \
  --cluster-update-strategy my-update-strategy

az fleet namespace show \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE
```

## Update namespace properties

```bash
az fleet namespace update \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE \
  --annotations "annotation1=value1 annotation2=value2" \
  --labels "team=myTeam label2=value2" \
  --cpu-requests 1m \
  --cpu-limits 4m \
  --memory-requests 1Mi \
  --memory-limits 4Mi \
  --ingress-policy AllowAll \
  --egress-policy AllowAll \
  --delete-policy Delete \
  --adoption-policy Never
```

## Update labels only

```bash
az fleet namespace update \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE \
  --labels "team=team-alpha label2=value2"
```

## Delete managed namespace

```bash
az fleet namespace delete \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $MANAGED_NAMESPACE
```

## Cleanup

```bash
az group delete -n "$GROUP" --yes --no-wait
```
