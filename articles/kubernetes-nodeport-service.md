# Kubernetes NodePort Services

If you set a Service's `type` field to `NodePort`, the Kubernetes control plane
allocates a port from the range set by the `--service-node-port-range` flag
(default: `30000-32767`). Every node proxies that same port number into your
Service, and the allocated port is reported in `.spec.ports[*].nodePort`.

## Create a Sample Application

```yaml
cat <<EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF
```

## Create the Deployment

```sh
kubectl apply -f nginx-deployment.yaml
```

## Create a NodePort Service

```yaml
cat <<EOF > nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

You can pin a specific node port by adding `nodePort: <30000-32767>` under the
port entry; otherwise one is allocated automatically.

## Create the NodePort Object

```sh
kubectl create -f nodeport.yaml
```

Or expose the deployment directly without writing a manifest:

```sh
kubectl expose deployment nginx-deployment \
  --type=NodePort \
  --name=nginx-service-nodeport
```

## Verification

```sh
# Show the service and its allocated nodePort
kubectl get service/nginx-service-nodeport

# List nodes with their internal/external IPs
kubectl get nodes -o wide | awk '{print $1" "$2" "$6}' | column -t
```

Reach the app at `http://<node-ip>:<nodePort>` (ensure the node's security
group/firewall allows the port).

```sh
# Discover the allocated port programmatically
NODE_PORT=$(kubectl get service nginx-service-nodeport \
  -o jsonpath='{.spec.ports[0].nodePort}')
echo "$NODE_PORT"
```

## Clean Up

```sh
kubectl delete service nginx-service-nodeport
kubectl delete deployment nginx-deployment
```

## Skills Practiced

- Understanding how NodePort allocates and proxies a port on every node
- Creating a NodePort Service via manifest and via `kubectl expose`
- Finding the allocated `nodePort` with `kubectl get` and JSONPath
- Reaching a workload through a node IP and cleaning up afterward

## Links

- [Kubernetes Service types (official docs)](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
