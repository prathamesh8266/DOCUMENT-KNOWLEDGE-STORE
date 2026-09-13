# CloudNativePG GitOps deployment

CloudNativePG runs standard PostgreSQL on Kubernetes and manages it through an
operator. The operator watches `Cluster` custom resources and creates the
PostgreSQL pods, services, secrets, and persistent volumes described by them.

This repository deploys CloudNativePG in two stages:

1. `cloudnativepg-operator` installs the CloudNativePG operator from its Helm
   repository.
2. `knowledge-postgres` deploys the `Cluster` resource stored in this Git
   repository.

The resulting flow is:

```text
Argo CD Application: cloudnativepg-operator
    -> CloudNativePG Helm chart
    -> operator, CRDs, RBAC, and admission webhooks

Argo CD Application: knowledge-postgres
    -> gitops/cloudnative-postgres/cluster/postgres-cluster.yaml
    -> CloudNativePG Cluster: knowledge-postgres
    -> PostgreSQL primary, replicas, services, secrets, and PVCs
```

## Repository layout

```text
gitops/
|-- cloudnative-postgres/
|   |-- applications/
|   |   |-- cloudnativepg-operator.yaml
|   |   `-- postgres-cluster-application.yaml
|   |-- cluster/
|   |   `-- postgres-cluster.yaml
|   `-- README.md
|-- projects/
|   `-- database.yaml
`-- repository/
    `-- repository.yaml
```

## What is deployed

The operator Application deploys:

- Two CloudNativePG operator replicas in `cnpg-system`.
- CloudNativePG CustomResourceDefinitions (CRDs).
- Cluster-level RBAC used by the operator.
- Mutating and validating admission webhooks.

The cluster Application deploys:

- The `database` namespace.
- A CloudNativePG cluster named `knowledge-postgres`.
- One PostgreSQL primary and two replicas.
- A database named `knowledge_platform` owned by `app_user`.
- A 2 GiB persistent volume for each PostgreSQL instance.

## Prerequisites

Before starting, confirm that the following are available:

- A running Kubernetes cluster.
- `kubectl` configured for the intended cluster.
- Argo CD installed in the `argocd` namespace.
- The repository pushed to GitHub.
- A default StorageClass named `standard`.
- The `psql` client if a connection from the local computer is required.

Check the current context and StorageClass:

```powershell
kubectl config current-context
kubectl get nodes
kubectl get storageclass
```

For the local Kind environment, the expected context is usually
`kind-rag-cluster`, and all four nodes should report `Ready`.

## Required preflight correction

The namespace must be named `database` consistently in all three places:

```yaml
# gitops/projects/database.yaml
destinations:
  - server: https://kubernetes.default.svc
    namespace: database
```

```yaml
# gitops/cloudnative-postgres/applications/postgres-cluster-application.yaml
destination:
  server: https://kubernetes.default.svc
  namespace: database
```

```yaml
# gitops/cloudnative-postgres/cluster/postgres-cluster.yaml
metadata:
  name: knowledge-postgres
  namespace: database
```

Do not use `databases` as the destination namespace. `databases` is not allowed
by the current AppProject and does not match the namespace in the Cluster
manifest.

## Before deploying

Argo CD reads the PostgreSQL manifest from the remote `master` branch. Commit
and push the GitOps files before creating the cluster Application:

```powershell
git status
git add gitops/cloudnative-postgres gitops/projects/database.yaml gitops/repository/repository.yaml
git commit -m "add CloudNativePG GitOps deployment"
git push origin master
```

Confirm that an older manually installed CloudNativePG release is not still
running:

```powershell
helm list -n cnpg-system
kubectl get pods -n cnpg-system
kubectl get clusters.postgresql.cnpg.io -A
```

There should be only one CloudNativePG installation. Two deployments with names
such as `cnpg-cloudnative-pg` and `cloudnative-pg` indicate that a manual Helm
installation and an Argo-managed installation are both present. They will
compete for the same leader-election lease.

If a manual `cnpg` release exists, first confirm that no PostgreSQL Cluster
resources depend on it. Only then remove the duplicate manual release:

```powershell
helm uninstall cnpg -n cnpg-system
```

## Step 1: Register the Git repository

Apply the Argo CD repository Secret:

```powershell
kubectl apply -f gitops/repository/repository.yaml
```

Confirm that Argo CD can see the repository configuration:

```powershell
kubectl get secret document-knowledge-store-repository -n argocd
```

The current repository is public, so the Secret only stores the repository URL.
Do not commit usernames, passwords, personal access tokens, or SSH private keys.

## Step 2: Create the database AppProject

Apply the AppProject before either Application:

```powershell
kubectl apply -f gitops/projects/database.yaml
```

Verify it:

```powershell
kubectl get appproject databases -n argocd
kubectl describe appproject databases -n argocd
```

The AppProject allows the CloudNativePG Helm repository and this Git repository.
It permits deployments to `cnpg-system` and `database`, plus the cluster-scoped
resources required by the operator, including CRDs, RBAC, and admission
webhooks.

## Step 3: Deploy the CloudNativePG operator

Apply the operator Application:

```powershell
kubectl apply -f gitops/cloudnative-postgres/applications/cloudnativepg-operator.yaml
```

Watch the Argo CD Application until it becomes `Synced` and `Healthy`:

```powershell
kubectl get application cloudnativepg-operator -n argocd -w
```

Press `Ctrl+C` after it is healthy, then verify the operator:

```powershell
kubectl get pods -n cnpg-system
kubectl get deployment -n cnpg-system
kubectl get crd clusters.postgresql.cnpg.io
kubectl get lease -n cnpg-system
```

Two `cloudnative-pg-*` pods are expected because the Application configures
`replicaCount: 2`. Only one replica holds the leader lease; the other is a
standby. This is normal.

Do not continue until the operator Application is healthy and the
`clusters.postgresql.cnpg.io` CRD exists.

## Step 4: Deploy the PostgreSQL cluster

Apply the cluster Application:

```powershell
kubectl apply -f gitops/cloudnative-postgres/applications/postgres-cluster-application.yaml
```

This Application reads the following path from the remote repository:

```text
gitops/cloudnative-postgres/cluster
```

Watch the Argo CD Application:

```powershell
kubectl get application knowledge-postgres -n argocd -w
```

Then watch the PostgreSQL cluster become ready:

```powershell
kubectl get cluster knowledge-postgres -n database -w
```

Initial image downloads and database initialization can take several minutes.
Press `Ctrl+C` after the cluster reports ready.

## Step 5: Verify PostgreSQL resources

Run these checks:

```powershell
kubectl get cluster -n database
kubectl get pods -n database -o wide
kubectl get services -n database
kubectl get pvc -n database
kubectl get secrets -n database
```

Expected results:

- The `knowledge-postgres` Cluster reports three instances and a healthy phase.
- Three PostgreSQL instance pods are running.
- Each instance has a bound PVC.
- `knowledge-postgres-rw` points to the writable primary.
- `knowledge-postgres-ro` provides read-only access to replicas.
- `knowledge-postgres-r` provides access to all instances.
- `knowledge-postgres-app` contains application connection credentials.

For more detail:

```powershell
kubectl describe cluster knowledge-postgres -n database
kubectl get events -n database --sort-by=.lastTimestamp
```

## Step 6: Read the generated credentials

CloudNativePG creates the application credentials as a Kubernetes Secret. Read
and decode them in PowerShell without writing them to a file:

```powershell
$encodedUsername = kubectl get secret knowledge-postgres-app -n database -o jsonpath="{.data.username}"
$databaseUsername = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedUsername))

$encodedPassword = kubectl get secret knowledge-postgres-app -n database -o jsonpath="{.data.password}"
$databasePassword = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedPassword))

$databaseUsername
$databasePassword
```

Do not place the decoded password in Git, a ConfigMap, application logs, or the
README.

## Step 7: Connect from the local computer

Forward the read-write PostgreSQL service to the local computer:

```powershell
kubectl port-forward -n database service/knowledge-postgres-rw 5432:5432
```

Keep that terminal running. In another PowerShell terminal, connect with
`psql`:

```powershell
$encodedPassword = kubectl get secret knowledge-postgres-app -n database -o jsonpath="{.data.password}"
$env:PGPASSWORD = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedPassword))
psql --host 127.0.0.1 --port 5432 --username app_user --dbname knowledge_platform
```

Inside Kubernetes, application services should use:

```text
Host: knowledge-postgres-rw.database.svc.cluster.local
Port: 5432
Database: knowledge_platform
Username: app_user
Password: value from the knowledge-postgres-app Secret
```

An application Deployment should consume the Secret directly rather than
hard-coding the password.

## GitOps behavior

Both Applications enable:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

This means:

- Argo CD automatically applies changes detected in the configured source.
- `selfHeal` restores resources changed manually in the cluster.
- `prune` removes managed resources deleted from the source.

The Application manifests themselves are currently bootstrap files. No parent
Application watches their directory, so changes to those files must be applied
again with `kubectl apply`. Changes to `cluster/postgres-cluster.yaml` are
automatically detected after they are committed and pushed to `master`.

## Updating the PostgreSQL cluster

To change the number of instances, storage, or resource limits:

1. Edit `gitops/cloudnative-postgres/cluster/postgres-cluster.yaml`.
2. Commit and push the change.
3. Watch Argo CD synchronize `knowledge-postgres`.
4. Watch the CloudNativePG Cluster status and events.

Example checks:

```powershell
kubectl get application knowledge-postgres -n argocd
kubectl get cluster knowledge-postgres -n database
kubectl get pods,pvc -n database
```

Do not manually edit operator-managed Deployments, Pods, Services, or PVCs.
Change the CloudNativePG `Cluster` resource and allow the operator to reconcile
the dependent resources.

## Troubleshooting

### Application destination is not permitted

If Argo CD reports that the destination is not permitted, make sure the
Application uses namespace `database`, not `databases`, and that the AppProject
contains the same destination.

### Source path does not exist

If Argo CD reports a manifest-generation or source-path error:

```powershell
git status
git log -1 --oneline
git ls-tree -r origin/master --name-only
```

Confirm that this path exists in the remote `master` branch:

```text
gitops/cloudnative-postgres/cluster/postgres-cluster.yaml
```

Local, uncommitted files cannot be read by Argo CD.

### No matches for kind Cluster

This means the CloudNativePG CRD is not installed or is not ready. Verify the
operator Application and CRD:

```powershell
kubectl get application cloudnativepg-operator -n argocd
kubectl get crd clusters.postgresql.cnpg.io
kubectl get pods -n cnpg-system
```

### Operator exists but no PostgreSQL pods appear

The operator alone does not create a database. Verify that the cluster
Application and Cluster resource exist:

```powershell
kubectl get application knowledge-postgres -n argocd
kubectl get clusters.postgresql.cnpg.io -A
```

### More than two operator pods exist

Check for a duplicate manual Helm installation:

```powershell
helm list -n cnpg-system
kubectl get deployments,pods -n cnpg-system
kubectl get lease -n cnpg-system
```

The expected Argo-managed Deployment is `cloudnative-pg` with two replicas.

### PostgreSQL pods remain Pending

Check PVCs, the StorageClass, node capacity, and recent events:

```powershell
kubectl get pvc -n database
kubectl get storageclass
kubectl describe pod -n database
kubectl get events -n database --sort-by=.lastTimestamp
```

The manifest currently requests the `standard` StorageClass. Change the
manifest if the target cluster uses a different StorageClass name.

### Admission webhook errors

Verify the operator pods, webhook service, and operator logs:

```powershell
kubectl get pods,services -n cnpg-system
kubectl get validatingwebhookconfiguration,mutatingwebhookconfiguration
kubectl logs deployment/cloudnative-pg -n cnpg-system --tail=200
```

## Important safety notes

- Keep database credentials in Kubernetes Secrets.
- Do not commit decoded passwords.
- Do not run a manual Helm installation alongside the Argo-managed operator.
- Review changes carefully because automated pruning is enabled.
- Deleting an Argo Application with its resource finalizer can delete the
  resources managed by that Application.
- Back up PostgreSQL before destructive tests or major configuration changes.
