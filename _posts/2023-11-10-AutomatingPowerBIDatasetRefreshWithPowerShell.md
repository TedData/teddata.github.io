---
layout: post
title: "Refresh PowerBI Datasets"
subtitle: 'Refresh the Workspace from PBI'
date: 2023-11-10
author: "Ted"
header-style: text
tags:
  - Power BI
  - PowerShell
---


# Automating Power BI Dataset Refresh with PowerShell

Power BI is a powerful business analytics service provided by Microsoft, allowing users to visualize and share insights across their organization. One crucial aspect of Power BI is the ability to keep datasets up to date through periodic refreshes. In this article, we'll explore a PowerShell script that automates the refresh of Power BI datasets within specified workspaces.

## Script Overview

The PowerShell script begins with a set of comments providing a synopsis and description of its purpose. Let's break down the key components:

### Parameters

The script includes two parameters:

- `$SpecifiedWorkspace`: Specifies the Power BI workspace to refresh. The default value is set to a sample workspace ID.
- `$refreshAllWorkspace`: Indicates whether to refresh datasets in all Power BI workspaces.

### Module Management

The script defines a function `Assert-ModuleExists` to ensure that the required PowerShell module, "MicrosoftPowerBIMgmt," is installed and up to date. This module is essential for interacting with the Power BI service.

### Power BI Connection

The script connects to the Power BI service account using the `Connect-PowerBIServiceAccount` cmdlet.

### Workspace Retrieval

Based on user input, the script retrieves a list of Power BI workspaces. If `$refreshAllWorkspace` is true, it gets all workspaces; otherwise, it fetches the specified workspace.

### Dataset Refresh

For each workspace, the script iterates through the datasets, displaying a message for each dataset being refreshed. It then builds the URI for dataset refresh using the Power BI REST API and triggers the refresh using `Invoke-PowerBIRestMethod`.

## Code

```powershell
<#
.SYNOPSIS
   This PowerShell script automates the refresh of Power BI datasets within specified workspaces.
.DESCRIPTION
   The script connects to the Power BI service account, retrieves a list of 
   Power BI workspaces based on user input, and refreshes all datasets within 
   each workspace. The user can either specify a particular workspace or choose 
   to refresh all workspaces. The script also ensures that the required Power 
   BI module ("MicrosoftPowerBIMgmt") is installed and up to date.
.PARAMETER SpecifiedWorkspace
   Specifies the Power BI workspace to refresh. Default value is set to a sample workspace ID.
.PARAMETER refreshAllWorkspace
   Indicates whether to refresh datasets in all Power BI workspaces.
#>

param (
    [string]$SpecifiedWorkspace = "113a0c56-9a98-44f6-91b7-c7964b34531f",
    [bool]$refreshAllWorkspace = $false
)

# Function to ensure a PowerShell module exists and is up to date
function Assert-ModuleExists([string]$ModuleName) {
    $module = Get-Module $ModuleName -ListAvailable -ErrorAction SilentlyContinue
    if (!$module) {
        Write-Host "Installing module $ModuleName ..."
        Install-Module -Name $ModuleName -Force -Scope CurrentUser
        Write-Host "Module installed"
    } elseif ($module.Version -lt '1.0.0') {
        Write-Host "Updating module $ModuleName ..."
        Update-Module -Name $ModuleName -Force -ErrorAction Stop
        Write-Host "Module updated"
    }
}

# Ensure that the required Power BI module exists and is up to date
Assert-ModuleExists -ModuleName "MicrosoftPowerBIMgmt"
# Connect to the Power BI service account
Connect-PowerBIServiceAccount

# Retrieve the list of Power BI workspaces based on user input
if ($refreshAllWorkspace) {
    $AllWorkspaces = Get-PowerBIWorkspace
} else {
    $AllWorkspaces = @(Get-PowerBIWorkspace -Id $SpecifiedWorkspace)
}

# Iterate through each workspace
foreach ($workspace in $AllWorkspaces) {
    $workspaceId = $workspace.Id
    # Retrieve datasets within the current workspace
    $datasets = Get-PowerBIDataset -WorkspaceId $workspaceId

    # Iterate through each dataset in the workspace
    foreach ($dataset in $datasets) {
        $datasetid = $dataset.Id
        # Display a message indicating the dataset and workspace being refreshed
        Write-Host "Refreshing dataset $($dataset.Name) in workspace $($workspace.Name)"

        # Build the URI for dataset refresh
        $uri = "groups/$workspaceId/datasets/$datasetid/refreshes"
        Write-Host $uri

        # Trigger the dataset refresh using Power BI REST API
        Invoke-PowerBIRestMethod -Url $uri -Method Post
    }
}

```

## Conclusion

Automation is key to efficiently managing Power BI datasets, and this PowerShell script provides a convenient way to refresh datasets across specified workspaces. By leveraging the Power BI REST API and PowerShell capabilities, users can save time and ensure that their Power BI reports are always based on the latest data.
