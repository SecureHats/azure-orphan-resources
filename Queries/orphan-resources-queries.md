# Azure Orphaned Resources - Queries

Here you can find all the orphan resources queries that build this Workbook.

#### Disks

Managed Disks with 'Unattached' state and not related to Azure Site Recovery.

```kql
Resources
| where type has "microsoft.compute/disks"
| extend diskState = tostring(properties.diskState)
| where managedBy == ""
| where not(name endswith "-ASRReplica" or name startswith "ms-asr-" or name startswith "asrseeddisk-")
| extend Details = pack_all()
| project id, resourceGroup, diskState, sku.name, properties.diskSizeGB, location, tags, subscriptionId, Details
```

> **_Note:_** Azure Site Recovery (aka: ASR) managed disks are excluded from the orphaned resource query.

> <sub> 1) Enable replication process created a temporary *'Unattached'* managed disk that begins with the prefix *"ms-asr-"*.<br/>
        2) When the replication start, a new managed disk that begin with the suffix *"-ASRReplica"* created in *'ActiveSAS'* state.<br/>
        3) When replicated on-premises VMware VMs and physicall servers to managed disks in Azure, these logs are used to create recovery points on Azure-managed disks that have prefix of *"asrseeddisk-"*.</sub>

#### Network Interfaces

Network Interfaces that are not attached to any resource.

```kql
Resources
| where type has "microsoft.network/networkinterfaces"
| where isnull(properties.privateEndpoint)
| where isnull(properties.privateLinkService)
| where properties.hostedWorkloads == "[]"
| where properties !has 'virtualmachine'
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, tags, subscriptionId, Details
```

> **_Note:_** Azure Netapp Volumes are excluded from the orphaned resource query.

> <sub> When creating a _Volume_ in _Azure Netapp Account_: <br/>
        1) A delegated subnet created in the virtaul network (vNET). <br/>
        2) A Network Interface created in the subnet with the fields: <br/><sub>
&nbsp;&nbsp;&nbsp;&nbsp;- "linkedResourceType": "Microsoft.Netapp/volumes" <br/>
&nbsp;&nbsp;&nbsp;&nbsp;- "hostedWorkloads": ["/subscriptions/<_**SubscriptionId**_>/resourceGroups/<_**RG-Name**_>/providers/Microsoft.NetApp/netAppAccounts/<_**NetAppAccount-Name**_>/capacityPools/<NetAppCapacityPool-Name>/volumes/<_**NetAppVolume-Name**_>" <br/>
&nbsp;&nbsp;&nbsp;&nbsp;- "bareMetalServer": { "id": "/subscriptions/<_**SubscriptionId**_>/resourceGroups/<_**RG-Name**_>/providers/Microsoft.Network/bareMetalServers/<_**baremetalTenant_svm_ID**_>"}</sub></sub>


#### Public IPs

Public IPs that are not attached to any resource (VM, NAT Gateway, Load Balancer, Application Gateway, Public IP Prefix, etc.).

```kql
Resources
| where type == "microsoft.network/publicipaddresses"
| where properties.ipConfiguration == "" and properties.natGateway == "" and properties.publicIPPrefix == ""
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, sku.name, tags ,Details
```

#### Resource Groups

Resource Groups without resources (including hidden types resources).

```kql
ResourceContainers
 | where type == "microsoft.resources/subscriptions/resourcegroups"
 | extend rgAndSub = strcat(resourceGroup, "--", subscriptionId)
 | join kind=leftouter (
     Resources
     | extend rgAndSub = strcat(resourceGroup, "--", subscriptionId)
     | summarize count() by rgAndSub
 ) on rgAndSub
 | where isnull(count_)
 | extend Details = pack_all()
 | project subscriptionId, Resource=id, location, tags ,Details
```

#### Network Security Groups (NSGs)

Network Security Group (NSGs) that are not attached to any network interface or subnet.

```kql
Resources
| where type == "microsoft.network/networksecuritygroups" and isnull(properties.networkInterfaces) and isnull(properties.subnets)
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Availability Set

Availability Sets that not associated to any Virtual Machine (VM) or Virtual Machine Scale Set (VMSS).

```kql
Resources
| where type =~ 'Microsoft.Compute/availabilitySets'
| where properties.virtualMachines == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Route Tables

Route Tables that not attached to any subnet.

```kql
resources
| where type == "microsoft.network/routetables"
| where isnull(properties.subnets)
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### Load Balancers

Load Balancers with empty backend address pools.

```kql
resources
| where type == "microsoft.network/loadbalancers"
| where properties.backendAddressPools == "[]"
| extend Details = pack_all()
| project subscriptionId, Resource=id, resourceGroup, location, tags, Details
```

#### App Service Plans

App Service plans without hosting Apps.

```kql
resources
| where type =~ "microsoft.web/serverfarms"
| where properties.numberOfSites == 0
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, Sku=sku.name, Tier=sku.tier, tags ,Details
```
        
#### Front Door WAF Policy

Front Door WAF Policy without associations. (Frontend Endpoint Links, Security Policy Links)

```kql
resources
| where type == "microsoft.network/frontdoorwebapplicationfirewallpolicies"
| where properties.frontendEndpointLinks== "[]" and properties.securityPolicyLinks == "[]"
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, Sku=sku.name, tags, Details
```
        
#### Traffic Manager Profiles

Traffic Manager without endpoints.

```kql
resources
| where type == "microsoft.network/trafficmanagerprofiles"
| where properties.endpoints == "[]"
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, tags, Details
```
