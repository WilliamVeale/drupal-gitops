# Drupal 11 GitOps Repository

This repository contains the FluxCD GitOps configuration for deploying Drupal 11 on Azure Kubernetes Service (AKS).

## Repository Structure

```
drupal-gitops/
├── apps/                      # Application manifests
│   ├── base/drupal/          # Base Drupal configuration
│   └── production/           # Production overlays
├── infrastructure/            # Infrastructure components
│   ├── base/                 # Base infrastructure
│   │   ├── storage/         # Azure Blob NFS StorageClass
│   │   ├── traefik/         # Traefik ingress controller
│   │   └── cert-manager/    # Certificate management
│   └── production/          # Production environment
└── clusters/production/      # Cluster configuration
    └── flux-system/         # FluxCD system files
```

## Prerequisites

1. **Azure Resources**:
   - AKS cluster with Blob CSI driver enabled
   - Azure Container Registry
   - Azure Database for MySQL Flexible Server
   - Azure Cache for Redis
   - Azure Blob Storage (Premium tier with NFS)
   - Azure Key Vault (for SOPS encryption)

2. **Tools**:
   - kubectl
   - flux CLI
   - sops
   - Azure CLI

## Deployment

### 1. Encrypt Secrets

Before committing, encrypt the secrets file:

```bash
# Ensure .sops.yaml is configured with your Azure Key Vault details
sops -e -i apps/production/secrets.yaml
```

### 2. Bootstrap FluxCD

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux
flux bootstrap git \
  --url=https://dev.azure.com/yourorg/yourproject/_git/drupal-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth
```

### 3. Configure SOPS Decryption

```bash
kubectl create secret generic sops-azure \
  --namespace=flux-system \
  --from-literal=sops.azure-kv.url=https://your-keyvault.vault.azure.net/keys/sops-key/version
```

### 4. Monitor Deployment

```bash
# Watch Flux reconciliation
flux get kustomizations --watch

# Check specific components
kubectl get pods -n drupal
kubectl get pods -n traefik
kubectl get pvc -n drupal
```

## Key Features

### Azure Blob CSI Driver Storage
- Uses NFS protocol for POSIX filesystem semantics
- Kernel-level metadata caching (actimeo=60) for performance
- No PHP storage modules required
- Managed Identity authentication

### GitOps Workflow
- Infrastructure deployed first (storage, ingress, certificates)
- Database migrations run as a separate Job before application deployment
- Application pods start only after successful migration
- Automated image updates via FluxCD image automation

### Security
- Secrets encrypted with SOPS and Azure Key Vault
- TLS certificates via cert-manager and Let's Encrypt
- Security headers and rate limiting via Traefik middlewares
- HTTPS-only with automatic HTTP redirect

## Configuration

### Update Domain Name

Update the domain in the following files:
- `apps/base/drupal/configmap.yaml` - DRUPAL_DOMAIN
- `apps/base/drupal/ingressroute.yaml` - Host() matchers
- `infrastructure/production/certificates.yaml` - dnsNames

### Update Container Registry

Update ACR references in:
- `apps/base/drupal/deployment.yaml`
- `apps/base/drupal/migration-job.yaml`
- `apps/base/drupal/cronjob.yaml`
- `clusters/production/image-automation.yaml`

### Adjust Resource Limits

Edit resource requests/limits in:
- `apps/base/drupal/deployment.yaml`
- `apps/base/drupal/migration-job.yaml`
- `apps/base/drupal/cronjob.yaml`

## Operations

### Manual Migration

```bash
kubectl delete job drupal-migration -n drupal
kubectl apply -f apps/base/drupal/migration-job.yaml
kubectl logs -n drupal job/drupal-migration -f
```

### Scale Application

```bash
kubectl scale deployment/drupal -n drupal --replicas=5
```

### Clear Cache

```bash
kubectl exec -it -n drupal deployment/drupal -- ./vendor/bin/drush cr
```

### View Logs

```bash
kubectl logs -n drupal -l app=drupal -f
```

## Troubleshooting

### Check Storage Mount

```bash
# Verify PVC is bound
kubectl get pvc -n drupal

# Check mount options
kubectl exec -it -n drupal deployment/drupal -- cat /proc/mounts | grep files
# Should show: actimeo=60,nconnect=8,noatime
```

### Verify Database Connection

```bash
kubectl exec -it -n drupal deployment/drupal -- \
  ./vendor/bin/drush sql:query "SELECT 1"
```

### Check Redis Connection

```bash
kubectl exec -it -n drupal deployment/drupal -- \
  ./vendor/bin/drush eval "var_dump(extension_loaded('redis'));"
```

## References

- [Drupal 11 Documentation](https://www.drupal.org/docs/11)
- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Azure Blob CSI Driver](https://github.com/kubernetes-sigs/blob-csi-driver)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
