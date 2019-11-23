# Deploy VMs on each Azure subscriptions from vhd

> :warning: This sample is just a use case. For detail to copy VM from VHD, see https://docs.microsoft.com/en-us/azure/virtual-machines/windows/create-vm-specialized#option-3-copy-an-existing-azure-vm.

First, you should login to the Azure account that has the base VHD and get the URI with SAS token.

```bash
# Login to Azure account that has the base VHD
az login

BASE_VHD_STORAGE_NAME="<Storage account name that has the VHD>"
BASE_VHD_STORAGE_CONTAINER="<Container name of the VHD>"
BASE_VHD_STORAGE_BLOB="<Blob name of the VHD>"

# Show an URI with SAS token of the base VHD blob
BASE_VHD_STORAGE_KEY=$(az storage account keys list \
    --account-name ${BASE_VHD_STORAGE_NAME} \
    --query "[0].value" \
    --output tsv)
EXPIRY=`date -u -d "6 hours" '+%Y-%m-%dT%H:%MZ'`
az storage blob generate-sas \
    --container-name ${BASE_VHD_STORAGE_CONTAINER} \
    --name ${BASE_VHD_STORAGE_BLOB} \
    --account-name ${BASE_VHD_STORAGE_NAME} \
    --account-key ${BASE_VHD_STORAGE_KEY} \
    --expiry ${EXPIRY} \
    --permissions r \
    --https-only \
    --full-uri \
    --output tsv

# Copy the URI with SAS token above
```

Then, log in to target account and copy vhd file to the target subscription.

```bash
az login

BASE_VHD_URI_WITH_SAS="<Paste the copied blob URI>"
LOCATION="japaneast"
VM_BASE_RESOURCE_GROUP="<Resource group name>"
VM_BASE_STORAGE_BLOB="<Blob name of the VHD>"

DEPLOYMENT_RESULT=$(az deployment create \
    --name deploymentToPrepareVMBase \
    --location ${LOCATION} \
    --template-uri https://raw.githubusercontent.com/dzeyelid/deploy-vms-on-each-subscriptions-from-vhd/master/01_prepare-vm-base.json \
    --parameters \
        resourceGroupName=${VM_BASE_RESOURCE_GROUP} \
        location=${LOCATION})

VM_BASE_STORAGE_ID=$(echo ${DEPLOYMENT_RESULT} | jq -r ".properties.outputs.storageAccountId.value")
VM_BASE_STORAGE_NAME=$(echo ${DEPLOYMENT_RESULT} | jq -r ".properties.outputs.storageAccountName.value")
VM_BASE_STORAGE_CONTAINER=$(echo ${DEPLOYMENT_RESULT} | jq -r ".properties.outputs.storageContainerName.value")
VM_BASE_STORAGE_ACCOUNT_KEY=$(echo ${DEPLOYMENT_RESULT} | jq -r ".properties.outputs.storageAccountKey.value")

az storage blob copy start \
    --account-name ${VM_BASE_STORAGE_NAME} \
    --account-key ${VM_BASE_STORAGE_ACCOUNT_KEY} \
    --destination-container ${VM_BASE_STORAGE_CONTAINER} \
    --destination-blob ${VM_BASE_STORAGE_BLOB} \
    --source-uri ${BASE_VHD_URI_WITH_SAS}
```

After the copying has finished, create VMs from the VHD.

```bash
VM_VHD_URI=$(az storage blob url \
    --account-name ${VM_BASE_STORAGE_NAME} \
    --container-name ${VM_BASE_STORAGE_CONTAINER} \
    --name ${VM_BASE_STORAGE_BLOB} \
    --output tsv)

LOCATION="japaneast"
PREFIX="<prefix>"
NUMBER_OF_VMS="<Number of Vms>"

az deployment create \
    --name deploymentVMs \
    --location ${LOCATION} \
    --template-uri https://raw.githubusercontent.com/dzeyelid/deploy-vms-on-each-subscriptions-from-vhd/master/02_create-vms.json \
    --parameters \
        prefix=${PREFIX} \
        location=${LOCATION} \
        numberOfVms=${NUMBER_OF_VMS} \
        vmVhdUri=${VM_VHD_URI} \
        storageAccountId=${VM_BASE_STORAGE_ID}
```
