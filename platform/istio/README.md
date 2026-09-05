1. add helm repo for istio
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update


2. list the charts in this repo
helm search repo istio


3. install the helm chart
helm install istio-base istio/base `
  --namespace istio-system `
  --create-namespace `
  --set defaultRevision=default `
  --wait

istio/base primarily installs Istio’s Custom Resource Definitions (CRDs) and other foundational cluster-wide resources.

Those CRDs allow Kubernetes to understand Istio objects such as:

Gateway
VirtualService
DestinationRule
ServiceEntry
PeerAuthentication
RequestAuthentication
AuthorizationPolicy

It does not run the Istio control plane; that is the job of the istio/istiod chart.


4. Confirm the release
helm list -n istio-system


5. install istio control plane

helm install istiod istio/istiod `
  --namespace istio-system `
  --wait

istio/base installs the CRD definitions.
You create resources using those CRDs, such as VirtualService and Gateway.
istiod watches those resources and converts them into configuration for Envoy proxies.
istio/base → Defines Gateway and VirtualService types
You/Argo CD → Creates Gateway and VirtualService objects
istiod → Reads them and configures Envoy

istiod generally does not create your routing resources—it processes the resources you create.

Istiod watches Kubernetes and Istio resources and tells Envoy:

Which services exist
Where their Pods are running
Which route to use
Whether mTLS is required
Which traffic policies apply
How to split traffic between versions
Gateway/VirtualService configuration
               ↓
             Istiod
               ↓ configuration
             Envoy
               ↓ actual request
       Application service


6. Ingress gateway
Next is the ingress gateway. Because the kind cluster maps 30080 and 30443, its Helm service must use those exact NodePorts. before create gateway-values.yaml
This installs the Istio ingress gateway infrastructure using Helm.
The chart creates resources such as:

Envoy Gateway Deployment
Gateway Pod
Kubernetes Service
ServiceAccount
Autoscaler

It creates the gateway infrastructure, but it does not define which applications receive traffic. The Gateway and VirtualService resources handle that next.

helm install istio-ingress istio/gateway `
  --namespace istio-ingress `
  --create-namespace `
  --values gateway-values.yaml `
  --wait


7. Enable Istio sidecar injection, Services deployed into this namespace will automatically receive an Envoy sidecar proxy. Only newly created Pods will receive Envoy sidecars. Existing Pods must be recreated if you want them added to the mesh.

`kubectl label namespace default istio-injection=enabled`

Enabling Istio injection across every namespace has disadvantages:

Every Pod receives an Envoy container, increasing CPU and memory usage.
Pods start more slowly.
Jobs and CronJobs may not terminate cleanly if their sidecar keeps running.
Databases, Kafka and monitoring tools may not need a mesh sidecar.
Networking and troubleshooting become more complicated.
Some system components may be incompatible with intercepted traffic.

For this project, enable it only for application namespaces:

default or rag-platform → injection enabled
istio-system            → not enabled
monitoring              → initially not enabled
data                     → initially not enabled
kafka                    → initially not enabled


8. confirm the label assigned to the installed gateway Pod:

kubectl get pods -n istio-ingress --show-labels

NAME                            READY   STATUS    RESTARTS   AGE     LABELS
istio-ingress-d85894d68-hf2tg   1/1     Running   0          6m11s   app.kubernetes.io/instance=istio-ingress,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=istio-ingress,app.kubernetes.io/part-of=istio,app.kubernetes.io/version=1.30.4,app=istio-ingress,helm.sh/chart=gateway-1.30.4,istio.io/dataplane-mode=none,istio=ingress,pod-template-hash=d85894d68,service.istio.io/canonical-name=istio-ingress,service.istio.io/canonical-revision=1.30.4,sidecar.istio.io/inject=true

The correct selector is: istio: ingress


9. Create gateway.yaml:

kubectl apply -f gateway.yaml