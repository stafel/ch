# ASKCOS Chemical Retrosynthesis - Ansible Deployment

This Ansible playbook deploys the ASKCOS (FourThievesVinegar fork) chemical retrosynthesis application to a Kubernetes cluster using Argo CD.

## Prerequisites

The target Kubernetes cluster must have the following already installed:

- **k3s** or any Kubernetes distribution
- **Traefik** as ingress controller
- **cert-manager** for TLS certificate management
- **Argo CD** for GitOps deployment
- **kubectl** configured to access the cluster
- **helm** CLI tool
- **Ansible** 2.14+ with `kubernetes.core` and `community.general` collections

### Required Ansible Collections

```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
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

## Usage

### Deploy Everything

```bash
ansible-playbook playbook.yaml
```

### Deploy with Custom Variables

```bash
ansible-playbook playbook.yaml \
  -e mongodb_root_password=mysecret \
  -e mongodb_password=mysecret \
  -e registry_username=myuser \
  -e registry_password=mypassword
```

### Use a Variables File

```bash
# Create vars.yaml with your secrets
ansible-playbook playbook.yaml -e @vars.yaml
```

### Encrypted Variables (Recommended)

For production, use Ansible Vault:

```bash
# Create encrypted vars file
ansible-vault create secrets.yaml

# Edit encrypted file
ansible-vault edit secrets.yaml

# Run playbook with vault
ansible-playbook playbook.yaml --ask-vault-pass
```

## What This Playbook Does

1. **Validates Prerequisites**: Checks that kubectl, helm are available and cluster is accessible

2. **Creates Namespace**: Creates the `chemsynth` namespace

3. **Creates TLS Certificate**: Uses cert-manager to create a certificate for `synth.maus.local`

4. **Creates IngressRoute**: Configures Traefik IngressRoute with TLS termination

5. **Creates All Required Secrets**:
   - MongoDB credentials
   - MySQL credentials
   - Redis credentials
   - RabbitMQ credentials
   - ASKCOS environment secrets
   - Container registry credentials

6. **Deploys Argo CD Application**: Creates the Argo CD Application that manages the ASKCOS Helm chart

## Post-Deployment Verification

### Check Certificate Status

```bash
kubectl -n chemsynth get certificate
kubectl -n chemsynth describe certificate synth-maus-local-tls
```

### Check Argo CD Application

```bash
argocd app get askcos-chemsynth
argocd app logs askcos-chemsynth
```

### Monitor Pods

```bash
kubectl -n chemsynth get pods -w
```

### Access the Application

Once all pods are ready, access ASKCOS at: https://synth.maus.local

## Customization

### Resource Configuration

Override resource requests/limits by setting these variables:

```yaml
mongodb_storage_size: 20Gi
mongodb_memory_request: 2Gi
mongodb_memory_limit: 4Gi
app_memory_request: 4Gi
nginx_memory_request: 512Mi
```

### Enable ML Servers

To enable ML servers, modify the Argo CD Application values in the playbook or create a custom values file.

### Custom Domain

Change the domain variable:

```yaml
domain: your-domain.com
```

### Different cert-manager Issuer

If using a self-signed issuer for testing:

```yaml
cert_manager_issuer: selfsigned-cluster-issuer
cert_manager_issuer_kind: ClusterIssuer
```

## Troubleshooting

### Certificate Not Issuing

```bash
# Check certificate events
kubectl -n chemsynth describe certificate

# Check cert-manager logs
kubectl -n cert-manager logs -l app=cert-manager

# Check challenges
kubectl -n chemsynth get challenges
```

### Pods Not Starting

```bash
# Check pod events
kubectl -n chemsynth get events --sort-by=.metadata.creationTimestamp

# Check pod logs
kubectl -n chemsynth logs <pod-name>

# Describe pod
kubectl -n chemsynth describe pod <pod-name>
```

### Argo CD Sync Issues

```bash
# Check sync status
argocd app get askcos-chemsynth

# Check sync logs
argocd app logs askcos-chemsynth

# Force sync
argocd app sync askcos-chemsynth
```

## Security Notes

1. **Never commit unencrypted secrets** to version control
2. Use Ansible Vault for production deployments
3. Rotate all `CHANGE_ME_*` passwords before production use
4. Consider using external secret managers (Vault, AWS Secrets Manager)

## Files

- `playbook.yaml` - Main Ansible playbook
- `docker-config.json.j2` - Template for Docker registry configuration
- `requirements.yml` - Ansible collection requirements
