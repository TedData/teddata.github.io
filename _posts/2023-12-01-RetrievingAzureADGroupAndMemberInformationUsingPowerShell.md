---
layout: post
title: "AzureAD Group Members Info."
subtitle: 'AD Group to CSV'
date: 2023-12-01
author: "Ted"
header-style: text
mathjax: true
tags:
  - PowerShell
  - Power BI
---

# Retrieving Azure AD Group and Member Information using PowerShell

PowerShell is a powerful scripting language that enables automation and management of Microsoft Azure resources. In this article, we'll explore a PowerShell script designed to retrieve information about Azure AD groups and their members using the Az module. The script connects to your Azure account, gathers details about each group, and exports the collected information to a CSV file.

## Script Overview

```powershell
<#
.SYNOPSIS
This PowerShell script retrieves Azure AD group and member information using the Az module.

.DESCRIPTION
The script connects to your Azure account, retrieves a list of Azure AD groups, 
and gathers details about each group and its members. The collected information 
is then exported to a CSV file.

.NOTES
Run the script with Administrator privileges.

#>

param (
    [string]$outputPath = "C:\Users\ted\Downloads"
)

# Install and import the Az module if not already installed
if (-not (Get-Module -Name Az -ListAvailable)) {
    Write-Output "Installing Az"
    Install-Module -Name Az -Force -AllowClobber
}

Write-Output "Import Az"
Import-Module Az

# Check if running with Administrator privileges
if (-not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Host "Please run the script with Administrator privileges." -ForegroundColor Red
    Exit
}

# Connect to your Azure account
Connect-AzAccount

# Retrieve a list of Azure AD groups
$groups = Get-AzADGroup
$getMembers = @()

# Iterate through each Azure AD group
foreach ($group in $groups) {
    # ... (Group information)

    Write-Output "Processing Group: $groupName"

    # Get members of the current Azure AD group
    $members = Get-AzADGroupMember -ObjectId $groupId

    # Iterate through each group member
    foreach ($member in $members.AdditionalProperties) {
        # ... (Member information)

        # Add the member information to the collection
        $getMembers += $getMembersInfo
    }
}

# Export results to a CSV file
$getMembers | Export-Csv -Path "$outputPath\PBI_ADGroup.csv" -NoTypeInformation
```

## Script Breakdown

- *Synopsis and Description:* Provides an overview of the script's purpose and functionality.
- *Notes:* Reminds users to run the script with Administrator privileges.

### Module Installation and Import

- Checks if the Az module is installed and installs it if not.
- Imports the Az module for Azure operations.

### Administrator Privileges Check

- Ensures the script is running with Administrator privileges.

### Azure Account Connection

- Connects to the Azure account using Connect-AzAccount.

### Azure AD Group Retrieval

- Retrieves a list of Azure AD groups using Get-AzADGroup.

### Group and Member Processing

- Iterates through each Azure AD group.
- Retrieves details about the group and its members.
- Creates a custom object for each member with relevant information.
- Adds member information to the collection.

### CSV Export

- Exports the collected information to a CSV file at the specified output path.

## Conclusion

This PowerShell script simplifies the process of gathering Azure AD group and member information, providing valuable insights into your Azure AD environment. Ensure you run the script with Administrator privileges for seamless execution. Feel free to customize the script to suit your specific requirements.
