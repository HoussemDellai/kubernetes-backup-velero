Velero is the leading tool for creating backups for Kubernets configuration and application manifest YAML files and Volumes.

This demo will guide through a sample demonstration for how to use Velero to backup an AKS cluster in Azure Storage Account.  

1) Creating an Azure Storage Account
2) Installing Velero on Kubernetes 
3) Creating a backup for YAML and Volumes
4) Simulating a cluster disaster like deleting an application
5) Restoring the application

The following script uses Powershell.
Guide using Bash is available here: https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure

```powershell
# Install velero on client machine (Windows here)
# doc: https://velero.io/docs/v1.5/basic-install/
choco install velero

$AZURE_BACKUP_SUBSCRIPTION_NAME="Microsoft Azure #1"
$AZURE_BACKUP_SUBSCRIPTION_ID="25a98a18-5e94-4b21-9d17-e8cf45bfd81f"
$AZURE_SUBSCRIPTION_ID=$(az account list --query '[?isDefault].id' -o tsv)
$AZURE_TENANT_ID=$(az account list --query '[?isDefault].tenantId' -o tsv)
$AZURE_BACKUP_RESOURCE_GROUP="Velero_Backups"
$AZURE_STORAGE_ACCOUNT_ID="storageforvelero"
$BLOB_CONTAINER="velero"
$AZURE_RESOURCE_GROUP="MC_aks-velero-demo_aks-velero-demo_westeurope"

# create a resource group on Azure to store the AKS cluster backup
az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location WestEurope

# create the storage account
az storage account create `
   --name $AZURE_STORAGE_ACCOUNT_ID `
   --resource-group $AZURE_BACKUP_RESOURCE_GROUP `
   --sku Standard_GRS `
   --encryption-services blob `
   --https-only true `
   --kind BlobStorage `
   --access-tier Hot

# create the container inside the storage account
az storage container create -n $BLOB_CONTAINER --public-access off --account-name $AZURE_STORAGE_ACCOUNT_ID

# create the Service Principal (SPN) for Velero
$AZURE_CLIENT_SECRET=$(az ad sp create-for-rbac --name "velero" --role "Contributor" --query 'password' -o tsv `
  --scopes  /subscriptions/$AZURE_SUBSCRIPTION_ID /subscriptions/$AZURE_BACKUP_SUBSCRIPTION_ID)

# get the SPN client Id
$AZURE_CLIENT_ID=$(az ad sp list --display-name "velero" --query '[0].appId' -o tsv)

# create a file with creadentials needed for Velero
$CredentialsVelero=@"
AZURE_SUBSCRIPTION_ID=$($AZURE_SUBSCRIPTION_ID)
AZURE_TENANT_ID=$($AZURE_TENANT_ID)
AZURE_CLIENT_ID=$($AZURE_CLIENT_ID)
AZURE_CLIENT_SECRET=$($AZURE_CLIENT_SECRET)
AZURE_RESOURCE_GROUP=$($AZURE_RESOURCE_GROUP)
AZURE_CLOUD_NAME=AzurePublicCloud
"@
$CredentialsVelero | Out-File -FilePath credentials-velero.yaml

# install velero on Kubernetes/AKS
velero install `
    --provider azure `
    --plugins velero/velero-plugin-for-microsoft-azure:v1.1.0 `
    --bucket $BLOB_CONTAINER `
    --secret-file .\credentials-velero.yaml `
    --backup-location-config resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_ID,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID `
    --snapshot-location-config apiTimeout=5m,resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID

# check the created resources
kubectl get all -n velero

# check the created CRDs
Kubectl get crd

# deploy sample nginx app
kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/velero/main/examples/nginx-app/base.yaml

# create a backup
velero backup create nginx-backup --include-namespaces nginx-example

# get the backups
kubectl get backups -n velero

# describe the backup
kubectl describe backup nginx-backup -n velero

# simulate a disaster
kubectl delete namespaces nginx-example
kubectl get all -n nginx-example

# restore a backup
velero restore create --from-backup nginx-backup

# describe the restored backup (change the name: nginx-backup-xxx)
velero restore describe nginx-backup-20200918100646

# check the restored resources
kubectl get all -n nginx-example

# to uninstall Velero
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```
