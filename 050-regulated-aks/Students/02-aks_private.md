#Create the Cluster

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

Now lay out the next critical component, the cluster's spoke (virtual network).

