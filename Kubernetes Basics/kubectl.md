[kubectl](https://kubernetes.io/docs/reference/kubectl/) is a command line tool that is used to manage a Kubernetes cluster.
It supports all major operating systems, the installation instructions can be found [here](https://kubernetes.io/docs/tasks/tools/#kubectl).

### Installation
Binaries for kubectl can be downloaded from the [Kubernetes Release Page](https://kubernetes.io/releases/download/#binaries).

On Windows it can be easily installed using [[winget]]:

```console

winget install -e --id Kubernetes.kubectl

```

### Configuration
See [[Kubeconfig]] for configuring access to a k8s cluster.

### Basic Commands
A collection of basic commands that can be used to manage a k8s cluster. By default commands are executed against the `default` namespace.

| Command                                                          | Description                                                                                                                                                                                                                           |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kubectl get all`                                                | Get information about all pods, services, deployments, daemonset etc                                                                                                                                                                  |
| `kubectl get <type>`                                             | Get information about a specific type. Options include:<br>- pods<br>- services<br>- deployments<br>- daemonset<br>- secrets                                                                                                          |
| `kubectl logs <pod>`                                             | Get the logs for the specified pod                                                                                                                                                                                                    |
| `kubectl apply -f <file>`                                        | Apply a yaml file to the cluster.<br>This can be used to, for example, create a pod                                                                                                                                                   |
| `kubectl delete <type> <name>`                                   | Delete an item, for example a secret                                                                                                                                                                                                  |
| `kubectl create namespace <namespace>`                           | Create a namespace                                                                                                                                                                                                                    |
| `kubectl port-forward <pod/service> <target port>:<origin port>` | Securely forward a port from inside your k8s cluster to your local machine. This is useful to, for example, access the traefik dashboard without exposing it to the internet. Ports can be forwarded for a service or a specific pod. |

### Switches
- -n <namespace> - execute the command against the specified namespace
