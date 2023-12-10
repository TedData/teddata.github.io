---
layout: post
title: "PowerBI Report Size Tracker"
subtitle: 'Report Size to CSV'
date: 2023-11-20
author: "Ted"
header-style: text
mathjax: true
tags:
  - PowerShell
  - Power BI
---

# Automating Power BI Reporting with PowerShell

PowerShell scripting provides a powerful toolset for automating tasks, and when it comes to managing and extracting information from the Power BI service, it becomes an invaluable asset. In this article, we'll explore a PowerShell script designed to automate interactions with the Power BI service, focusing on retrieving information about Power BI reports and exporting the data to a CSV file.

```powershell
<#
.SYNOPSIS
This PowerShell script automates interactions with the Power BI service, 
focusing on retrieving information about Power BI reports and exporting 
the data to a CSV file.

.DESCRIPTION
The script performs the following tasks:
1. Ensures the existence of the MicrosoftPowerBIMgmt module, installing 
   or updating it if necessary.
2. Connects to the Power BI service using the Connect-PowerBIServiceAccount cmdlet.
3. Iterates through Power BI workspaces and their reports.
4. Retrieves information about each report, including its ID, name, 
   workspace ID, workspace name, and size.
5. Calculates and displays the size of each report in bytes.
6. Exports the collected data to a CSV file named PBI_ReportsSize.csv 
   in the specified output path.
#>

param (
    [string]$outputPath = "C:\Users\ted\Downloads"
)

# Function to Ensure Module Existence
function Install-OrUpdate-Module {
    param(
        [string]$ModuleName
    )

    # Check if the module is available
    $module = Get-Module $ModuleName -ListAvailable -ErrorAction SilentlyContinue

    if (!$module) {
        Write-Host "Installing module $ModuleName ..."
        Install-Module -Name $ModuleName -Force -Scope CurrentUser
        Write-Host "Module installed"
    }
    else {
        Write-Host "Module $ModuleName found."

        # Update the module if it's not the desired version
        if ($module.Version -lt '1.0.0' -or $module.Version -le '1.0.410') {
            Write-Host "Updating module $ModuleName ..."
            Update-Module -Name $ModuleName -Force -ErrorAction Stop
            Write-Host "Module updated"
        }
    }
}

# Check and install/update required module
Install-OrUpdate-Module -ModuleName "MicrosoftPowerBIMgmt"

# Connect to Power BI Service Account
Connect-PowerBIServiceAccount

# Initialize array to store report information
$PBI_ReportsSize = @()

# Iterate through Power BI workspaces
foreach ($workspace in Get-PowerBIWorkspace) {
    $workspaceId = $workspace.Id

    # Iterate through reports in each workspace
    foreach ($report in Get-PowerBIReport -WorkspaceId $workspaceId) {
        $uri_getSize = "https://api.powerbi.com/v1.0/myorg/admin/reports/$($report.Id)"
        $reportContent = Invoke-PowerBIRestMethod -Url "$uri_getSize/Export" -Method Get

        # Calculate report size and handle the case when the size is 0
        $reportSize = if ($reportContent.Length -eq 0) { "" } else { $reportContent.Length }

        # Display information about each report
        Write-Host "$($report.Name): $($reportSize) bytes"

        # Create a custom object to store report information
        $ReportsSizeInfo = [PSCustomObject]@{
            "ReportId"    = $report.Id
            "Report"      = $report.Name
            "WorkspaceId" = $workspaceId
            "Workspace"   = $workspace.Name
            "ReportSize"  = $reportSize
        }

        # Add report information to the array
        $PBI_ReportsSize += $ReportsSizeInfo
    }
}

# Export collected data to a CSV file
$PBI_ReportsSize | Export-Csv -Path "$outputPath\PBI_ReportSize.csv" -NoTypeInformation

# Display completion message
Write-Host "Completed"
```

## Script Overview

The script begins with a synopsis and description outlining its purpose. It ensures the presence of the MicrosoftPowerBIMgmt module and connects to the Power BI service using the Connect-PowerBIServiceAccount cmdlet. The script then iterates through Power BI workspaces and their reports, retrieving information such as report ID, name, workspace ID, workspace name, and size.

## Module Installation or Update

To guarantee the required module's existence, the script includes a function, Install-OrUpdate-Module, which checks for the module and installs or updates it as needed. This ensures that the script has the necessary tools to interact with the Power BI service.

## Retrieving Report Information

The script utilizes the Get-PowerBIWorkspace and Get-PowerBIReport cmdlets to iterate through workspaces and reports. It retrieves information about each report, including its size, by invoking the Power BI REST API.

## Calculating and Displaying Report Size

The script calculates the size of each report and displays it in bytes. It handles cases where the size is 0, providing a clear overview of the reports and their sizes.

## Exporting Data to CSV

Finally, the script exports the collected report information to a CSV file named PBI_ReportSize.csv in the specified output path.

## Conclusion

This PowerShell script streamlines the process of interacting with the Power BI service, making it easier to retrieve essential information about reports and their sizes. By automating these tasks, users can save time and ensure consistency in their reporting processes.
