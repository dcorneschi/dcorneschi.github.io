# DOKS Node Pools

## Overview

DigitalOcean Kubernetes (DOKS) supports **node pools**, which are equivalent to
EKS node groups — a group of worker nodes with the same size and configuration,
managed as a unit.

## Key Features

- **Multiple pools per cluster** — mix different node sizes/types
- **Auto-scaling** — min/max node counts per pool
- **Taints and labels** — custom scheduling rules
- **Independent scaling** — scale each pool separately

## Common Operations

### List node pools

```bash
doctl kubernetes cluster node-pool list my-doks-cluster
```

### Create a new pool

```bash
doctl kubernetes cluster node-pool create my-doks-cluster \
  --name pool-02 \
  --size s-2vcpu-4gb \
  --count 3 \
  --auto-scale \
  --min-nodes 1 \
  --max-nodes 5
```

### Delete a pool

```bash
doctl kubernetes cluster node-pool delete my-doks-cluster pool-01
```

### Add taints to a pool

```bash
doctl kubernetes cluster node-pool update my-doks-cluster pool-01 \
  --taint "workload=gpu:NoSchedule"
```

## Difference from EKS

The main difference from EKS is that DOKS node pools are simpler — no launch
templates or complex IAM configurations needed.

## Kubeconfig Management

### Save/update cluster kubeconfig

```bash
doctl kubernetes cluster kubeconfig save my-doks-cluster
```

### Remove an old context from kubectl

```bash
kubectl config delete-context <old-context-name>
kubectl config delete-cluster <old-cluster-name>
kubectl config delete-user <old-user-name>
```

## Skills Practiced

- Listing, creating, and deleting DOKS node pools with `doctl`
- Enabling per-pool auto-scaling and applying taints
- Managing cluster access with `doctl kubernetes cluster kubeconfig save`
- Cleaning up stale contexts from kubeconfig
