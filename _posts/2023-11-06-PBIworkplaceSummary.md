---
layout: post
title: "Exports PowerBI workplace Summary"
subtitle: 'PBI Workspace Summary to CSV'
date: 2023-11-06
author: "Ted"
header-style: text
tags:
  - Power BI
  - PowerShell
---

# Power BI Workspace Summary Export using PowerShell

PowerShell is a powerful scripting language that enables automation and management tasks in various environments. In this script, we demonstrate how to leverage PowerShell to connect to the Power BI service account, retrieve detailed information about all workspaces within the organization, and export a comprehensive summary to a CSV file.

## Script Overview

```powershell
# PowerShell Script for Power BI Workspace Summary Export

<#
Description:
This PowerShell script connects to the Power BI service account, 
retrieves information about all workspaces in the organization, 
and exports a summary to a CSV file.

Functionality:
1. Connects to the Power BI service account.
2. Retrieves details of all Power BI workspaces in the organization.
3. Creates a summary including workspace properties such as ID, Name, 
   Type, State, IsReadOnly, IsOrphaned, IsOnDedicatedCapacity, 
   CapacityId, UsersCount, and UserNames.
4. Exports the summary to a CSV file.

Summary:
The script provides an overview of Power BI workspaces within the 
organization, capturing essential details for analysis and documentation.
#>

# Set output path for CSV export
$output_path = "C:\Users\ted\Downloads"

# Connect to Power BI service account
Connect-PowerBIServiceAccount

# Get all Power BI workspaces in the organization
$allWorkspaces = Get-PowerBIWorkspace -Scope Organization

# Create an array to store results
$results = foreach ($workspace in $allWorkspaces) {
    [PSCustomObject]@{
        "Id"                    = $workspace.Id
        "Name"                  = $workspace.Name
        "Type"                  = $workspace.Type
        "State"                 = $workspace.State
        "IsReadOnly"            = $workspace.IsReadOnly
        "IsOrphaned"            = $workspace.IsOrphaned
        "IsOnDedicatedCapacity" = $workspace.IsOnDedicatedCapacity
        "CapacityId"            = $workspace.CapacityId
        "UsersCount"            = ($workspace.Users.Count | Measure-Object -Sum).Sum
        "UserNames"             = ($workspace.Users.UserPrincipalName -replace '@.*\.com') -join ', '  # Extracting and joining user names
    }
}

# Export results to CSV
$results | Export-Csv -Path "$output_path\workspaceSummary.csv" -NoTypeInformation
```

## Explanation

The script begins with a detailed description of its purpose, providing insights into its functionalities and intended outcomes.

The code then sets the output path for the CSV file where the workspace summary will be exported.

It establishes a connection to the Power BI service account using Connect-PowerBIServiceAccount.

The script fetches all Power BI workspaces within the organization using Get-PowerBIWorkspace with the scope set to Organization.

For each workspace, it creates a custom object containing essential properties such as ID, Name, Type, State, etc.

The script then exports the collected workspace information to a CSV file located at the specified output path.

## Summary

This PowerShell script serves as a valuable tool for Power BI administrators and users, providing an automated means to gather detailed information about all workspaces within an organization. The exported CSV file contains crucial details, facilitating analysis, documentation, and overall workspace management.
