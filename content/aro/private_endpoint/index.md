---
date: '2023-06-27T22:07:09.774151'
title: Configure a Private ARO cluster with Azure File via a Private Endpoint
tags: ["ARO", "Azure"]
authors:
  - Connor Wooley
  - Dustin Scott
---

There are two way to configure this set up
1. Self provision the storage account and file share (static method)
  - Requires pre-existing storage account and file share
2. Auto provision the storage account and file share (dynamic method)
  - CSI will create the storage account and file share

> **WARNING** please note that this approach does not work on FIPS-enabled clusters.  This is due to the CIFS
protocol being largely non-compliant with FIPS cryptographic requirements.  Please see the following for more 
information:

- [Red Hat Article on CIFS/FIPS](https://access.redhat.com/solutions/256053)
- [Microsoft Article on CIFS/FIPS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/fail-to-mount-azure-file-share#fipsnodepool)

## Pre Requisites

- private aro cluster 


## Set Environment Variables

```bash
export AZR_REGION=eastus \
  AZR_RESOURCE_GROUP=<my-rg> \
  AZR_VNET=<my-vnet> \
  AZR_CLUSTER_NAME=<my-cluster-name> \
  AZR_STORAGE_ACCOUNT_NAME=<my-storage-account> \
  AZR_SERVICES_SUBNET=<private-endpoint-subnet-name> \
  AZR_FILE_SHARE=<your-file-share-name>
  OC_STORAGE_ACCOUNT_SECRET_NAME=<your-name>

```

## Self-Provision Storage Account and File Share (Static Method) 

* Note if you would like to dynamically provision the storage account using the CSI provisioner skip the first step

1. Create the storage account and attach the private endpoint to it  

```bash
az storage account create \
    --name $AZR_STORAGE_ACCOUNT_NAME \
    --resource-group $AZR_RESOURCE_GROUP \
    --location $AZR_REGION \
    --sku Standard_LRS \
    --kind StorageV2
```

2. Create the file share in your storage account

```bash
az storage share create \
    --account-name $AZR_STORAGE_ACCOUNT_NAME \
    --name $AZR_FILE_SHARE \
    --quota 5
```

## Create/Configure the Private Endpoint

*NOTE make sure the share quota matches the quota set later when creating the PVC

1. Create a services subnet in the cluster rg and vnet for the Private Endpoint

```bash
az network vnet subnet create \
    --name $AZR_SERVICES_SUBNET \
    --resource-group $AZR_RESOURCE_GROUP \
    --vnet-name $AZR_VNET \
    --address-prefix 10.0.12.0/23 
```

- address prefix will be configured based on your vnet IP addressing

*NOTE we recommend creating separate subnets for services, especially when you are using a private ARO environment 

3. Create private endpoint 

```bash
az network private-endpoint create \
  --name $AZR_CLUSTER_NAME \
  --resource-group $AZR_RESOURCE_GROUP \
  --vnet-name $AZR_VNET \
  --subnet $AZR_SERVICES_SUBNET \
  --private-connection-resource-id $(az resource show -g $AZR_RESOURCE_GROUP -n $AZR_STORAGE_ACCOUNT_NAME --resource-type "Microsoft.Storage/storageAccounts" --query "id" -o tsv) \
  --location $AZR_REGION \
  --group-id file \
  --connection-name $AZR_STORAGE_ACCOUNT_NAME
```

## DNS Resolution for Private Connection

1. Configure the private DNS zone for the private link connection

In order to use the private endpoint connection you will need to create a private DNS zone, if not configured correctly, the connection will attempt to use the public IP (file.core.windows.net) whereas the private connection's domain is prefixed with 'privatelink'

```bash
az network private-dns zone create \
  --resource-group $AZR_RESOURCE_GROUP \
  --name "privatelink.file.core.windows.net"
  
az network private-dns link vnet create \
  --resource-group $AZR_RESOURCE_GROUP \
  --zone-name "privatelink.file.core.windows.net" \
  --name $AZR_CLUSTER_NAME \
  --virtual-network $AZR_VNET \
  --registration-enabled false
```


  If you are using a custom DNS server on your network, clients must be able to resolve the FQDN for the storage account endpoint to the private endpoint IP address. You should configure your DNS server to delegate your private link subdomain to the private DNS zone for the VNet, or configure the A records for `StorageAccountA.privatelink.file.core.windows.net` with the private endpoint IP address.

  When using a custom or on-premises DNS server, you should configure your DNS server to resolve the storage account name in the privatelink subdomain to the private endpoint IP address. You can do this by delegating the privatelink subdomain to the private DNS zone of the VNet or by configuring the DNS zone on your DNS server and adding the DNS A records.


*For MAG customers:*

  [GOV Private Endpoint DNS](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns#government)

  [Custom DNS Config](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns#virtual-network-workloads-without-custom-dns-server)


2. Retrieve the private IP from the private link connection:

```bash
PRIVATE_IP=`az resource show \
  --ids $(az network private-endpoint show --name $AZR_CLUSTER_NAME --resource-group $AZR_RESOURCE_GROUP --query 'networkInterfaces[0].id' -o tsv) \
  --api-version 2019-04-01 \
  -o json | jq -r '.properties.ipConfigurations[0].properties.privateIPAddress'`
```
3. Create the DNS records for the private link connection:

```bash
az network private-dns record-set a create \
  --name $AZR_STORAGE_ACCOUNT_NAME \
  --zone-name privatelink.file.core.windows.net \
  --resource-group $AZR_RESOURCE_GROUP

az network private-dns record-set a add-record \
  --record-set-name $AZR_STORAGE_ACCOUNT_NAME \
  --zone-name privatelink.file.core.windows.net \
  --resource-group $AZR_RESOURCE_GROUP \
  -a $PRIVATE_IP
```

4. test private endpoint connectivity
  - on a Vm in the vnet run 

```bash 
nslookup $AZR_STORAGE_ACCOUNT_NAME.file.core.windows.net
```
- Should return:
```
Server:		x.x.x.x
Address:	x.x.x.x#x

Non-authoritative answer:
<storage_account_name>.file.core.windows.net	canonical name = <storage_account_name>.privatelink.file.core.windows.net.
Name:	<storage_account_name>.privatelink.file.core.windows.net
Address: x.x.x.x
```

## Configure Cluster Storage Resources

1. Login to your cluster

2. Create a secret object containing azure file creds

```bash
AZR_STORAGE_KEY=$(az storage account keys list --account-name $AZR_STORAGE_ACCOUNT_NAME --query "[0].value" -o tsv)

oc create secret generic $OC_STORAGE_ACCOUNT_SECRET_NAME --from-literal=azurestorageaccountname=$AZR_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$AZR_STORAGE_KEY
```

1. Create a static storage class (see below for dynamic method)

> **NOTE** only needed if using the static provisioning method

- The CSI can either create volumes in pre created storage accounts or dynamically create the storage account with a volume inside the dynamic storage account

- Using an existing storage account

    ```yaml
    allowVolumeExpansion: true
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: <static_sc_name>
    parameters:
      resourceGroup: <cluster_resource_group>
      server: <storage_account>.file.core.windows.net
      skuName: Standard_LRS
      storageAccount: <storage_account>
      secretName: <secret_name>
      secretNamespace: default
      shareName: <file_share_name>
    provisioner: file.csi.azure.com
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    ```

- Create a dynamic storage class

> **NOTE** only needed if using the dynamic provisioning method (see above for static method)

    ```yaml
    allowVolumeExpansion: true
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: <dynamic_sc_name>
    parameters:
      resourceGroup: <cluster-resource-group>
      skuName: Standard_LRS
      secretName: <secret_name>
      secretNamespace: default
      networkEndpointType: privateEndpoint
    provisioner: file.csi.azure.com
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    ```

4. create PVC object that maps to the PV created

- PVCs are scoped at the namespace level so make sure you are creating this volume claim in the appropriate project

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <claim_name>
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "5Gi" 
  storageClassName: <storage_class_name> 
```



5. Mount Azure file share in pod

  - create pod that mounts existing pv
  - optionally patch or create the Mount Path in the containers block of a deployment manifest as well as the PVC object in the volumes block

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-name 
spec:
  containers:
    ...
    volumeMounts:
    - mountPath: "/data" 
      name: <name_your_volume>
  volumes:
    - name: <name_your_volume>
      persistentVolumeClaim:
        claimName: <claim_name> 
```


## Testing

1. Exec into a pod with the mounted volume

```bash
oc exec -it <pod_name> -- /bin/bash
```

2. Create a file in the file share's mount path

```bash
cd <file_share_mount_path>
touch test
```

3. In your Azure portal or using the CLI, verify the created file exists in your Storage Account's File Share 

- Azure portal

  1. Search for Storage Account or find the Storage Account icon on the services blade
  2. Find your storage account
  3. select File Share
  4. Open your File Share and verify the 'test' file is in the file share

- CLI

```bash
az storage file list --account-name $AZR_STORAGE_ACCOUNT_NAME --account-key $AZR_STORAGE_KEY --share-name $AZR_FILE_SHARE
```



