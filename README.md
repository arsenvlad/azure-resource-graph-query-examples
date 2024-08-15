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
| extend roleDefinitionId = tolower(tostring(properties.roleDefinitionId))
| join kind=leftouter (authorizationresources 
                        | where type == 'microsoft.authorization/roledefinitions'
                        | extend roleDefinitionName = tostring(properties.roleName)
                        | extend rdId = tolower(id)
                      ) on $left.roleDefinitionId == $right.rdId
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
| project id, tenantId, subscriptionId, principalType, principalId, createdOn, scopeType, scope, roleDefinitionName, roleDefinitionId
| summarize dcount(id) by tenantId, subscriptionId
```

## Compute Quota and Usage per subscription

The Azure Resource Graph table "quotaresources" is mentioned in this document: <https://learn.microsoft.com/en-us/azure/quotas/how-to-guide-monitoring-alerting>

It returns data similar to the response of this ARM REST API: <https://learn.microsoft.com/en-us/rest/api/compute/usage/list>

```kql
quotaresources
| where type =~ "microsoft.compute/locations/usages"
| mv-expand json = properties.value limit 1000
| extend currentValue = toint(json['currentValue'])
| extend limitValue = toint(json['limit'])
| extend limitName = tostring(json['name'].localizedValue)
| extend unit = tostring(json['unit'])
| extend usedPercent = toint((todouble(currentValue) / todouble(limitValue)) * 100)
| where limitName contains "vCPUs"
| where limitName <> "Total Regional vCPUs"
| where currentValue > 0 // only rows that have current usage
//| where subscriptionId in ("1","2") // only specific subscriptions
//| where location in ("westus3","eastus") // only specific locations
| join kind=inner (resourcecontainers | where type =~ "microsoft.resources/subscriptions" | project subId = subscriptionId, subName = name) on $left.subscriptionId == $right.subId
| project subscriptionId, subName, location, limitName, currentValue, limitValue, usedPercent
```

## Key Vault access policies

```kql
resources
| where type == "microsoft.keyvault/vaults"
| extend accessPolicies = tostring(properties.accessPolicies)
| extend accessPolicyCount = array_length(parse_json(accessPolicies))
| project subscriptionId, resourceGroup, name, location, accessPolicyCount
| order by accessPolicyCount desc

resources
| where type == "microsoft.keyvault/vaults"
| extend accessPolicies = parse_json(properties.accessPolicies)
| mv-expand accessPolicies limit 2000
| project subscriptionId, name, location, resourceGroup, accessPolicyObjectId = accessPolicies.objectId, accessPolicyTenantId = accessPolicies.tenantId, accessPolicyPermissions = accessPolicies.permissions
| summarize accessPolicyCount = count() by subscriptionId, resourceGroup, name, location
| order by accessPolicyCount desc
```
