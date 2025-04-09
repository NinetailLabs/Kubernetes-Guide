This will guide you through setting up a MS SQL Server in your Kubernetes cluster.

Start by setting up a namespace for your server
```powershell
kubectl create namespace sql-server
```
## PVC
First we will set up a PVC for our server to store it's data.
```yaml
#sql-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sql-server-data
  namespace: sql-server
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

Apply to your cluster
```powershell
 kubectl apply -f '.\sql-pvc.yml'
```

You can verify your PVC was created by running 
```powershell
kubectl get pvc -n sql-server
```

## Helm Chart
Start by adding the SQL Server chart
```powershell
helm repo add simcube https://simcubeltd.github.io/simcube-helm-charts/
```

Then update the repository
```powershell
helm repo update
```

Next search for the appropriate chart
```powershell
helm search repo mssql
```

Next create a value file to configure the chart
```yaml
# sql-values.yml
acceptEula:
  value: "Y"
image:
  repository: mcr.microsoft.com/mssql/server
  tag: 2022-latest
  pullPolicy: IfNotPresent
edition:
  value: Developer
sapassword: <sa-password>
persistence:
  enabled: true
  existingDataClaim: sql-server-data  # Reference to the pre-created PVC
service:
  type: ClusterIP
  port: 1433
resources:
  limits:
    cpu: 4
    memory: 8Gi
  requests:
    cpu: 2
    memory: 4Gi
securityContext:
  runAsUser: 10001
  runAsGroup: 10001
```

And install the chart
```powershell
helm install sqlserver simcube/mssqlserver-2022 --namespace sql-server --values .\sql-values.yml
```

## Database Access
To access the database using a tool such as Azure Data Studio or SSMS you need to forward the appropriate port

Start by getting the pod name
```powershell
get pods -n sql-server
```

And then forward the port

```powershell
kubectl port-forward -n sql-server <pod-name> 4444:1433
```

Note that I am forwarding to port 4444 locally as my local SQL Server instance is taking port 1433.

To connect to the server, set the server as `127.0.0.1,4444`, use `sa` for the username and the password you configured in the `sql-values.yml` file.

You can access your server using the following connection string
```
Server=sqlserver-mssqlserver-2022.sql-server.svc.cluster.local;Database=<database_name>;User Id=sa;Password=<your-password>;TrustServerCertificate=True;Encrypt=False;
```

Note that it is best practice to set up separate users for each database.

## Set up SQL User
TODO