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
Commands are for PowerShell.

## How the pipeline promotes a change

- A pull request runs the tests and lints the chart.
- A push to `main` builds the image once, tags it with the commit SHA, pushes it,
  and deploys it to the `staging` namespace.
- A manual run of the workflow deploys to the `prod` namespace. The `prod`
  environment waits for an approval before it proceeds (see below).
- Staging and prod deploy the same SHA tag, so a prod release ships the exact
  build that was tested on staging, and a rollback is a manual run on an older SHA.
  Prod never builds an image, so run it from `main` after a push whose `build`
  job succeeded.

All four services deploy into one namespace per environment (`staging`, `prod`),
which keeps their Service names stable so they resolve each other.

### Turning on the prod approval gate

In each repository open **Settings > Environments > prod** and add yourself or
the team under **Required reviewers**. A prod deploy then pauses until someone
approves it.

## Deploying to prod

Each repository's prod deploy is a separate manual run, and the repositories do
not trigger each other. Deploy the three backends first. The UI's nginx resolves
the `restaurants`, `orders` and `notifications` Services when it starts, so it
crashes if they are not in the namespace yet.

```powershell
foreach ($r in "restaurants", "orders", "notifications") {
  gh workflow run ci-cd.yml --repo Cloud-Computing-FRI/delivery-app-$r --ref main
}
# once those three have finished (approve each in the Actions tab if reviewers are set):
gh workflow run ci-cd.yml --repo Cloud-Computing-FRI/delivery-app-ui --ref main
```

Follow a run with `gh run list --repo Cloud-Computing-FRI/delivery-app-orders -L 3`.
Afterwards check the namespace and read the public address:

```powershell
kubectl -n prod get pods
kubectl -n prod get svc ui    # EXTERNAL-IP of the LoadBalancer
```

Only redeploy the services that changed. Staging is the same, except it deploys
automatically on every push to `main`.

## Required GitHub secrets

Set these once as organization secrets under the organization's
**Settings > Secrets and variables > Actions**, with **Repository access** set to
the four service repositories. The workflows read them as `secrets.<NAME>`, so
the repositories need no secrets of their own.

On the GitHub Free plan, organization secrets only reach **public** repositories.
In a private repository they read as empty, and `azure/login` fails with
`Not all values are present. Ensure 'client-id' and 'tenant-id' are supplied.`
Keep the four repositories public, or set the secrets per repository.

| Secret | Purpose |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub username (also the image namespace) |
| `DOCKERHUB_TOKEN` | Docker Hub access token with push rights |
| `AZURE_CLIENT_ID` | Client ID of the user-assigned managed identity (OIDC) |
| `AZURE_TENANT_ID` | Azure tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |
| `AKS_RESOURCE_GROUP` | Resource group the AKS cluster belongs to |
| `AKS_CLUSTER_NAME` | AKS cluster name |

The two Docker Hub values come from Docker Hub itself. Create the token under
**Account Settings > Personal access tokens** at https://hub.docker.com. The
Docker Hub repositories must be public, because the cluster pulls images without
credentials.

### Retrieving the values with the Azure CLI

Sign in, then read the tenant and subscription IDs:

```powershell
az login
az account show --query tenantId -o tsv   # AZURE_TENANT_ID
az account show --query id -o tsv          # AZURE_SUBSCRIPTION_ID
```

Find the AKS cluster and its resource group, and keep them in variables that the
commands below reuse:

```powershell
$rg = az aks list --query "[0].resourceGroup" -o tsv   # AKS_RESOURCE_GROUP
$cluster = az aks list --query "[0].name" -o tsv       # AKS_CLUSTER_NAME
"$rg / $cluster"
```

If the subscription contains more than one cluster, replace `[0]` with the index of
yours (`az aks list -o table` shows them all). The variables only live in the
current session, so run the later blocks in the same one.

Create the managed identity the workflow logs in as, then read its client ID:

```powershell
az identity create --name delivery-ci --resource-group $rg
az identity show --name delivery-ci --resource-group $rg --query clientId -o tsv   # AZURE_CLIENT_ID
```

### Federated credentials

Login uses OIDC, so no client secret is stored. Add one federated credential per
repository and environment so GitHub can exchange a token.

Manage these on the managed identity with `az identity`, not in Entra ID. A
university account often gets a 401 "You do not have access" on the Entra ID
pages, and the managed identity does not need them.

The credential subject must use GitHub's ID-based format, which is what the
tokens for these repositories carry:

```
repo:<org>@<orgId>/<repo>@<repoId>:environment:<staging|prod>
```

A plain `repo:Cloud-Computing-FRI/<repo>:environment:<env>` subject fails the
login with `AADSTS700213: No matching federated identity record found`. The
`subject claim` line in the `azure/login` log shows the exact value GitHub sends.
Look up the organization and repository IDs with `gh`:

```powershell
$orgId = gh api orgs/Cloud-Computing-FRI --jq .id
$repos = "delivery-app-restaurants", "delivery-app-orders", "delivery-app-notifications", "delivery-app-ui"
foreach ($repo in $repos) {
  $repoId = gh api "repos/Cloud-Computing-FRI/$repo" --jq .id
  if (-not $repoId) { Write-Warning "No repo id for $repo, skipping"; continue }
  foreach ($env in "staging", "prod") {
    az identity federated-credential delete --identity-name delivery-ci `
      --resource-group $rg --name "github-$repo-$env-id" --yes 2>$null
    az identity federated-credential create `
      --name "github-$repo-$env-id" `
      --identity-name delivery-ci `
      --resource-group $rg `
      --issuer https://token.actions.githubusercontent.com `
      --subject "repo:Cloud-Computing-FRI@${orgId}/${repo}@${repoId}:environment:${env}" `
      --audiences api://AzureADTokenExchange
  }
}
```

The braces around the variables in the subject are required in PowerShell;
otherwise `$repo:environment` parses as a scope-qualified variable. Check the
result: there should be 8 credentials and no subject containing `@:` (an empty
ID). Azure can take a minute to apply new credentials.

```powershell
az identity federated-credential list --identity-name delivery-ci `
  --resource-group $rg --query "[].{name:name, subject:subject}" -o table
```

### Cluster access

Give the identity access to pull the kubeconfig and deploy into the cluster:

```powershell
$clientId = az identity show --name delivery-ci --resource-group $rg --query clientId -o tsv
$clusterId = az aks show --name $cluster --resource-group $rg --query id -o tsv
az role assignment create --assignee $clientId `
  --role "Azure Kubernetes Service Cluster User Role" --scope $clusterId
```

### Loading the secrets with the GitHub CLI

Creating organization secrets needs the `admin:org` scope on your GitHub token:

```powershell
gh auth refresh -s admin:org
```

With the values in hand, set each secret once for the whole organization and
limit it to the four service repositories:

```powershell
$clientId = az identity show --name delivery-ci --resource-group $rg --query clientId -o tsv
$tenantId = az account show --query tenantId -o tsv
$subId = az account show --query id -o tsv
$repos = "delivery-app-restaurants,delivery-app-orders,delivery-app-notifications,delivery-app-ui"

gh secret set AZURE_CLIENT_ID       --org Cloud-Computing-FRI --visibility selected --repos $repos --body $clientId
gh secret set AZURE_TENANT_ID       --org Cloud-Computing-FRI --visibility selected --repos $repos --body $tenantId
gh secret set AZURE_SUBSCRIPTION_ID --org Cloud-Computing-FRI --visibility selected --repos $repos --body $subId
gh secret set AKS_RESOURCE_GROUP    --org Cloud-Computing-FRI --visibility selected --repos $repos --body $rg
gh secret set AKS_CLUSTER_NAME      --org Cloud-Computing-FRI --visibility selected --repos $repos --body $cluster
gh secret set DOCKERHUB_USERNAME    --org Cloud-Computing-FRI --visibility selected --repos $repos --body "<dockerhub-username>"
gh secret set DOCKERHUB_TOKEN       --org Cloud-Computing-FRI --visibility selected --repos $repos --body "<dockerhub-token>"
```

## Managing the cluster with kubectl

Point `kubectl` at the AKS cluster. This merges the credentials into your
`~/.kube/config`:

```powershell
$rg = az aks list --query "[0].resourceGroup" -o tsv
$cluster = az aks list --query "[0].name" -o tsv
az aks get-credentials --resource-group $rg --name $cluster
```

Each environment is a namespace (`staging`, `prod`). Inspect what is running:

```powershell
kubectl -n staging get deploy,svc,pods
kubectl -n staging logs -f deploy/orders
kubectl -n staging rollout status deploy/orders
kubectl -n staging describe pod <pod-name>
```

The UI is a `LoadBalancer` Service in both environments. Read its public address:

```powershell
kubectl -n staging get svc ui   # EXTERNAL-IP, then open http://<EXTERNAL-IP>
kubectl -n prod get svc ui
```

Restart a service:

```powershell
kubectl -n staging rollout restart deploy/orders
```

Helm tracks each service as its own release, so you can review history and roll
back a bad deploy:

```powershell
helm -n prod list
helm -n prod history <release>
helm -n prod rollback <release> <revision>
```

## Troubleshooting

A deploy step that hangs and then fails with `resource Deployment/<ns>/<name> not
ready` only says the pods never became ready. Find out why with
`kubectl -n <ns> get pods`:

| Pod status | Cause and fix |
| --- | --- |
| `ImagePullBackOff` | The image tag does not exist. Prod deploys the SHA of the commit it runs on, and only a push to `main` builds that image. Check that `https://hub.docker.com/r/<username>/<repo>/tags` has the SHA, push a commit to `main` if it does not, then run prod again from `main`. |
| `CrashLoopBackOff` on the UI | `kubectl -n <ns> logs deploy/ui --previous` shows `host not found in upstream`. A backend Service (`restaurants`, `orders` or `notifications`) is missing from that namespace. Deploy the backends first, then re-run the UI job. |

After fixing the cause, use **Re-run failed jobs** on the workflow run. Helm
upgrades a `failed` release in place, so no cleanup is needed.

A `LoadBalancer` Service stuck on `EXTERNAL-IP <pending>` shows the reason in
`kubectl -n <ns> describe svc ui`. If it is `PublicIPCountLimitReached`, the
subscription is at its limit of 3 public IPs in the region. The cluster's own
outbound IP counts as one, so only two LoadBalancer Services fit (the staging and
prod UIs). Free an IP by deleting an unused LoadBalancer Service or namespace; the
pending Service picks one up on its own after a few minutes.

## Running it locally

For a local kind cluster, install each chart into one namespace with its
`values-dev.yaml`, then port-forward the UI. The per-repo chart notes print the
exact commands after an install.
