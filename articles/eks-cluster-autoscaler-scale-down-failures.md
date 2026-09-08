# Checking Cluster Autoscaler Scale-Down Failures Across EKS Clusters

## Overview

This guide provides multiple methods and scripts to check for failed scale-down
messages across all your EKS clusters using `kubectl` and the AWS CLI.

## Prerequisites

```bash
# Required tools: kubectl, aws cli, jq (optional, for JSON parsing)

# Ensure AWS credentials are configured
aws configure list

# Test kubectl access
kubectl version
```

## Method 1: Single Cluster Check

### Basic check

```bash
# Check cluster-autoscaler logs for scale down failures
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | \
    grep -i "scale.*down" | \
    grep -i "fail\|error\|unable"
```

### Detailed single cluster check

```bash
#!/bin/bash
# check_single_cluster.sh

echo "Checking scale-down issues for current cluster..."
echo "================================================"

CURRENT_CONTEXT=$(kubectl config current-context)
echo "Context: $CURRENT_CONTEXT"
echo ""

# Check if cluster-autoscaler exists
if ! kubectl -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
    echo "Cluster Autoscaler not found in kube-system namespace"
    exit 1
fi

echo "Cluster Autoscaler Pod Status:"
kubectl -n kube-system get pods -l app=cluster-autoscaler

echo ""
echo "Recent Scale-Down Failures:"
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=1000 | \
    grep -iE "(failed|unable|cannot|error).*scale.*down" | \
    tail -10

echo ""
echo "Nodes that cannot be removed:"
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=1000 | \
    grep -i "cannot.*remove" | \
    tail -5
```

## Method 2: Loop Through All Clusters

```bash
#!/bin/bash
# check_all_clusters_scale_down.sh

REGION=${1:-us-east-1}

CLUSTERS=$(aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text)

echo "Checking failed scale-down messages across all EKS clusters"
echo "Region: $REGION"
echo "============================================================"

for cluster in $CLUSTERS; do
    echo ""
    echo "Cluster: $cluster"
    echo "----------------------------------------"

    aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster" > /dev/null 2>&1

    if ! kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
        echo "Cluster Autoscaler not installed"
        continue
    fi

    failed_messages=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
        grep -i "scale.*down" | \
        grep -i "fail\|error\|unable\|cannot" | \
        wc -l | tr -d ' ')

    if [ "$failed_messages" -gt 0 ]; then
        echo "Found $failed_messages failed scale-down messages"
        echo ""
        echo "Recent failures:"
        kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
            grep -i "scale.*down" | \
            grep -i "fail\|error\|unable\|cannot" | \
            tail -5
    else
        echo "No failed scale-down messages"
    fi
done

echo ""
echo "============================================================"
echo "Check complete!"
```

## Method 3: Parallel Checking

Faster for checking many clusters simultaneously.

```bash
#!/bin/bash
# parallel_check_scale_down.sh

REGION=${1:-us-east-1}
MAX_PARALLEL=${2:-5}

check_cluster() {
    cluster=$1
    region=$2

    aws eks update-kubeconfig --name "$cluster" --region "$region" --alias "$cluster" > /dev/null 2>&1

    if ! kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
        echo "[$cluster] Cluster Autoscaler not found"
        return
    fi

    failures=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
        grep -iE "scale.*down.*(fail|error|unable|cannot)" | wc -l | tr -d ' ')

    if [ "$failures" -gt 0 ]; then
        echo "[$cluster] $failures failed scale-down messages"
        sample=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
            grep -iE "scale.*down.*(fail|error|unable|cannot)" | tail -1)
        echo "  Sample: ${sample:0:100}..."
    else
        echo "[$cluster] No issues"
    fi
}

export -f check_cluster
export REGION

echo "Checking scale-down issues across all clusters (parallel)"
echo "Region: $REGION, Max parallel: $MAX_PARALLEL"
echo "=========================================================="

aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text | \
    tr '\t' '\n' | \
    xargs -P "$MAX_PARALLEL" -I {} bash -c 'check_cluster "$@"' _ {} "$REGION"

echo ""
echo "Check complete!"
```

## Method 4: Detailed Analysis

Comprehensive analysis with categorized errors.

```bash
#!/bin/bash
# detailed_scale_down_check.sh

REGION=${1:-us-east-1}

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; BLUE='\033[0;34m'; NC='\033[0m'

echo -e "${BLUE}Detailed Scale-Down Analysis Across All EKS Clusters${NC}"
echo "Region: $REGION"
echo "===================================================="

CLUSTERS=$(aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text)

for cluster in $CLUSTERS; do
    echo ""
    echo -e "${YELLOW}Cluster: $cluster${NC}"

    aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster" > /dev/null 2>&1

    if ! kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
        echo -e "${YELLOW}Cluster Autoscaler not found${NC}"
        continue
    fi

    pod_status=$(kubectl --context="$cluster" -n kube-system get pods -l app=cluster-autoscaler -o jsonpath='{.items[0].status.phase}' 2>/dev/null)
    echo "Pod Status: $pod_status"

    logs=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=1000 2>/dev/null)

    echo ""
    echo "1. Failed scale-downs:"
    failed=$(echo "$logs" | grep -i "failed.*scale.*down" | wc -l | tr -d ' ')
    if [ "$failed" -gt 0 ]; then
        echo -e "${RED}   Found: $failed${NC}"
        echo "$logs" | grep -i "failed.*scale.*down" | tail -2 | sed 's/^/   /'
    else
        echo -e "${GREEN}   None${NC}"
    fi

    echo ""
    echo "2. Unable to scale down:"
    unable=$(echo "$logs" | grep -i "unable.*scale.*down" | wc -l | tr -d ' ')
    if [ "$unable" -gt 0 ]; then
        echo -e "${RED}   Found: $unable${NC}"
        echo "$logs" | grep -i "unable.*scale.*down" | tail -2 | sed 's/^/   /'
    else
        echo -e "${GREEN}   None${NC}"
    fi

    echo ""
    echo "3. Scale down errors:"
    errors=$(echo "$logs" | grep -iE "scale.*down.*error" | wc -l | tr -d ' ')
    if [ "$errors" -gt 0 ]; then
        echo -e "${RED}   Found: $errors${NC}"
        echo "$logs" | grep -iE "scale.*down.*error" | tail -2 | sed 's/^/   /'
    else
        echo -e "${GREEN}   None${NC}"
    fi

    echo ""
    echo "4. Nodes that cannot be removed:"
    cannot=$(echo "$logs" | grep -i "node.*cannot.*be.*removed" | wc -l | tr -d ' ')
    if [ "$cannot" -gt 0 ]; then
        echo -e "${RED}   Found: $cannot${NC}"
        echo "$logs" | grep -i "node.*cannot.*be.*removed" | tail -2 | sed 's/^/   /'
    else
        echo -e "${GREEN}   None${NC}"
    fi

    echo ""
    echo "5. Pod Disruption Budget blocks:"
    pdb=$(echo "$logs" | grep -i "pod disruption budget" | wc -l | tr -d ' ')
    if [ "$pdb" -gt 0 ]; then
        echo -e "${YELLOW}   Found: $pdb${NC}"
    else
        echo -e "${GREEN}   None${NC}"
    fi

    total_issues=$((failed + unable + errors + cannot))

    echo ""
    if [ "$total_issues" -gt 0 ]; then
        echo -e "${RED}Total scale-down issues: $total_issues${NC}"
    else
        echo -e "${GREEN}No scale-down issues detected${NC}"
    fi
done

echo ""
echo -e "${BLUE}Analysis complete!${NC}"
```

## Method 5: Export to CSV

```bash
#!/bin/bash
# export_scale_down_issues.sh

OUTPUT_FILE="scale_down_issues_$(date +%Y%m%d_%H%M%S).csv"
REGION=${1:-us-east-1}

echo "Exporting scale-down issues to CSV..."
echo "Region: $REGION"
echo "Output: $OUTPUT_FILE"

echo "Cluster,Region,Status,Autoscaler_Found,Failed_Scale_Downs,Unable_Scale_Downs,PDB_Blocks,Last_Error_Sample" > "$OUTPUT_FILE"

CLUSTERS=$(aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text)

for cluster in $CLUSTERS; do
    echo "Processing: $cluster"

    aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster" > /dev/null 2>&1

    if kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
        autoscaler_found="Yes"
        status="Running"

        logs=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=1000 2>/dev/null)

        failed=$(echo "$logs" | grep -i "failed.*scale.*down" | wc -l | tr -d ' ')
        unable=$(echo "$logs" | grep -i "unable.*scale.*down" | wc -l | tr -d ' ')
        pdb=$(echo "$logs" | grep -i "pod disruption budget" | wc -l | tr -d ' ')

        last_error=$(echo "$logs" | \
            grep -iE "(failed|unable|cannot|error).*scale.*down" | \
            tail -1 | tr ',' ';' | tr '\n' ' ' | cut -c1-200)

        echo "$cluster,$REGION,$status,$autoscaler_found,$failed,$unable,$pdb,\"$last_error\"" >> "$OUTPUT_FILE"
    else
        echo "$cluster,$REGION,Not Found,No,0,0,0," >> "$OUTPUT_FILE"
    fi
done

echo ""
echo "Results exported to: $OUTPUT_FILE"
```

## Method 6: Check Specific Error Patterns

```bash
#!/bin/bash
# check_specific_scale_down_errors.sh

REGION=${1:-us-east-1}

check_cluster_errors() {
    cluster=$1
    region=$2

    echo "Cluster: $cluster"
    echo "----------------"

    aws eks update-kubeconfig --name "$cluster" --region "$region" --alias "$cluster" > /dev/null 2>&1

    if ! kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
        echo "  Cluster Autoscaler: Not found"
        echo ""
        return
    fi

    logs=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=2000 2>/dev/null)

    echo "  Pod Disruption Budgets blocking: $(echo "$logs" | grep -ic "pod disruption budget")"
    echo "  Nodes with local storage: $(echo "$logs" | grep -ic "local storage")"
    echo "  Nodes with system pods: $(echo "$logs" | grep -ic "system pod")"
    echo "  Non-replicated pods: $(echo "$logs" | grep -ic "non-replicated")"
    echo "  Kube-system pod issues: $(echo "$logs" | grep -i "kube-system" | grep -ic "cannot")"
    echo "  EmptyDir volume issues: $(echo "$logs" | grep -ic "emptydir")"
    echo ""
}

export -f check_cluster_errors

echo "Checking Specific Scale-Down Error Patterns"
echo "Region: $REGION"
echo "==========================================="

aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text | \
    tr '\t' '\n' | \
    xargs -I {} bash -c 'check_cluster_errors "$@"' _ {} "$REGION"
```

## Method 7: Real-Time Monitoring

Monitor scale-down events as they happen.

```bash
#!/bin/bash
# monitor_scale_down_realtime.sh

CLUSTER=${1}
REGION=${2:-us-east-1}

if [ -z "$CLUSTER" ]; then
    echo "Usage: $0 <cluster-name> [region]"
    exit 1
fi

echo "Real-Time Scale-Down Monitoring — Cluster: $CLUSTER, Region: $REGION"
echo "Press Ctrl+C to stop"

aws eks update-kubeconfig --name "$CLUSTER" --region "$REGION" --alias "$CLUSTER"

kubectl --context="$CLUSTER" -n kube-system logs -f deployment/cluster-autoscaler 2>/dev/null | \
    grep --line-buffered -iE "scale.*down" | \
    grep --line-buffered --color=always -iE "(fail|error|unable|cannot|success|completed|removing)"
```

## Method 8: Summary Report

```bash
#!/bin/bash
# scale_down_summary_report.sh

REGION=${1:-us-east-1}
OUTPUT_FILE="scale_down_report_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "=========================================="
    echo "Scale-Down Health Report"
    echo "=========================================="
    echo "Date: $(date)"
    echo "Region: $REGION"
    echo ""

    CLUSTERS=$(aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text)
    total_clusters=0
    clusters_with_issues=0
    clusters_without_autoscaler=0
    total_issues=0

    echo "Cluster Details:"
    echo "----------------"

    for cluster in $CLUSTERS; do
        ((total_clusters++))
        aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster" > /dev/null 2>&1

        if ! kubectl --context="$cluster" -n kube-system get deployment cluster-autoscaler > /dev/null 2>&1; then
            ((clusters_without_autoscaler++))
            printf "%-30s : Cluster Autoscaler not installed\n" "$cluster"
            continue
        fi

        issues=$(kubectl --context="$cluster" -n kube-system logs deployment/cluster-autoscaler --tail=1000 2>/dev/null | \
            grep -iE "(failed|unable|cannot|error).*scale.*down" | wc -l | tr -d ' ')

        if [ "$issues" -gt 0 ]; then
            ((clusters_with_issues++))
            ((total_issues += issues))
            printf "%-30s : %d issues\n" "$cluster" "$issues"
        else
            printf "%-30s : Healthy\n" "$cluster"
        fi
    done

    echo ""
    echo "=========================================="
    echo "Summary:"
    echo "=========================================="
    echo "Total clusters checked          : $total_clusters"
    echo "Clusters without autoscaler     : $clusters_without_autoscaler"
    echo "Clusters with scale-down issues : $clusters_with_issues"
    echo "Total scale-down issues found   : $total_issues"

} | tee "$OUTPUT_FILE"

echo ""
echo "Report saved to: $OUTPUT_FILE"
```

## Method 9: Kubernetes Events Check

```bash
#!/bin/bash
# check_scale_down_events.sh

CLUSTER=${1}
REGION=${2:-us-east-1}

if [ -z "$CLUSTER" ]; then
    CLUSTERS=$(aws eks list-clusters --region "$REGION" --query 'clusters[]' --output text)
else
    CLUSTERS=$CLUSTER
fi

echo "Checking Kubernetes Events for Scale-Down Issues — Region: $REGION"
echo "================================================"

for cluster in $CLUSTERS; do
    echo ""
    echo "Cluster: $cluster"
    echo "----------------"

    aws eks update-kubeconfig --name "$cluster" --region "$REGION" --alias "$cluster" > /dev/null 2>&1

    echo "FailedScaleDown events:"
    kubectl --context="$cluster" get events --all-namespaces \
        --sort-by='.lastTimestamp' \
        --field-selector reason=FailedScaleDown 2>/dev/null | tail -10

    echo ""
    echo "Recent autoscaler events:"
    kubectl --context="$cluster" get events -n kube-system \
        --sort-by='.lastTimestamp' 2>/dev/null | \
        grep -i "autoscaler\|scale" | \
        tail -10
    echo ""
done
```

## Quick One-Liners

### Check current cluster

```bash
# Failed scale-downs
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -iE "(failed|unable|error).*scale.*down"

# Count issues
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -iE "scale.*down.*(fail|error)" | wc -l

# Recent scale-down activity
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=1000 | grep -i "scale.*down" | tail -20

# Why nodes can't be removed
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=1000 | grep -i "cannot.*remove"

# Pod disruption budget blocks
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=1000 | grep -i "pod disruption budget"
```

### Check all contexts

```bash
# Count issues in all contexts
for ctx in $(kubectl config get-contexts -o name); do
    echo "$ctx:"
    kubectl --context="$ctx" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
        grep -iE "scale.*down.*(fail|error)" | wc -l
done

# Find clusters with issues
for ctx in $(kubectl config get-contexts -o name); do
    count=$(kubectl --context="$ctx" -n kube-system logs deployment/cluster-autoscaler --tail=500 2>/dev/null | \
        grep -iE "scale.*down.*(fail|error)" | wc -l | tr -d ' ')
    [ "$count" -gt 0 ] && echo "$ctx: $count issues"
done
```

## Common Scale-Down Failure Reasons

### 1. Pod Disruption Budgets (PDB)

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -i "pod disruption budget"
```

Fix: review and adjust PDB settings for your applications.

### 2. Local storage / emptyDir

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -i "local storage\|emptydir"
```

Fix: use persistent volumes, or allow local-storage scale-down with
`--skip-nodes-with-local-storage=false`.

### 3. System pods

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -i "system pod"
```

Fix: use `--skip-nodes-with-system-pods=false` if appropriate.

### 4. Non-replicated pods

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -i "non-replicated"
```

Fix: use Deployments/ReplicaSets instead of bare pods.

### 5. Node still has pods

```bash
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=500 | grep -i "node.*still.*has.*pod"
```

Fix: ensure pods can be safely evicted and rescheduled.

## Troubleshooting Commands

```bash
# View Cluster Autoscaler configuration
kubectl -n kube-system get deployment cluster-autoscaler -o yaml | grep -A 20 "args:"

# Check unneeded nodes
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=100 | grep "unneeded"

# Check scale-down delay
kubectl -n kube-system logs deployment/cluster-autoscaler --tail=100 | grep "scale-down-delay"

# View node status
kubectl get nodes -o wide
kubectl describe nodes | grep -A 5 "Taints\|Unschedulable"
```

## Best Practices

1. **Regular monitoring** — run checks at least weekly.
2. **Automation** — schedule checks with cron or CI/CD.
3. **Alerting** — set up alerts for persistent issues.
4. **Documentation** — track common issues and fixes.
5. **PDB review** — regularly review and optimize PDB configurations.
6. **Log retention** — increase retention for better analysis.
7. **Multi-region** — check every region where you have clusters.

## Scheduling Regular Checks

```bash
# crontab -e

# Every day at 9 AM
0 9 * * * /path/to/scale_down_summary_report.sh us-east-1 >> /var/log/scale-down-checks.log 2>&1

# Every 6 hours
0 */6 * * * /path/to/check_all_clusters_scale_down.sh us-east-1 >> /var/log/scale-down-checks.log 2>&1
```

For a serverless option, wrap these checks in an AWS Lambda function and send
the results to SNS or Slack.

## Additional Resources

- [Cluster Autoscaler FAQ — what prevents CA from removing a node](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-types-of-pods-can-prevent-ca-from-removing-a-node)
- [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [EKS Best Practices — Autoscaling](https://aws.github.io/aws-eks-best-practices/cluster-autoscaling/)

## Related

- [Cluster Autoscaler on EKS](articles/eks-cluster-autoscaler-setup.md)
- [Kubernetes Cluster Autoscaler Tuning](articles/kubernetes-cluster-autoscaler-tuning.md)
- [Cluster Autoscaler Scale-Up Troubleshooting](articles/kubernetes-cluster-autoscaler-scale-up-troubleshooting.md)

## Skills Practiced

- Grepping cluster-autoscaler logs for scale-down failure patterns
- Iterating over all EKS clusters in a region (sequential and parallel)
- Categorizing failure reasons (PDB, local storage, system/non-replicated pods)
- Exporting findings to CSV/reports and scheduling recurring checks
