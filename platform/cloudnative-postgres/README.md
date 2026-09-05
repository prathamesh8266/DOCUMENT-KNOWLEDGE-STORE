CloudNativePG is not a different PostgreSQL database. It runs normal PostgreSQL but manages it through a Kubernetes operator.
| Normal PostgreSQL deployment                | CloudNativePG                                       |
| ------------------------------------------- | --------------------------------------------------- |
| You create the StatefulSet, Service and PVC | Operator creates and manages them                   |
| You manually configure replication          | Operator manages primary and replicas               |
| Manual failover                             | Automatic failover                                  |
| Manual PostgreSQL upgrades                  | Operator coordinates updates                        |
| Manual backup configuration                 | Declarative backup and recovery support             |
| You track which Pod is primary              | Operator provides read-write and read-only Services |
| Basic Pod monitoring                        | PostgreSQL-aware status and metrics                 |

The flow is:
Create a CloudNativePG Cluster resource
                 ↓
CloudNativePG operator watches it
                 ↓
Operator creates and manages normal PostgreSQL Pods

For example, if the primary PostgreSQL Pod fails:
Normal deployment → You investigate and recover it
CloudNativePG      → Operator can promote a healthy replica

Choosing CloudNativePG to demonstrate Kubernetes operator patterns, reconciliation, persistence, failover and database observability. The application still connects using the standard PostgreSQL protocol and drivers.

1. Add the repo from helm

helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update


2. Install the chart
helm install cnpg cnpg/cloudnative-pg `
  --namespace cnpg-system `
  --create-namespace `
  --wait


3. Create a Custom resources using the cloudnatice CRD's
kubectl apply -f postgres-cluster.yaml


4. Check if deployed
kubectl get po -n cnpg-system
kubectl get cluster,pods,pvc -n database