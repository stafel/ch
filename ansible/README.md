# Ansible Playbook for ASKCOS v2 Deployment

This Ansible playbook creates the required Kubernetes secrets for deploying ASKCOS v2 Chemical Retrosynthesis tool using Argo CD.

## Overview

The playbook performs the following:

1. **Namespace Creation**: Creates the `chemsynth` namespace
2. **Auto-Generated Passwords**: Generates secure random passwords for all database services
3. **Secret Creation**: Creates all required Kubernetes secrets:
   - `mongodb-credentials` - MongoDB root and user passwords
   - `mysql-credentials` - MySQL root password (for Keycloak, optional)
   - `redis-credentials` - Redis password
   - `rabbitmq-credentials` - RabbitMQ username and password
   - `askcos-env` - ASKCOS application secrets (V1_USERNAME, V1_PASSWORD)
   - `gitlab-registry` - Container registry credentials (required)
4. **Argo CD Deployment**: Deploys the Argo CD Application manifest

## Prerequisites

- Ansible 2.14+
- Python 3.6+
- `kubernetes.core` Ansible collection
- `community.general` Ansible collection
- kubectl configured with access to your Kubernetes cluster
- Cluster has Traefik, cert-manager, and Argo CD pre-installed

## Installation

### Install Ansible Collections

```bash
ansible-galaxy install -r requirements.yml
```

Or install manually:

```bash
ansible-galaxy collection install kubernetes.core community.general
```

## Usage

### Deployment

GitLab registry credentials are required to pull the private ASKCOS images. Database passwords are auto-generated:

```bash
ansible-playbook playbook.yaml \
  -e v1_password=YOUR_V1_PASSWORD \
  -e registry_server=registry.gitlab.com \
  -e registry_username=your_username \
  -e registry_password=your_password \
  -e registry_email=your@email.com
```

### Using a Variables File

Create a `vars.yaml` file:

```yaml
v1_username: "askcos"
v1_password: "your_v1_password_here"
registry_server: "registry.gitlab.com"
registry_username: "your_username"
registry_password: "your_password"
registry_email: "your@email.com"
```

Then run:

```bash
ansible-playbook playbook.yaml -e @vars.yaml
```

## Secrets Created

### Auto-Generated Secrets (32-character random passwords)

- **mongodb-credentials**
  - `mongodb-root-password`: Auto-generated
  - `mongodb-password`: Auto-generated
  - `mongodb-username`: `askcos` (hardcoded)

- **mysql-credentials**
  - `mysql-root-password`: Auto-generated
  - Note: MySQL is only used for Keycloak in v2, disabled by default

- **redis-credentials**
  - `redis-password`: Auto-generated

- **rabbitmq-credentials**
  - `rabbitmq-username`: `askcos` (configurable)
  - `rabbitmq-password`: Auto-generated

### User-Provided Secrets

- **askcos-env**
  - `V1_USERNAME`: Configurable (default: `askcos`)
  - `V1_PASSWORD`: **REQUIRED** - You must provide this

### Required Registry Secret

- **gitlab-registry**
  - Docker registry credentials for pulling private images
  - Uses `kubernetes.io/dockerconfigjson` type

## Argo CD Application Deployment

The playbook automatically deploys the Argo CD Application from `k8s/argocd/application.yaml`.

Argo CD will then deploy all manifests from `k8s/chemsynth/`:
- namespace.yaml
- certificate.yaml
- ingressroute.yaml
- app/configmap.yaml
- app/deployment.yaml
- web/deployment.yaml
- precompute/deployment.yaml
- celery/deployment.yaml
- mongodb/statefulset.yaml
- redis/statefulset.yaml
- rabbitmq/statefulset.yaml

## Security Notes

1. **Generated Passwords**: All database passwords are generated using Ansible's `password` lookup with 32-character random strings
2. **No Secrets in Repo**: No sensitive data is stored in the repository
3. **Display**: Generated passwords are displayed at the end of the playbook run - save them securely
4. **Kubernetes Secrets**: All secrets are stored as Kubernetes Secrets (base64 encoded)

## Verification

After running the playbook:

```bash
# Check namespace
kubectl get namespace chemsynth

# Check secrets
kubectl -n chemsynth get secrets

# Check Argo CD application
argocd app get askcos-chemsynth

# Monitor sync
argocd app logs askcos-chemsynth
```

## Customization

### Password Length

Modify `passwd_len` variable in the playbook (default: 32):

```yaml
vars:
  passwd_len: 64  # Use 64-character passwords
```

### RabbitMQ Username

```yaml
vars:
  rabbitmq_username: "custom_rabbitmq_user"
```

### MongoDB Username

The MongoDB username is currently hardcoded to `askcos` in the manifests. To change it, update the `MONGO_INITDB_ROOT_USERNAME` and `MONGO_USER` values in the MongoDB StatefulSet manifest.

## Troubleshooting

### kubectl not found

```bash
# Install kubectl
# On Ubuntu/Debian:
sudo apt-get install -y kubectl

# On macOS:
brew install kubectl
```

### Python module kubernetes not found

```bash
pip install kubernetes
```

### Ansible collection not found

```bash
ansible-galaxy collection install kubernetes.core community.general
```

### Connection to Kubernetes cluster failed

```bash
# Verify kubectl configuration
kubectl cluster-info

# Check current context
kubectl config current-context

# If using k3s, ensure KUBECONFIG is set
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## Files

- `playbook.yaml` - Main Ansible playbook
- `docker-config.json.j2` - Jinja2 template for Docker registry configuration
- `requirements.yml` - Ansible collection requirements
- `README.md` - This file

## References

- ASKCOS v2: https://gitlab.com/mlpds_mit/askcosv2/askcos2_core
- Ansible kubernetes.core collection: https://galaxy.ansible.com/kubernetes/core
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
