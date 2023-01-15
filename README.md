terraform-azurerm-aks
Deploys a Kubernetes cluster on AKS with monitoring support through Azure Log Analytics

This Terraform module deploys a Kubernetes cluster on Azure using AKS (Azure Kubernetes Service) and adds support for monitoring with Log Analytics.

-> NOTE: If you have not assigned client_id or client_secret, A SystemAssigned identity will be created.
Notice on Upgrade to V6.x

We've added a CI pipeline for this module to speed up our code review and to enforce a high code quality standard, if you want to contribute by submitting a pull request, please read Pre-Commit & Pr-Check & Test section, or your pull request might be rejected by CI pipeline.

A pull request will be reviewed when it has passed Pre Pull Request Check in the pipeline, and will be merged when it has passed the acceptance tests. Once the ci Pipeline failed, please read the pipeline's output, thanks for your cooperation.
Notice on Upgrade to V5.x

V5.0.0 is a major version upgrade and a lot of breaking changes have been introduced. Extreme caution must be taken during the upgrade to avoid resource replacement and downtime by accident.

Running the terraform plan first to inspect the plan is strongly advised.
Terraform and terraform-provider-azurerm version restrictions

Now Terraform core's lowest version is v1.2.0 and terraform-provider-azurerm's lowest version is v3.21.0.
variable user_assigned_identity_id has been renamed.

variable user_assigned_identity_id has been renamed to identity_ids and it's type has been changed from string to list(string).
addon_profile in outputs is no longer available.

It has been broken into the following new outputs:

    aci_connector_linux
    aci_connector_linux_enabled
    azure_policy_enabled
    http_application_routing_enabled
    ingress_application_gateway
    ingress_application_gateway_enabled
    key_vault_secrets_provider
    key_vault_secrets_provider_enabled
    oms_agent
    oms_agent_enabled
    open_service_mesh_enabled

The following variables have been renamed from enable_xxx to xxx_enabled

    enable_azure_policy has been renamed to azure_policy_enabled
    enable_http_application_routing has been renamed to http_application_routing_enabled
    enable_ingress_application_gateway has been renamed to ingress_application_gateway_enabled
    enable_log_analytics_workspace has been renamed to log_analytics_workspace_enabled
    enable_open_service_mesh has been renamed to open_service_mesh_enabled
    enable_role_based_access_control has been renamed to role_based_access_control_enabled

nullable = true has been added to the following variables so setting them to null explicitly will use the default value

    log_analytics_workspace_enable
    os_disk_type
    private_cluster_enabled
    rbac_aad_managed
    rbac_aad_admin_group_object_ids
    network_policy
    enable_node_public_ip

var.admin_username's default value has been removed

In v4.x var.admin_username has a default value azureuser and has been removed in V5.0.0. Since the admin_username argument in linux_profile block is a ForceNew argument, any value change to this argument will trigger a Kubernetes cluster replacement SO THE EXTREME CAUTION MUST BE TAKEN. The module's callers must set var.admin_username to azureuser explicitly if they didn't set it before.
module.ssh-key has been removed

The file named private_ssh_key which contains the tls private key will be deleted since the local_file resource has been removed. Now the private key is exported via generated_cluster_private_ssh_key in output and the corresponding public key is exported via generated_cluster_public_ssh_key in output.

A moved block has been added to relocate the existing tls_private_key resource to the new address. If the var.admin_username is not null, no action is needed.

Resource tls_private_key's creation now is conditional. Users may see the destruction of existing tls_private_key in the generated plan if var.admin_username is null.
system_assigned_identity in the output has been renamed to cluster_identity

The system_assigned_identity was:

output "system_assigned_identity" {
  value = azurerm_kubernetes_cluster.main.identity
}

Now it has been renamed to cluster_identity, and the block has been changed to:

output "cluster_identity" {
  description = "The `azurerm_kubernetes_cluster`'s `identity` block."
  value       = try(azurerm_kubernetes_cluster.main.identity[0], null)
}

The callers who used to read the cluster's identity block need to remove the index in their expression, from module.aks.system_assigned_identity[0] to module.aks.cluster_identity.
The following outputs are now sensitive. All outputs referenced them must be declared as sensitive too

    client_certificate
    client_key
    cluster_ca_certificate
    generated_cluster_private_ssh_key
    host
    kube_admin_config_raw
    kube_config_raw
    password
    username

Usage in Terraform 1.2.0

Please view folders in examples.

The module supports some outputs that may be used to configure a kubernetes provider after deploying an AKS cluster.

provider "kubernetes" {
  host                   = module.aks.host
  client_certificate     = base64decode(module.aks.client_certificate)
  client_key             = base64decode(module.aks.client_key)
  cluster_ca_certificate = base64decode(module.aks.cluster_ca_certificate)
}

There're some examples in the examples folder. You can execute terraform apply command in examples's sub folder to try the module. These examples are tested against every PR with the E2E Test.
Pre-Commit & Pr-Check & Test
Configurations

    Configure Terraform for Azure

We assumed that you have setup service principal's credentials in your environment variables like below:

export ARM_SUBSCRIPTION_ID="<azure_subscription_id>"
export ARM_TENANT_ID="<azure_subscription_tenant_id>"
export ARM_CLIENT_ID="<service_principal_appid>"
export ARM_CLIENT_SECRET="<service_principal_password>"

On Windows Powershell:

$env:ARM_SUBSCRIPTION_ID="<azure_subscription_id>"
$env:ARM_TENANT_ID="<azure_subscription_tenant_id>"
$env:ARM_CLIENT_ID="<service_principal_appid>"
$env:ARM_CLIENT_SECRET="<service_principal_password>"

We provide a docker image to run the pre-commit checks and tests for you: mcr.microsoft.com/azterraform:latest

To run the pre-commit task, we can run the following command:

$ docker run --rm -v $(pwd):/src -w /src mcr.microsoft.com/azterraform:latest make pre-commit

On Windows Powershell:

$ docker run --rm -v ${pwd}:/src -w /src mcr.microsoft.com/azterraform:latest make pre-commit

In pre-commit task, we will:

    Run terraform fmt -recursive command for your Terraform code.
    Run terrafmt fmt -f command for markdown files and go code files to ensure that the Terraform code embedded in these files are well formatted.
    Run go mod tidy and go mod vendor for test folder to ensure that all the dependencies have been synced.
    Run gofmt for all go code files.
    Run gofumpt for all go code files.
    Run terraform-docs on README.md file, then run markdown-table-formatter to format markdown tables in README.md.

Then we can run the pr-check task to check whether our code meets our pipeline's requirement(We strongly recommend you run the following command before you commit):

$ docker run --rm -v $(pwd):/src -w /src mcr.microsoft.com/azterraform:latest make pr-check

On Windows Powershell:

$ docker run --rm -v ${pwd}:/src -w /src mcr.microsoft.com/azterraform:latest make pr-check

To run the e2e-test, we can run the following command:

docker run --rm -v $(pwd):/src -w /src -e ARM_SUBSCRIPTION_ID -e ARM_TENANT_ID -e ARM_CLIENT_ID -e ARM_CLIENT_SECRET mcr.microsoft.com/azterraform:latest make e2e-test

On Windows Powershell:

docker run --rm -v ${pwd}:/src -w /src -e ARM_SUBSCRIPTION_ID -e ARM_TENANT_ID -e ARM_CLIENT_ID -e ARM_CLIENT_SECRET mcr.microsoft.com/azterraform:latest make e2e-test

To follow Ensure AKS uses disk encryption set policy we've used azurerm_key_vault in example codes, and to follow Key vault does not allow firewall rules settings we've limited the ip cidr on it's network_acls. On default we'll use the ip return by https://api.ipify.org?format=json api as your public ip, but in case you need use other cidr, you can assign on by passing an environment variable:

docker run --rm -v $(pwd):/src -w /src -e TF_VAR_key_vault_firewall_bypass_ip_cidr="<your_cidr>" -e ARM_SUBSCRIPTION_ID -e ARM_TENANT_ID -e ARM_CLIENT_ID -e ARM_CLIENT_SECRET mcr.microsoft.com/azterraform:latest make e2e-test

On Windows Powershell:

docker run --rm -v ${pwd}:/src -w /src -e TF_VAR_key_vault_firewall_bypass_ip_cidr="<your_cidr>" -e ARM_SUBSCRIPTION_ID -e ARM_TENANT_ID -e ARM_CLIENT_ID -e ARM_CLIENT_SECRET mcr.microsoft.com/azterraform:latest make e2e-test

Prerequisites

    Docker

Authors

Originally created by Damien Caro and Malte Lantin
License

MIT
Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the Microsoft Open Source Code of Conduct. For more information see the Code of Conduct FAQ or contact opencode@microsoft.com with any additional questions or comments.
Module Spec

The following sections are generated by terraform-docs and markdown-table-formatter, please DO NOT MODIFY THEM MANUALLY!
Requirements
Name 	Version
terraform 	>= 1.2
azurerm 	>= 3.27
tls 	>= 3.1
Providers
Name 	Version
azurerm 	>= 3.27
tls 	>= 3.1
Modules

No modules.
Resources
Name 	Type
azurerm_kubernetes_cluster.main 	resource
azurerm_log_analytics_solution.main 	resource
azurerm_log_analytics_workspace.main 	resource
tls_private_key.ssh 	resource
azurerm_resource_group.main 	data source
Inputs
Name 	Description 	Type 	Default 	Required
aci_connector_linux_enabled 	Enable Virtual Node pool 	bool 	false 	no
aci_connector_linux_subnet_name 	(Optional) aci_connector_linux subnet name 	string 	null 	no
admin_username 	The username of the local administrator to be created on the Kubernetes cluster. Set this variable to null to turn off the cluster's linux_profile. Changing this forces a new resource to be created. 	string 	null 	no
agents_availability_zones 	(Optional) A list of Availability Zones across which the Node Pool should be spread. Changing this forces a new resource to be created. 	list(string) 	null 	no
agents_count 	The number of Agents that should exist in the Agent Pool. Please set agents_count null while enable_auto_scaling is true to avoid possible agents_count changes. 	number 	2 	no
agents_labels 	(Optional) A map of Kubernetes labels which should be applied to nodes in the Default Node Pool. Changing this forces a new resource to be created. 	map(string) 	{} 	no
agents_max_count 	Maximum number of nodes in a pool 	number 	null 	no
agents_max_pods 	(Optional) The maximum number of pods that can run on each agent. Changing this forces a new resource to be created. 	number 	null 	no
agents_min_count 	Minimum number of nodes in a pool 	number 	null 	no
agents_pool_name 	The default Azure AKS agentpool (nodepool) name. 	string 	"nodepool" 	no
agents_size 	The default virtual machine size for the Kubernetes agents 	string 	"Standard_D2s_v3" 	no
agents_tags 	(Optional) A mapping of tags to assign to the Node Pool. 	map(string) 	{} 	no
agents_type 	(Optional) The type of Node Pool which should be created. Possible values are AvailabilitySet and VirtualMachineScaleSets. Defaults to VirtualMachineScaleSets. 	string 	"VirtualMachineScaleSets" 	no
api_server_authorized_ip_ranges 	(Optional) The IP ranges to allow for incoming traffic to the server nodes. 	set(string) 	null 	no
automatic_channel_upgrade 	(Optional) The upgrade channel for this Kubernetes Cluster. Possible values are patch, rapid, node-image and stable. By default automatic-upgrades are turned off. Note that you cannot use the patch upgrade channel and still specify the patch version using kubernetes_version. See the documentation for more information 	string 	null 	no
azure_policy_enabled 	Enable Azure Policy Addon. 	bool 	false 	no
client_id 	(Optional) The Client ID (appId) for the Service Principal used for the AKS deployment 	string 	"" 	no
client_secret 	(Optional) The Client Secret (password) for the Service Principal used for the AKS deployment 	string 	"" 	no
cluster_log_analytics_workspace_name 	(Optional) The name of the Analytics workspace 	string 	null 	no
cluster_name 	(Optional) The name for the AKS resources created in the specified Azure Resource Group. This variable overwrites the 'prefix' var (The 'prefix' var will still be applied to the dns_prefix if it is set) 	string 	null 	no
disk_encryption_set_id 	(Optional) The ID of the Disk Encryption Set which should be used for the Nodes and Volumes. More information can be found in the documentation. Changing this forces a new resource to be created. 	string 	null 	no
enable_auto_scaling 	Enable node pool autoscaling 	bool 	false 	no
enable_host_encryption 	Enable Host Encryption for default node pool. Encryption at host feature must be enabled on the subscription: https://docs.microsoft.com/azure/virtual-machines/linux/disks-enable-host-based-encryption-cli 	bool 	false 	no
enable_node_public_ip 	(Optional) Should nodes in this Node Pool have a Public IP Address? Defaults to false. 	bool 	false 	no
http_application_routing_enabled 	Enable HTTP Application Routing Addon (forces recreation). 	bool 	false 	no
identity_ids 	(Optional) Specifies a list of User Assigned Managed Identity IDs to be assigned to this Kubernetes Cluster. 	list(string) 	null 	no
identity_type 	(Optional) The type of identity used for the managed cluster. Conflict with client_id and client_secret. Possible values are SystemAssigned, UserAssigned, SystemAssigned, UserAssigned(to enable both). If UserAssigned or SystemAssigned, UserAssigned is set, an identity_ids must be set as well. 	string 	"SystemAssigned" 	no
ingress_application_gateway_enabled 	Whether to deploy the Application Gateway ingress controller to this Kubernetes Cluster? 	bool 	false 	no
ingress_application_gateway_id 	The ID of the Application Gateway to integrate with the ingress controller of this Kubernetes Cluster. 	string 	null 	no
ingress_application_gateway_name 	The name of the Application Gateway to be used or created in the Nodepool Resource Group, which in turn will be integrated with the ingress controller of this Kubernetes Cluster. 	string 	null 	no
ingress_application_gateway_subnet_cidr 	The subnet CIDR to be used to create an Application Gateway, which in turn will be integrated with the ingress controller of this Kubernetes Cluster. 	string 	null 	no
ingress_application_gateway_subnet_id 	The ID of the subnet on which to create an Application Gateway, which in turn will be integrated with the ingress controller of this Kubernetes Cluster. 	string 	null 	no
key_vault_secrets_provider_enabled 	(Optional) Whether to use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster. For more details: https://docs.microsoft.com/en-us/azure/aks/csi-secrets-store-driver 	bool 	false 	no
kubernetes_version 	Specify which Kubernetes release to use. The default used is the latest Kubernetes version available in the region 	string 	null 	no
load_balancer_profile_enabled 	(Optional) Enable a load_balancer_profile block. This can only be used when load_balancer_sku is set to standard. 	bool 	false 	no
load_balancer_profile_idle_timeout_in_minutes 	(Optional) Desired outbound flow idle timeout in minutes for the cluster load balancer. Must be between 4 and 120 inclusive. 	number 	30 	no
load_balancer_profile_managed_outbound_ip_count 	(Optional) Count of desired managed outbound IPs for the cluster load balancer. Must be between 1 and 100 inclusive 	number 	null 	no
load_balancer_profile_managed_outbound_ipv6_count 	(Optional) The desired number of IPv6 outbound IPs created and managed by Azure for the cluster load balancer. Must be in the range of 1 to 100 (inclusive). The default value is 0 for single-stack and 1 for dual-stack. Note: managed_outbound_ipv6_count requires dual-stack networking. To enable dual-stack networking the Preview Feature Microsoft.ContainerService/AKS-EnableDualStack needs to be enabled and the Resource Provider re-registered, see the documentation for more information. https://learn.microsoft.com/en-us/azure/aks/configure-kubenet-dual-stack?tabs=azure-cli%2Ckubectl#register-the-aks-enabledualstack-preview-feature 	number 	null 	no
load_balancer_profile_outbound_ip_address_ids 	(Optional) The ID of the Public IP Addresses which should be used for outbound communication for the cluster load balancer. 	set(string) 	null 	no
load_balancer_profile_outbound_ip_prefix_ids 	(Optional) The ID of the outbound Public IP Address Prefixes which should be used for the cluster load balancer. 	set(string) 	null 	no
load_balancer_profile_outbound_ports_allocated 	(Optional) Number of desired SNAT port for each VM in the clusters load balancer. Must be between 0 and 64000 inclusive. Defaults to 0 	number 	0 	no
load_balancer_sku 	(Optional) Specifies the SKU of the Load Balancer used for this Kubernetes Cluster. Possible values are basic and standard. Defaults to standard. Changing this forces a new kubernetes cluster to be created. 	string 	"standard" 	no
local_account_disabled 	(Optional) - If true local accounts will be disabled. Defaults to false. See the documentation for more information. 	bool 	null 	no
location 	Location of cluster, if not defined it will be read from the resource-group 	string 	null 	no
log_analytics_solution_id 	(Optional) Existing azurerm_log_analytics_solution ID. Providing ID disables creation of azurerm_log_analytics_solution. 	string 	null 	no
log_analytics_workspace 	(Optional) Existing azurerm_log_analytics_workspace to attach azurerm_log_analytics_solution. Providing the config disables creation of azurerm_log_analytics_workspace. 	

object({
    id   = string
    name = string
  })

	null 	no
log_analytics_workspace_enabled 	Enable the integration of azurerm_log_analytics_workspace and azurerm_log_analytics_solution: https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard 	bool 	true 	no
log_analytics_workspace_resource_group_name 	(Optional) Resource group name to create azurerm_log_analytics_solution. 	string 	null 	no
log_analytics_workspace_sku 	The SKU (pricing level) of the Log Analytics workspace. For new subscriptions the SKU should be set to PerGB2018 	string 	"PerGB2018" 	no
log_retention_in_days 	The retention period for the logs in days 	number 	30 	no
maintenance_window 	(Optional) Maintenance configuration of the managed cluster. 	

object({
    allowed = list(object({
      day   = string
      hours = set(number)
    })),
    not_allowed = list(object({
      end   = string
      start = string
    })),
  })

	null 	no
microsoft_defender_enabled 	(Optional) Is Microsoft Defender on the cluster enabled? Requires var.log_analytics_workspace_enabled to be true to set this variable to true. 	bool 	false 	no
net_profile_dns_service_ip 	(Optional) IP address within the Kubernetes service address range that will be used by cluster service discovery (kube-dns). Changing this forces a new resource to be created. 	string 	null 	no
net_profile_docker_bridge_cidr 	(Optional) IP address (in CIDR notation) used as the Docker bridge IP address on nodes. Changing this forces a new resource to be created. 	string 	null 	no
net_profile_outbound_type 	(Optional) The outbound (egress) routing method which should be used for this Kubernetes Cluster. Possible values are loadBalancer and userDefinedRouting. Defaults to loadBalancer. 	string 	"loadBalancer" 	no
net_profile_pod_cidr 	(Optional) The CIDR to use for pod IP addresses. This field can only be set when network_plugin is set to kubenet. Changing this forces a new resource to be created. 	string 	null 	no
net_profile_service_cidr 	(Optional) The Network Range used by the Kubernetes service. Changing this forces a new resource to be created. 	string 	null 	no
network_plugin 	Network plugin to use for networking. 	string 	"kubenet" 	no
network_policy 	(Optional) Sets up network policy to be used with Azure CNI. Network policy allows us to control the traffic flow between pods. Currently supported values are calico and azure. Changing this forces a new resource to be created. 	string 	null 	no
node_resource_group 	The auto-generated Resource Group which contains the resources for this Managed Kubernetes Cluster. Changing this forces a new resource to be created. 	string 	null 	no
oidc_issuer_enabled 	Enable or Disable the OIDC issuer URL. Defaults to false. 	bool 	false 	no
only_critical_addons_enabled 	(Optional) Enabling this option will taint default node pool with CriticalAddonsOnly=true:NoSchedule taint. Changing this forces a new resource to be created. 	bool 	null 	no
open_service_mesh_enabled 	Is Open Service Mesh enabled? For more details, please visit Open Service Mesh for AKS. 	bool 	null 	no
orchestrator_version 	Specify which Kubernetes release to use for the orchestration layer. The default used is the latest Kubernetes version available in the region 	string 	null 	no
os_disk_size_gb 	Disk size of nodes in GBs. 	number 	50 	no
os_disk_type 	The type of disk which should be used for the Operating System. Possible values are Ephemeral and Managed. Defaults to Managed. Changing this forces a new resource to be created. 	string 	"Managed" 	no
pod_subnet_id 	(Optional) The ID of the Subnet where the pods in the default Node Pool should exist. Changing this forces a new resource to be created. 	string 	null 	no
prefix 	(Required) The prefix for the resources created in the specified Azure Resource Group 	string 	n/a 	yes
private_cluster_enabled 	If true cluster API server will be exposed only on internal IP address and available only in cluster vnet. 	bool 	false 	no
private_cluster_public_fqdn_enabled 	(Optional) Specifies whether a Public FQDN for this Private Cluster should be added. Defaults to false. 	bool 	false 	no
private_dns_zone_id 	(Optional) Either the ID of Private DNS Zone which should be delegated to this Cluster, System to have AKS manage this or None. In case of None you will need to bring your own DNS server and set up resolving, otherwise cluster will have issues after provisioning. Changing this forces a new resource to be created. 	string 	null 	no
public_ssh_key 	A custom ssh key to control access to the AKS cluster. Changing this forces a new resource to be created. 	string 	"" 	no
rbac_aad 	(Optional) Is Azure Active Directory ingration enabled? 	bool 	true 	no
rbac_aad_admin_group_object_ids 	Object ID of groups with admin access. 	list(string) 	null 	no
rbac_aad_azure_rbac_enabled 	(Optional) Is Role Based Access Control based on Azure AD enabled? 	bool 	null 	no
rbac_aad_client_app_id 	The Client ID of an Azure Active Directory Application. 	string 	null 	no
rbac_aad_managed 	Is the Azure Active Directory integration Managed, meaning that Azure will create/manage the Service Principal used for integration. 	bool 	false 	no
rbac_aad_server_app_id 	The Server ID of an Azure Active Directory Application. 	string 	null 	no
rbac_aad_server_app_secret 	The Server Secret of an Azure Active Directory Application. 	string 	null 	no
rbac_aad_tenant_id 	(Optional) The Tenant ID used for Azure Active Directory Application. If this isn't specified the Tenant ID of the current Subscription is used. 	string 	null 	no
resource_group_name 	The resource group name to be imported 	string 	n/a 	yes
role_based_access_control_enabled 	Enable Role Based Access Control. 	bool 	false 	no
scale_down_mode 	(Optional) Specifies the autoscaling behaviour of the Kubernetes Cluster. If not specified, it defaults to Delete. Possible values include Delete and Deallocate. Changing this forces a new resource to be created. 	string 	"Delete" 	no
secret_rotation_enabled 	Is secret rotation enabled? This variable is only used when key_vault_secrets_provider_enabled is true and defaults to false 	bool 	false 	no
secret_rotation_interval 	The interval to poll for secret rotation. This attribute is only set when secret_rotation is true and defaults to 2m 	string 	"2m" 	no
sku_tier 	The SKU Tier that should be used for this Kubernetes Cluster. Possible values are Free and Paid 	string 	"Free" 	no
storage_profile_blob_driver_enabled 	(Optional) Is the Blob CSI driver enabled? Defaults to false 	bool 	false 	no
storage_profile_disk_driver_enabled 	(Optional) Is the Disk CSI driver enabled? Defaults to true 	bool 	true 	no
storage_profile_disk_driver_version 	(Optional) Disk CSI Driver version to be used. Possible values are v1 and v2. Defaults to v1. 	string 	"v1" 	no
storage_profile_enabled 	Enable storage profile 	bool 	false 	no
storage_profile_file_driver_enabled 	(Optional) Is the File CSI driver enabled? Defaults to true 	bool 	true 	no
storage_profile_snapshot_controller_enabled 	(Optional) Is the Snapshot Controller enabled? Defaults to true 	bool 	true 	no
tags 	Any tags that should be present on the AKS cluster resources 	map(string) 	{} 	no
ultra_ssd_enabled 	(Optional) Used to specify whether the UltraSSD is enabled in the Default Node Pool. Defaults to false. 	bool 	false 	no
vnet_subnet_id 	(Optional) The ID of a Subnet where the Kubernetes Node Pool should exist. Changing this forces a new resource to be created. 	string 	null 	no
workload_identity_enabled 	Enable or Disable Workload Identity. Defaults to false. 	bool 	false 	no
Outputs
Name 	Description
aci_connector_linux 	The aci_connector_linux block of azurerm_kubernetes_cluster resource.
aci_connector_linux_enabled 	Has aci_connector_linux been enabled on the azurerm_kubernetes_cluster resource?
admin_client_certificate 	The client_certificate in the azurerm_kubernetes_cluster's kube_admin_config block. Base64 encoded public certificate used by clients to authenticate to the Kubernetes cluster.
admin_client_key 	The client_key in the azurerm_kubernetes_cluster's kube_admin_config block. Base64 encoded private key used by clients to authenticate to the Kubernetes cluster.
admin_cluster_ca_certificate 	The cluster_ca_certificate in the azurerm_kubernetes_cluster's kube_admin_config block. Base64 encoded public CA certificate used as the root of trust for the Kubernetes cluster.
admin_host 	The host in the azurerm_kubernetes_cluster's kube_admin_config block. The Kubernetes cluster server host.
admin_password 	The password in the azurerm_kubernetes_cluster's kube_admin_config block. A password or token used to authenticate to the Kubernetes cluster.
admin_username 	The username in the azurerm_kubernetes_cluster's kube_admin_config block. A username used to authenticate to the Kubernetes cluster.
aks_id 	The azurerm_kubernetes_cluster's id.
aks_name 	The aurerm_kubernetes-cluster's name.
azure_policy_enabled 	The azurerm_kubernetes_cluster's azure_policy_enabled argument. Should the Azure Policy Add-On be enabled? For more details please visit Understand Azure Policy for Azure Kubernetes Service
azurerm_log_analytics_workspace_id 	The id of the created Log Analytics workspace
azurerm_log_analytics_workspace_name 	The name of the created Log Analytics workspace
azurerm_log_analytics_workspace_primary_shared_key 	Specifies the workspace key of the log analytics workspace
client_certificate 	The client_certificate in the azurerm_kubernetes_cluster's kube_config block. Base64 encoded public certificate used by clients to authenticate to the Kubernetes cluster.
client_key 	The client_key in the azurerm_kubernetes_cluster's kube_config block. Base64 encoded private key used by clients to authenticate to the Kubernetes cluster.
cluster_ca_certificate 	The cluster_ca_certificate in the azurerm_kubernetes_cluster's kube_config block. Base64 encoded public CA certificate used as the root of trust for the Kubernetes cluster.
cluster_fqdn 	The FQDN of the Azure Kubernetes Managed Cluster.
cluster_identity 	The azurerm_kubernetes_cluster's identity block.
cluster_portal_fqdn 	The FQDN for the Azure Portal resources when private link has been enabled, which is only resolvable inside the Virtual Network used by the Kubernetes Cluster.
cluster_private_fqdn 	The FQDN for the Kubernetes Cluster when private link has been enabled, which is only resolvable inside the Virtual Network used by the Kubernetes Cluster.
generated_cluster_private_ssh_key 	The cluster will use this generated private key as ssh key when var.public_ssh_key is empty or null. Private key data in PEM (RFC 1421) format.
generated_cluster_public_ssh_key 	The cluster will use this generated public key as ssh key when var.public_ssh_key is empty or null. The fingerprint of the public key data in OpenSSH MD5 hash format, e.g. aa:bb:cc:.... Only available if the selected private key format is compatible, similarly to public_key_openssh and the ECDSA P224 limitations.
host 	The host in the azurerm_kubernetes_cluster's kube_config block. The Kubernetes cluster server host.
http_application_routing_enabled 	The azurerm_kubernetes_cluster's http_application_routing_enabled argument. (Optional) Should HTTP Application Routing be enabled?
http_application_routing_zone_name 	The azurerm_kubernetes_cluster's http_application_routing_zone_name argument. The Zone Name of the HTTP Application Routing.
ingress_application_gateway 	The azurerm_kubernetes_cluster's ingress_application_gateway block.
ingress_application_gateway_enabled 	Has the azurerm_kubernetes_cluster turned on ingress_application_gateway block?
key_vault_secrets_provider 	The azurerm_kubernetes_cluster's key_vault_secrets_provider block.
key_vault_secrets_provider_enabled 	Has the azurerm_kubernetes_cluster turned on key_vault_secrets_provider block?
kube_admin_config_raw 	The azurerm_kubernetes_cluster's kube_admin_config_raw argument. Raw Kubernetes config for the admin account to be used by kubectl and other compatible tools. This is only available when Role Based Access Control with Azure Active Directory is enabled and local accounts enabled.
kube_config_raw 	The azurerm_kubernetes_cluster's kube_config_raw argument. Raw Kubernetes config to be used by kubectl and other compatible tools.
kubelet_identity 	The azurerm_kubernetes_cluster's kubelet_identity block.
location 	The azurerm_kubernetes_cluster's location argument. (Required) The location where the Managed Kubernetes Cluster should be created.
node_resource_group 	The auto-generated Resource Group which contains the resources for this Managed Kubernetes Cluster.
oidc_issuer_url 	The OIDC issuer URL that is associated with the cluster.
oms_agent 	The azurerm_kubernetes_cluster's oms_agent argument.
oms_agent_enabled 	Has the azurerm_kubernetes_cluster turned on oms_agent block?
open_service_mesh_enabled 	(Optional) Is Open Service Mesh enabled? For more details, please visit Open Service Mesh for AKS.
password 	The password in the azurerm_kubernetes_cluster's kube_config block. A password or token used to authenticate to the Kubernetes cluster.
username 	The username in the azurerm_kubernetes_cluster's kube_config block. A username used to authenticate to the Kubernetes cluster.