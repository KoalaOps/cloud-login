# Koala Cloud Login

Unified cloud authentication for **AWS**, **GCP**, **Azure (beta)**, plus registry login for **ECR**, **GAR**, **ACR**, **GHCR**, and **Docker Hub** — with optional Kubernetes context setup.

## Highlights

- 🔐 **One action** for AWS / GCP / Azure (beta) auth
- 🔍 **Auto-detect by image** (e.g., `*.dkr.ecr.*.amazonaws.com`, `*.docker.pkg.dev`, `*.azurecr.io`, `ghcr.io`, `docker.io`)
- ☸️ **Kube context** setup when `cluster` is provided (EKS/GKE/AKS)
- 🐳 **Registry login** for ECR/GAR/ACR/GHCR/Docker Hub
- 📦 **ECR repo ensure** (optional) for AWS
- 🔄 Works with OIDC (recommended) or keys/creds

> **Location model:** Use a single `location` input.
>
> - **AWS:** region (e.g., `us-east-1`)
> - **GCP:** location (`us`, `europe`, `asia`, `us-central1`, or `us-central1-a`)
> - **Azure:** use `azure_resource_group` for AKS; `location` is not used for Azure.

---

## Quick Start (auto-detect from image)

```yaml
permissions:
  id-token: write   # for OIDC (AWS/GCP/Azure)
  contents: read
  packages: write   # only if you'll push to GHCR

- name: Cloud login (auto)
  uses: KoalaOps/cloud-login@v1
  with:
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-svc
    login_to_container_registry: true
    # Provide possible credentials (only the matching provider is used)
    aws_role_to_assume: ${{ vars.AWS_BUILD_ROLE }}
    gcp_workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    gcp_service_account: ${{ vars.WIF_SA }}
    azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
    azure_client_secret: ${{ secrets.AZURE_CLIENT_SECRET }}
    azure_tenant_id: ${{ secrets.AZURE_TENANT_ID }}
```

---

## Common Examples

### AWS (OIDC) + ECR login + EKS kubeconfig

```yaml
permissions:
  id-token: write
  contents: read

- uses: KoalaOps/cloud-login@v1
  with:
    provider: aws
    location: us-east-1
    cluster: eks-prod
    login_to_container_registry: true
    # Ensure one or more repos exist (comma-separated)
    repositories: backend,frontend
    aws_role_to_assume: ${{ vars.AWS_DEPLOY_ROLE }}
```

### AWS (Access keys) + ECR login (no cluster)

```yaml
- uses: KoalaOps/cloud-login@v1
  with:
    provider: aws
    location: eu-central-1
    login_to_container_registry: true
    repositories: my-svc
    aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### GCP (GKE + GAR login)

```yaml
permissions:
  id-token: write
  contents: read

- uses: KoalaOps/cloud-login@v1
  with:
    provider: gcp
    account: my-project
    location: us-central1
    cluster: gke-prod
    login_to_container_registry: true
    gcp_workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    gcp_service_account: ${{ vars.WIF_SA }}
```

### Azure (AKS + ACR, Service Principal)

```yaml
permissions:
  id-token: write
  contents: read

- uses: KoalaOps/cloud-login@v1
  with:
    provider: azure
    account: ${{ vars.AZURE_SUBSCRIPTION_ID }}   # or set azure_subscription_id
    azure_resource_group: my-rg
    cluster: aks-prod
    login_to_container_registry: true
    acr_registry: myacrname
    azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
    azure_client_secret: ${{ secrets.AZURE_CLIENT_SECRET }}
    azure_tenant_id: ${{ secrets.AZURE_TENANT_ID }}
```

### GHCR (login only)

```yaml
permissions:
  id-token: write
  contents: read
  packages: write

- uses: KoalaOps/cloud-login@v1
  with:
    provider: github
    login_to_container_registry: true
    # github_token defaults to GITHUB_TOKEN
```

### Docker Hub (login only)

```yaml
- uses: KoalaOps/cloud-login@v1
  with:
    provider: dockerhub
    login_to_container_registry: true
    dockerhub_username: ${{ secrets.DOCKERHUB_USERNAME }}
    dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

---

## Inputs

| Input                         | Description                                                                                  | Required When                                                   |
| ----------------------------- | -------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `image`                       | Image URL to auto-detect provider/account/location/registry                                  | If you prefer auto-detect                                       |
| `provider`                    | One of `aws`, `gcp`, `azure`, `github`, `dockerhub`                                          | If not using `image`                                            |
| `account`                     | AWS account ID / GCP project ID / Azure subscription ID                                      | GCP with GKE or GAR; Azure; optional for AWS (resolved via STS) |
| `location`                    | **AWS:** region. **GCP:** location (`us`, `europe`, `asia`, `us-central1`, `us-central1-a`). | AWS/GCP when cluster or registry login is used                  |
| `cluster`                     | Kubernetes cluster name (EKS/GKE/AKS)                                                        | If you want kubecontext configured                              |
| `login_to_container_registry` | `true`/`false` (default `false`)                                                             | If you want registry login                                      |
| `repositories`                | AWS ECR repos to ensure exist (comma-separated)                                              | Optional (AWS only)                                             |
| `acr_registry`                | Azure ACR name (e.g., `myacrname`)                                                           | Required if Azure + registry login                              |

**AWS auth**  
`aws_role_to_assume` (recommended), or `aws_access_key_id` + `aws_secret_access_key`  
Optional: `aws_session_duration` (default `3600`)

**GCP auth**  
`gcp_workload_identity_provider`, `gcp_service_account`, or `gcp_credentials_json`

**Azure auth**  
`azure_client_id`, `azure_client_secret`, `azure_tenant_id` _(Service Principal)_  
Alternative: Managed Identity (via `azure/login@v2` without client/secret; subscription ID still needed)  
`azure_subscription_id` (alias for `account`), `azure_resource_group` (for AKS)

**GHCR / Docker Hub**  
`github_token` (defaults to `GITHUB_TOKEN`)  
`dockerhub_username`, `dockerhub_token`

---

## Outputs

| Output            | Description                                                                                                               |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `account_id`      | Resolved cloud account/subscription/project (or `unknown` for GHCR/Docker Hub)                                            |
| `registry_url`    | Base registry URL (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com`, `us-docker.pkg.dev/my-project`, `myacr.azurecr.io`) |
| `kubectl_context` | The current kubectl context (if `cluster` given)                                                                          |
| `authenticated`   | `"true"` if the action completed without auth errors                                                                      |

---

## How it works

1. **Parse** (optional) — if `image` is set, auto-detect provider/account/location/repo.
2. **Normalize & validate** minimal inputs based on what you want (kubecontext, registry login).
3. **Authenticate** to the cloud using OIDC (recommended) or credentials.
4. **Kubecontext** — if `cluster` is set, configure EKS/GKE/AKS context.
5. **Registry login** — if enabled, log in to ECR/GAR/ACR/GHCR/Docker Hub; ensure ECR repos if requested.
6. **Outputs** — emit `account_id`, `registry_url`, `kubectl_context`, `authenticated`.

---

## Permissions & roles (quick notes)

- **GHCR:** pushing requires `permissions: packages: write`.
- **AWS OIDC:** allow `sts:AssumeRoleWithWebIdentity`; ECR login requires basic ECR permissions; repo creation needs `ecr:CreateRepository`.
- **GCP:** GAR needs `roles/artifactregistry.writer` (push) or `reader` (pull); GKE needs `container.clusterViewer`.
- **Azure:** ACR `AcrPull`/`AcrPush` as needed; AKS cluster user permissions for kubecontext.

> Runners should have `aws`, `gcloud`, `az`, and `kubectl` available (GitHub‐hosted Ubuntu runners do).

---

## Troubleshooting

- **GAR host mismatch:** Use a correct `location`. Host is `<location>-docker.pkg.dev` (except the multi-region literals `us|europe|asia`).
- **“Region required” errors:** Provide `location` whenever you use EKS/GKE or registry login on AWS/GCP.
- **Azure ACR:** You must provide `acr_registry` (we don’t guess).
