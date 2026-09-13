1. Add the repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update


2. Install the charts
helm install argocd argo/argo-cd `
  --namespace argocd `
  --create-namespace `
  --wait


3. Update the chart using the values inside argocd-values.yaml folder
helm upgrade argocd argo/argo-cd `
  --namespace argocd `
  --reuse-values `
  --values argocd-values.yaml `
  --wait


4. Apply the virtual service where it tell the request from /argocd should go the the argocd service


5. Get argocd creds (windows)
$encodedPassword = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encodedPassword))
user: admin
password: uTt2LzWyl29OXpVl
url: http://localhost:8080/argocd

6. 