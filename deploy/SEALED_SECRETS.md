# Sealed Secrets Setup Guide

This guide explains how to work with Sealed Secrets in this Helm deployment for encrypting sensitive data before committing to Git.

## Overview

Sealed Secrets allows you to encrypt Kubernetes secrets and store them safely in Git. The Sealed Secrets controller decrypts them when deploying to the cluster.

## Prerequisites

1. **Install kubeseal CLI** on your local machine:
   ```bash
   # For Linux
   wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.5/kubeseal-0.24.5-linux-amd64.tar.gz
   tar -xvzf kubeseal-0.24.5-linux-amd64.tar.gz
   sudo install -m 755 kubeseal /usr/local/bin/kubeseal

   # For macOS
   brew install kubeseal

   # For other platforms, see: https://github.com/bitnami-labs/sealed-secrets#kubeseal
   ```

2. **Access to Kubernetes cluster** with Sealed Secrets controller installed

3. **Sealed Secrets controller public key** (automatically generated when controller is deployed)

## Workflow for Encrypting Secrets

### Step 1: Get the Sealed Secrets Controller Certificate

First, ensure the Sealed Secrets controller is running in your cluster:

```bash
kubectl get pods -n kube-system | grep sealed-secrets
```

Fetch the controller's public key:

```bash
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > pub-cert.pem
```

### Step 2: Create a Secret Manifest

Create a temporary YAML file with your plain text secret (this file should NOT be committed):

```yaml
# secret-template.yaml (DO NOT COMMIT THIS FILE)
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: team2-ns
type: Opaque
stringData:
  JWT_SECRET: "your-actual-jwt-secret-here"
```

### Step 3: Encrypt the Secret

Use kubeseal to encrypt the secret:

```bash
kubeseal --cert=pub-cert.pem \
  --format=yaml \
  --namespace=team2-ns \
  < secret-template.yaml \
  > encrypted-secret.yaml
```

This will generate an encrypted SealedSecret YAML.

### Step 4: Extract Encrypted Values

From the generated `encrypted-secret.yaml`, copy the encrypted values from the `encryptedData` section and paste them into the appropriate `values.yaml` files.

For example, for the JWT secret:
```yaml
secrets:
  jwtSecret: "AgBy3i4OJSWK1P8C...[encrypted-value]..."
```

### Step 5: Clean Up

Delete the temporary files:
```bash
rm secret-template.yaml encrypted-secret.yaml pub-cert.pem
```

## Current Secrets Configuration

### API Gateway
- **Secret**: JWT_SECRET
- **Values file**: `deploy/helm/api-gateway/values.yaml`
- **Field**: `secrets.jwtSecret`

### Order Service
- **Secret**: DATABASE_PASSWORD
- **Values file**: `deploy/helm/order-service/values.yaml`
- **Field**: `secrets.databasePassword`

### Postgres
- **Secret**: POSTGRES_PASSWORD
- **Values file**: `deploy/helm/postgres/values.yaml`
- **Field**: `secrets.postgresPassword`

## Deployment Order

Important: Sealed Secrets controller must be deployed before applications that use SealedSecrets.

1. Deploy Sealed Secrets controller (via Helm dependency)
2. Deploy applications with SealedSecrets

## Updating Secrets

To update an existing secret:

1. Follow steps 1-5 above with the new secret value
2. Update the corresponding `values.yaml` file
3. Commit and push the changes
4. ArgoCD will automatically update the SealedSecret and regenerate the Secret

## Security Notes

- **Never commit plain text secrets** to Git
- The encrypted values in `values.yaml` are safe to commit
- Store `pub-cert.pem` securely if needed for automation
- Regularly rotate secrets and update encrypted values

## Troubleshooting

### SealedSecret not decrypting
```bash
kubectl describe sealedsecret <name> -n <namespace>
kubectl logs -n kube-system deployment/sealed-secrets-controller
```

### kubeseal command not found
Ensure kubeseal is installed and in your PATH.

### Certificate fetch fails
Ensure the Sealed Secrets controller is running and accessible.
