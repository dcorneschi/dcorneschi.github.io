# CKA Practice - ConfigMaps and Secrets

Practice creating and using ConfigMaps and Secrets in Kubernetes.

## Step 1: Create a ConfigMap and Secret

Your application needs a database configuration and a password stored securely.

**Goal:**

- Create a ConfigMap named `app-config` with:
  - `DB_HOST=localhost`
  - `DB_PORT=3306`
- Create a Secret named `db-secret` with:
  - `DB_PASSWORD=supersecret` (base64 encoded automatically by kubectl)

**Commands to run:**

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=3306

# Create Secret
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=supersecret
```

**Validation:**

```bash
kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o yaml
```

## Step 2: Use ConfigMap and Secret in a Pod

Now that you have the ConfigMap and Secret, make them available to a Pod.

**Goal:**

- Create a Pod named `app-pod` using the `nginx` image
- Inject the ConfigMap values as environment variables
- Inject the Secret value as an environment variable

Example manifest snippet:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_HOST
  - name: DB_PORT
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_PORT
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

**Validation:**

```bash
kubectl exec app-pod -- env | grep DB_
```

Check that all three environment variables are set.

## Summary

In this scenario, you learned how to:

- Store non-sensitive and sensitive configuration separately
- Create a ConfigMap from literal values
- Create a Secret from literal values
- Inject them into a Pod as environment variables

> **Real-world tip:** Avoid hardcoding sensitive data in YAML files. Use `kubectl create secret` or external secret management solutions to handle passwords securely.
