# Delivery app (Cloud Computing, UL FRI)

A small food delivery application split into isolated microservices. Each service
lives in its own repository, uses its own database, and talks to the others only
over HTTP through adapters.

| Repository | Role |
| --- | --- |
| [delivery-app-restaurants](https://github.com/Cloud-Computing-FRI/delivery-app-restaurants) | Restaurants and their menus |
| [delivery-app-orders](https://github.com/Cloud-Computing-FRI/delivery-app-orders) | Placing and tracking orders |
| [delivery-app-notifications](https://github.com/Cloud-Computing-FRI/delivery-app-notifications) | Order notifications |
| [delivery-app-ui](https://github.com/Cloud-Computing-FRI/delivery-app-ui) | Web frontend and `/api` proxy |

Every repository includes its own Helm chart under `helm/` and the same CI/CD
workflow under `.github/workflows/ci-cd.yml`. This page documents the parts that
are shared across all four, so the individual repositories do not repeat them.

## How the pipeline promotes a change

- A pull request runs the tests and lints the chart.
- A push to `main` builds the image once, tags it with the commit SHA, pushes it,
  and deploys it to the `staging` namespace.
- A manual run of the workflow deploys to the `prod` namespace. The `prod`
  environment waits for an approval before it proceeds (see below).
- Staging and prod deploy the same SHA tag, so a prod release ships the exact
  build that was tested on staging, and a rollback is a redeploy of an older SHA.

All four services deploy into one namespace per environment (`staging`, `prod`),
which keeps their Service names stable so they resolve each other.

### Turning on the prod approval gate

In each repository open **Settings > Environments > prod** and add yourself or
the team under **Required reviewers**. A prod deploy then pauses until someone
approves it.

## Required GitHub secrets

Set these in every service repository under
**Settings > Secrets and variables > Actions**.

| Secret | Purpose |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub username (also the image namespace) |
| `DOCKERHUB_TOKEN` | Docker Hub access token with push rights |
| `AZURE_CLIENT_ID` | Client ID of the user-assigned managed identity (OIDC) |
| `AZURE_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `AKS_RESOURCE_GROUP` | Resource group the AKS cluster belongs to |
| `AKS_CLUSTER_NAME` | AKS cluster name |

### Retrieving the values with the Azure CLI

Sign in and select the subscription you deploy into:

```bash
az login
az account set --subscription "<subscription name or id>"
```

Read the tenant and subscription IDs:

```bash
az account show --query tenantId -o tsv   # AZURE_TENANT_ID
az account show --query id -o tsv         # AZURE_SUBSCRIPTION_ID
```

Find the AKS cluster and its resource group, and keep them in variables that the
commands below reuse:

```bash
rg=$(az aks list --query "[0].resourceGroup" -o tsv)   # AKS_RESOURCE_GROUP
cluster=$(az aks list --query "[0].name" -o tsv)       # AKS_CLUSTER_NAME
echo "$rg / $cluster"
```

If the subscription contains more than one cluster, replace `[0]` with the index of
yours (`az aks list -o table` shows them all). The variables only live in the
current shell, so run the later blocks in the same session.

Create the managed identity the workflow logs in as, then read its client ID:

```bash
az identity create --name delivery-ci --resource-group "$rg"
az identity show  --name delivery-ci --resource-group "$rg" --query clientId -o tsv   # AZURE_CLIENT_ID
```

Because login uses OIDC, no client secret is stored. Add one federated
credential per repository and environment so GitHub can exchange a token:

```bash
for repo in delivery-app-restaurants delivery-app-orders delivery-app-notifications delivery-app-ui; do
  for env in staging prod; do
    az identity federated-credential create \
      --name "${repo}-${env}" \
      --identity-name delivery-ci \
      --resource-group "$rg" \
      --issuer https://token.actions.githubusercontent.com \
      --subject "repo:Cloud-Computing-FRI/${repo}:environment:${env}" \
      --audience api://AzureADTokenExchange
  done
done
```

Give that identity access to pull the kubeconfig and deploy into the cluster:

```bash
clientId=$(az identity show --name delivery-ci --resource-group "$rg" --query clientId -o tsv)
clusterId=$(az aks show --name "$cluster" --resource-group "$rg" --query id -o tsv)
az role assignment create --assignee "$clientId" \
  --role "Azure Kubernetes Service Cluster User Role" --scope "$clusterId"
# For an AKS cluster with Azure RBAC, also grant a write role such as
# "Azure Kubernetes Service RBAC Writer" scoped to the staging and prod namespaces.
```

The two Docker Hub values come from Docker Hub itself. Create the token under
**Account Settings > Personal access tokens** at https://hub.docker.com.

### Loading the secrets with the GitHub CLI

With the values in hand, push them to all four repositories at once:

```bash
clientId=$(az identity show --name delivery-ci --resource-group "$rg" --query clientId -o tsv)
tenantId=$(az account show --query tenantId -o tsv)
subId=$(az account show --query id -o tsv)

for repo in delivery-app-restaurants delivery-app-orders delivery-app-notifications delivery-app-ui; do
  gh secret set AZURE_CLIENT_ID       --repo "Cloud-Computing-FRI/$repo" --body "$clientId"
  gh secret set AZURE_TENANT_ID       --repo "Cloud-Computing-FRI/$repo" --body "$tenantId"
  gh secret set AZURE_SUBSCRIPTION_ID --repo "Cloud-Computing-FRI/$repo" --body "$subId"
  gh secret set AKS_RESOURCE_GROUP    --repo "Cloud-Computing-FRI/$repo" --body "$rg"
  gh secret set AKS_CLUSTER_NAME      --repo "Cloud-Computing-FRI/$repo" --body "$cluster"
  gh secret set DOCKERHUB_USERNAME    --repo "Cloud-Computing-FRI/$repo" --body "<dockerhub-username>"
  gh secret set DOCKERHUB_TOKEN       --repo "Cloud-Computing-FRI/$repo" --body "<dockerhub-token>"
done
```

## Managing the cluster with kubectl

Point `kubectl` at the AKS cluster. This merges the credentials into your
`~/.kube/config`:

```bash
rg=$(az aks list --query "[0].resourceGroup" -o tsv)
cluster=$(az aks list --query "[0].name" -o tsv)
az aks get-credentials --resource-group "$rg" --name "$cluster"
```

Each environment is a namespace (`staging`, `prod`). Inspect what is running:

```bash
kubectl -n staging get deploy,svc,pods
kubectl -n staging logs -f deploy/orders
kubectl -n staging rollout status deploy/orders
kubectl -n staging describe pod <pod-name>
```

Reach the UI. On `staging` forward it to your machine, and on `prod` read its
public address:

```bash
kubectl -n staging port-forward svc/ui 8080:80   # then open http://localhost:8080
kubectl -n prod get svc ui                        # EXTERNAL-IP of the LoadBalancer
```

Restart or roll a service:

```bash
kubectl -n staging rollout restart deploy/orders
```

Helm tracks each service as its own release, so you can review history and roll
back a bad deploy:

```bash
helm -n prod list
helm -n prod history orders
helm -n prod rollback orders <revision>
```

## Running it locally

For a local kind cluster, install each chart into one namespace with its
`values-dev.yaml`, then port-forward the UI. The per-repo chart notes print the
exact commands after an install.
