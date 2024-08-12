# Azure Resource Graph query examples

Learn more about [Azure Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview)

## Private Endpoints per VNet

Sample query that returns number of private endpoints per VNet.

 ```kql
resources
| where type == "microsoft.network/privateendpoints"
| extend subnetId = tostring(properties.subnet.id)
| mv-expand privateLinkServiceConnections = properties.privateLinkServiceConnections
| extend privateLinkServiceId = tostring(privateLinkServiceConnections.properties.privateLinkServiceId)
| extend privateLinkServiceConnectionStatus = tostring(privateLinkServiceConnections.properties.privateLinkServiceConnectionState.status)
| join kind=inner (resources | where type == "microsoft.network/virtualnetworks" | mv-expand subnets = properties.subnets | extend vnetSubnetId = tostring(subnets.id) | project vnetId = id, vnetSubnetId) on $left.subnetId == $right.vnetSubnetId
| project subscriptionId, id, name, location, vnetId, subnetId, privateLinkServiceId, privateLinkServiceConnectionStatus
| summarize dcount(id) by subscriptionId, vnetId, location
```

## Private Endpoint connections per Private Link Service

Sample query that returns number of private endpoints connected to a private link service (with has limit of 1000). Comment out the summarize line to see the details of each PLS-PE connection.

```kql
resources
| where type == "microsoft.network/privatelinkservices"
| extend pec = properties.privateEndpointConnections
| mv-expand pec
| extend provisioningState = tostring(pec.properties.provisioningState)
| extend status = tostring(pec.properties.privateLinkServiceConnectionState.status)
| extend privateEndpointId = tostring(pec.properties.privateEndpoint.id)
| extend privateEndpointLocation = tostring(pec.properties.privateEndpointLocation)
| project subscriptionId, id, location, provisioningState, status, privateEndpointId, privateEndpointLocation
| summarize dcount(privateEndpointId) by subscriptionId, id, location
```

## Role Assignments per subscription

```kql
authorizationresources
| where type == 'microsoft.authorization/roleassignments'
| extend roleDefinitionId = tostring(properties.roleDefinitionId)
| extend principalType = tostring(properties.principalType)
| extend principalId = tostring(properties.principalId)
| extend createdOn = todatetime(properties.createdOn)
| extend scope = tostring(properties.scope)
| extend scopeParts = split(scope,"/")
| extend scopeType = case(scope contains "managementGroups", "managementGroup",
                          scope contains "resourceGroups" and array_length(scopeParts) == 5, "resourceGroup",
                          scope contains "resourceGroups" and array_length(scopeParts) > 5, "resource",
                          scope contains "subscriptions" and array_length(scopeParts) == 3, "subscription",
                          scope == "/", "tenant",
                          "")
| project id, tenantId, subscriptionId, principalType, principalId, createdOn, scopeType, scope, roleDefinitionId
| summarize dcount(id) by tenantId, subscriptionId
```
