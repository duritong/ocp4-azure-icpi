# OCP 4 - Customized IPI installation on Azure

This repository contains guidance and terraform manifests to do a customized IPI installation on Azure.

This is not a supported way of installing OpenShift 4, but it is close to what the `openshift-install` does
when you are doing an IPI installation on Azure.

## General idea

Basically, we follow what the IPI would do, but are driving the steps on our own. This allows us
to adapt the automatically deployed infrastructure, rename components differently and so on.

Please be aware that this won't let you customize deployments to a complete different way how a
cluster looks like. The core of OCP 4 are operators and they will still enforce things they want.
But if we can configure them, then

What we will do:

* Prepare the Azure Account and credentials
* Create install config and adapt to what you want and is possible at this stage
* Create the manifests and adapt them
* Create the ignition configs out of the manifests
* Set terraform variables based on the requirements
* Run terraform
* watch boostrap
* destroy bootstrap
* watch operators
* done

## Requirements

* az cli
* terraform
* OpenShift installer
* OpenShift cli binary

## Do the job

The next sections should give you all the necessary steps that are tuneable.


### Prepare Account

```shell
RG_NAME="ocp4"
az ad sp create-for-rbac --role Contributor --name $RG_NAME | jq --arg sub_id "$(az account show | jq -r '.id')" '{subscriptionId:$sub_id,clientId:.appId, clientSecret:.password,tenantId:.tenant}' | jq -r 'keys[] as $k | "export \($k)=\(.[$k])"' | tee app_env
source app_env
export AZURE_AUTH_LOCATION=$(pwd)/osServicePrincipal.json
echo "{\"subscriptionId\": \"${subscriptionId}\", \"clientId\": \"${clientId}\", \"clientSecret\": \"${clientSecret}\", \"tenantId\": \"${tenantId}\" }"  | jq > $AZURE_AUTH_LOCATION
az role assignment create --role "User Access Administrator"     --assignee-object-id $(az ad sp list --filter "appId eq '${clientId}'" | jq '.[0].objectId' -r)
az ad app permission add --id ${clientId} --api 00000002-0000-0000-c000-000000000000 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role
# Grant permission for default directory in AD -> App registrations for that account for API Permissions
```

### Run installer

```shell
rm -rf $RG_NAME && mkdir $RG_NAME # ensure we have a clean directory
cd $RG_NAME
./openshift-install create install-config --dir=.
# tune install-config - ./install-config.yaml
./openshift-install create manifests --dir=.
# adapt resource group name in .openshift_install_state.json /*/*.yml or whatever you like to change
# IMPORTANT: some parts are already k8s secrets (e.g. 99_cloud-creds-secret.yaml) which contain the value base64 encoded, thus you need to do the base64 dance twice.
sed -i "s/$(grep resourceGroupName: ./manifests/cluster-infrastructure-02-config.yml | cut -d:  -f 2 | tr -d ' ')/${RG_NAME}/g" .openshift_install_state.json */*.yaml
# Also edit the base64 blobs of the following files in .openshift_install_state.json as well as their content in the manifests .yaml
# cloud-provider-config.yaml
# 99_cloud-creds-secret.yaml
# 99_openshift-cluster-api_master-machines-0.yaml
# 99_openshift-cluster-api_master-machines-1.yaml
# 99_openshift-cluster-api_master-machines-2.yaml
# 99_openshift-cluster-api_worker-machineset-0.yaml
# 99_openshift-cluster-api_worker-machineset-1.yaml
# 99_openshift-cluster-api_worker-machineset-2.yaml
# example command
# echo -n BASE64_BLOB | base64 -d | sed MAGIC | base74 -w 0

./openshift-install create ignition-configs --dir=.

# Verify content of the files above in the generated bootstrap.ign iginition file

# Create terraform config
cat >./terraform.tfvars<<EOF
azure_subscription_id = "${subscriptionId}"
azure_client_id = "${clientId}"
azure_client_secret = "${clientSecret}"
azure_tenant_id = "${tenantId}"
azure_bootstrap_vm_type = "Standard_D4s_v3"
azure_master_vm_type = "Standard_D4s_v3"
azure_master_root_volume_size = 64
azure_region = "$(jq -r -c .azure.region metadata.json)"
azure_resource_group_name = "${RG_NAME}"
azure_base_domain_resource_group_name = "imagestore"
azure_image_url = "$(curl -s https://raw.githubusercontent.com/openshift/installer/release-4.2/data/data/rhcos.json | jq -r -c .azure.url)"
azure_master_availability_zones = [ 1,2,3 ]
cluster_id = "$(jq -r -c .clusterName metadata.json)"
base_domain = "$(jq -r .ignition.config.append[0].source master.ign  | sed 's@.*/api-int.[^\.]*\.\(.*\):22623/.*@\1@')
cluster_domain = "$(jq -r .ignition.config.append[0].source $(pwd)/master.ign  | sed 's@.*/api-int.\(.*\):22623/.*@\1@')"
machine_cidr = "$(jq -r '.["*installconfig.InstallConfig"]["config"]["networking"]["machineCIDR"]' .openshift_install_state.json)"
master_count = 3
ignition_bootstrap_file = "$(pwd)/bootstrap.ign"
ignition_master_file = "$(pwd)/master.ign"
EOF

OCP_INSTALLER_DIR=$(pwd)
clone this repository and change to the directory
# now would be the time to adapt the manifests

terraform init
terraform plan -var-file=${OCP_INSTALLER_DIR}/terraform.tfvars -out ${OCP_INSTALLER_DIR}/plan.out
terraform apply ${OCP_INSTALLER_DIR}/plan.out

./openshift-install wait-for bootstrap-complete --dir=${OCP_INSTALLER_DIR}
# It is safe to destroy the boostrap node as soon as the previous commands complete

terraform destroy -target=module.bootstrap -auto-approve -var-file=${OCP_INSTALLER_DIR}/terraform.tfvars

# Let operators do their work
./openshift-install wait-for install-complete --dir=${OCP_INSTALLER_DIR}

# Done!
```

### Destroy

```shell
./openshift-install destroy cluster --dir=${OCP_INSTALLER_DIR}
```
## Getting terraform manifests

Terraform manifests can be found in the installer in the following directory structure: `data/data/azure`.
They have only been slightly adapted to make them useable out of the box. Eventually you might adapt
certain manifests as you like to pre-configure your infrastructure.


## Random bits

### Import Image

Adapted from: https://github.com/openshift/installer/blob/d0f7654bc4a0cf73392371962aef68cd9552b5dd/docs/user/azure/install.md

```shell
az group create --name os4-common --location westeurope
az storage account create --location westeurope --name os4storagemh --kind StorageV2 --resource-group os4-common
az storage container create --name vhd --account-name os4storagemh
ACCOUNT_KEY=$(az storage account keys list --account-name os4storagemh --resource-group os4-common --query "[0].value" -o tsv)

# Get latest image from: https://github.com/openshift/installer/blob/release-4.2/data/data/rhcos.json

VHD_NAME=VHD_NAME=rhcos-42.80.20191002.0.vhd
az storage blob copy start --account-name "os4storage" --account-key "$ACCOUNT_KEY" --destination-blob "$VHD_NAME" --destination-container vhd --source-uri "https://openshifttechpreview.blob.core.windows.net/rhcos/$VHD_NAME"
az storage blob show -c vhd  --name "$VHD_NAME" --account-name "os4storage"
RHCOS_VHD=$(az storage blob url --account-name os4storage -c vhd --name "$VHD_NAME" -o tsv)
az image create --resource-group rhcos_images --name rhcosimage --os-type Linux --storage-sku Premium_LRS --source "$RHCOS_VHD" --location westeurope
```

## License

Apache License 2.0
