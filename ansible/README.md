# ASKCOS Chemical Retrosynthesis - Ansible Secrets Setup

This Ansible playbook creates the required Kubernetes secrets for deploying ASKCOS (FourThievesVinegar fork) chemical retrosynthesis application. The actual application deployment is managed by Argo CD.

## Prerequisites

The target Kubernetes cluster must have the following already installed and configured:

- k3s or any Kubernetes distribution
- kubectl configured to access the cluster
- Ansible 2.14+ with `kubernetes.core` collection

## Quick Start

### 1. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Variables

Edit `playbook.yaml` and change the user-facing credentials:

```yaml
# playbook.yaml
v1_password: "CHANGE_ME_v1_password"
```

Or create a `vars.yaml` file with your values:

```yaml
# vars.yaml
v1_password: "your_v1_password"
```

For container registry access (optional - only needed for private registries):

```yaml
# vars.yaml
registry_enabled: true
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

2. **Auto-generates secure passwords** for non-user-facing database credentials:
   - MongoDB root password (32 chars, random)
   - MongoDB user password (32 chars, random)
   - MySQL root password (32 chars, random)
   - Redis password (32 chars, random)
   - RabbitMQ password (32 chars, random)

3. **Creates All Required Secrets** in the `chemsynth` namespace:
   - `mongodb-credentials` - MongoDB root and user passwords (auto-generated)
   - `mysql-credentials` - MySQL root password (auto-generated)
   - `redis-credentials` - Redis password (auto-generated)
   - `rabbitmq-credentials` - RabbitMQ password (auto-generated)
   - `askcos-env` - ASKCOS application secrets (V1 credentials - you provide)
   - `gitlab-registry` - Container registry credentials (optional, only if `registry_enabled: true`)

## Post-Secrets Creation: Deploy with Argo CD

After running this playbook, deploy the Argo CD Application:

```bash
kubectl apply -f k8s/argocd/application.yaml
```

The Argo CD Application will deploy:
- Namespace: `chemsynth`
- Certificate: TLS certificate for `synth.maus.local` via cert-manager
- IngressRoute: Traefik configuration for HTTPS access
- All ASKCOS components (MongoDB, Redis, RabbitMQ, App, Nginx, Celery)

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

### Container Registry

By default, the registry secret creation is **disabled** (`registry_enabled: false`). 
This allows anonymous access to public container images.

To enable private registry access:

```yaml
registry_enabled: true
registry_username: "your_username"
registry_password: "your_token_or_password"
registry_email: "your.email@example.com"
```

## Security Notes

1. **Auto-generated passwords**: All database passwords are generated with secure random values (32 characters, including letters, digits, hexdigits, and special characters).
2. **Only user-facing credentials need manual input**: Only `v1_password` must be provided by you.
3. **Never commit unencrypted secrets** to version control
4. Use Ansible Vault for production deployments
5. The playbook only creates secrets - it does not deploy any application resources

## Files

- `playbook.yaml` - Main Ansible playbook for secret creation
- `docker-config.json.j2` - Template for Docker registry configuration
- `requirements.yml` - Ansible collection requirements
