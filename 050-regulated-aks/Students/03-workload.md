# Deploy Application

## Place the Cluster Under GitOps Management

GITOPS_REPOURL=$(git config --get remote.origin.url)

### Flux is configured and deployed

Your GitHub repo will be the source of truth for your cluster's configuration. Typically this would be a private repo, but for ease of demonstration, it'll be connected to a public repo (all firewall permissions are set to allow this specific interaction.) You'll be updating a configuration resource for Flux so that it knows to point to _your own repo_.

#### Steps

1. Update kustomization files to use images from your container registry. For this look-up the ACR created in the Resource Group **rg-bu0001a0005** and assign it to the variable **ACR_NAME**.

   ```bash
   # Replace the text YOUR_ACR_NAME with the name of your ACR created in the Resource Group rg-bu0001a0005
   ACR_NAME=YOUR-ACR-NAME
   cd cluster-manifests
   sed -i "s/REPLACE_ME_WITH_YOUR_ACRNAME/${ACR_NAME}/g" */kustomization.yaml

   git commit -a -m "Update bootstrap deployments to use images from my ACR instead of public container registries."
   ```

1. Update Flux to pull from your repo instead of the mspnp repo.

   ```bash
   sed -i "s/REPLACE_ME_WITH_YOUR_GITHUBACCOUNTNAME/${GITHUB_ACCOUNT_NAME}/" flux-system/gotk-sync.yaml

   git commit -a -m "Update Flux to pull from my fork instead of the upstream Microsoft repo."
   ```

1. Update Key Vault placeholders in your CSI Secret Store provider.

   You'll be using the [Secrets Store CSI Driver for Kubernetes](https://docs.microsoft.com/azure/aks/csi-secrets-store-driver) to mount the ingress controller's certificate which you stored in Azure Key Vault. Once mounted, your ingress controller will be able to use it. To make the CSI Provider aware of this certificate, it must be described in a `SecretProviderClass` resource. You'll update the supplied manifest file with this information now.

   ```bash
   KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0005 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)

   sed -i -e "s/KEYVAULT_NAME/${KEYVAULT_NAME}/" -e "s/KEYVAULT_TENANT/${TENANTID_AZURERBAC}/" ingress-nginx/akv-tls-provider.yaml

   git commit -a -m "Update SecretProviderClass to reference my ingress wildcard certificate."
   ```

1. Push those three commits to your repo.

   ```bash
   git push
   cd ..
   ```

1. Connect to a jump box node via Azure Bastion.

   If this is the first time you've used Azure Bastion, here is a detailed walk through of this process.

   1. Open the [Azure Portal](https://portal.azure.com).
   1. Navigate to the **rg-bu0001a0005** resource group.
   1. Click on the Virtual Machine Scale Set resource named **vmss-jumpboxes**.
   1. Click **Instances**.
   1. Click the name of any of the two listed instances. E.g. **vmss-jumpboxes_0**
   1. Click **Connect** -> **Bastion** -> **Use Bastion**.
   1. Fill in the username field with one of the users from your customized `jumpBoxCloudInit.yml` file. E.g. **opsuser01**
   1. Select **SSH Private Key from Local File** and select your private key file (e.g. `opsuser01.key`) for that specific user.
   1. Provide your SSH passphrase in **SSH Passphrase** if your private key is protected with one.
   1. Click **Connect**.
   1. For enhanced "copy-on-select" & "paste-on-right-click" support, your browser may request your permission to support those features. It's recommended that you _Allow_ that feature. If you don't, you'll have to use the **>>** flyout on the screen to perform copy & paste actions.
   1. Welcome to your jump box!


   > :warning: The jump box deployed in this walkthrough has only ephemeral disks attached, in which content written to disk will not survive planned or unplanned restarts of the host. Never store anything of value on these jump boxes. They are expected to be fully ephemeral in nature, and in fact could be scaled-to-zero when not in use.

1. _From your Azure Bastion connection_, log into your Azure RBAC tenant and select your subscription.

   The following command will perform a device login. Ensure you're logging in with the Azure AD user that has access to your AKS resources (i.e. the one you did your deployment with.)

   ```bash
   az login
   # This will give you a link to https://microsoft.com/devicelogin where you can enter 
   # the provided code and perform authentication.

   # Ensure you're on the correct subscription
   az account show

   # If not, select the correct subscription
   # az account set -s <subscription name or id>
   ```

   > :warning: Your organization may have a conditional access policies in place that forbids access to Azure resources [from non corporate-managed devices](https://docs.microsoft.com/azure/active-directory/conditional-access/require-managed-devices). This jump box as deployed in these steps might trigger that policy. If that is the case, you'll need to work with your IT Security organization to provide an alterative access mechanism or temporary solution.

1. _From your Azure Bastion connection_, get your AKS credentials and set your `kubectl` context to your cluster.

   ```bash
   AKS_CLUSTER_NAME=$(az deployment group show -g rg-bu0001a0005 -n cluster-stamp --query properties.outputs.aksClusterName.value -o tsv)

   az aks get-credentials -g rg-bu0001a0005 -n $AKS_CLUSTER_NAME
   ```

1. _From your Azure Bastion connection_, test cluster access and authenticate as a cluster admin user.

   The following command will force you to authenticate into your AKS cluster's control plane. This will start yet another device login flow. For this one (**Azure Kubernetes Service AAD Client**), log in with a user that is a member of your cluster admin group in the Azure AD tenet you selected to be used for Kubernetes Cluster API RBAC. Also this is where any specified Azure AD conditional access policies would take effect if they had been applied, and ideally you would have first used PIM JIT access to be assigned to the admin group. Remember, the identity you log in here with is the identity you're performing cluster control plane (Cluster API) management commands (e.g. `kubectl`) as.

   ```bash
   kubectl get nodes
   ```

   If all is successful you should see something like:

   ```output
   NAME                                  STATUS   ROLES   AGE   VERSION
   aks-npinscope01-26621167-vmss000000   Ready    agent   20m   v1.22.x
   aks-npinscope01-26621167-vmss000001   Ready    agent   20m   v1.22.x
   aks-npooscope01-26621167-vmss000000   Ready    agent   20m   v1.22.x
   aks-npooscope01-26621167-vmss000001   Ready    agent   20m   v1.22.x
   aks-npsystem-26621167-vmss000000      Ready    agent   20m   v1.22.x
   aks-npsystem-26621167-vmss000001      Ready    agent   20m   v1.22.x
   aks-npsystem-26621167-vmss000002      Ready    agent   20m   v1.22.x
   ```

   > :watch: The access tokens obtained in the prior two steps are subject to a Microsoft Identity Platform TTL (e.g. six hours). If your `az` or `kubectl` commands start erroring out after hours of usage with a message related to permission/authorization, you'll need to re-execute the `az login` and `az aks get-credentials` (overwriting your context) to refresh those tokens.

1. _From your Azure Bastion connection_, confirm admission policies are applied to the AKS cluster.

   Azure Policy was configured in the cluster deployment with a set of starter policies. Your cluster pods will be covered using the [Azure Policy add-on for AKS](https://docs.microsoft.com/azure/aks/use-pod-security-on-azure-policy). Some of these policies might end up in the denial of a specific Kubernetes API request operation to ensure the pod's specification is compliance with your organization's security best practices. Moreover [data is generated by Azure Policy](https://docs.microsoft.com/azure/governance/policy/how-to/get-compliance-data) to assist the app team in the process of assessing the current compliance state of the AKS cluster. You've assign the [Azure Policy for Kubernetes built-in restricted initiative](https://docs.microsoft.com/azure/aks/use-pod-security-on-azure-policy#built-in-policy-initiatives) as well as ten more [built-in individual Azure policies](https://docs.microsoft.com/azure/aks/policy-samples#microsoftcontainerservice) that enforce that pods perform resource requests, define trusted container registries, allow root filesystem access in read-only mode, enforce the usage of internal load balancers, and enforce https-only Kuberentes Ingress objects.

   ```bash
   kubectl get ConstraintTemplate
   ```

   A similar output as the one showed below should be returned

   ```output
   NAME                                     AGE
   k8sazureallowedcapabilities              21m
   k8sazureallowedseccomp                   21m
   … more …
   k8sazureserviceallowedports              21m
   k8sazurevolumetypes                      21m
   ```

1. _From your Azure Bastion connection_, confirm your ingress controller's Pod Managed Identity exists.

   ```bash
   kubectl describe AzureIdentity,AzureIdentityBinding -n ingress-nginx
   ```

   This will show you the AzureIdentity Kubernetes resources that were created via the cluster's ARM template. This means that any workload in the `ingress-nginx` namespace that wishes to identify itself as the Azure resource `podmi-ingress-controller` can do so by adding a `aadpodidbinding: podmi-ingress-controller` label to their pod deployment. In this walkthrough, our ingress controller will be using that identity. This identity will be used with the Secret Store driver for Key Vault to reference your wildcard ingress TLS certificate.

1. _From your Azure Bastion connection_, bootstrap Flux. 🛑

   ```bash
   GITHUB_ACCOUNT_NAME=YOUR-GITHUB-ACCOUNT-NAME-GOES-HERE

   git clone https://github.com/$GITHUB_ACCOUNT_NAME/aks-baseline-regulated.git
   cd aks-baseline-regulated/cluster-manifests

   # Apply the Flux CRDs before applying the rest of flux (to avoid a race condition in the sync settings)
   kubectl apply -f flux-system/gotk-crds.yaml
   kubectl apply -k flux-system
   ```

   > The Flux CLI does have a built-in bootstrap feature. To ensure everyone using this walkthrough has a consistent experience (not one based on what version of Flux cli they may have installed), we've performed that bootstrap process more "manually" above via pre-generated manifest files. Also, you might consider doing the same with your production clusters; as all manifests you apply should be well-known, intentional, and auditable. Doing so will eliminate any guess work from what is or is not being deployed and leads to ultimate repeatability. It does mean that you'll manually be managing some manifests that could otherwise be managed in a more opaque way, and we suggest getting comfortable with that notion. You'll see this play out not just in Flux, but Falco, Kured, etc. Ensure you understand the risks associated with (and _benefits_ gained from) whatever _convenance_ solutions you bring to your cluster (Helm, cli "installer" commands, etc.) and you're comfortable with that layer of indirection bring introduced.

   Validate that Flux has been bootstrapped.

   ```bash
   kubectl wait --namespace flux-system --for=condition=available deployment/source-controller --timeout=90s

   # If you have flux cli installed you can also inspect using the following commands
   # (The default jump box image created with this walkthrough has the flux cli installed.)
   flux check --components source-controller,kustomize-controller
   ```

## Deploy the workload

The next few steps will walk through considerations that are specific to the first workload in the cluster. Workloads are a mix of potential infrastructure changes (e.g. Azure Application Gateway routes, Azure resources for the workload itself -- such as CosmosDB for state storage and Azure Cache for Redis for cache.), privileged cluster changes (i.e. creating target namespace, creating and assigning any specific cluster or namespace roles, etc.), deciding on how that "last mile" deployment of these workloads will be handled (e.g. using the `snet-management-agents` subnet adjacent to this cluster), and workload teams which are responsible for creating the container image(s), building deployment manifests, etc. Many regulations have a clear separation of duties requirements, be sure in your case you have documented and understood change management process. How you partition this work will not be described here because there isn't a one-size-fits-most solution. Allocate time to plan, document, and educate on these concerns.

#### Steps

1. Use your Azure Container Registry build agents to build and quarantine the workload.

   ```bash
   # [This takes about three minutes to run.]
   az acr build -t quarantine/a0005/chain-api:1.0 -r $ACR_NAME_QUARANTINE --platform linux/amd64 --agent-pool acragent -f SimpleChainApi/Dockerfile https://github.com/mspnp/aks-endpoint-caller#main:SimpleChainApi
   ```

   You are using your own dedicated task agents here, in a dedicated subnet, for this process. Securing your workload pipeline components are critical to having a compliant solution. Ensure your build pipeline matches your desired security posture. Consider performing image building in an Azure Container Registry that is network-isolated from your clusters (unlike what we're showing here where it's within the same virtual network for simplicity.) Ensure build logs are captured. That build ACR instance might also serve as your quarantine instance as well. Once the build is complete and post-build audits are complete, then it can be imported to your "live" registry.

1. Release the workload image from quarantine.

   ```bash
   # [This takes about one minute to run.]
   az acr import --source quarantine/a0005/chain-api:1.0 -r $ACR_NAME_QUARANTINE -t live/a0005/chain-api:1.0 -n $ACR_NAME
   ```

1. Update workload ACR references in your kustomization files.

   ```bash
   cd workload
   sed -i "s/REPLACE_ME_WITH_YOUR_ACRNAME/${ACR_NAME}/g" */*/kustomization.yaml

   git commit -a -m "Update the four workload images to use my Azure Container Registry instance."
   ```

1. Push this change to your repo.

   ```bash
   git push
   ```

1. _From your Azure Bastion connection_, deploy the sample workloads to cluster.

   The sample workload will be deployed across two namespaces. An "in-scope" namespace (`a0005-i`) and an "out-of-scope" (`a0005-o`) namespace to represent a logical separation of components in this solution. The workloads that are in `a0005-i` are assumed to be directly or indirectly handling data that is in regulatory scope. The workloads that are in `a0005-o` are supporting workloads, but they themselves do not handle in-scope regulatory data. While this entire cluster is subject to being in regulatory scope, consider making it clear in your namespacing, labeling, etc. what services actively engage in the handling of critical data, vs those that are in a supportive role and should never handle or be able to handle that data. Ideally you'll want to minimize the workload in your in-scope clusters to just those workloads dealing with the data under regulatory compliance; _running non-scoped workloads in an alternate cluster_. Sometimes that isn't practical, therefor when you co-mingle the workloads, you need to treat almost everything as in scope, but that doesn't mean you can't treat the truly in-scope components with added segregation and care.

   In addition to namespaces, the cluster also has dedicated node pools for the "in-scope" components. This helps ensure that out-of-scope workload components (where possible), do not run on the same hardware as the in-scope components. Ideally your in-scope node pools will run just those workloads that deal with in-scope regulatory data and the security agents to support the your regulatory obligations. These two node pools benefit from being on separate subnets as well, which allows finer control as the Azure Network level (NSG rules and Azure Firewall rules).

   > :notebook: See [Azure Architecture Center guidance for PCI-DSS 3.2.1 Requirement 2.2.1 in AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-network#requirement-221).

   ```bash
   cd ../workload

   # Get the workload ACR endpoint changes you committed above.
   git pull

   # Deploy "in-scope" components.  These will live in the a0005-i namespace and will be
   # scheduled on the aks-npinscope01 node pool - dedicated to just those workloads.
   kubectl apply -k a0005-i/web-frontend
   kubectl apply -k a0005-i/microservice-c

   # Deploy "out-of-scope" components. These will live in the a0005-o namespace and will
   # be scheduled on the aks-npooscope01 node pool - used for all non in-scope components.
   kubectl apply -k a0005-o/microservice-a
   kubectl apply -k a0005-o/microservice-b
   ```

#### Workload networking

You are responsible to control the exposure of your in-cluster endpoints and to control what traffic is allowed to ingress and egress. **Kubernetes, by default, is a full-trust platform at the network level.** That means the default unit of isolation in Kubernetes is the cluster -- from a networking perspective. To achieve a zero-trust network environment, you'll need to apply in-cluster (and Azure network) constructs to build that foundation. This is done through the use of Kubernetes Network Policies and/or service mesh constructs, across all of your namespaces -- not just your workloads' namespace(s). Expect to invest time in documenting the exact network flows of your applications' components _and your baseline tooling_ to build out the in-cluster restrictions to model those expected network flows.

#### Network policies

The foundation of in-cluster network security is Kubernetes Network Policies. This cluster is deployed with Azure NPM (Azure Network Policy Manager) which enforces standard Kubernetes NetworkPolicy resources across your cluster. Kubernetes Network Policies are a Layer 3 and Layer 4 construct that allow you to define what traffic is allowed into and out of a pod. Network Policies are namespaced resources, and as such you need to manage them across your namespaces. This reference implementation deploys a default "deny-all" policy to establish immediate zero-trust in the workload namespaces. Then the workload overlays just the network traffic it needs to be functional and observable. While we only applied them to the workload namespaces, all namespaces that you control (ie. _not_ `kube-system`, `gatekeeper-system`, etc.), should have zero-trust policies applied. Time wasn't invested to do that in this walkthrough because your bootstrapped solutions are likely to be bespoke to your cluster and you'll need to apply the policies that make sense for them.

For a namespace in which services will be talking to other services, **the recommended zero-trust network policy for AKS can be found in [networkpolicy-denyall.yaml](cluster-manifests/a0005-i/networkpolicy-denyall.yaml)**. This blocks ALL traffic (in and out) other than outbound to kube-dns (which is CoreDNS in AKS). If you don't need DNS resolution across all workloads in the namespace, then you can remove that from the deny all and apply it selectively to pods that do require it (if any).

> :notebook: See [Azure Architecture Center guidance for PCI-DSS 3.2.1 Requirement 1.1.4 in AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-pci/aks-pci-network#requirement-114).

#### Service mesh

The workload (split across four components - `web-frontend`, `microservice-a`, `microservice-b`, and `microservice-c`) is deployed across two separate node pools and across two separate namespaces. But because this workload represents a set of connected microservices, this workload has joined the cluster's service mesh. Doing so provides the following benefits.

* Network access is removed by default from all access outside of the mesh.
* Network access is limited to defined HTTP routes, with explicitly defined sources and destinations.
* mTLS encryption between all components in the mesh.
* mTLS encryption between your ingress controller and the endpoint into the mesh.
* mTLS rotation happening every 24 hours.


### Next step

:arrow_forward: [End-to-End Validation](./13-validation.md)
