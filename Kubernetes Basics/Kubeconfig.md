Kubeconfig files are used to manage access to one or more k8s clusters. More details about how to use and manage kubeconfig files can be found [here](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

A step by step guide to setting up multiple clusters in a kubeconfig can be found [here](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

Using PowerShell you can set an environment variable for the current terminal to use a specific kubeconfig file as follows:

```powershell
$env:KUBECONFIG="..\kubeconfig-microcluster.yml"
```

To check cluster information you can run

```console
kubectl cluster-info
```