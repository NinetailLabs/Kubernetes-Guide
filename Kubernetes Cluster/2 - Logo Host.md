The description provided here will guide you through the process of installing the LogoHost pod. This pod is proprietary and not available outside of NineTail Labs, however the details provided will still inform you on how to use a private container and [[Helm]] chart stored on DockerHub.

## Secret Configuration
Start by creating a new namespace for your pod
```powershell
kubectl create namespace <your-namespace>
```

Next well create a docker secret in the namespace to allow us to pull the image from our private DockerHub repository by running this command in powershell
```powershell
$DOCKER_SERVER="https://index.docker.io/v1/"
$DOCKER_USERNAME="<your-username>"
$DOCKER_PASSWORD="<your-PAT>"

kubectl create secret docker-registry dockerhub-secret `
  --docker-server=$DOCKER_SERVER `
  --docker-username=$DOCKER_USERNAME `
  --docker-password=$DOCKER_PASSWORD `
  --namespace=your-namespace
```

Next we'll configure the value file for the [[Helm]] chart
```yaml
# values.yml
config:
  logoName: logo-name.extension
  pageTitle: page-name

ingress:
  hostname: your-domain
```

After this you have to log into your DockerHub for [[Helm]]
```powershell
helm registry login -u <username> registry-1.docker.io
```

And finally deploy your chart
```powershell
helm install chart-name oci://registry-1.docker.io/dockerhub-username/chart-name:2025.04.09-e26b16a --namespace your-namespace -f '.\values.yml'
```