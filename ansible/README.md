# ASKCOS Chemical Retrosynthesis - Ansible Secrets Setup

This Ansible playbook creates the required Kubernetes secrets for deploying ASKCOS (FourThievesVinegar fork) chemical retrosynthesis application. The actual application deployment is managed by Argo CD.

## Prerequisites

The target Kubernetes cluster must have the following already installed:

- **k3s** or any Kubernetes distribution
- **Traefik** as ingress controller
- **cert-manager** for TLS certificate management
- **Argo CD** for GitOps deployment
- **kubectl** configured to access the cluster
- **Ansible** 2.14+ with `kubernetes.core` collection

### Required Ansible Collections

```bash
ansible-galaxy collection install kubernetes.core
```

## Quick Start

### 1. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Variables

Edit `playbook.yaml` and change all `CHANGE_ME_*` variables:

- Database passwords (mongodb, mysql, redis, rabbitmq)
- ASKCOS secrets (oauth2_secret_key, v1_password)
- Container registry credentials (gitlab_username, gitlab_access_token)

Or create a `vars.yaml` file with your values:

```yaml
# vars.yaml
mongodb_root_password: "your_secure_password"
mongodb_password: "your_secure_password"
mysql_root_password: "your_secure_password"
redis_password: "your_secure_password"
rabbitmq_password: "your_secure_password"
oauth2_secret_key: "your_oauth2_secret"
v1_password: "your_v1_password"
registry_username: "your_gitlab_username"
registry_password: "your_gitlab_access_token"
registry_email: "your.email@example.com"
```

Then run the playbook with:
```bash
ansible-playbook playbook.yaml -e @vars.yaml
```

### 3. Run the Playbook

```bash
# Dry run (check what will be done)
ansible-playbook playbook.yaml --check

# Actual deployment
ansible-playbook playbook.yaml
```

## What This Playbook Does

This playbook **only creates Kubernetes secrets**. It does NOT deploy the application.

1. **Validates Prerequisites**: Checks that kubectl is available and cluster is accessible

2. **Creates All Required Secrets**:
   - `mongodb-credentials` - MongoDB root and user passwords
   - `mysql-credentials` - MySQL root password
   - `redis-credentials` - Redis password
   - `rabbitmq-credentials` - RabbitMQ password
   - `askcos-env` - ASKCOS application secrets (OAUTH2, V1 credentials)
   - `gitlab-registry` - Container registry credentials for pulling images

## Post-Secrets Creation: Deploy with Argo CD

After running this playbook, deploy the Argo CD Application:

```bash
# Deploy Argo CD Application
kubectl apply -f k8s/argocd/application.yaml
```

The Argo CD Application will deploy:
- Namespace: `chemsynth`
- Certificate: TLS certificate for `synth.maus.local` via cert-manager
- IngressRoute: Traefik configuration for HTTPS access
- ASKCOS Helm chart from `ASKCOS/askcos-deploy` with custom values

### Verify Deployment

```bash
# Check Argo CD sync status
argocd app get askcos-chemsynth

# Monitor pods
kubectl -n chemsynth get pods -w

# Check certificate
kubectl -n chemsynth get certificate

# Access application
# https://synth.maus.local
```

## Customization

### Encrypted Variables (Recommended for Production)

For production, use Ansible Vault:

```bash
# Create encrypted vars file
ansible-vault create secrets.yaml

# Edit encrypted file
ansible-vault edit secrets.yaml

# Run playbook with vault
ansible-playbook playbook.yaml --ask-vault-pass
```

### Custom Namespace

Change the namespace variable:

```yaml
namespace: your-namespace
```

## Troubleshooting

### Secrets Not Created

```bash
# Check if namespace exists
kubectl get ns chemsynth

# Check secret creation
kubectl -n chemsynth get secrets

# Check kubectl access
kubectl cluster-info
```

### Permission Issues

Ensure your kubectl context has permissions to create secrets in the target namespace.

## Security Notes

1. **Never commit unencrypted secrets** to version control
2. Use Ansible Vault for production deployments
3. Rotate all `CHANGE_ME_*` passwords before production use
4. The playbook only creates secrets - it does not deploy any application resources

## Files

- `playbook.yaml` - Main Ansible playbook for secret creation
- `docker-config.json.j2` - Template for Docker registry configuration
- `requirements.yml` - Ansible collection requirements
