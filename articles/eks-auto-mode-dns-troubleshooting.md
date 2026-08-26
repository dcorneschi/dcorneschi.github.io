# Troubleshoot DNS in EKS Auto Mode

How to diagnose and fix DNS resolution issues in Amazon EKS Auto Mode — pod-level checks, CoreDNS log inspection via debug containers, and node-level DNS verification.

EKS Auto Mode integrates CoreDNS as a built-in component. Since SSH/SSM access is not available on Auto Mode-launched EC2 managed instances, you need debug containers and `kubectl debug` to access node logs.

## Verify DNS Resolution Inside a Pod

### 1. Exec into the Pod

```bash
kubectl exec -it <pod-name> -- sh
```

### 2. Check /etc/resolv.conf

```bash
cat /etc/resolv.conf
```

Expected output:

```
search default.svc.cluster.local svc.cluster.local cluster.local ec2.internal
nameserver 172.20.0.10
options ndots:5
```

The `nameserver` should point to the `kube-dns` service cluster IP.

### 3. Resolve an Internal Service

```bash
nslookup kubernetes.default 172.20.0.10
```

```
Server:         172.20.0.10
Address:        172.20.0.10#53

Name:   kubernetes.default.svc.cluster.local
Address: 172.20.0.1
```

### 4. Resolve a Fully Qualified Service Name

```bash
dig +short my-service.default.svc.cluster.local
```

Replace `my-service` with your service name and `default` with its namespace.

### 5. Resolve a Public Domain

```bash
nslookup amazon.com 172.20.0.10
```

```
Server:         172.20.0.10
Address:        172.20.0.10#53

Non-authoritative answer:
Name:   amazon.com
Address: 54.239.28.85
```

### 6. Test TCP Connectivity

```bash
telnet google.com 80
```

If DNS resolves but connections fail, the problem is likely network policies or security groups rather than DNS.

## Check CoreDNS Logs via Debug Containers

Since you cannot SSH into Auto Mode nodes, use `kubectl debug` to access node-level logs.

### 1. (Optional) Deploy a DNS Test Pod

Schedule a pod on the target node to generate DNS traffic:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test-pod
  labels:
    app: dns-test
spec:
  nodeName: <NODE_NAME>
  containers:
  - name: dns-tester
    image: curlimages/curl:latest
    command: ["/bin/sh"]
    args:
      - -c
      - |
        while true; do
          echo "Testing DNS resolution..."
          curl -Is --connect-timeout 5 http://google.com || echo "google.com failed"
          sleep 2
          curl -Is --connect-timeout 5 http://amazon.com || echo "amazon.com failed"
          sleep 2
          curl -Is --connect-timeout 5 http://my-service || echo "my-service failed"
          sleep 10
        done
  restartPolicy: Always
```

Replace `<NODE_NAME>` with the target node's instance ID.

### 2. Launch a Debug Container on the Node

```bash
kubectl debug node/<NODE_NAME> -it --profile=sysadmin --image=public.ecr.aws/amazonlinux/amazonlinux:2023
```

### 3. Stream CoreDNS Logs

Inside the debug container:

```bash
yum install -y util-linux-core
nsenter -t 1 -m journalctl -f -u coredns
```

Example output:

```
[INFO] 10.0.16.19:43366 - 43927 "AAAA IN amazon.com. udp 39 false 1232" NOERROR qr,rd,ra 125 0.000687584s
[INFO] 10.0.16.19:56306 - 47907 "AAAA IN example.default.svc.cluster.local. udp 74 false 1232" NXDOMAIN qr,aa,rd 144 0.000162575s
[INFO] 10.0.16.19:56306 - 54143 "A IN example. udp 36 false 1232" SERVFAIL qr,aa,rd,ra 25 0.000310053s
```

### Interpreting CoreDNS Response Codes

| Code | Meaning | Action |
|------|---------|--------|
| `NOERROR` | Query succeeded | No issue |
| `NXDOMAIN` | Domain does not exist | Verify the service exists: `kubectl get svc -A \| grep <name>` |
| `SERVFAIL` | Server failed to resolve | Check upstream DNS, network policies, or missing service |

### 4. Verify Services Exist

```bash
kubectl get svc -A | grep <service-name>
kubectl get endpoints -A | grep <service-name>
```

### 5. Check Network Policies

Network policies might block DNS traffic (UDP/TCP port 53):

```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy -A
```

## Verify DNS on the Node Itself

From inside the debug container:

### Check Node resolv.conf

```bash
nsenter -t 1 -m cat /etc/resolv.conf
```

Expected output for Auto Mode nodes:

```
nameserver 127.0.0.53
options edns0 trust-ad
search ec2.internal
```

Auto Mode nodes use `systemd-resolved` (127.0.0.53) as the local DNS resolver.

### Test Public DNS from the Node

```bash
yum install -y bind-utils
nsenter -t 1 -n nslookup amazon.com 127.0.0.53
```

```
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   amazon.com
Address: 54.239.28.85
```

If this fails, the issue is at the VPC/node networking level rather than Kubernetes DNS.

## Common DNS Issues in EKS Auto Mode

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pod cannot resolve any name | CoreDNS not running or pod network broken | Check CoreDNS logs, verify node health |
| Internal service NXDOMAIN | Service doesn't exist or wrong namespace | `kubectl get svc -A` to verify |
| External domain SERVFAIL | Network policy blocking egress DNS | Review NetworkPolicies for UDP/53 egress |
| Intermittent timeouts | DNS throttling or node resource pressure | Check CoreDNS pod resource usage, consider NodeLocal DNSCache |

## Related Resources

- [EKS Auto Mode troubleshooting — debug containers and node logs](https://docs.aws.amazon.com/eks/latest/userguide/auto-troubleshoot.html#auto-node-debug-logs)
- [Retrieve node logs for a managed node using kubectl and S3](https://docs.aws.amazon.com/eks/latest/userguide/auto-get-logs.html)
- [Node monitoring agent (NodeDiagnostic resource)](https://docs.aws.amazon.com/eks/latest/userguide/auto-troubleshoot.html#auto-node-monitoring-agent)
