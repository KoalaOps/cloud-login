# Cloud Login

Universal cloud authentication action that handles AWS, GCP, and Azure (beta) with automatic cluster and registry setup.

## Features

- 🔐 **Multi-cloud support** - AWS, GCP, and Azure (beta) in one action
- ☸️ **Automatic kubectl setup** - Configures kubeconfig for your cluster
- 🐳 **Registry authentication** - ECR, GAR, ACR login
- 📦 **ECR repository creation** - Ensures repositories exist (AWS)
- 🔄 **Cross-account support** - Handle multi-account setups

## Usage

```yaml
- name: Authenticate to cloud and cluster
  uses: KoalaOps/cloud-login@v1
  with:
    provider: aws
    account: "123456789"
    region: us-east-1
    cluster: production
    aws_role_to_assume: ${{ vars.AWS_DEPLOY_ROLE }}
```

## Inputs

### Common Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `provider` | Cloud provider (aws, gcp, azure) | ✅ | - |
| `account` | Account/Project/Subscription ID | ❌ | - |
| `region` | Region/Location | ✅ | - |
| `cluster` | Kubernetes cluster name | ❌ | - |
| `login_to_container_registry` | Enable container registry login | ❌ | `false` |

**Note for Azure:** The `repositories` input expects the ACR registry name (e.g., "myregistry"), not a list of repositories.

### AWS-Specific

| Input | Description | Required |
|-------|-------------|----------|
| `aws_role_to_assume` | IAM role ARN for OIDC | ❌* |
| `aws_access_key_id` | AWS access key | ❌* |
| `aws_secret_access_key` | AWS secret key | ❌* |
| `repositories` | AWS: ECR repos to ensure exist. Azure: ACR registry name | ❌ |

*Either use OIDC (role_to_assume) or access keys

### GCP-Specific

| Input | Description | Required |
|-------|-------------|----------|
| `gcp_workload_identity_provider` | Workload Identity provider | ❌* |
| `gcp_service_account` | Service account email | ❌* |
| `gcp_credentials_json` | Service account JSON key | ❌* |

*Either use Workload Identity or service account key

### Azure-Specific (Beta)

| Input | Description | Required |
|-------|-------------|----------|
| `azure_client_id` | Service principal client ID | ❌* |
| `azure_client_secret` | Service principal secret | ❌* |
| `azure_tenant_id` | Azure AD tenant ID | ❌* |
| `azure_subscription_id` | Azure subscription ID | ❌ |

*Either use managed identity (no credentials) or service principal

## Outputs

| Output | Description |
|--------|-------------|
| `account_id` | Resolved account/project ID |
| `registry_url` | Container registry URL (if enabled) |
| `kubectl_context` | Kubernetes context name |
| `authenticated` | Whether authentication succeeded (true/false) |

## Examples

### AWS with OIDC and ECR
```yaml
- name: Login to AWS
  uses: KoalaOps/cloud-login@v1
  with:
    provider: aws
    account: "123456789"
    region: us-east-1
    cluster: eks-production
    aws_role_to_assume: ${{ vars.AWS_ROLE }}
    login_to_container_registry: true
    repositories: backend,frontend
```

### GCP with Workload Identity
```yaml
- name: Login to GCP
  uses: KoalaOps/cloud-login@v1
  with:
    provider: gcp
    account: my-project
    region: us-central1
    cluster: gke-cluster
    gcp_workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    gcp_service_account: ${{ vars.WIF_SA }}
    login_to_container_registry: true
```

### Azure with Service Principal (Beta)
```yaml
- name: Login to Azure
  uses: KoalaOps/cloud-login@v1
  with:
    provider: azure
    account: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    region: my-resource-group  # Resource group for AKS
    cluster: aks-cluster
    azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
    azure_client_secret: ${{ secrets.AZURE_CLIENT_SECRET }}
    azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
    login_to_container_registry: true
    repositories: myacrregistry  # ACR registry name
```

### Azure with Managed Identity (Beta)
```yaml
- name: Login to Azure with MI
  uses: KoalaOps/cloud-login@v1
  with:
    provider: azure
    account: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    region: my-resource-group  # Resource group for AKS
    cluster: aks-cluster
    # No credentials needed - uses managed identity
    login_to_container_registry: true
```

### Parse tuple and authenticate
```yaml
- name: Parse cloud config
  uses: KoalaOps/parse-cloud-provider-cluster@v1
  id: cloud
  with:
    cloud_provider_cluster: ${{ inputs.cloud_tuple }}

- name: Authenticate
  uses: KoalaOps/cloud-login@v1
  with:
    provider: ${{ steps.cloud.outputs.provider }}
    account: ${{ steps.cloud.outputs.account }}
    region: ${{ steps.cloud.outputs.location }}
    cluster: ${{ steps.cloud.outputs.cluster }}
```

## Authentication Methods

### AWS
1. **OIDC (Recommended)**: Use `aws_role_to_assume`
2. **Access Keys**: Use `aws_access_key_id` and `aws_secret_access_key`

### GCP
1. **Workload Identity (Recommended)**: Use provider and service account
2. **Service Account Key**: Use `gcp_credentials_json`

### Azure (Beta)
1. **Managed Identity (Recommended)**: No credentials needed
2. **Service Principal**: Use client ID, secret, and tenant

## Building Blocks Used

This orchestrator action internally uses:

### For AWS
- `KoalaOps/login-aws@v1` - Complete AWS authentication
  - Which uses `KoalaOps/ensure-ecr-repository@v1` for ECR repos
  - Which uses `aws-actions/configure-aws-credentials@v4` for auth
  - Which uses `aws-actions/amazon-ecr-login@v2` for ECR login

### For GCP
- `KoalaOps/login-gcp-gke@v1` - Complete GCP authentication
  - Which uses `google-github-actions/auth@v2` for auth
  - Which uses `google-github-actions/get-gke-credentials@v2` for GKE

### For Azure (Beta)
- `azure/login@v2` - Azure authentication
- `azure/aks-set-context@v4` - AKS cluster setup
- Azure CLI for ACR login

## Notes

- Requires appropriate permissions in the runner environment
- For OIDC/Workload Identity, ensure `id-token: write` permission
- Registry login requires appropriate IAM permissions
- ECR repository creation requires `ecr:CreateRepository` permission