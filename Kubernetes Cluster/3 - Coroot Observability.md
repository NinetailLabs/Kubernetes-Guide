Coroot provides a community edition observability dashboard that can be set up in your cluster to monitor your cluster with minimal effort. This guide will go through the steps to easily get coroot set up in your cluster.

## Installation
To set up Coroot simply follow the steps provided [here](https://docs.coroot.com/). The steps are also reproduced here for convenience.

Add the Coroot helm chart repo:

```
helm repo add coroot https://coroot.github.io/helm-chartshelm repo update coroot
```

Next, install the Coroot Operator:

```
helm install -n coroot --create-namespace coroot-operator coroot/coroot-operator
```

Install the Coroot Community Edition. This chart creates a minimal [Coroot Custom Resource](https://docs.coroot.com/installation/k8s-operator):

```
helm install -n coroot coroot coroot/coroot-ce --set "clickhouse.shards=2,clickhouse.replicas=2"
```

The Coroot pods can take a while to fully start, you can monitor their progress by running
```powershell
kubectl get pods -n coroot
```

Once all the pods are running forward the Coroot port to your machine:

```
kubectl port-forward -n coroot service/coroot-coroot 8080:8080
```

Then, you can access Coroot at [http://localhost:8080](http://localhost:8080)

## Internet Facing Dashboard
To expose the dashboard to the internet we will have to create an ingress
```yaml
# o11y-ingressroute.yml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: coroot
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: coroot-http
  namespace: coroot
  annotations:
    traefik.ingress.kubernetes.io/service.passHostHeader: "true"
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    middlewares:
    - name: https-redirect
    services:
    - name: coroot-coroot
      port: 8080
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: coroot-https
  namespace: coroot
  annotations:
    traefik.ingress.kubernetes.io/service.passHostHeader: "true"
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    services:
    - name: coroot-coroot
      port: 8080
      # Enable WebSocket protocol
      responseForwarding:
        flushInterval: "1ms"
      sticky:
        cookie: {}
  tls: {}
```

Apply the ingressroute to your cluser
```powershell
kubectl apply -f '.\o11y-ingressroute.yml'
```
