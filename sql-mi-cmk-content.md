This example architecture describes how customers can manage their own Transparent Data Encryption (TDE) keys for SQL Managed Instances in a cross-region, auto-failover group using using Azure Key Vault. When using customer-managed keys (CMK), also referred to as Bring Your Own Key (BYOK), the customer is responsible for the security, availability, and optional rotation of the keys.This is a critical responsibility because if the key is lost, the [databases and backups are permanently lost](source) as well. This article will detail the process and provide options so that you have all the information you need to be able to make an informed decision about which method is optimal for your business.

## Potential use cases

Because of the nature of [how TDE works](source), losing the key would be disastrous. Any database that is critical to your business would be a potential use case for securing your customer-managed TDE keys. 


## Architecture

![](./media/mesh.png)

>For greater redundancy of the TDE keys, SQL Managed Instance is configured to use the Key Vault in the same region as ir for primary, and the Key Vault in the remote region as secondary.

>The secondary Key Vault instance, while in a remote region, has a [Private Endpoint](https://docs.microsoft.com/azure/private-link/private-endpoint-overview) in the same region as the SQL instance. So, as far as a SQL Instance is concerned, requests made to both primary and secondary Key Vaults are logically within the same VNET and region. Many organizations choose to use Private Endpoint over accessing the public endpoint, therefore, this is the recommended method.

Flow

1. Every 10 minutes, SQL Managed Instance will check to make sure it can access the TDE wrapper at the Key Vault defined as primary. 

2. If a Key Vault becomes unavailable, and is set as the primary on a SQL Instance, that instance will check the Key Vault set as secondary. If that Key Vault is also unavailable, SQL Managed Instance will mark the databases as "inaccessible" (source).

### Components

- [Azure Key Vault](https://azure.microsoft.com/services/key-vault/)
- [Azure SQL Managed Instance](https://docs.microsoft.com/azure/azure-sql/managed-instance/)
- [Private Link Service](https://docs.microsoft.com/azure/private-link/)


### Alternatives
The alternative to Customer-Managed TDE keys is Service-Managed TDE Keys. Microsoft handles securing and rotating the keys for you and the entire process is abstracted away from the customer. 

## Considerations

### Key Management Considerations
Your method of Key Rotation will differ depending on what you are using to create your TDE asymmetric keys. When you bring your own TDE wrapper key, you have to decide how you will create this key. Your options are:

1. Have Azure Key Vault create the key. This option ensures that the private key material never leaves Azure Key Vault and cannot be seen by any human or system. While the private key is not exportable, it can be backed up and restored to another Azure Key Vault. This is important to know because in order to have the same key material in multiple Key Vaults as required by this design, you will have to use the bakup / restore feature.  There are several limitation with this option. Both Key Vaults must be in the same Azure [geography](source) or the restore will not work. The only way around this is to keep the Key Vaults in separate subscriptions and [move the subscription to another region](source). 

2. Generate the asymmetric keys offline using a utility such as OpenSSL and import the key into Azure Key Vault. When you import a key into Key Vault, you can [mark is as exportable](source) so you can either throw away the keys once you've imported them into Key Vault or you can store them somewhere else (on-prem, another Key Vault, etc.). This option gives you the most flexibility but can be the least secure without properly ensuring the keys don't get in the wrong hands. The system generating the keys and the method used to place the keys in Azure Key Vault are not controlled by Azure. This process can be automated using [Azure DevOps](https://docs.microsoft.com/azure/devops/), [Azure Automation](https://docs.microsoft.com/azure/automation/), or any orchestration tool of your choice.


3. Use a [supported on-premises Hardware Security Module (HSM)](https://docs.microsoft.com/en-us/azure/key-vault/keys/hsm-protected-keys#supported-hsms) to generate your keys. Using a supported HSM, you can import keys into Azure Key Vault, securely. The same-geography limitation does not apply here.

### Availability
By adding Azure Key Vault to your architecture, it becomes a critical component and at least one of the Key Vaults in the design must be accessible. Additionally, the keys necessary for TDE must be accessible. Azure Monitor Insights provides comprehansive monitoring of Azure Key Vault. More information can be found here: [source](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/key-vault-insights-overview)


### Operations
Because Azure SQL Managed Instance and Azure Key Vault are managed services, when it comes to operating this solution, the only components that require managing are the ones you're choosing to manage: Your TDE wrapper key. 


### Performance
SQL Managed Instance auto-failover groups performce significantly better when using paired regions as opposed to not using paired regions (link to source)


### Scalability
Because the focus of this document is the self-management of the TDE wrapper keys, we won't discuss the scalability of SQL Managed Instance but rather Azure Key Vault. The requests being made to Azure Key Vault are minimal (every 10 minutes) [source](source), therefore, scaling of the Key Vault will not be an issue. However, if there does happen to be throttling, or another incident which prevents the Key Vault from being responsive enough to service SQL MI, SQL will fail over to the secondary Key Vault to service its requests. 


### Security
The biggest security consideration is ensuring you keep your TDE wrapper key safe. Microsoft recommends using service-managed keys to take this burden off the customer.


### Resiliency
Each SQL Instance is configured to use two Key Vaults. If an instance's primary Key Vault is unavailable, it will attempt to find the key with a matching thumbprint in the secondary Key Vault. If there is no secondary Key Vault configured, it will continue to try the primary for 10 minutes. If it does not because accessible, SQL Managed Instance will mark the databases as *down*. (source)


### DevOps
Azure DevOps can be used to automate the process used for [Key Rotation](source).


## Deploy This Scenario
This scenario can be deployed by using the following ARM templates:
[ARM Templates](source).


## Pricing
Enter Key Vault pricing, HSM pricing, etc.


## Next Steps
???

## Related Resources
??