# Automating creation of DNS records
Azure Portal offers creating records in Azure Private DNS as part of GUI wizard. Unfortunatelly this works only for private zones in the same subscription therefore is not ideal for enterprise scenario when custom DNS server in central hub subscription is used. In such case all spoke subscriptions are pointed to custom DNS server which than forwards requestes to on-premises and to private DNS zones configured in hub subscription. Since wizard in spole does not support configuring zone in different subscription (and even if it does enterprise customers will not allow users to access it anyway) different approach is currently needed.

There are few ways to automate this environment including using Azure Functions, Azure Automation, Logic Apps and either use scheduled jobs or react on Azure control plane event flows. All of those require knowledge of such systems including maintenance. Therefore we used Azure Policy to achieve this automation as it is tool central team is familiar with and it is used to fulfil many other governance requirements.

Policy uses DeployIfNotExists effect and do to certain limitations (references and outputs) it uses linked templates (nested templates cannot be used in this scenario). 

sqlPolicyDefinition.json file contains policy rules and master template. It is using nonsense existence condition as policy currently does not support evaluating resources outside of subscription where resource is created (we cannot test existence of DNS record on hub subscription while reacting on Private Link creation in spoke subscription). This way policy always triggers.

Note policy assignment will create managed identity to deploy resources. We will use Network Contributor role. Note users of spoke subscription will have no access to this account so networking environment stays protected.

There are few items you might want to modify:
- Different service - this example is used for Azure SQL. In order to create policy for different type of resource, replace policy rule to match eg. Microsoft.Storage and change domain parameter value when calling plinkPolicyDnsRecord. There are no modifications needed in plinkPolicyGetPrivateIp.json and plinkPolicyDnsRecord.json
- Linked templates referenced are stored on my GitHub. I would advice to download plinkPolicyGetPrivateIp.json and plinkPolicyDnsRecord.json and publish in private blob storage account and access those via Shared Access Signature. Change uri field in TemplateLink.

I will publish this policy definition and assign on subscription level (in practice you will probably use management group level that includes all spoke subscriptions). Note managed identity is scoped to management group that includes all spokes and hub so policy engine can access private DNS zone in hub subscription.

```bash
subscriptionName=mojesub2

az policy definition create -n 'dns-automation-private-link-sql' \
    --display-name 'Private Link DNS automation for Azure SQL' \
    --description 'This policy reacts on creation of private link for SQL in spoke subscriptions and provisions DNS record in hub.' \
    --rules ./sqlPolicyDefinition.json \
    --mode All

az policy assignment create -n mojesub2-plink-dns-sql \
    --scope /subscriptions/$(az account show -s $subscriptionName --query id -o tsv) \
    --policy $(az policy definition show -n 'dns-automation-private-link-sql' --query id -o tsv) \
    --assign-identity \
    --role Contributor \
    --identity-scope /providers/Microsoft.Management/managementGroups/tomaskubica \
    --location westeurope
```

