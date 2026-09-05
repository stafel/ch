# ASKCOS Argo CD Deployment for k3s with Traefik and cert-manager

This directory contains Kubernetes manifests for deploying ASKCOS (FourThievesVinegar fork) on a k3s cluster with Traefik ingress and cert-manager for TLS certificates.

## Architecture Overview

- **Namespace**: `chemsynth`
- **Domain**: `synth.maus.local`
- **Ingress**: Traefik IngressRoute with TLS termination
- **Certificate**: cert-manager managed certificate
- **Deployment**: Argo CD Application managing Helm chart from ASKCOS/askcos-deploy

## Prerequisites

### 1. k3s Cluster Setup

Ensure you have a k3s cluster with:
- Traefik installed as the ingress controller (default in k3s)
- cert-manager installed for TLS certificate management
- Argo CD installed for GitOps deployment

```bash
# Install k3s with Traefik (default)
curl -sfL https://get.k3s.io | sh -s - --disable traefik

# Install Traefik separately (recommended for better control)
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n kube-system

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. cert-manager Issuer Setup

Create a ClusterIssuer for Let's Encrypt production (or self-signed for testing):

**For Let's Encrypt Production:**
```yaml
# cert-manager-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@maus.local
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

**For Self-Signed (Testing):**
```yaml
# cert-manager-selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: cert-manager
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

Apply the issuer:
```bash
kubectl apply -f cert-manager-issuer.yaml
```

## Deployment Structure

```
k8s/
├── chemsynth/
│   ├── namespace.yaml          # Namespace definition
│   ├── certificate.yaml        # cert-manager Certificate for synth.maus.local
│   └── ingressroute.yaml       # Traefik IngressRoute
└── argocd/
    └── application.yaml        # Argo CD Application manifest
```

## Required Secrets (NOT in this repo)

The following secrets must be created manually before deployment:

### 1. Database Credentials

```bash
# MongoDB credentials
kubectl -n chemsynth create secret generic mongodb-credentials \
  --from-literal=mongodb-root-password='YOUR_MONGODB_ROOT_PASSWORD' \
  --from-literal=mongodb-password='YOUR_MONGODB_USER_PASSWORD'

# MySQL credentials
kubectl -n chemsynth create secret generic mysql-credentials \
  --from-literal=mysql-root-password='YOUR_MYSQL_ROOT_PASSWORD'

# Redis credentials (if using password)
kubectl -n chemsynth create secret generic redis-credentials \
  --from-literal=redis-password='YOUR_REDIS_PASSWORD'

# RabbitMQ credentials
kubectl -n chemsynth create secret generic rabbitmq-credentials \
  --from-literal=rabbitmq-password='YOUR_RABBITMQ_PASSWORD'
```

### 2. ASKCOS Application Secrets

```bash
# ASKCOS environment secrets
kubectl -n chemsynth create secret generic askcos-env \
  --from-literal=OAUTH2_SECRET_KEY='YOUR_OAUTH2_SECRET_KEY' \
  --from-literal=V1_PASSWORD='YOUR_V1_PASSWORD' \
  --from-literal=CONTACT_EMAIL='admin@maus.local' \
  --from-literal=SUPPORT_EMAILS='admin@maus.local'
```

### 3. Container Registry Credentials

If using private container registry (GitLab):

```bash
# Create registry secret
kubectl -n chemsynth create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=YOUR_GITLAB_USERNAME \
  --docker-password=YOUR_GITLAB_ACCESS_TOKEN \
  --docker-email=your.email@example.com
```

### 4. TLS Certificate (Optional - if not using cert-manager)

If you want to provide your own certificate instead of using cert-manager:

```bash
kubectl -n chemsynth create secret tls synth-maus-local-tls \
  --cert=path/to/certificate.crt \
  --key=path/to/private.key
```

## Deployment Steps

### Step 1: Create the chemsynth namespace

```bash
kubectl apply -f k8s/chemsynth/namespace.yaml
```

### Step 2: Deploy Certificate Resource

```bash
kubectl apply -f k8s/chemsynth/certificate.yaml
```

Verify certificate is ready:
```bash
kubectl -n chemsynth get certificate
kubectl -n chemsynth describe certificate synth-maus-local
```

### Step 3: Deploy IngressRoute

```bash
kubectl apply -f k8s/chemsynth/ingressroute.yaml
```

### Step 4: Create Required Secrets

Create all the secrets mentioned in the "Required Secrets" section above.

### Step 5: Deploy Argo CD Application

```bash
kubectl apply -f k8s/argocd/application.yaml
```

### Step 6: Access Argo CD Dashboard

Port-forward to access Argo CD:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access at: https://localhost:8080
Default credentials: `admin` / (get password with `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`)

### Step 7: Monitor Deployment

```bash
# Check Argo CD sync status
argocd app get askcos-chemsynth

# View logs
argocd app logs askcos-chemsynth

# Check pods
kubectl -n chemsynth get pods

# Check services
kubectl -n chemsynth get svc
```

## DNS Configuration

Ensure `synth.maus.local` resolves to your cluster's external IP or load balancer:

```bash
# If using local testing, add to /etc/hosts
 echo "<K3S_NODE_IP> synth.maus.local" | sudo tee -a /etc/hosts
```

For production, configure DNS A record pointing to your load balancer IP.

## Customization

### Custom Values for Helm Chart

The Argo CD Application manifest includes default values. To customize:

1. Create a `values.yaml` file in this directory
2. Update the Argo CD Application to reference it:

```yaml
spec:
  source:
    helm:
      values: |-
        # Your custom values here
      valueFiles:
        - values.yaml
```

### Scaling Resources

Adjust resource requests/limits in the Argo CD Application manifest based on your cluster capacity.

### Enabling Additional Components

The default configuration disables ML servers and reduces celery workers. To enable:

```yaml
mlserver:
  - name: fast-filter
    image:
      repository: askcos/fast-filter
      tag: "1.0"
    replicaCount: 1
    service:
      type: ClusterIP
      port: 8501
  # Add other ML servers as needed
```

## Troubleshooting

### Certificate Not Issuing

```bash
# Check certificate status
kubectl -n chemsynth describe certificate synth-maus-local

# Check cert-manager logs
kubectl -n cert-manager logs -l app=cert-manager

# Check challenge status
kubectl -n chemsynth get challenges
```

Common issues:
- DNS not pointing to cluster IP
- Ingress controller not properly configured
- Let's Encrypt rate limits

### Pods Not Starting

```bash
# Check pod events
kubectl -n chemsynth get events --sort-by=.metadata.creationTimestamp

# Check pod logs
kubectl -n chemsynth logs <pod-name>

# Describe pod for details
kubectl -n chemsynth describe pod <pod-name>
```

Common issues:
- Missing secrets
- Insufficient resources
- Image pull errors (registry credentials)

### Argo CD Sync Issues

```bash
# Check sync status
argocd app get askcos-chemsynth

# Check sync logs
argocd app logs askcos-chemsynth

# Force sync
argocd app sync askcos-chemsynth
```

## Security Considerations

1. **Secrets Management**: All sensitive data is stored in Kubernetes Secrets. Consider using:
   - External secret managers (AWS Secrets Manager, HashiCorp Vault)
   - Sealed Secrets for encrypted secrets in Git
   - SOPS for encrypted manifests

2. **Network Security**: 
   - Consider adding NetworkPolicies to restrict pod communication
   - Use PodSecurityPolicies or OPA/Gatekeeper for security constraints

3. **Access Control**:
   - Configure RBAC for Argo CD
   - Restrict access to Argo CD dashboard
   - Use SSO for Argo CD authentication

## Resource Requirements

Minimum recommended resources for production:

- **MongoDB**: 2 vCPUs, 4GB RAM, 20GB storage
- **MySQL**: 1 vCPU, 1GB RAM, 10GB storage
- **Redis**: 512MB RAM
- **RabbitMQ**: 512MB RAM
- **ASKCOS App**: 2 vCPUs, 4GB RAM
- **Celery Workers**: 1-2 vCPUs per worker, 2-16GB RAM depending on worker type
- **ML Servers**: 1-2 vCPUs, 2-8GB RAM each

## Updates

To update ASKCOS:

1. Update the `targetRevision` in the Argo CD Application to the desired git tag/branch
2. Argo CD will automatically sync the changes

Or manually trigger sync:
```bash
argocd app sync askcos-chemsynth
```

## References

- ASKCOS Helm Chart: https://github.com/ASKCOS/askcos-deploy/tree/main/helm/askcos
- FourThievesVinegar askcos2_core: https://github.com/FourThievesVinegar/askcos2_core
- Argo CD Documentation: https://argo-cd.readthedocs.io/
- cert-manager Documentation: https://cert-manager.io/docs/
- Traefik Documentation: https://doc.traefik.io/traefik/
