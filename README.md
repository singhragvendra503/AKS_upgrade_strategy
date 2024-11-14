# Azure Kubernetes Service (AKS) Upgrade Strategy for Zero Downtime

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Zero-Downtime Upgrade Strategy](#zero-downtime-upgrade-strategy)
4. [Upgrade Control Plane](#upgrade-control-plane)
5. [Create a New Node Pool](#create-a-new-node-pool)
6. [Pod Disruption Budgets (PDBs) for Critical Applications](#pod-disruption-budgets-pdbs-for-critical-applications)
7. [Verification and Rollback Strategy](#verification-and-rollback-strategy)

---

## Overview
This guide provides a structured approach for upgrading an AKS cluster with zero downtime. The focus is on separating control plane and node upgrades to maintain stability and continuity for applications, as well as using Pod Disruption Budgets (PDBs) for critical workloads.

## Prerequisites
- *Azure CLI*: Ensure Azure CLI is installed and updated.
- *Kubernetes CLI (kubectl)*: Install kubectl and ensure it's configured to access your AKS cluster.
- *Azure Account Access*: Sufficient permissions to modify AKS clusters.

## Zero-Downtime Upgrade Strategy

1. *Separate Control Plane and Node Upgrades*: Upgrade the control plane independently of the nodes to minimize potential issues.
2. *Implement Pod Disruption Budgets (PDBs)*: For critical applications, apply PDBs to prevent excessive disruption.
3. *Upgrade Node Pools Sequentially*: Upgrade the node pools one at a time to avoid disrupting the entire application workload.
4. *Create New Node Pool Instead of In-Place Upgrades*: Creating a new node pool and migrating workloads is generally safer and avoids potential issues from in-place upgrades.

## Upgrade Control Plane

1. *Check Current Control Plane Version*:
   ```bash
   az aks show --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME> --query kubernetesVersion -o tsv
   ```

2. *List Available Kubernetes Versions*:
   ```bash
   az aks get-upgrades --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME> --output table
   ```

3. *Upgrade the Control Plane*:
   ```bash
   az aks upgrade --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME> --kubernetes-version <VERSION> --control-plane-only --no-wait
   ```
   > *Note*: Replace <VERSION> with the desired Kubernetes version.

4. *Monitor Control Plane Upgrade Progress*:
   ```bash
   az aks show --resource-group <RESOURCE_GROUP> --name <CLUSTER_NAME> --query "provisioningState"
   ```

## Create a New Node Pool

1. *Create a New Node Pool with the Desired Kubernetes Version*:
   ```bash
   az aks nodepool add --resource-group <RESOURCE_GROUP> --cluster-name <CLUSTER_NAME> --name <NEW_NODE_POOL_NAME> --kubernetes-version <VERSION> --node-count <COUNT>
   ```
   
   > *Note*: Adjust <NEW_NODE_POOL_NAME>, <VERSION>, and <COUNT> to fit your requirements.

2. *Drain and Delete the Old Node Pool* (Optional, if replacing an old pool):
   - *Drain the old node pool*:
     ```bash
     kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
     ```

   - *Delete the old node pool*:
     ```bash
     az aks nodepool delete --resource-group <RESOURCE_GROUP> --cluster-name <CLUSTER_NAME> --name <OLD_NODE_POOL_NAME>
     ```


3. *Verify New Node Pool Status*:
   ```bash
   kubectl get nodes
   ```


## Pod Disruption Budgets (PDBs) for Critical Applications

To minimize disruption for critical workloads, apply PDBs as follows:

1. *Define a Pod Disruption Budget*:
   - Example YAML file (pdb.yaml):
    ``` yaml
     apiVersion: policy/v1
     kind: PodDisruptionBudget
     metadata:
       name: critical-app-pdb
     spec:
       minAvailable: 1
       selector:
         matchLabels:
           app: critical-app
    ```

2. *Apply the PDB*:
   ```bash
   kubectl apply -f pdb.yaml
   ```

3. *Verify the PDB*:
   ```bash
   kubectl get pdb
   ```

> *Note*: Modify minAvailable or maxUnavailable parameters based on your application’s redundancy requirements.

## Verification and Rollback Strategy

1. *Check Node and Pod Status*:
   ```bash
   kubectl get nodes
   kubectl get pods --all-namespaces
   ```

2. *Test Application Functionality*: Verify that applications are functioning correctly by checking relevant endpoints or using health-check tools.

3. *Rollback (if necessary)*:
   - *Revert Control Plane*: If issues occur, contact Microsoft support for control plane rollback.
   - *Node Pool Reversion*: If the new node pool has issues, recreate an old node pool with a known stable version.

---

This strategy will help ensure a zero-downtime upgrade with a focused approach on separating control plane and node upgrades, as well as using PDBs to safeguard critical workloads.