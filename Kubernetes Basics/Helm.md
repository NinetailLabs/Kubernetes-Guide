[Helm](https://helm.sh/) is a command line tool that is used to easily install packages on a Kubernetes cluster.

### Installation
The guide to install Helm can be found [here](https://helm.sh/docs/intro/install/) with the binaries available from [GitHub](https://github.com/helm/helm/releases).

On Windows it can be easily installed by using [[winget]]:

```console
winget install Helm.Helm
```

### Install a chart
To install a chart start by adding it's repository

```powershell
helm repo add <name> <repository url>
```

Then run the installation

```powershell
helm install <name> <chart>
```

It is also possible to install a chart directly from a repository without adding the repository

```powershell
helm install <name> <repository url>
```

To install from a private [[DockerHub]] repository you will need to log in first using a personal access token (PAT) with at least read access

```powershell
helm registry login -u <username> registry-1.docker.io
```

Note that the username is not your full email address!

You will be prompted for a password, enter your PAT to finish the login. Once logged in you can pull charts from your private repositories.

#### Flags
- `--namespace <namespace>` - Install the chart into a specific namespace
- `-f <file>` - Install with overrides from a value file