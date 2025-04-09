The first step once your cluster becomes available is to set up an ingress to allow internet traffic into your cluster, for this I will be using [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/). This guide will walk you through the basic steps needed to set up Traefik as your ingress with a wildcard [Let's Encrypt](https://letsencrypt.org/) certificate acquired using the DNS challenge method from [Cloudflare](https://www.cloudflare.com/). We will also enable the Traefik dashboard and make it accessible via [[kubectl]] port-forwarding with basic auth. The dashboard will not be exposed to the internet. 

## Cloudflare Token
The first step is to configure a Cloudflare [token](https://dash.cloudflare.com/profile/api-tokens). The token should be set up as follows:
- Permissions
	- Zone:Zone - Read
	- Zone:DNS - Edit
- Zone Resources
	- Include:Specific Zone:ZoneToUse (This can also be set to all zones if wanted/required)

For increased security you can limit the IP address allowed to use the token as well as the TTL for the token.

Once you have your token run this command to set up a secret in k8s.
Start by setting up the Traefik namespace:

```powershell
kubectl create namespace traefik
```

Then add your token secret to the namespace:

```powershell
kubectl create secret generic cloudflare-api-token --from-literal=api-token='<your-token>' -n traefik
```

This will create a generic secret called `cloudflare-api-token` in the `traefik` namespace that Traefik can use to run commands against Cloudflare.

You can verify the secret by running:
```powershell
kubectl get secrets -n traefik
```

## Persistent Volume Claim (PVC)
Next we will set up a PVC that Traefik can use to store some information. We will set up a `classic` PVC for this purpose as it is the cheapest and we don't need a lot of throughput.

```yaml
# traefik-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: traefik-acme
  namespace: traefik
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 128Mi
  storageClassName: csi-cinder-classic
```

Save the file and run
```powershell
kubectl apply -f '.\traefik-pvc.yml'
```

After this you should see the volume in the OVH control panel or in Powershell
```powershell
kubectl get pvc -n traefik
```

## Dashboard Auth
We'll set up basic auth for our dashboard even though we won't be exposing it to the internet. To do this we'll create an auth secret using a job pod in our cluster (if you are using Linux, you can just run the command directly)

```yaml
#auth-generator-job.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: auth-generator
  namespace: traefik
spec:
  template:
    spec:
      containers:
      - name: htpasswd-generator
        image: httpd:2.4-alpine
        command:
        - /bin/sh
        - -c
        - "htpasswd -cbB /tmp/auth admin <your password> && cat /tmp/auth && echo '---' && echo 'Copy the output above for your secret' && sleep 60"
      restartPolicy: Never
  backoffLimit: 1
```

Execute the job in the cluster
```powershell
kubectl apply -f '.\auth-generator-job.yml'
```

Find the name of the pod
```powershell
kubectl get pods -n traefik
```

Read the logs for the pod to get your secret
```powershell
kubectl logs -n traefik <pod-name>
```

The secret will be in the form `admin:[hash]`, copy this secret into a file called `auth.txt` then run
```powershell
kubectl create secret generic dashboard-auth-secret --from-file=users=auth.txt -n traefik
```

Finally erase the job pod by running
```powershell
kubectl delete pod -n <pod-name>
```

## Traefik Installation
Create a `values.yml` file to configure Traefik for our [[Helm]] installation
```yaml
# traefik-values.yml
# Traefik values for Helm
# Dashboard and API configuration
api:
  dashboard: true
  insecure: false

dashboard:
  enabled: true

# Basic deployment settings
deployment:
  enabled: true
  replicas: 1
  additionalVolumes:
    - name: acme-data
      persistentVolumeClaim:
        claimName: traefik-acme
  additionalVolumeMounts:
    - name: acme-data
      mountPath: /data
  podSecurityContext:
    fsGroup: 65532
  securityContext:
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: false
    runAsNonRoot: true
    runAsUser: 65532

# Dashboard access route
ingressRoute:
  dashboard:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: traefik
    entryPoints: ["traefik"]

# Port configuration
ports:
  traefik:
    exposedPort: 9000
    port: 9000
    protocol: TCP
  web:
    exposedPort: 80
    port: 8000
    protocol: TCP
  websecure:
    exposedPort: 443
    port: 8443
    protocol: TCP

# Core configuration
additionalArguments:
  - "--entrypoints.traefik.address=:9000"
  - "--providers.kubernetesingress.ingressclass=traefik"
  - "--log.level=DEBUG"
  - "--certificatesresolvers.letsencrypt.acme.email=<your-email>"
  - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
  - "--certificatesresolvers.letsencrypt.acme.dnschallenge=true"
  - "--certificatesresolvers.letsencrypt.acme.dnschallenge.provider=cloudflare"
  - "--certificatesresolvers.letsencrypt.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53"
  - "--providers.file.directory=/data"
  - "--providers.file.watch=true"

# Disable default persistence as we're using additionalVolumes
persistence:
  enabled: false

# Service configuration
service:
  enabled: true
  type: LoadBalancer

# RBAC settings
rbac:
  enabled: true
  namespaced: false

# Provider configuration
providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
  kubernetesIngress:
    enabled: true
    allowExternalNameServices: true
  file:
    directory: "/data"
    watch: true

# Environment variables for CloudFlare API
env:
  - name: CLOUDFLARE_EMAIL
    value: "<your-email>"
  - name: CLOUDFLARE_DNS_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-token
        key: api-token

# Create middleware for authentication
middleware:
  dashboard-auth:
    basicAuth:
      secret: dashboard-auth-secret

```

This configures Traefik with out secrets for the dashboard, opens port 80 and 443 as well as setting up Let's Encrypt using Cloudflare DNS challange.

Add the helm charts for Traefik
```powershell
helm repo add traefik https://traefik.github.io/charts
```

Then update the repo
```powershell
helm repo update traefik
```

And finally install Traefik using the chart
```powershell
helm install traefik traefik/traefik --namespace traefik -f '.\traefik-pvc.yml'
```

Check the status of your external IP address by running
```powershell
kubectl get services -n traefik
```

Getting an external IP address can take a little while as OVH will be configuring an IP, gateway and load balancer for you.

Now get the Traefik pod
```powershell
kubectl get pods -n traefik
```

And port forward to the pod to get access to the Traefik dashboard
```powershell
kubectl port-forward -n traefik <pod-name> 9000:9000
```

You can view your dashboard by going to `http://127.0.0.1:9000/dashboard/` in a browser. Note the trailing `/`, otherwise the dashboard won't show.

## TLS Store
Now we'll create a TLS store for our certificates in Kubernetes. We will define it so that Traefik will request a wildcard certificate for our domain, allowing us to use one certificate for all our subdomains.

```yaml
#tlsstore.yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: traefik
spec:
  defaultGeneratedCert:
    resolver: letsencrypt
    domain:
      main: "*.<your-domain>"
      sans:
        - "<your-domain>"
```

And apply it
```powershell
kubectl apply -f '.\tlsstore.yml'
```

Traefik will try to serve a default certificate, so we'll set one up.

In an elevated PowerShell run
```powershell
$cert = New-SelfSignedCertificate -DnsName "<your-domain>", "*.<your-domain>"
```

Then get the certificate as a .pfx by executing
```powershell
$certPassword = ConvertTo-SecureString -String "password" -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath default-cert.pfx -Password $certPassword
```

Next get the .crt by running. For this command you need to have `openssl` installed
```powershell
openssl pkcs12 -in default-cert.pfx -out default-cert.crt -nokeys -password pass:password
openssl pkcs12 -in default-cert.pfx -out default-cert.key -nocerts -nodes -password pass:password
```

And the create a secret in Kubernetes
```powershell
kubectl create secret tls default-cert --cert=default-cert.crt --key=default-cert.key -n traefik
```

Finally restart your Traefik deployment to allow it to pick up the changes
```powershell
kubectl rollout restart deployment traefik -n traefik
```

## Testing
Finally we will want to test our Traefik setup.
First, configure your DNS to point to the public IP of your load balancer.
Next, we'll create a test pod to see if everything works.

Start by creating a new namespace for our test deployment
```powershell
 kubectl create namespace web-test
```

Next, create a configmap to store a basic HTML structure
```yaml
#test-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-html
  namespace: web-test
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Ingress Test Page</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 40px; line-height: 1.6; }
        h1 { color: #333; }
        .container { border: 1px solid #ddd; padding: 20px; border-radius: 5px; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>Ingress Test Page</h1>
        <p>Congratulations! If you're seeing this page, your ingress configuration is working correctly.</p>
        <p>Pod Info:</p>
        <ul>
          <li>Pod Name: <script>document.write(window.location.hostname)</script></li>
          <li>Current Time: <script>document.write(new Date().toLocaleString())</script></li>
        </ul>
      </div>
    </body>
    </html>
```

And create it in your cluster
```powershell
# test-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-html
  namespace: web-test
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Ingress Test Page</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 40px; line-height: 1.6; }
        h1 { color: #333; }
        .container { border: 1px solid #ddd; padding: 20px; border-radius: 5px; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>Ingress Test Page</h1>
        <p>Congratulations! If you're seeing this page, your ingress configuration is working correctly.</p>
        <p>Pod Info:</p>
        <ul>
          <li>Pod Name: <script>document.write(window.location.hostname)</script></li>
          <li>Current Time: <script>document.write(new Date().toLocaleString())</script></li>
        </ul>
      </div>
    </body>
    </html>
```

Then set up a basic deployment
```yaml
# test-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-test
  namespace: web-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-test
  template:
    metadata:
      labels:
        app: web-test
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: test-html
```

Deploy it
```powershell
kubectl apply -f '.\test-deployment.yml'
```

Next we'll set up a service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-test
  namespace: web-test
spec:
  selector:
    app: web-test
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

And apply
```powershell
kubectl apply -f '.\test-service.yml'
```

And finally we set up the ingress route
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: https-redirect
  namespace: web-test
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: web-test-http
  namespace: web-test
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    middlewares:
    - name: https-redirect
    services:
    - name: web-test
      port: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: web-test-https
  namespace: web-test
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`<your-domain>`)
    kind: Rule
    services:
    - name: web-test
      port: 80
  tls:
    certResolver: letsencrypt
```

And apply it
```powershell
kubectl apply -f '.\test-ingressroute.yml'
```

In your browser you should now be able to navigate to your domain and see a test page.
Your Traefik dashboard will also show your new routes, services and middleware.

If you see the page and it is secured using a Let's Encrypt certificate, then congratulations, you have successfully set up Traefik!