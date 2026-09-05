# ASKCOS Chemical Retrosynthesis Deployment (v2)

Argo CD compatible Kubernetes deployment for **ASKCOS v2** (FourThievesVinegar fork of MIT MLPDS ASKCOS v2) chemical retrosynthesis tool with standard Kubernetes Ingress and cert-manager TLS certificates.

---

## Architecture Overview

| Aspect | Detail |
|--------|--------|
| **Namespace** | `chemsynth` |
| **Domain** | `synth.maus.local` |
| **Ingress** | Standard Kubernetes Ingress (works with Traefik, Nginx, etc.) |
| **Certificate** | cert-manager managed TLS certificate |
| **Deployment** | Argo CD Application managing all ASKCOS v2 components |
| **API Gateway** | FastAPI-based (not Django) |
| **Frontend** | Vue.js 3 with Vite |

---

## Prerequisites

The Kubernetes cluster must have the following installed and configured:

- k3s or any Kubernetes distribution
- Ingress controller (Traefik, Nginx Ingress, etc.)
- cert-manager for TLS certificate management
- Argo CD for GitOps deployment
- ClusterIssuer for cert-manager (e.g., `letsencrypt-prod`)

---

## Deployment Structure

```
.
├── README.md                          # This file
├── ansible/
│   ├── playbook.yaml                  # Creates required secrets
│   ├── docker-config.json.j2          # Registry config template
│   ├── README.md                      # Ansible documentation
│   └── requirements.yml               # Ansible collection requirements
└── k8s/
    ├── argocd/
    │   └── application.yaml            # Argo CD Application manifest
    └── chemsynth/
        ├── namespace.yaml              # Namespace definition
        ├── certificate.yaml            # cert-manager Certificate
        ├── ingressroute.yaml            # Standard Kubernetes Ingress
        ├── app/
        │   ├── configmap.yaml           # ASKCOS v2 Configuration
        │   └── deployment.yaml          # ASKCOS FastAPI App v2
        ├── web/
        │   └── deployment.yaml          # ASKCOS Vue.js Web v2
        ├── precompute/
        │   └── deployment.yaml          # Precompute Service v2
        ├── celery/
        │   └── deployment.yaml          # Celery Workers v2
        ├── mongodb/
        │   └── statefulset.yaml         # MongoDB 6.0 StatefulSet + Service
        ├── redis/
        │   └── statefulset.yaml         # Redis 7.0 StatefulSet + Service
        └── rabbitmq/
            └── statefulset.yaml         # RabbitMQ 3.11 StatefulSet + Service
```

---

## Components Deployed (ASKCOS v2)

| Component | Type | Image | Port | Purpose |
|-----------|------|-------|------|---------|
| `namespace` | Namespace | - | - | Isolated namespace for ASKCOS |
| `certificate` | Certificate | - | - | TLS certificate for synth.maus.local |
| `ingress` | Ingress | - | - | Standard Kubernetes Ingress with TLS |
| `mongodb` | StatefulSet | `mongo:6.0.9-jammy` | 27017 | MongoDB database with persistent storage |
| `redis` | StatefulSet | `redis:7.0-alpine` | 6379 | Redis cache with persistent storage |
| `rabbitmq` | StatefulSet | `rabbitmq:3.11-alpine` | 5672, 15672 | RabbitMQ message broker with persistent storage |
| `askcos-app` | Deployment | `registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core/app:2.0` | 9100 | ASKCOS FastAPI API Gateway v2 |
| `askcos-web` | Deployment | `registry.gitlab.com/mlpds_mit/askcosv2/askcos-vue-nginx:2.0` | 80 | Vue.js Web Frontend v2 and API reverse proxy |
| `askcos-precompute` | Deployment | `registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core/precompute:2.0` | - | Precomputation Service v2 |
| `askcos-celery-worker` | Deployment | `registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core/celery:2.0` | - | Celery workers for background tasks v2 |

---

## Quick Start

### Step 1: Create Secrets

Use the Ansible playbook to create all required Kubernetes secrets:

```bash
# Registry credentials are required to pull the private ASKCOS images.
ansible-playbook ansible/playbook.yaml \
  -e v1_password=YOUR_V1_PASSWORD \
  -e registry_server=registry.gitlab.com \
  -e registry_username=your_username \
  -e registry_password=your_password \
  -e registry_email=your@email.com
```

### Step 2: Deploy Argo CD Application

```bash
kubectl apply -f k8s/argocd/application.yaml
```

### Step 3: Monitor Deployment

```bash
# Check Argo CD sync status
argocd app get askcos-chemsynth

# View sync logs
argocd app logs askcos-chemsynth

# Check all resources in chemsynth namespace
kubectl -n chemsynth get all,statefulsets,pvc

# Check pods specifically
kubectl -n chemsynth get pods -w
```

### Step 4: Access Application

Once all pods are ready, access ASKCOS v2 at:

**https://synth.maus.local**

---

## Required Secrets

The following secrets are created by the Ansible playbook in the `chemsynth` namespace:

| Secret | Keys | Source |
|--------|------|--------|
| `mongodb-credentials` | `mongodb-root-password`, `mongodb-password`, `mongodb-username` | Auto-generated |
| `mysql-credentials` | `mysql-root-password` | Auto-generated (optional, for Keycloak) |
| `redis-credentials` | `redis-password` | Auto-generated |
| `rabbitmq-credentials` | `rabbitmq-username`, `rabbitmq-password` | Auto-generated |
| `askcos-env` | `V1_USERNAME`, `V1_PASSWORD` | User-provided |
| `gitlab-registry` | `.dockerconfigjson` | Required to pull private ASKCOS images |

---

## ASKCOS v2 Architecture Changes

### Key Differences from v1:

| Aspect | v1 | v2 |
|--------|----|----|
| **API Gateway** | Django | FastAPI |
| **Frontend** | Django templates | Vue.js 3 with Vite |
| **Web Server** | Separate Nginx | Integrated in web image |
| **Precompute** | Part of app | Separate service |
| **Celery Queues** | Basic | Enhanced with 7 queue types |
| **Images** | Various | All from `registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core` |

### Service Ports:

| Service | Port | Protocol |
|---------|------|-----------|
| askcos-app | 9100 | HTTP |
| askcos-web | 80 | HTTP (TLS terminates at the Ingress) |
| mongodb | 27017 | MongoDB |
| redis | 6379 | Redis |
| rabbitmq | 5672 | AMQP |
| rabbitmq | 15672 | HTTP |

---

## Configuration

### ASKCOS v2 Environment Variables

Key configuration in `k8s/chemsynth/app/configmap.yaml`:

```yaml
MODULE_CONFIG_PATH: "configs/module_config_full.py"
VERSION_NUMBER: "2026.07"
PROTOCOL: "https"
GATEWAY_URL: "https://synth.maus.local"
VITE_ORGANIZATION: "ChemSynth"
VITE_GUEST_LOGIN: "True"
VITE_EMAIL_REQUIRED: "False"
```

### Celery Queues

The celery workers handle the following queues:
- `atom_mapping_worker`
- `cr_network_worker`
- `descriptors_worker`
- `sites_worker`
- `tffp_worker`
- `tb_coordinator_mcts`
- `tb_c_worker`

---

## Resource Requirements

### Minimum (Development/Test)

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|-----------|-------------|-----------|----------------|--------------|---------|
| MongoDB | 500m | 2 | 2 Gi | 4 Gi | 20 Gi |
| Redis | 250m | 500m | 512 Mi | 1 Gi | 1 Gi |
| RabbitMQ | 250m | 500m | 1 Gi | 2 Gi | 2 Gi |
| ASKCOS App | 500m | 2 | 2 Gi | 4 Gi | - |
| ASKCOS Web | 250m | 500m | 512 Mi | 1 Gi | - |
| Precompute | 1 | 2 | 4 Gi | 8 Gi | 10 Gi + 5 Gi |
| Celery Worker | 1 | 2 | 4 Gi | 8 Gi | - |
| **Total** | **~4.5** | **~9** | **~14.5 Gi** | **~29 Gi** | **~38 Gi** |

### Recommended (Production)

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit | Storage |
|-----------|-------------|-----------|----------------|--------------|---------|
| MongoDB | 1 | 4 | 4 Gi | 8 Gi | 50 Gi |
| Redis | 500m | 2 | 1 Gi | 2 Gi | 2 Gi |
| RabbitMQ | 500m | 2 | 2 Gi | 4 Gi | 5 Gi |
| ASKCOS App | 1 | 4 | 4 Gi | 8 Gi | - |
| ASKCOS Web | 500m | 2 | 1 Gi | 2 Gi | - |
| Precompute | 2 | 4 | 8 Gi | 16 Gi | 20 Gi + 10 Gi |
| Celery Worker | 2 | 4 | 8 Gi | 16 Gi | - |
| **Total** | **~8** | **~22** | **~26 Gi** | **~52 Gi** | **~87 Gi** |

---

## Customization

### Scaling Components

Scale components by modifying replica counts in their deployment manifests:

```bash
# Scale celery workers
kubectl -n chemsynth scale deployment askcos-celery-worker --replicas=4

# Scale app instances
kubectl -n chemsynth scale deployment askcos-app --replicas=2
```

### Resource Limits

Adjust resource requests/limits in the individual manifests under `k8s/chemsynth/`.

### Configuration

Modify `k8s/chemsynth/app/configmap.yaml` for ASKCOS v2 settings.

---

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

### Application Logs

```bash
# View ASKCOS app logs
kubectl -n chemsynth logs -l app=askcos-app --tail=100

# View web frontend logs
kubectl -n chemsynth logs -l app=askcos-web --tail=100

# View celery worker logs
kubectl -n chemsynth logs -l app=askcos-celery --tail=100

# View precompute logs
kubectl -n chemsynth logs -l app=askcos-precompute --tail=100
```

---

## DNS Configuration

Ensure `synth.maus.local` resolves to your cluster's external IP or load balancer.

```bash
# Test DNS resolution
ping synth.maus.local
nslookup synth.maus.local
dig synth.maus.local
```

---

## Updates

To update ASKCOS v2, modify the image tags in the deployment manifests under `k8s/chemsynth/`.

```bash
# Update to a new version
kubectl -n chemsynth set image deployment/askcos-app \
  askcos-app=registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core/app:NEW_VERSION

# Or let Argo CD handle it by updating the manifests
```

---

## Security Considerations

1. **Secrets Management**: All sensitive data is stored in Kubernetes Secrets. Use Ansible Vault or external secret managers for production.
2. **Network Security**: Consider adding NetworkPolicies to restrict pod communication.
3. **Access Control**: Configure RBAC for Argo CD and restrict dashboard access.
4. **Persistent Storage**: All StatefulSets use persistent volumes for data durability.
5. **Auto-Generated Passwords**: All database passwords are auto-generated with secure random values.

---

## References

- [ASKCOS v2 Core](https://gitlab.com/mlpds_mit/askcosv2/askcos2_core)
- [FourThievesVinegar askcos2_core](https://github.com/FourThievesVinegar/askcos2_core)
- [ASKCOS Documentation](https://gitlab.com/mlpds_mit/askcosv2/askcos-docs/-/wikis/home)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)

---

## Version Information

| Aspect | Version |
|--------|---------|
| **Deployment Version** | v2.0 |
| **ASKCOS Version** | 2026.07 |
| **Architecture** | FastAPI + Vue.js 3 + Celery |
| **Images** | `registry.gitlab.com/mlpds_mit/askcosv2/askcos2_core:*:2.0` |

---

## License

This deployment configuration is provided as-is for the ASKCOS chemical retrosynthesis tool.

---

## Support

For issues with this deployment configuration, please check:
1. The [GitHub repository](https://github.com/stafel/ch)
2. The [ASKCOS v2 documentation](https://gitlab.com/mlpds_mit/askcosv2/askcos-docs/-/wikis/home)

For ASKCOS-specific questions, refer to the official ASKCOS documentation and support channels.
