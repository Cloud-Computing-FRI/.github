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

## Prod database (Azure Database for PostgreSQL)

Staging runs a Postgres pod per service, seeded automatically from
`helm/files/init.sql`. Prod does not: its `values-prod.yaml` sets
`postgresql.enabled: false` and reads `DATABASE_URL` from a Secret that must
exist before the deploy (`db-restaurants`, `db-orders`, `db-notifications`).
Without it the pods stay in `CreateContainerConfigError` (`secret "db-..." not
found`). Set it up once, in this order.

### 1. Create the server (portal)

In the Azure portal open **Azure Database for PostgreSQL flexible servers** and
click **Create**:

- Resource group `rg-1`, region the same as the cluster (Austria East).
- Server name: it becomes `<name>.postgres.database.azure.com` and must be unique
  across all of Azure. A plain name like `delivery-app-pg` is usually taken
  ("server name must be available"), so add a random suffix.
- Workload type **Development** (Burstable B1ms, 32 GB), high availability off.
- Authentication **PostgreSQL authentication only**. Pick an admin username
  (`pgadmin`) and a password, and save the password: it is not shown again. Use
  letters and digits only, or URL-encode special characters (`@` -> `%40`,
  `/` -> `%2F`, `#` -> `%23`) when it goes into the connection string below.
- Networking: **Public access**. Leave "Allow public access from any Azure
  service" off, since the firewall rule below is enough.

Creation takes several minutes. The server costs about 21 USD a month while
running. Stopping it (**Stop** in the portal, or `az postgres flexible-server
stop`) halts the compute charge, but Azure restarts a stopped server after 7 days.

### 2. Create the databases (portal)

Open the server, then **Databases** > **Add**, once each for `restaurants`,
`orders` and `notifications`. The charts expect exactly those names.

### 3. Allow the cluster through the firewall (portal)

Pods reach the server from the cluster's outbound IP. Find it:

```powershell
$ipId = az aks show -g $rg -n $cluster --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs[0].id" -o tsv
az network public-ip show --ids $ipId --query ipAddress -o tsv
```

On the server open **Networking** > **Add firewall rule**, name it `aks-egress`,
enter that address as both start and end IP, and save. Without this rule requests
hang and the services log `psycopg.errors.ConnectionTimeout`, so the UI shows
"Can't reach the restaurants service". If the cluster is stopped and started the
address can change, and the rule must be updated.

### 4. Create the Secrets and restart the services

```powershell
$server = "<server-name>.postgres.database.azure.com"
$user = "pgadmin"
$pass = "<admin-password>"
foreach ($db in "restaurants", "orders", "notifications") {
  kubectl -n prod delete secret "db-$db" --ignore-not-found
  kubectl -n prod create secret generic "db-$db" `
    --from-literal=DATABASE_URL="postgresql://${user}:${pass}@${server}:5432/${db}?sslmode=require"
}
kubectl -n prod rollout restart deploy/restaurants deploy/orders deploy/notifications
```

`sslmode=require` is needed because Azure rejects unencrypted connections. Pods
read the Secret only when they start, so rerun the `rollout restart` after every
change to it.

### 5. Create the tables

The databases are empty. Run each service's `helm/files/init.sql` once against
its own database, from a temporary pod that reads the Secret created above (run
from the folder that contains the four repository checkouts):

```powershell
foreach ($s in "restaurants", "orders", "notifications") {
  kubectl -n prod create configmap "db-init-$s" --from-file=init.sql="delivery-app-$s/helm/files/init.sql"
  @"
apiVersion: v1
kind: Pod
metadata:
  name: db-init-$s
spec:
  restartPolicy: Never
  containers:
    - name: psql
      image: postgres:16-alpine
      command: ["sh", "-c", 'psql "`$DATABASE_URL" -v ON_ERROR_STOP=1 -f /init/init.sql']
      env:
        - name: DATABASE_URL
          valueFrom: {secretKeyRef: {name: db-$s, key: DATABASE_URL}}
      volumeMounts:
        - {name: init, mountPath: /init}
  volumes:
    - name: init
      configMap: {name: db-init-$s}
"@ | kubectl -n prod apply -f -
}

foreach ($s in "restaurants", "orders", "notifications") {
  kubectl -n prod wait --for=jsonpath='{.status.phase}'=Succeeded "pod/db-init-$s" --timeout=120s
  kubectl -n prod logs "db-init-$s"
  kubectl -n prod delete pod,configmap "db-init-$s"
}
```

The logs show `CREATE TABLE` for each service, plus the seed rows for
restaurants. The scripts are not idempotent: running them again fails with
`relation ... already exists`, which changes nothing. Check the result:

```powershell
curl.exe http://<prod-EXTERNAL-IP>/api/restaurants   # the three seeded restaurants
curl.exe http://<prod-EXTERNAL-IP>/api/orders        # [] until an order is placed
```

## Deploying to prod

Each repository's prod deploy is a separate manual run, and the repositories do
not trigger each other. Set up the prod database first (above), then deploy the
three backends before the UI. The UI's nginx resolves
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
| `CreateContainerConfigError` on a backend in prod | The `db-<service>` Secret does not exist. Create it (see "Prod database"). |

Pods that are `Running` but whose API calls fail (the UI says "Can't reach the
restaurants service", or `/api/...` returns 500 or hangs) are a database problem.
`kubectl -n prod logs deploy/restaurants --tail=40` ends with the cause:

| Log error | Cause and fix |
| --- | --- |
| `Name or service not known` | The host in the Secret does not exist. Use the exact server name from the portal and recreate the Secrets. |
| `ConnectionTimeout` / `connection timeout expired` | The cluster's outbound IP is not in the server firewall. Add it. |
| `password authentication failed for user` | Wrong password or username in the Secret. Reset the password on the server and recreate the Secrets. |
| `relation "..." does not exist` | The tables are missing. Run `init.sql` (step 5 of "Prod database"). |

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
