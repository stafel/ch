# ASKCOS Chemical Retrosynthesis Deployment

Argo CD compatible Kubernetes deployment for ASKCOS (FourThievesVinegar fork) chemical retrosynthesis tool with Traefik ingress and cert-manager TLS certificates.

## Architecture Overview

- **Namespace**: `chemsynth`
- **Domain**: `synth.maus.local`
- **Ingress**: Traefik IngressRoute with TLS termination
- **Certificate**: cert-manager managed certificate
- **Deployment**: Argo CD Application managing all ASKCOS components

## Prerequisites

The Kubernetes cluster must have the following installed and configured:

- k3s or any Kubernetes distribution
- Traefik as ingress controller
- cert-manager for TLS certificate management
- Argo CD for GitOps deployment
- ClusterIssuer for cert-manager (e.g., `letsencrypt-prod` ClusterIssuer)

## Deployment Structure

```
.
├── README.md                          # This file
├── ansible/
│   ├── playbook.yaml                  # Creates required secrets
│   ├── docker-config.json.j2           # Registry config template
│   ├── README.md                       # Ansible documentation
│   └── requirements.yml                # Ansible collection requirements
└── k8s/
    ├── argocd/
    │   └── application.yaml            # Argo CD Application manifest
    └── chemsynth/
        ├── namespace.yaml              # Namespace definition
        ├── certificate.yaml            # cert-manager Certificate
        ├── ingressroute.yaml            # Traefik IngressRoute
        ├── mongodb/
        │   └── statefulset.yaml         # MongoDB StatefulSet + Service
        ├── redis/
        │   └── statefulset.yaml         # Redis StatefulSet + Service
        ├── rabbitmq/
        │   └── statefulset.yaml         # RabbitMQ StatefulSet + Service
        ├── app/
        │   └── deployment.yaml           # ASKCOS App Deployment + Service
        ├── nginx/
        │   ├── deployment.yaml           # Nginx Deployment + Service
        │   └── configmap.yaml            # Nginx ConfigMap
        └── celery/
            └── deployment.yaml           # Celery Workers Deployment
```

## Components Deployed

| Component | Type | Purpose |
|-----------|------|---------|
| namespace | Namespace | Isolated namespace for ASKCOS |
| certificate | Certificate | TLS certificate for synth.maus.local |
| ingressroute | IngressRoute | Traefik HTTPS ingress configuration |
| mongodb | StatefulSet | MongoDB database with persistent storage |
| redis | StatefulSet | Redis cache with persistent storage |
| rabbitmq | StatefulSet | RabbitMQ message broker with persistent storage |
| askcos-app | Deployment | ASKCOS Django application |
| askcos-nginx | Deployment | Nginx reverse proxy with static files |
| askcos-celery-worker | Deployment | Celery workers for background tasks |

## Required Secrets (NOT in this repo)

The following secrets must be created in the `chemsynth` namespace before deployment. Use the Ansible playbook to create them:

```bash
ansible-playbook ansible/playbook.yaml -e @vars.yaml
```

Or create them manually - see `ansible/README.md` for details.

Required secrets:
- `mongodb-credentials` - MongoDB root and user passwords
- `mysql-credentials` - MySQL root password
- `redis-credentials` - Redis password
- `rabbitmq-credentials` - RabbitMQ password
- `askcos-env` - ASKCOS application secrets (OAUTH2, V1 credentials)
- `gitlab-registry` - Container registry credentials for pulling images

## Deployment Steps

### Step 1: Create Secrets

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

# Check all resources in chemsynth namespace
kubectl -n chemsynth get all

# Check pods specifically
kubectl -n chemsynth get pods

# Check services
kubectl -n chemsynth get svc

# Check statefulsets
kubectl -n chemsynth get statefulsets

# Check deployments
kubectl -n chemsynth get deployments
```

### Step 4: Access Application

Once all pods are ready, access ASKCOS at: https://synth.maus.local

## DNS Configuration

Ensure `synth.maus.local` resolves to your cluster's external IP or load balancer.

## Customization

### Resource Configuration

Adjust resource requests/limits in the individual manifests under `k8s/chemsynth/`.

### Scaling

Scale components by modifying replica counts:
- Celery workers: Edit `k8s/chemsynth/celery/deployment.yaml`
- App instances: Edit `k8s/chemsynth/app/deployment.yaml`

### Enabling Additional Celery Workers

To add more specialized celery workers, create additional Deployment manifests in `k8s/chemsynth/celery/`.

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

### StatefulSet Issues

```bash
kubectl -n chemsynth describe statefulset <name>
kubectl -n chemsynth logs <pod-name>
kubectl -n chemsynth get pvc
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
4. **Persistent Storage**: All StatefulSets use persistent volumes for data durability.

## Resource Requirements

Minimum recommended resources:

| Component | CPU | Memory | Storage |
|-----------|-----|--------|---------|
| MongoDB | 1 vCPU | 1-2 Gi | 20 Gi |
| Redis | 0.5 vCPU | 128 Mi | 1 Gi |
| RabbitMQ | 0.5 vCPU | 256 Mi | 2 Gi |
| ASKCOS App | 1-2 vCPU | 2 Gi | - |
| Nginx | 0.25 vCPU | 256 Mi | - |
| Celery Worker | 1-2 vCPU | 2-4 Gi | - |

## Updates

To update ASKCOS, modify the image tags in the deployment manifests under `k8s/chemsynth/`. Argo CD will automatically sync the changes.

## References

- ASKCOS Helm Chart: https://github.com/ASKCOS/askcos-deploy/tree/main/helm/askcos
- FourThievesVinegar askcos2_core: https://github.com/FourThievesVinegar/askcos2_core
- Argo CD Documentation: https://argo-cd.readthedocs.io/
- cert-manager Documentation: https://cert-manager.io/docs/
- Traefik Documentation: https://doc.traefik.io/traefik/
