# AKS Upgrade Guide

## Prerequisites

#### 1. *Cluster Access and Permissions:* Ensure you have Owner or Contributor permissions on the AKS cluster.
#### 2. IP Address Availability Check:

- Use Azure CLI to check available IPs for Azure CNI:
```bash
az network vnet list --query "[].{Name:name, AvailableIPs:ipConfigurations[].privateIpAddress | length(@)}"
```

- Confirm that there are enough IPs available in the subnet to accommodate the additional node pool.

#### 3. Compute Resource Verification:
- Ensure sufficient resources in your Azure subscription to handle the new node pool:
    - Go to `Azure Portal > Subscriptions > Usage + quotas`.
    - Verify you have enough compute resources, like vCPU and memory, to provision a new node pool of your desired size.
   
#### 4. Backup AKS Cluster Configuration:
- It’s good practice to back up all configuration files:
```bash
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml
```
## Upgrade Plan: Control Plane and Node Pools

#### 1. Control Plane Upgrade
- Start by upgrading the AKS control plane to version 1.30.4:
```bash
az aks upgrade --resource-group <resource-group> --name <aks-cluster-name> --kubernetes-version 1.30.4 --control-plane-only
```
- Monitor the control plane upgrade process to confirm successful completion:
```bash
az aks show --resource-group <resource-group> --name <aks-cluster-name> --query "kubernetesVersion"
```
- Verify the control plane upgrade by checking the API server status:
```bash
kubectl version --short
```
#### 2. New Node Pool Creation (for Zero Downtime)

- Create a new node pool with version 1.30.4 to avoid disrupting running workloads:
```bash
az aks nodepool add --resource-group <resource-group> --cluster-name <aks-cluster-name> --name <new-nodepool-name> --node-count <desired-count> --kubernetes-version 1.30.4
```

- Ensure that the new node pool has autoscaling enabled and desired size:
```bash
az aks nodepool update --resource-group <resource-group> --cluster-name <aks-cluster-name> --name <new-nodepool-name> --enable-cluster-autoscaler --min-count <min-count> --max-count <max-count>
```

#### 3. Cordoning and Draining Old Node Pools

- Cordon nodes in the old node pool to prevent new workloads from being scheduled:
```bash
kubectl cordon <node-name>
```

- Drain workloads from each node in the old pool:
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
- Repeat this for each node in the old node pool.
#### 4. Migrating Workloads to New Node Pool

- Check that all workloads are successfully rescheduled onto the new node pool.
- Verify service availability and no downtime during the transition:

```bash
kubectl get pods --all-namespaces -o wide
```
#### 5. Deleting the Old Node Pool
- After confirming that all workloads are running smoothly on the new node pool, delete the old node pool:

```bash
az aks nodepool delete --resource-group <resource-group> --cluster-name <aks-cluster-name> --name <old-nodepool-name>
```

## Verification Steps Post-Upgrade
#### 1. Cluster Status Check

- Check the cluster health and ensure all nodes are running on version 1.30.4:
```bash
kubectl get nodes -o wide
```
#### 2. Application Verification

- Test the key application workflows to ensure no downtime occurred.
- Run any test scripts or manual checks for critical applications.

## Rollback Steps
If any issues arise during the upgrade, you can perform the following rollback steps:

#### 1. Rollback Node Pool

- Delete the new node pool if workloads need to return to the old node pool:
```bash 
az aks nodepool delete --resource-group <resource-group> --cluster-name <aks-cluster-name> --name <new-nodepool-name>
```
- Uncordon and re-enable scheduling on the old node pool:
```bash
kubectl uncordon <node-name>
```
#### 2. Revert Control Plane Version (if supported by AKS)

- AKS may allow downgrades; confirm with Azure Support, as control plane version downgrades are not always possible.

#### 3. Restore from Backup (if necessary)

- If needed, restore configurations from the backup file:
```bash
kubectl apply -f cluster-backup.yaml
```