# ASKCOS Argo CD Deployment for Chemical Retrosynthesis

This directory contains Kubernetes manifests for deploying ASKCOS (FourThievesVinegar fork) chemical retrosynthesis application using Argo CD with Traefik ingress and cert-manager for TLS certificates.

## Architecture Overview

- **Namespace**: `chemsynth`
- **Domain**: `synth.maus.local`
- **Ingress**: Traefik IngressRoute with TLS termination
- **Certificate**: cert-manager managed certificate
- **Deployment**: Argo CD Application managing Helm chart from ASKCOS/askcos-deploy

## Prerequisites

The Kubernetes cluster must have the following installed and configured:

- k3s or any Kubernetes distribution
- Traefik as ingress controller
- cert-manager for TLS certificate management
- Argo CD for GitOps deployment
- ClusterIssuer for cert-manager (e.g., `letsencrypt-prod` ClusterIssuer)

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

The following secrets must be created in the `chemsynth` namespace before deployment. Use the Ansible playbook in the `ansible/` directory to create them:

```bash
ansible-playbook ansible/playbook.yaml -e @vars.yaml
```

Or create them manually:

```bash
# MongoDB credentials
kubectl -n chemsynth create secret generic mongodb-credentials \
  --from-literal=mongodb-root-password='YOUR_PASSWORD' \
  --from-literal=mongodb-password='YOUR_PASSWORD' \
  --from-literal=mongodb-username=askcos

# MySQL credentials
kubectl -n chemsynth create secret generic mysql-credentials \
  --from-literal=mysql-root-password='YOUR_PASSWORD'

# Redis credentials
kubectl -n chemsynth create secret generic redis-credentials \
  --from-literal=redis-password='YOUR_PASSWORD'

# RabbitMQ credentials
kubectl -n chemsynth create secret generic rabbitmq-credentials \
  --from-literal=rabbitmq-password='YOUR_PASSWORD'

# ASKCOS environment secrets
kubectl -n chemsynth create secret generic askcos-env \
  --from-literal=OAUTH2_SECRET_KEY='YOUR_SECRET' \
  --from-literal=V1_USERNAME=askcos \
  --from-literal=V1_PASSWORD='YOUR_PASSWORD' \
  --from-literal=CONTACT_EMAIL='admin@maus.local' \
  --from-literal=SUPPORT_EMAILS='admin@maus.local'

# Container registry credentials
kubectl -n chemsynth create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_TOKEN \
  --docker-email=your.email@example.com
```

## Deployment Steps

### Step 1: Create Secrets

Use the Ansible playbook:
```bash
ansible-playbook ansible/playbook.yaml -e @vars.yaml
```

### Step 2: Deploy Argo CD Application

```bash
kubectl apply -f k8s/argocd/application.yaml
```

### Step 3: Monitor Deployment

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

### Step 4: Access Application

Once all pods are ready, access ASKCOS at: https://synth.maus.local

## DNS Configuration

Ensure `synth.maus.local` resolves to your cluster's external IP or load balancer.

## Customization

### Custom Values for Helm Chart

The Argo CD Application includes default values. To customize, modify the values in `k8s/argocd/application.yaml`.

### Scaling Resources

Adjust resource requests/limits in the Argo CD Application manifest based on your cluster capacity.

### Enabling Additional Components

The default configuration disables ML servers and reduces celery workers. To enable ML servers, update the `mlserver` section in the Argo CD Application values.

## Troubleshooting

### Certificate Not Issuing

```bash
kubectl -n chemsynth describe certificate
kubectl -n cert-manager logs -l app=cert-manager
kubectl -n chemsynth get challenges
```

### Pods Not Starting

```bash
kubectl -n chemsynth get events --sort-by=.metadata.creationTimestamp
kubectl -n chemsynth logs <pod-name>
kubectl -n chemsynth describe pod <pod-name>
```

### Argo CD Sync Issues

```bash
argocd app get askcos-chemsynth
argocd app logs askcos-chemsynth
argocd app sync askcos-chemsynth
```

## Security Considerations

1. **Secrets Management**: All sensitive data is stored in Kubernetes Secrets. Use Ansible Vault or external secret managers for production.
2. **Network Security**: Consider adding NetworkPolicies to restrict pod communication.
3. **Access Control**: Configure RBAC for Argo CD and restrict dashboard access.

## Resource Requirements

Minimum recommended resources:

- **MongoDB**: 2 vCPUs, 4GB RAM, 20GB storage
- **MySQL**: 1 vCPU, 1GB RAM, 10GB storage
- **Redis**: 512MB RAM
- **RabbitMQ**: 512MB RAM
- **ASKCOS App**: 2 vCPUs, 4GB RAM
- **Celery Workers**: 1-2 vCPUs per worker, 2-16GB RAM depending on worker type

## Updates

To update ASKCOS, update the `targetRevision` in the Argo CD Application to the desired git tag/branch. Argo CD will automatically sync the changes.

## References

- ASKCOS Helm Chart: https://github.com/ASKCOS/askcos-deploy/tree/main/helm/askcos
- FourThievesVinegar askcos2_core: https://github.com/FourThievesVinegar/askcos2_core
- Argo CD Documentation: https://argo-cd.readthedocs.io/
- cert-manager Documentation: https://cert-manager.io/docs/
- Traefik Documentation: https://doc.traefik.io/traefik/
