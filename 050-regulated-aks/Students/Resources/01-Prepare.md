
# Prerequisites
This is the starting point for the end-to-end instructions on deploying the [AKS Baseline for Regulated Workloads reference implementation](/README.md). There is required access and tooling you'll need in order to accomplish this. Follow the instructions below and on the subsequent pages so that you can get your environment and subscription ready to proceed with the AKS cluster creation.

### A. Azure AD Tenant
Azure AD [User Administrator](https://docs.microsoft.com/azure/active-directory/users-groups-roles/directory-assign-admin-roles#user-administrator-permissions) is _required_ to create a "break glass" AKS admin Active Directory Security Group and User. For this exercise, consider [creating a new tenant](https://docs.microsoft.com/azure/active-directory/fundamentals/active-directory-access-create-new-tenant#create-a-new-tenant-for-your-organization) to use while evaluating this implementation. 
Alternatively, you could get your Azure AD admin to create this for you when instructed to do so.

#### Azure AD tenant selection

AKS allows for a separation between Azure management control plane access control and Kubernetes control plane access control. This deployment process, creating and associating Azure resources with each other, is an example of Azure management control plane access. This is a relationship between your Azure AD tenant associated with your Azure subscription and is what grants you the permissions to create networks, clusters, managed identities, and create relationships between them. Kubernetes has it's own control plane, exposed via the Kubernetes Cluster API endpoint, and honors the Kubernetes RBAC authorization model. This endpoint is where `kubectl` commands are executed against, for example.

AKS allows for disparate Azure AD tenants between these two control planes; one tenant can be used for Azure management control plane and another for Kuberentes Cluster API authorization. You can also use the same tenant for both. Some regulated environments may prefer a clear tenant separation to address impact radius and potential lateral movement; at the _significant_ added complexity and cost of managing multiple identity stores. This reference implementation will work with either model. Most customers, even in regulated environments, use a single Azure AD tenant model with added features such as Conditional Access Policies. Ensure your final implementation is aligned with how your organization and compliance requirements dictate identity management.

#### AD Objects

Following the steps below will result in an Azure AD configuration that will be used for Kubernetes control plane (Cluster API) authorization.

| Object                         | Purpose                                                 |
|--------------------------------|---------------------------------------------------------|
| A Cluster Admin Security Group | Will be mapped to `cluster-admin` Kubernetes role.      |
| A Cluster Admin User           | Represents at least one break-glass cluster admin user. |
| Cluster Admin Group Membership | Association between the Cluster Admin User(s) and the Cluster Admin Security Group. Ideally there would be NO standing group membership associations made, but for the purposes of this material, you should have assigned the admin user(s) created above. |
| _Additional Security Groups_   | _Optional._ A security group (and its memberships) for the other built-in and custom Kubernetes roles you plan on using. |

#### Steps

1. Log in to the tenant where Kubernetes Cluster API authorization will be associated with. 🛑

   Capture the Azure AD Tenant ID that will be associated with your cluster's Kubernetes RBAC for Cluster API access. This is _typically_ the same tenant as your Azure RBAC, see [Azure AD tenant selection](#Azure-AD-tenant-selection) above for more details. However, if you do not have access to manage Azure AD groups and permissions, you may create a temporary tenant specifically for this walkthrough so that you're not blocked at this point.

   ```bash
   az login -t <Replace-With-ClusterApi-AzureAD-TenantId> --allow-no-subscriptions
   TENANTID_K8SRBAC=$(az account show --query tenantId -o tsv)
   ```

1. Create/identify the Azure AD security group that is going to map to the [Kubernetes Cluster Admin](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) role `cluster-admin`.

   ```bash
   AADOBJECTNAME_GROUP_CLUSTERADMIN=cluster-admins-bu0001a000500
   AADOBJECTID_GROUP_CLUSTERADMIN=$(az ad group create --display-name $AADOBJECTNAME_GROUP_CLUSTERADMIN --mail-nickname $AADOBJECTNAME_GROUP_CLUSTERADMIN --description "Principals in this group are cluster admins in the bu0001a000500 cluster." --query objectId -o tsv)
   ```

1. Create a "break-glass" cluster administrator user for your AKS cluster.

   This steps creates a dedicated account that you can use for cluster administrative access. This account should have no standing permissions on any Azure resources; a compromise of this account then cannot be parlayed into Azure management control plane access. If using the same tenant that your Azure resources are managed with, some organizations employ an alt-account strategy. In that case, your cluster admins' alt account(s) might satisfy this step.

   ```bash
   TENANTDOMAIN_K8SRBAC=$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d '@' -f 2 | sed 's/\"//')
   AADOBJECTNAME_USER_CLUSTERADMIN=bu0001a000500-admin
   AADOBJECTID_USER_CLUSTERADMIN=$(az ad user create --display-name=${AADOBJECTNAME_USER_CLUSTERADMIN} --user-principal-name ${AADOBJECTNAME_USER_CLUSTERADMIN}@${TENANTDOMAIN_K8SRBAC} --force-change-password-next-login --password ChangeMebu0001a0005AdminChangeMe --query objectId -o tsv)
   ```

1. Add the cluster admin user(s) to the cluster admin security group.

   ```bash
   az ad group member add -g $AADOBJECTID_GROUP_CLUSTERADMIN --member-id $AADOBJECTID_USER_CLUSTERADMIN
   ```

### B. Workload Identity
* Follow the instructions to [enable Azure AD Pod Identity](https://docs.microsoft.com/azure/aks/use-azure-ad-pod-identity#before-you-begin). You do not need to install the preview CLI extension or follow other instructions on this page.

#### Steps

   ```bash
   az feature register --namespace "Microsoft.ContainerService" -n "EnablePodIdentityPreview"

   # Keep running until all say "Registered." (This may take up to 20 minutes.)
   az feature list -o table --query "[?name=='Microsoft.ContainerService/EnablePodIdentityPreview'].{Name:name,State:properties.state}"

   # When all say "Registered" then re-register the AKS resource provider
   az provider register --namespace Microsoft.ContainerService
   ```
 ### C. Clone Repository
   Fork this repository and clone it locally. 🛑

   ```bash
   GITHUB_ACCOUNT_NAME=YOUR-GITHUB-ACCOUNT-NAME-GOES-HERE

   git clone https://github.com/${GITHUB_ACCOUNT_NAME}/aks-baseline-regulated.git
   cd aks-baseline-regulated
   ```
   
### D. TLS Certificates 

#### Steps

1. Ensure [OpenSSL is installed](https://github.com/openssl/openssl#download) in order to generate the example self-signed certs used in this implementation. _OpenSSL is already installed in Azure Cloud Shell._

1. To support end-to-end TLS encryption, the following TLS certificates are procured.

| Common Name                 | Purpose                                          | Notes |
|-----------------------------|--------------------------------------------------|-------|
| `bicycle.contoso.com`       | Attached to the public IP on the Application Gateway | This is client-facing for the endpoint your workload will respond at. Typically this will be an EV certificate generated by a public CA. |
| `*.aks-ingress.contoso.com` | Attached to the ingress controller in the cluster    | This is not client-facing and doesn't need to be procured by a public CA. This provides TLS encryption between Application Gateway and your ingress controller. |

:warning: Do not use the certificates created by these instructions for actual deployments. The use of self-signed certificates are provided for ease of illustration purposes only. For your cluster, use your organization's requirements for procurement and lifetime management of TLS certificates, _even for development purposes_.

> :notebook: See [Azure Architecture Center guidance for PCI-DSS 3.2.1 Requirement 3.6 in AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-data#requirement-36) and [TLS encryption architecture considerations](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-ra-code-assets#tls-encryption).

1. Create the certificate for Azure Application Gateway with a common name of `bicycle.contoso.com`.

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=bicycle.contoso.com/O=Contoso Bicycle" -addext "subjectAltName = DNS:bicycle.contoso.com" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"
   openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:
   ```

1. Base64 encode the client-facing certificate.

   :bulb: No matter if you used a certificate from your organization or you generated one from above, you'll need the certificate (as `.pfx`) to be Base64 encoded for proper storage in Key Vault later.

   ```bash
   APP_GATEWAY_LISTENER_CERTIFICATE_BASE64=$(cat appgw.pfx | base64 | tr -d '\n')
   ```

1. Generate the certificate for the ingress controller with a common name of `*.aks-ingress.contoso.com`.

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out ingress-internal-aks-ingress-contoso-com-tls.crt -keyout ingress-internal-aks-ingress-contoso-com-tls.key -subj "/CN=*.aks-ingress.contoso.com/O=Contoso AKS Ingress"

   # Combined as PEM structure (required by Azure Application Gateway for backend pools)
   cat ingress-internal-aks-ingress-contoso-com-tls.crt ingress-internal-aks-ingress-contoso-com-tls.key > ingress-internal-aks-ingress-contoso-com-tls.pem
   ```

1. Base64 encode the ingress controller certificate.

   :bulb: No matter if you used a certificate from your organization or you generated one from above, you'll need the public certificate (as `.crt` or `.cer`) to be Base64 encoded for proper storage in Key Vault later.

   ```bash
   INGRESS_CONTROLLER_CERTIFICATE_BASE64=$(cat ingress-internal-aks-ingress-contoso-com-tls.crt | base64 | tr -d '\n')
   ```

### E. Resource Groups

The following three resource groups will be created in the steps below.

| Name                            | Purpose                                   |
|---------------------------------|-------------------------------------------|
| rg-enterprise-networking-hubs   | Contains all of your organization's regional hubs. A regional hub resources in this implementation include an the hub Virtual Network, egress firewall, Azure Bastion, and Log Analytics for network logging. They may also contain your VPN Gateways, which are not addressed in this implementation. |
| rg-enterprise-networking-spokes | Contains all of your organization's regional spokes and related networking resources. All spokes will peer with their regional hub and subnets will egress through the regional firewall in the hub. |
| rg-bu0001a0005                  | Contains the regulated cluster resources. |
| networkWatcherRG                | Contains regional Network Watchers. _(Most subscriptions already have this.)_ |

Both Azure Kubernetes Service and Azure Image Builder Service use a concept of a dynamically-created _infrastructure_ resource group. So in addition to the four resource groups mentioned above, as you follow these instructions, you'll end up with six resource groups; two of which are automatically created and their lifecycle tied to their owning service. You will not see these two infrastructure resource groups get created until later in the walkthrough when their owning service is created.

### F. Azure Policies
To help govern our resources, there are policies we apply over the scope of these resource groups. These policies will also be created in the steps below.

| Policy Name                    | Scope                           | Purpose                                                                                           |
|--------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| Enable Microsoft Defender Standard | Subscription                | Ensures that Microsoft Defender for Containers, DNS, Key Vault, and Resource Manager are always enabled. |
| Allowed resource types         | rg-enterprise-networking-hubs   | Restricts the hub resource group to just relevant networking resources.                           |
| VNet must have Network Watcher | rg-enterprise-networking-hubs   | Audit policy that will trigger if a network is deployed to a region that doesn't have a Network Watcher. _(This is only created if your subscription doesn't already have Network Watchers in place.)_ |
| Allowed resource types         | rg-enterprise-networking-spokes | Restricts the spokes resource group to just relevant networking resources.                        |
| VNet must have Network Watcher | rg-enterprise-networking-spokes | Audit policy that will trigger if a network is deployed to a region that doesn't have a Network Watcher. _(This is only created if your subscription doesn't already have Network Watchers in place.)_ |
| Allowed resource types         | rg-bu0001a0005                  | Restricts the workload resource group to just resources necessary for this specific architecture. |
| Allowed resource types         | networkWatcherRG                | Restricts the Network Watcher resource group to just Network Watcher resources. _(Audit only mode to prevent conflict with any existing policy that manages this common resource group.)_ |
| No public AKS clusters         | rg-bu0001a0005                  | Restricts the creation of AKS clusters to only those with private Kubernetes API server.   |
| No out-of-date AKS clusters    | rg-bu0001a0005                  | Restricts the creation of AKS clusters to only recent versions.                            |
| No AKS clusters without RBAC   | rg-bu0001a0005                  | Restricts the creation of AKS clusters to only those that are Azure AD RBAC enabled.       |
| No AKS clusters without Azure Policy | rg-bu0001a0005            | Restricts the creation of AKS clusters to only those that have the Azure Policy Add-on enabled.   |
| No AKS clusters without BYOK OS & Data Disk Encryption | rg-bu0001a0005  | Restricts the creation of AKS clusters to only those that have customer-managed disk encryption enabled. (_This is in audit only mode, as not all customers may wish to do this._) |
| No AKS clusters without encryption-at-host | rg-bu0001a0005      | Restricts the creation of AKS clusters to only those that have the Encryption-At-Host feature enabled. (_This is in audit only mode, as not all customers may wish to do this._) |
| No AKS clusters without Microsoft Defender for Containers | rg-bu0001a0005                | Restricts the creation of AKS clusters to only those that have the Microsoft Defender for Containers feature enabled. |
| No App Gateways without WAF    | rg-bu0001a0005                  | Restricts the creation of Azure Application Gateway to only the WAF SKU. |
| No VMSS with public IPs        | rg-bu0001a0005                  | Only VMSS that do not have public IPs can be created in this resource group. |

#### Steps
Perform subscription-level deployment.

   This will deploy the resource groups, Azure Policies, and Microsoft Defender for Cloud configuration all as identified above.

   ```bash
   # [This may take up to six minutes to run.]
   az deployment sub create -f subscription.json -l centralus -p networkWatcherRGRegion="${NETWORK_WATCHER_RG_REGION}"
   ```

   If you do not have permissions on your subscription to enable Microsoft Defender (which requires the Azure RBAC role of _Subscription Owner_ or _Security Admin_), then instead execute the following variation of the same command. This will not enable Microsoft Defender services nor will Azure Policy attempt to enable the same (the policy will still be created, but in audit-only mode). Your final implementation should be to a subscription with these security services activated.

   ```bash
   # [This may take up to five minutes to run.]
   az deployment sub create -f subscription.json -l centralus -p enableAzureDefender=false enforceAzureDefenderAutoDeployPolicies=false networkWatcherRGRegion="${NETWORK_WATCHER_RG_REGION}"
   ```
#### Other Azure Policies

Consider evaluating additional Azure Policies to help guard your subscription from undesirable resource deployments. Here are some to consider.

* [PCI-DSS 3.2.1 Blueprint](https://docs.microsoft.com/azure/governance/blueprints/samples/pci-dss-3.2.1/)
* Allowed locations
* Allowed locations for resource groups
* External accounts with read permissions should be removed from your subscription
* External accounts with write permissions should be removed from your subscription
* External accounts with owner permissions should be removed from your subscription
* Network interfaces should not have public IPs

### G. Networking in this architecture

Egressing your spoke traffic through a hub network (following the hub-spoke model), is a critical component of this AKS architecture. Your organization's networking team will likely have a specific strategy already in place for this; such as a _Connectivity_ subscription already configured for regional egress. In this walkthrough, we are going to implement this recommended strategy in an illustrative manner, however you will need to adjust based on your specific situation when you implement this cluster for production. Hubs are usually a centrally-managed and governed resource in an organization, and not typically workload specific. The steps that follow create the hub (and spokes) as a stand-in for the work that you'd coordinate with your networking team.

> :notebook: See [Azure Architecture Center guidance for PCI-DSS 3.2.1 Requirement 1 in AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-network#requirement-1install-and-maintain-a-firewall-configuration-to-protect-cardholder-data) and [Networking configuration](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-ra-code-assets#networking-configuration).

#### IP addressing

* Regional Hubs are allocated to `10.200.[0-9].0` in this reference implementation. The `eastus2` hub (created below) will be `10.200.0.0/24`.
* Regional Spokes (created later) in this reference implementation are allocated to `10.240.0.0/16` and `10.241.0.0/28`.

Since this walkthrough is expected to be deployed isolated from existing infrastructure and not joined to any of your existing networks; these IP addresses should not come in conflict with any existing networking you have, even if those IP addresses overlap with ones you already have. However, if you need to join existing networks, even for the purposes this walkthrough, you'll need to adjust the IP space in the ARM templates as per your requirements as to not conflict.

#### Steps

1. Create the regional network hub.

   ```bash
   # [This takes about eight minutes to run.]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v0.json -p location=eastus2
   ```

   The hub deployment will output the following:

      * `hubVnetId` - which you'll query in future steps when creating connected regional spokes. E.g. `/subscriptions/[subscription id]/resourceGroups/rg-enterprise-networking-hubs/providers/Microsoft.Network/virtualNetworks/vnet-eastus2-hub`


### Next step

:arrow_forward: [Deploy the regional hub network](./05-networking-hub.md).
