---
layout: post
title: "PowerBI Gateway DataSource Script"
subtitle: 'Gateway and DataSource User to CSV'
date: 2023-12-04
author: "Ted"
header-style: text 
header-mask: 0.3
mathjax: true
tags:
  - PowerShell
  - Power BI
---

# PowerShell Script for Power BI Gateway and DataSource User Information

PowerShell, a robust scripting language, empowers users to automate tasks efficiently. In this script, we delve into the realm of Power BI, extracting detailed information about Gateways, DataSources, and their associated users.

## Script Overview

### Module Management

The script begins by ensuring the presence of the required module "MicrosoftPowerBIMgmt." It checks for installation or updates to maintain compatibility:

```powershell
function Install-OrUpdate-Module([string]$ModuleName) {
    # Check if the module is installed and its version is outdated
    $module = Get-Module $ModuleName -ListAvailable -ErrorAction SilentlyContinue

    if (!$module -or ($module.Version -ne '1.0.0' -and $module.Version -le '1.0.410')) {
        $action = if (!$module) { "Installing" } else { "Updating" }
        Write-Host "$action module $ModuleName ..."
        Install-Module -Name $ModuleName -Force -Scope CurrentUser
        Write-Host "Module $ModuleName $action complete"
    }
}
```
# Install or update the required PowerShell module for Power BI management
```
Install-OrUpdate-Module -ModuleName "MicrosoftPowerBIMgmt"
Connect-PowerBIServiceAccount
```

### Gateway Information Retrieval

The script proceeds to gather information about Power BI Gateways by querying the Power BI service's REST API. Key details such as Gateway ID, Name, Type, Public Key, and Version are extracted:

```powershell
$url = "https://api.powerbi.com/v1.0/myorg/gateways"
$GetGateways = Invoke-PowerBIRestMethod -Url $url -Method Get | ConvertFrom-Json
```

### DataSource Processing

For each Gateway, the script iterates through associated DataSources, collecting essential information like DataSource ID, Name, Credential Type, and Connection Details:

```powershell
foreach ($Gateway in $GetGateways.value){
    # Extract gateway information
    $gatewayAnnotation = $Gateway.gatewayAnnotation | ConvertFrom-Json
    $gatewayId = $Gateway.id
    $gatewayName = $Gateway.name
    $gatewatType = $Gateway.type
    $gatewatPublicKey = $Gateway.publicKey
    $gatewayContactInformation = $gatewayAnnotation.gatewayContactInformation
    $gatewayVersion = $gatewayAnnotation.gatewayVersion
    $gatewayWitnessString = $gatewayAnnotation.gatewayWitnessString
    $gatewayMachine = $gatewayAnnotation.gatewayMachine

    # Retrieve datasource information for the current gateway
    $url_getGatewayId = "https://api.powerbi.com/v1.0/myorg/gateways/$($gatewayId)/datasources"
    $GetDatasources = Invoke-PowerBIRestMethod -Url $url_getGatewayId -Method Get | ConvertFrom-Json
```

### User Information Extraction

The script then dives into each DataSource, retrieving user-specific details such as DisplayName, Principal Type, Identifier, and DataSource Access Rights:

```powershell
# Iterate through each user for the current datasource
foreach ($GetDatasourcesUser in $GetDatasourcesUsers.value){
    $displayName = $GetDatasourcesUser.displayName
    $principalType = $GetDatasourcesUser.principalType
    $identifier = $GetDatasourcesUser.identifier
    $datasourceAccessRight = $GetDatasourcesUser.datasourceAccessRight

    # Create objects to store user information
    $result_userInfo = [PSCustomObject]@{
        "datasourceName" = $datasourceName
        "datasourceId" = $datasourceId
        "userName" = $displayName
        "userType" = $principalType
        "identifier" = $identifier
        "datasourceAccessRight" = $datasourceAccessRight
    }
    $result_user += $result_userInfo
}
```

### Results Export

Finally, the collected information is stored in custom objects for Gateways and Users. The script exports these details to separate CSV files for analysis:

```powershell
# Export results to CSV files
$result_gateway | Export-Csv -Path "$outputPath\PBI_Gateways.csv" -NoTypeInformation
$result_user | Export-Csv -Path "$outputPath\PBI_DataSourceUsers.csv" -NoTypeInformation
```

## Conclusion

In conclusion, this PowerShell script offers a comprehensive solution for auditing Power BI Gateway and DataSource information. Leveraging the Power BI Management module and REST API, users can gain insights into their Power BI infrastructure, aiding in efficient management and troubleshooting.
