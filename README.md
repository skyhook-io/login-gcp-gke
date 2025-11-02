# Login GCP GKE

Authenticates to Google Cloud Platform and configures kubectl for GKE clusters.

## Features

- 🔐 **Workload Identity Federation** - Secure keyless authentication
- 🔑 **Service Account Keys** - Traditional authentication
- ☸️ **GKE Integration** - Automatic kubectl configuration  
- 🐳 **GAR Support** - Google Artifact Registry login
- 🔄 **Multi-cluster** - Handle multiple GKE clusters

## Usage

```yaml
- name: Login to GCP and GKE
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.WIF_SERVICE_ACCOUNT }}
    project_id: my-project
    location: us-central1
    cluster_name: production
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `project_id` | GCP project ID | ✅ | - |
| `location` | GKE cluster location (region or zone, e.g. `us-central1` or `us-central1-a`) | ✅ | - |
| `cluster_name` | GKE cluster name | ❌ | - |
| `workload_identity_provider` | WIF provider | ❌* | - |
| `service_account` | Service account email | ❌* | - |
| `credentials_json` | Service account JSON key | ❌* | - |
| `gar_location` | GAR location (defaults to cluster location) | ❌ | cluster location |
| `skip_gar_login` | Skip GAR Docker login | ❌ | `false` |

*Use either Workload Identity (provider + service_account) OR credentials_json

## Outputs

| Output | Description |
|--------|-------------|
| `project_id` | GCP project ID |
| `gar_registry` | GAR registry URL if configured |
| `gar_logged_in` | `true` if GAR login was performed, `false` otherwise |
| `kubectl_context` | Kubernetes context name |

## Examples

### Workload Identity Federation (Recommended)
```yaml
- name: GCP auth with WIF
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github/providers/github
    service_account: github-actions@my-project.iam.gserviceaccount.com
    project_id: my-project
    location: us-central1
    cluster_name: prod-cluster
```

### Service Account Key
```yaml
- name: GCP auth with key
  uses: skyhook-io/login-gcp-gke@v1
  with:
    credentials_json: ${{ secrets.GCP_CREDENTIALS }}
    project_id: my-project
    location: europe-west1
    cluster_name: staging-cluster
```

### With GAR Docker Login
```yaml
- name: GCP with container registry
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.WIF_SA }}
    project_id: my-project
    location: us-central1
    cluster_name: prod
    skip_gar_login: false  # Enable GAR login

- name: Push to GAR
  run: |
    docker build -t us-docker.pkg.dev/my-project/images/app:latest .
    docker push us-docker.pkg.dev/my-project/images/app:latest
```

### With Custom GAR Location
```yaml
- name: GCP with different GAR location
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.WIF_SA }}
    project_id: my-project
    location: us-central1           # GKE cluster in us-central1
    cluster_name: prod
    gar_location: europe-west1       # GAR in europe-west1
    skip_gar_login: false

- name: Push to European GAR
  run: |
    docker build -t europe-docker.pkg.dev/my-project/images/app:latest .
    docker push europe-docker.pkg.dev/my-project/images/app:latest
```

### GCP Only (No GKE)
```yaml
- name: GCP auth only
  uses: skyhook-io/login-gcp-gke@v1
  with:
    credentials_json: ${{ secrets.GCP_CREDENTIALS }}
    project_id: my-project
    location: us-central1
    # No cluster_name - just GCP auth
```

### GAR Only (No GKE)
```yaml
- name: GCP with GAR only
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.WIF_SA }}
    project_id: my-project
    location: us-central1  # Required even without GKE
    gar_location: europe  # Multi-regional GAR
    skip_gar_login: false
    # No cluster_name - just GAR login

- name: Push to European multi-regional GAR
  run: |
    docker build -t europe-docker.pkg.dev/my-project/images/app:latest .
    docker push europe-docker.pkg.dev/my-project/images/app:latest
```

### Multiple Clusters (Matrix)
```yaml
jobs:
  deploy:
    strategy:
      matrix:
        cluster:
          - name: prod-us
            location: us-central1
          - name: prod-eu
            location: europe-west1
          - name: prod-asia
            location: asia-northeast1
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Login to GCP and GKE
        uses: skyhook-io/login-gcp-gke@v1
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.WIF_SA }}
          project_id: my-project
          location: ${{ matrix.cluster.location }}
          cluster_name: ${{ matrix.cluster.name }}

      - name: Deploy to cluster
        run: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/my-app
```

## Prerequisites

### For Workload Identity
1. Create Workload Identity Pool
2. Configure provider for GitHub
3. Create service account with necessary permissions
4. Add `id-token: write` permission to workflow

### For Service Account Key
1. Create service account
2. Generate JSON key
3. Store as GitHub secret

## Permissions Required

### GCP Permissions
- `iam.serviceAccounts.getAccessToken` (for WIF)
- `container.clusters.get` (for GKE)
- `container.clusters.list` (for GKE)

### GitHub Permissions
```yaml
permissions:
  id-token: write  # For WIF
  contents: read
```

## Google Artifact Registry (GAR) Endpoints

### Multi-Regional vs Regional Repositories

| Location Type | Example Values | Registry URL | Use Case |
|--------------|----------------|--------------|----------|
| Multi-regional | `us`, `europe`, `asia` | `{location}-docker.pkg.dev` | Lower latency across regions |
| Regional | `us-central1`, `europe-west1` | `{location}-docker.pkg.dev` | Single region compliance |
| Zonal* | `us-central1-a` | `{zone%-?}-docker.pkg.dev` | Converted to regional |

*Zones are automatically converted to their parent region for GAR (e.g., `us-central1-a` → `us-central1-docker.pkg.dev`)

### How the Action Determines Registry Host

1. If `gar_location` is provided, use that
2. Otherwise, use the `location` parameter
3. Always format as `{location}-docker.pkg.dev` (works for both multi-regional and regional)

## GKE Cluster Location Support

### Regional vs Zonal Clusters

- **Regional clusters** (e.g., `location: us-central1`): High availability across multiple zones
- **Zonal clusters** (e.g., `location: us-central1-a`): Single zone, lower cost

The action automatically detects the location type and uses the appropriate `gcloud` commands.

## Using Outputs

### Example: Using the outputs in subsequent steps

```yaml
- name: Login to GCP and GKE
  id: gcp
  uses: skyhook-io/login-gcp-gke@v1
  with:
    workload_identity_provider: ${{ vars.WIF_PROVIDER }}
    service_account: ${{ vars.WIF_SA }}
    project_id: my-project
    location: us-central1
    cluster_name: production

- name: Use outputs
  run: |
    echo "Project: ${{ steps.gcp.outputs.project_id }}"
    echo "Context: ${{ steps.gcp.outputs.kubectl_context }}"
    echo "Registry: ${{ steps.gcp.outputs.gar_registry }}"

    # Push to the registry
    docker build -t ${{ steps.gcp.outputs.gar_registry }}/my-app:latest .
    docker push ${{ steps.gcp.outputs.gar_registry }}/my-app:latest

    # Deploy to the cluster
    kubectl --context=${{ steps.gcp.outputs.kubectl_context }} apply -f k8s/
```

## Troubleshooting

### Workload Identity Federation (WIF) Errors

#### "Unable to acquire impersonation credentials"

Check your Workload Identity Pool configuration:

```bash
gcloud iam workload-identity-pools providers describe github \
  --workload-identity-pool=github \
  --location=global
```

Verify the attribute mappings include:
- `google.subject` → `assertion.sub`
- `attribute.repository` → `assertion.repository`

#### "Permission denied on resource"

Ensure your service account binding is correct:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  YOUR-SA@PROJECT.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/attribute.repository/ORG/REPO"
```

#### "Resource not found"

Common causes:
- Incorrect `workload_identity_provider` format
- Pool or provider doesn't exist
- Wrong project number in the provider string

Correct format:
```
projects/{PROJECT_NUMBER}/locations/global/workloadIdentityPools/{POOL}/providers/{PROVIDER}
```

## Notes

- Workload Identity is more secure than service account keys
- GAR login configures Docker for the appropriate registry endpoint
- Works with both regional and zonal GKE clusters
- Compatible with Autopilot and Standard GKE clusters
- GCP-only mode (no cluster) still configures GAR if requested