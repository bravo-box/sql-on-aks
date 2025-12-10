# SQL on AKS and Arc-Enabled AKS

This is a quickstart to run SQL in AKS or Arc-enabled AKS. The focus of this will be around the edge (Azure Local) as there are many options for running SQL in Azure for example: SQL Managed Instance and Azure SQL. This code works on both Azure Kubernetes Service and Arc-Enabled Kubernetes Service.

Deployment is relatively straight-forward however there is some infrastructure setup that needs to be in place prior to deploying the SQL YAML.

## What you will cover in this quickstart

1. Connect to Arc-enabled Kubernetes on Azure Local
2. Ensure storage classes are in place
3. Create your deployment namespace
4. Deploy SQL YAML
5. Create a load balancer for access to the SQL Server

### Connectivity to the arc-enabled cluster

Connecting to Azure Local requires a proxy connection. This Microsoft Learn article explains the connection, [Cluster Connect](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-cluster-connect)

```cli
az connectedk8s proxy -n <cluster_name> -g <cluster_rg>
```

expected output:

```text
Proxy is listening on port 47011
Merged "<cluster_name>" as current context in /<path>/.kube/config
Start sending kubectl requests on '<cluster_name>' context using kubeconfig at /<path>/.kube/config
Press Ctrl+C to close proxy.
```

__NOTE__: you will need to open a new terminal window to run your kubectl commands as the proxy connection continues to run in the window.

Check to see that you have access to your Arc-enabled Kubernetes cluster on the Azure Local

```cli
kubectl get ns
```

example output:

```text
NAME                      STATUS   AGE
azure-arc                 Active   22d
azuremonitor-containers   Active   22d
default                   Active   22d
flux-system               Active   20d
gatekeeper-system         Active   22d
gitops                    Active   20d
kube-node-lease           Active   22d
kube-public               Active   22d
kube-system               Active   22d
mdc                       Active   22d
```

### Validate the storage class that you have on your Azure Local, this will be need to be updated in the sql-server-aks.yaml file (row 23 and 71)

```cli
kubectl get storageclass
```

example output:

```text
NAME                PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
default (default)   disk.csi.akshci.com   Delete          Immediate           true                   22d
sql-storage         disk.csi.akshci.com   Retain          Immediate           true                   10m
```

If you need to deploy a new storage class

```cli
kubectl apply -f storage_class.yaml
```

### Create the namespace for your SQL deployment

```cli
kubectl apply -f new-namespace.yaml
```

This will create a name space called sql-demo, if you want a different name update in the new-namespace.yaml file, also make sure that you then update the namespace in sql-server-aks.yaml (row 4, 13, 29, 79)

Once you have your storage class and namespace created you will be ready to roll out the containers.

### Deploying SQL Container from YAML

__NOTE__: before you proceed ensure that the SA_PASSWORD on row 8 of the sql-server-aks.yaml

```cli
kubectl apply -f sql-server-aks.yaml
```

__NOTE__: to check the status of your deployment

```cli
kubectl get pods -n sql-demo
```

expected output:

```text
NAME      READY   STATUS    RESTARTS   AGE
mssql-0   1/1     Running   0          81m
```

check pvc binding

```cli
kubectl get pvc -n sql-demo
```

example output:

```text
mssql-data-mssql-0   Bound    pvc-ad134dca-6d82-4e24-a3fe-b7da90444930   20Gi       RWO            default        <unset>                 84m
mssql-pvc            Bound    pvc-c8d89563-fe7d-4e7e-9f45-ea6d4709e826   20Gi       RWO            default        <unset>                 84m
```

check the status of the service

```cli
kubectl get service -n sql-demo
```

example output:

```cli
NAME    TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
mssql   LoadBalancer   10.99.75.8   <pending>     1433:32666/TCP   83m
```

At this point you will notice that there is a ```<pending>``` external ip on the Azure Local. If you were deploying directly to AKS you would likely have a public IP. Ensure that you are securing that public IP with an NSG.

### Creating Load Balancer on Azure Local

Azure Local uses MetalLB, you will need to ensure that the arcnetworking extension is deployed. Here is the Microsoft Learn documentation to walk through [*Deploying MetalLB for Arc-Enabled Kubernetes*](https://learn.microsoft.com/en-us/azure/aks/aksarc/deploy-load-balancer-portal)

Once you have the extension deployed, you can proceed in creating a loadbalancer so you can connect to the SQL instance.

```cli
lbname=sql-lb
resUri=$(az connectedk8s show -n k8s-oside -g fse-v-demo-rg --query id -o tsv)
ipRange=192.168.1.222/32
advertiseMode=both
svcSelector={"app":"mssql"}

az k8s-runtime load-balancer create --load-balancer-name $lbname --resource-uri $resUri --addresses $ipRange --advertise-mode $advertiseMode --service-selector $svcSelector
```

example output:

```json
{
  "id": "/subscriptions/<subid>/resourceGroups/<rg_name>/providers/Microsoft.Kubernetes/ConnectedClusters/<clustername>/providers/Microsoft.KubernetesRuntime/loadBalancers/sql-lb",
  "name": "sql-lb",
  "properties": {
    "addresses": [
      "192.168.1.222/32"
    ],
    "advertiseMode": "Both",
    "provisioningState": "Succeeded",
    "serviceSelector": {
      "app": "mssql"
    }
  },
  "resourceGroup": "<rg_name>",
  "systemData": {
    "createdAt": "2025-12-10T21:08:57.7891663Z",
    "createdBy": "your_upn",
    "createdByType": "User",
    "lastModifiedAt": "2025-12-10T21:08:57.7891663Z",
    "lastModifiedBy": "your_upn",
    "lastModifiedByType": "User"
  },
  "type": "microsoft.kubernetesruntime/loadbalancers"
}
```

You should now be able to connect to the SQL Server in AKS, using the ip listed above and connecting on port 1433.

### Using the Azure Portal to deploy

All the above can be done from the portal as well, it will require however that you create a bearer token first. In your terminal once you have established the proxy connection. Create a variable for your AAD entity object and then we will create the bearer token.

```cli
AAD_ENTITY_OBJECT=$(az ad signed-in-user show --query id -o tsv)

kubectl create token $AAD_ENTITY_OBJECT -n default
```

Copy the token and use in the portal. Once you can see the services in the Azure portal you can add the YAML files directly.
