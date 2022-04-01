# Cluster Setup

The first foundational networking component, the regional hub, [has been deployed](./05-networking-hub.md). Before we dive into the cluster spoke and cluster itself, we need to take a quick detour to plan and talk about cluster control plane access.

### A. Create the AKS Jump Box Image


You are going to be using Azure Image Builder to generate a Kubernetes-specific jump box image. The image construction will be performed in a dedicated network spoke with limited Internet exposure. These steps below will deploy a new dedicated image-building spoke, connected through our hub to sequester network traffic throughout the process. It will then deploy an image template and all infrastructure components for Azure Image Builder to operate. Finally you will build an image to use for your jump box.

* The network spoke will be called `vnet-spoke-bu0001a0005-00` and have a range of `10.241.0.0/28`.
* The hub's firewall will be updated to allow only the necessary outbound traffic from this spoke to complete the operation.
* The final image will be placed into the workload's resource group.

#### Steps

##### Deploy the spoke

1. Create the AKS jump box image builder network spoke.

   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-region.v0 --query properties.outputs.hubVnetId.value -o tsv)

   # [This takes about one minute to run.]
   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0005-00.json -p location=eastus2 hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

1. Update the regional hub deployment to account for the requirements of the spoke.

   Now that the first spoke network is created, the hub network's firewall needs to be updated to support the Azure Image Builder process that will execute in there. The hub firewall does NOT have any default permissive egress rules, and as such, each needed egress endpoint needs to be specifically allowed. This deployment builds on the prior with the added allowances in the firewall.

   > :eyes: If you're curious to see what changed in the regional hub, [view the diff](https://diffviewer.azureedge.net/?l=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/networking/hub-region.v0.json&r=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/networking/hub-region.v1.json).

   ```bash
   RESOURCEID_SUBNET_AIB=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-00 --query properties.outputs.imageBuilderSubnetResourceId.value -o tsv)

   # [This takes about five minutes to run.]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v1.json -p location=eastus2 aksImageBuilderSubnetResourceId="${RESOURCEID_SUBNET_AIB}"
   ```

##### Build and deploy the jump box image

Now that we have our image building network created, egressing through our hub, and all NSG/firewall rules applied, it's time to build and deploy our jump box image. We are using the general purpose AKS jump box image as described in the [AKS Jump Box Image Builder repository](https://github.com/mspnp/aks-jumpbox-imagebuilder); which comes with baked-in tooling such as Azure CLI, kubectl, helm, flux, etc.. The network rules applied in the prior steps support its specific build-time requirements. If you use this infrastructure to build a modified version of this image template, you may need to add additional network allowances or remove unneeded allowances.

1. Download the ARM templates from the AKS Jump Box Image Builder repository.

   Ideally core templates like this would be part of your private Bicep registry. For this walk through, we are simply downloading the remote Bicep templates locally for execution.

   ```bash
   wget -B https://raw.githubusercontent.com/mspnp/aks-jumpbox-imagebuilder/main/ -x -nH --cut-dirs=3 -i jumpbox/jumpbox-bicep.txt -P jumpbox
   ```

1. Deploy custom Azure RBAC roles. _Optional._

   Azure Image Builder requires permissions to be granted to its runtime identity. The following deploys two _custom_ Azure RBAC roles that encapsulate those exact permissions necessary. If you do not have permissions to create Azure RBAC roles in your subscription, you can skip this step. However, in the next step below, you'll then be required to apply existing built-in Azure RBAC roles to the service's identity, which are more permissive than necessary, but would be fine to use for this walkthrough.

   ```bash
   # [This takes about one minute to run.]
   az deployment sub create -f jumpbox/createsubscriptionroles.bicep -l centralus -n DeployAibRbacRoles
   ```

1. Create the AKS jump box image template. (🛑 _if not using the custom roles created above._)

   Next you are going to deploy the image template and Azure Image Builders's managed identity. This is being done directly into our workload resource group for simplicity. You can choose to deploy this to a separate resource group if you wish. This "golden image" generation process would typically happen out-of-band to the cluster management.

   ```bash
   #ROLEID_NETWORKING=4d97b98b-1d4f-4787-a291-c67834d212e7 # Network Contributor -- Only use this if you did not, or could not, create custom roles. This is more permission than necessary.)
   ROLEID_NETWORKING=$(az deployment sub show -n DeployAibRbacRoles --query 'properties.outputs.roleResourceIds.value.customImageBuilderNetworkingRole.guid' -o tsv)
   #ROLEID_IMGDEPLOY=b24988ac-6180-42a0-ab88-20f7382dd24c  # Contributor -- only use this if you did not, or could not, create custom roles. This is more permission than necessary.)
   ROLEID_IMGDEPLOY=$(az deployment sub show -n DeployAibRbacRoles --query 'properties.outputs.roleResourceIds.value.customImageBuilderImageCreationRole.guid' -o tsv)

   # [This takes about one minute to run.]
   az deployment group create -g rg-bu0001a0005 -f jumpbox/azuredeploy.bicep -p buildInSubnetResourceId=${RESOURCEID_SUBNET_AIB} location=eastus2 imageBuilderNetworkingRoleGuid="${ROLEID_NETWORKING}" imageBuilderImageCreationRoleGuid="${ROLEID_IMGDEPLOY}" -n CreateJumpBoxImageTemplate
   ```

1. Build the general-purpose AKS jump box image.

   Now you'll build the actual VM golden image you will use for your jump box. This uses the image template created in the prior step and is executed by Azure Image Builder under the authority of the managed identity (and its role assignments) also created in the prior step.

   ```bash
   IMAGE_TEMPLATE_NAME=$(az deployment group show -g rg-bu0001a0005 -n CreateJumpBoxImageTemplate --query 'properties.outputs.imageTemplateName.value' -o tsv)

   # [This takes about >> 30 minutes << to run.]
   az image builder run -n $IMAGE_TEMPLATE_NAME -g rg-bu0001a0005
   ```

   > A successful run of the command above is typically shown with no output or a success message. An error state will be typically be presented if there was an error. To see if your image was built successfully, you can go to the **rg-bu0001a0005** resource group in the portal and look for a created VM Image resource. It will have the same name as the Image Template resource created in Step 2.

### B. Configuring Jump Box Users

Following the steps below, you'll end up with a SSH public-key-based solution that leverages [cloud-init](https://docs.microsoft.com/azure/virtual-machines/linux/using-cloud-init). The results will be captured in `jumpBoxCloudInit.yml` which you will later convert to Base64 for use in your cluster's ARM template.

#### Steps

1. Open `jumpBoxCloudInit.yml` in your preferred editor.
1. Add/remove/modify users following the two examples in that file. You need **one** user defined in this file to complete this walk through (_more than one user is fine_, but not necessary). 🛑
   1. `name:` set to whatever you login account name you wish. (You'll need to remember this later.)
   1. `sudo:` - Suggested to leave at `False`. This means the user cannot `sudo`. If this user needs sudo access, use [sudo rule strings](https://cloudinit.readthedocs.io/en/latest/topics/examples.html?highlight=sudo#including-users-and-groups) to restrict what sudo access is allowed.
   1. `lock_passwd:` - Leave at `True`. This disables password login, and as such the user can only connect via an SSH authorized key. Your jump box should enforce this as well on its ssh daemon. If you deployed using the image builder in the prior step, it does this enforcement there as well.
   1. In `ssh-authorized-keys` replace the example public key for the user. This must be an RSA key of at least 2048 bits and **must be secured with a passphrase**. This key will be added to that user's `~/.ssh/authorized_keys` file on the jump box via the cloud-init bootstrap process. If you need to generate a key pair you can execute this command:

      ```bash
      ssh-keygen -t rsa -b 4096 -f opsuser01.key
      cat opsuser01.key.pub
      
        **Enter a passphrase when requested** (_do not leave empty_) and note where the public and private key file was saved. The _public_ key file _contents_ (`opsuser01.key.pub` in the example above) is what is added to the `ssh-authorized-keys` array in `jumpBoxCloudInit.yml`. You'll need the username, the private key file (`opsuser01.key`), and passphrase later in this walkthrough.
      ```

1. Save the `jumpBoxCloudInit.yml` file. You _cannot_ use the provided example keys in this file as you do not have the private key to go with them, **you must update this file following the instructions above or you will not be able to complete this walkthrough.**
1. You can commit this file change if you wish, as the only values in here are public keys, which are not secrets. **Never commit any private SSH keys.**

### C. Deploy the Cluster Spoke

Your `rg-enterprise-networking-spokes` will be populated with the dedicated regional spoke network in which your cluster (and its direct adjacent resources will be connected to). This spoke will have limited Internet exposure and will support Network Security Groups (NSGs) at various levels to further limit network traffic as necessary.

* The network spoke will be called `vnet-spoke-bu0001a0005-01` and have a range of `10.240.0.0/16`.
* The spoke is broken into multiple subnets, each with a clearly defined purpose, appropriate IP range, and maximally restrictive NSG.
* DNS will be forwarded to the hub to support firewall inspection/logging and to support more complex network considerations such as DNS forwarders to your organization's DNS servers.
* The hub's firewall will be updated to allow only the necessary outbound traffic from this spoke's specific resource, management, and workload needs.

#### Steps

1. Deploy the cluster spoke.

   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-region.v0 --query properties.outputs.hubVnetId.value -o tsv)

   # [This takes about five minutes to run.]
   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0005-01.json -p location=eastus2 hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

1. Update the regional hub deployment to account for the runtime requirements of the virtual network.

   This is an evolution of same hub template you used before, but now updated with Azure Firewall rules specific to this AKS cluster infrastructure.

   > :eyes: If you're curious to see what changed in the regional hub, [view the diff](https://diffviewer.azureedge.net/?l=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/networking/hub-region.v1.json&r=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/networking/hub-region.v2.json).

   ```bash
   RESOURCEID_SUBNET_AIB=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-00 --query properties.outputs.imageBuilderSubnetResourceId.value -o tsv)
   RESOURCEID_SUBNET_NODEPOOLS="['$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-01 --query "properties.outputs.nodepoolSubnetResourceIds.value | join ('\',\'',@)" -o tsv)']"
   RESOURCEID_SUBNET_JUMPBOX=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-01 --query properties.outputs.jumpboxSubnetResourceId.value -o tsv)

   # [This takes about seven minutes to run.]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v2.json -p location=eastus2 aksImageBuilderSubnetResourceId="${RESOURCEID_SUBNET_AIB}" nodepoolSubnetResourceIds="${RESOURCEID_SUBNET_NODEPOOLS}" aksJumpboxSubnetResourceId="${RESOURCEID_SUBNET_JUMPBOX}"
   ```
### D. Deploy the Cluster
  
  * The cluster and all adjacent resources are deployed.
  * This includes core infrastructure such as Azure Key Vault, Azure Container Registry, and Azure Application Gateway.
  * Private Link configuration
  * Jump box (Azure Bastion) access
* A wildcard TLS certificate (`*.aks-ingress.contoso.com`) is imported into Azure Key Vault that will be used by your workload's ingress controller to expose an HTTPS endpoint to Azure Application Gateway.
* A Pod Managed Identity (`podmi-ingress-controller`) is deployed to the `ingress-nginx` namespace and ready to be bound via the name `podmi-ingress-controller`.
  * The same managed identity is granted the ability to pull the ingress controller's own TLS certificate from Key Vault.

#### Steps

1. Get the already-deployed, virtual network resource ID that this cluster will be attached to.

   ```bash
   RESOURCEID_VNET_CLUSTERSPOKE=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-01 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   ```

1. Identify your jump box image.

   ```bash
   # If you used a pre-existing image and not the one built by this walk through, replace the command below with the resource id of that image.
   RESOURCEID_IMAGE_JUMPBOX=$(az deployment group show -g rg-bu0001a0005 -n CreateJumpBoxImageTemplate --query 'properties.outputs.distributedImageResourceId.value' -o tsv)
   ```

1. Convert your jump box cloud-init (users) file to Base64.

   ```bash
   CLOUDINIT_BASE64=$(base64 jumpBoxCloudInit.yml | tr -d '\n')
   ```

   If you need to perform this in Powershell, you can achieve the same with this.

   ```powershell
   [Convert]::ToBase64String([IO.File]::ReadAllBytes('jumpBoxCloudInit.yml'))
   ```

1. Deploy the cluster ARM template.

   > _Alteratively 🛑_, you could set these values in [`azuredeploy.parameters.prod.json`](../../azuredeploy.parameters.prod.json) file instead of the individual key-value pairs shown below. You'll be redeploying a slight evolution of this template a later time in this walkthrough, and you might find it easier to have these variables captured in the parameters file as they will not change for the second deployment.

   ```bash
   # [This takes about 20 minutes to run.]
   az deployment group create -g rg-bu0001a0005 -f cluster-stamp.json -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE} clusterAdminAadGroupObjectId=${AADOBJECTID_GROUP_CLUSTERADMIN} k8sControlPlaneAuthorizationTenantId=${TENANTID_K8SRBAC} appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_BASE64} aksIngressControllerCertificate=${INGRESS_CONTROLLER_CERTIFICATE_BASE64} jumpBoxImageResourceId=${RESOURCEID_IMAGE_JUMPBOX} jumpBoxCloudInitAsBase64=${CLOUDINIT_BASE64}

   # Or if you updated and wish to use the parameters file …
   #az deployment group create -g rg-bu0001a0005 -f cluster-stamp.json -p "@azuredeploy.parameters.prod.json"
   ```

1. Update cluster deployment with managed identity assignments.

   **cluster-stamp.v2.json** is a _tiny_ evolution of the **cluster-stamp.json** ARM template you literally just deployed in the step above. Because we are using Azure AD Pod Identity v1 as a Microsoft-managed add-on, the mechanism to associate identities with the cluster is via ARM template instead of via Kubernetes manifest deployments (as you would do with the vanilla open source solution). However, due to a current limitation of the add-on, managed identities for Pod Managed Identities CANNOT be associated to the cluster when the cluster is first being created, only as an update to an existing cluster. So this deployment will re-deploy with the Pod Managed Identity association as the _only change_. Pod Managed Identity v2 is in development and it will support assignment at cluster-creation time. This implementation will evolve to use Azure AD Pod Identity v2 when available and we'll remove this step and add the assignment directly in `cluster-stamp.json`.

   > :eyes: If you're curious to see what changed in the cluster stamp, [view the diff](https://diffviewer.azureedge.net/?l=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/cluster-stamp.json&r=https://raw.githubusercontent.com/mspnp/aks-baseline-regulated/main/cluster-stamp.v2.json).

   ```bash
   # [This takes about five minutes to run.]
   az deployment group create -g rg-bu0001a0005 -f cluster-stamp.v2.json -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE} clusterAdminAadGroupObjectId=${AADOBJECTID_GROUP_CLUSTERADMIN} k8sControlPlaneAuthorizationTenantId=${TENANTID_K8SRBAC} appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_BASE64} aksIngressControllerCertificate=${AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64} jumpBoxImageResourceId=${RESOURCEID_IMAGE_JUMPBOX} jumpBoxCloudInitAsBase64=${CLOUDINIT_BASE64}

   # Or if you used the parameters file …
   #az deployment group create -g rg-bu0001a0005 -f cluster-stamp.v2.json -p "@azuredeploy.parameters.prod.json"
   ```

#### Import the wildcard certificate for the AKS ingress controller to Azure Key Vault

Once web traffic hits Azure Application Gateway, public-facing TLS is terminated. This supports WAF inspection rules and other request manipulation features of Azure Application Gateway. The next hop for this traffic is to the internal layer 4 load balancer and then to the in-cluster ingress controller. Starting at Application Gateway, all subsequent network hops are done via your private virtual network and is no longer traversing any public networks. That said, we still desire to provide TLS as an added layer of protection when traversing between Azure Application Gateway and our ingress controller. That'll bring TLS encryption _into_ your cluster from Application Gateway.

#### Steps

1. Give your user temporary permissions to import certificates into Key Vault.

   ```bash
   KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0005 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   az keyvault set-policy --certificate-permissions import --upn $(az account show --query user.name -o tsv) -n $KEYVAULT_NAME
   ```

1. Import the AKS ingress controller's certificate.

   You currently cannot import certificates into Key Vault directly via ARM templates. As such, post deployment of our Azure resources (which includes Key Vault), you need to upload your ingress controller's wildcard certificate to Key Vault. This is the `.pem` file you created on a prior page. Your ingress controller will authenticate to Key Vault (via the Pod Managed Identity created above) and use this certificate as its default TLS certificate, presenting exclusively to your Azure Application Gateway.

   ```bash
   az keyvault certificate import -f ingress-internal-aks-ingress-contoso-com-tls.pem -n ingress-internal-aks-ingress-contoso-com-tls --vault-name $KEYVAULT_NAME
   ```

1. Remove the temporary import certificates permissions for current user.

   > The Azure Key Vault Policy for your user was a temporary policy to allow you to import the certificate for this walkthrough. In actual deployments, you would manage access policies like these via your ARM templates using [Azure RBAC for Key Vault data plane](https://docs.microsoft.com/azure/key-vault/general/secure-your-key-vault#data-plane-and-access-policies).

   ```bash
   az keyvault delete-policy --upn $(az account show --query user.name -o tsv) -n $KEYVAULT_NAME
   ```


### Next step

:arrow_forward: [Prepare to bootstrap the cluster](./10-pre-bootstrap.md)
