[DockerHub](https://hub.docker.com) is a repository for storing and building docker containers. 
They also provide the ability to store helm charts using [OCI](https://www.docker.com/blog/announcing-docker-hub-oci-artifacts-support/)
## Secrets
To access private repositories from docker hub the following command can be used to create a secret in your k8s cluster

```powershell
$DOCKER_SERVER="https://index.docker.io/v1/"
$DOCKER_USERNAME="your username"
$DOCKER_PASSWORD="PAT"

kubectl create secret docker-registry dockerhub-secret `
  --docker-server=$DOCKER_SERVER `
  --docker-username=$DOCKER_USERNAME `
  --docker-password=$DOCKER_PASSWORD `
  --namespace=namespace_for_secret
```
