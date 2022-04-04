
# Validate and Monitor the Setup

Now that you have a [simulated regulated workload deployed](./03-workload.md), you can start validating and exploring this reference implementation of the [AKS Baseline Cluster for Regulated Workloads](/).

## Validate the workload is deployed

This section will help you to validate the workload is exposed correctly and responding to HTTPS requests.

### Steps

1. Get the public IP of Azure Application Gateway.

   ```bash
   APPGW_PUBLIC_IP=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0005-01 --query properties.outputs.appGwPublicIpAddress.value -o tsv)
   ```

1. Create a DNS `A` Record 🛑

   > :bulb: You can simulate this via a local hosts file modification.

   Map the Azure Application Gateway public IP address to the application domain name. To do that, please edit your hosts file (`C:\Windows\System32\drivers\etc\hosts` or `/etc/hosts`) and add the following record to the end: `${APPGW_PUBLIC_IP} bicycle.contoso.com`

1. Browse to the site (e.g. <https://bicycle.contoso.com>).

1. Review the emitted page details.

   This page shows a series of attempted network traffic attempts in your cluster. Due to Azure Firewall, NSGs, service mesh rules, and Kuberentes Network Policies, only a select number of requests will be successful, the rest will be blocked. Generally speaking, the allowed network flow is Ingress -> web-frontend -> microservice-a -> microservice-b -> microservice-c. All other attempted connections are denied. Likewise, most Internet-bound traffic was denied, and only microservice-a was able to make the public request.

1. Browse to the site with the following appended to the URL: `?sql=DELETE%20FROM` (e.g. <https://bicycle.contoso.com/?sql=DELETE%20FROM>).

1. Observe that your request was blocked by Application Gateway's WAF rules and your workload never saw this potentially dangerous request.

## Access resource logs & Microsoft Defender for Cloud data

You can access these logs all directly from the attached Log Analytics workspace(s), but when you do you'll need to filter to specific resources. For simplicity the steps below direct you to the pre-filtered view offered by the Azure Portal when viewing within the context of each service.

### View traffic blocked by Azure Firewall

Your Azure Firewall has already blocked traffic originating in the cluster headed to **https://example.org** as part of the workload. To see those denied requests, execute the following query:

```kusto
AzureDiagnostics
| where Category == "AzureFirewallApplicationRule"
| where msg_s contains "example.org:443"
| order by TimeGenerated desc
```

### View Azure Firewall DNS proxy logs

Your Azure Firewall is acting as a DNS proxy for your spokes. To see DNS requests that the firewall has serviced, you can execute the following query.

```kusto
AzureDiagnostics
| where Category == "AzureFirewallDnsProxy"
| parse kind=regex msg_s with "DNS Request: " SourceIp ":.+ IN " RequestedName " ..p.+s"
| project TimeGenerated, SourceIp, RequestedName
| order by TimeGenerated desc
```

### View image imports in Azure Container Registry

To view all the image imports that have happened against your container registry, you can execute the following query.

ContainerRegistryRepositoryEvents
| where OperationName == "importImage"
| order by TimeGenerated desc

### View image pulls in Azure Container Registry

To view all the pulls that have happened against your container registry, you can execute the following query.

```kusto
ContainerRegistryRepositoryEvents
| where OperationName == "Pull"
| order by TimeGenerated desc
```
### Kubernetes API Server access logs

This reference implementation logs all AKS control plane interactions in the associated Log Analytics workspace. Specifically this is enabled through the use of `kube-audit-admin` Diagnostics setting that was enabled on the cluster.

```kusto
AzureDiagnostics 
| where Category == 'kube-audit-admin'
| order by TimeGenerated desc
```

This returns all Kubernetes API Server interaction happening in your cluster, other than most `GET` requests. Basically any interaction that might potentially have the capability to modifying the system. Even an "idle" cluster can fill this log very quickly (don't be surprised to see over 200 messages in a 30 minute window). Most regulations do not require it, but if you disable `kube-audit-admin` and instead simply enable `kube-audit` the system will _also_ log all of the `GET` (read) requests as well. This will _dramatically_ increase the number of logs, but you will then see 100% of the requests to the Kubernetes API Server. _Never enable both at the same time._

### Workload logs

The example workload uses the standard dotnet logger interface, which are captured in `ContainerLogs` in Azure Monitor. You could also include additional logging and telemetry frameworks in your workload, such as Application Insights. Execute the following query to view application logs.

```kusto
ContainerLog
| where Image contains "chain-api"
| order by TimeGenerated desc
```

### Azure Policy logs

Azure policy definitions sync with your cluster about once every 15 minutes. To see when they sync you can execute the following query.

```kusto
ContainerLog
| where Image contains "policy-kubernetes-addon"
| where LogEntry contains "Syncing policies with cluster"
| order by TimeGenerated desc
```

And audit results will be sent to Azure Policy about once every 30 minutes. To see when they sync you can execute the following query.

```kusto
ContainerLog
| where Image contains "policy-kubernetes-addon"
| where LogEntry contains "Sending audit result"
| order by TimeGenerated desc
```

### Azure Application Gateway Access logs

All traffic that the gateway services can be viewed via the following query. This includes source and destination information.

```kusto
AzureDiagnostics 
| where Category == "ApplicationGatewayAccessLog"
| order by TimeGenerated desc
```

### View Web Application Firewall (WAF) logs

Blocked requests (along with other gateway data) will be visible in the attached Log Analytics workspace. Execute the following query to show WAF logs, for example. If you executed the intentionally malicious request on the previous page, you should already see logs in here.

```kusto
AzureDiagnostics 
| where Category == "ApplicationGatewayFirewallLog"
| order by TimeGenerated desc
```

### View all requests for Azure Key Vault Secrets

Both your cluster and Application Gateway will be pulling secrets from your Key Vault. To see that traffic, you can execute the following query.

```kusto
AzureDiagnostics 
| where OperationName == "SecretGet"
| order by TimeGenerated desc
```

You should see requests for `sslcert` & `appgw-ingress-internal-aks-ingress-contoso-com-tls` from Application Gateway and `ingress-internal-aks-ingress-contoso-com-tls` from your AKS cluster.

Other common operations you might be interested in are `SecretResourcePut`, `Authentication`, and `CertificateImport`.

### Microsoft Defender for Cloud - Regulatory compliance

If your subscription has the **Azure Security Benchmark** Azure Policy initiative applied, and **Industry & regulatory standards** enabled (e.g. **PCI DSS 3.2.1**), the **Regulatory compliance** dashboard will allow you to see compliance status for controls that have been mapped by Azure to the specific standard. The view is updated about once every 24 hours, this includes its initial scan. So if you enabled this as part of the walkthrough (steps found on the [Subscription page](./04-subscription.md)), you may not yet see any content in here.

1. Open the [Regulatory compliance dashboard](https://portal.azure.com/#blade/Microsoft_Azure_Security/SecurityMenuBlade/22) in Microsoft Defender for Cloud.
1. Review the Industry summary and any recommendations.
