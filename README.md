# Azure terraform UPI with external DNS

Assumptions:

1. IPs are managed externally - no automation possible
2. We use static IPs for bootstrap, masters & LBs


Requirements:

1. Azure SP with Contributor Rights
2. Predefined: vnet (master & worker subnet), RG
3. ssh key
4. az, openshift-install, ansible, jq, yq, oc, kubectl
5. ocp pull secret

Workflow:

1. Define IPs
2. Create install-config.yaml (see install-config.yaml.sample)
3. Create osServicePrincipal.json (see steps)


Steps:

```bash
# make openshift-installer talk to AZ with SP
export AZURE_AUTH_LOCATION=$(pwd)/osServicePrincipal.json
echo "{\"subscriptionId\": \"${subscriptionId}\", \"clientId\": \"${clientId}\", \"clientSecret\": \"${clientSecret}\", \"tenantId\": \"${tenantId}\" }"  | jq > $AZURE_AUTH_LOCATION

mkdir install
cp install-config.yaml install
./openshift-install create manifests --dir=./install
./openshift-install create ignition-configs --dir=../install

# Verify content of the files above in the generated bootstrap.ign iginition file

# Create terraform config
cat >./install/terraform.tfvars<<EOF
azure_subscription_id = "${subscriptionId}"
azure_client_id = "${clientId}"
azure_client_secret = "${clientSecret}"
azure_tenant_id = "${tenantId}"
azure_bootstrap_vm_type = "Standard_D4s_v3"
azure_master_vm_type = "Standard_D8s_v3"
azure_master_root_volume_size = 1024
azure_region = "$(jq -r -c .azure.region metadata.json)"
azure_resource_group_name = "${RG_NAME}"
azure_base_domain_resource_group_name = "external"
azure_image_url = "$(curl -s https://raw.githubusercontent.com/openshift/installer/release-4.7/data/data/rhcos.json | jq -r -c .azure.url)"
azure_master_availability_zones = [ 1,2,3 ]
cluster_id = "$(jq -r -c .clusterName metadata.json)"
base_domain = "$(jq -r .ignition.config.append[0].source master.ign  | sed 's@.*/api-int.[^\.]*\.\(.*\):22623/.*@\1@')
cluster_domain = "$(jq -r .ignition.config.append[0].source $(pwd)/master.ign  | sed 's@.*/api-int.\(.*\):22623/.*@\1@')"
machine_cidr = "$(jq -r '.["*installconfig.InstallConfig"]["config"]["networking"]["machineCIDR"]' .openshift_install_state.json)"
master_count = 3

ignition_bootstrap_file = "$(pwd)/install/bootstrap.ign"
ignition_master_file = "$(pwd)/install/master.ign"
azure_api_ip=1.2.3.4
azure_bootstrap_ip=1.2.3.5
azure_master_ips=['2.3.4.5','2.3.4.6.','2.3.4.7']
EOF

cd terraform
terraform init
terraform plan -var-file=./install/terraform.tfvar -out ../install/plan.out
terraform apply ../install/plan.out

./openshift-install wait-for bootstrap-complete --dir=../install

terraform destroy -target=module.bootstrap -auto-approve -var-file=../terraform.tfvars

# wait for apps LB to popup and move it to the right ip

./openshift-install wait-for install-complete --dir=../install
```
