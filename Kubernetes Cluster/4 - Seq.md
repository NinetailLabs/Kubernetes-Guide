[Seq](https://datalust.co/seq) is a structured log server. It is self hosted and can be used for semantic logging and dashboarding.

Start by creating a namespace to store our Seq resources
```powershell
kubectl create namespace seq-logging
```

## PVC
A PVC is required for SEQ to store its data.
To create the PVC create the following yaml file
```yaml
# seq-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: seq-data-pvc
  namespace: seq-logging
  labels:
    app: seq
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: csi-cinder-classic
```

Apply using 
```powershell
kubectl apply -f '.\seq-pvc.yml'
```

### Helm Deployment
We'll deploy Seq using a helm chart.
Start by adding the repository to [[Helm]]
```powershell
helm repo add datalust https://helm.datalust.co
helm repo update
```

Next, create the values file to configure the chart
```yaml
# Seq values configuration
seq:
  admin:
    username: admin
    password: <your-password>
    existingSecret: ""

persistence:
  enabled: true
  existingClaim: seq-data-pvc
  size: 5Gi

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1
    memory: 1Gi

service:
  type: ClusterIP
  port: 80

# Disable standard Kubernetes ingress since we're using Traefik IngressRoute
ingress:
  ui:
    enabled: false
  ingestion:
    enabled: false

# Configure logging level
env:
  - name: ACCEPT_EULA
    value: "Y"
  - name: SEQ_CACHE_SYSTEMRAMTARGET
    value: "0.2"

```

Install Seq by executing
```powershell
helm install my-seq datalust/seq --namespace seq-logging
```

Port forward to your Seq cluster and set up an admin user to secure the login
```powershell
kubectl port-forward service/my-seq -n seq-logging 5555:80
```

Note that we are forwarding to port 5555 locally as port 80 is reserved and might be in use.
Access the Seq dashboard by going to 127.0.0.1:5555

## Expose Dashboard
Once our dashboard is secured we can now set up an ingress route to allow access to it from the internet (if required)

Create the ingress route definition
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: seq-logging
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: seq-http
  namespace: seq-logging
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    middlewares:
    - name: https-redirect
    services:
    - name: my-seq
      port: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: seq-https
  namespace: seq-logging
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    services:
    - name: my-seq
      port: 80
  tls: {}
```

And execute
```powershell
 kubectl apply -f '.\seq-ingressroute.yaml'
```

