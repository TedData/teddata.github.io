---
layout: post
title: "Power BI Service Data Export Script"
subtitle: 'PBI Workspace Metadata to CSV'
date: 2023-11-03
author: "Ted"
header-style: text
tags:
  - Power BI
  - PowerShell
---

# Automating Power BI Service Interaction with PowerShell

## Introduction

This article explores a PowerShell script designed to automate interactions with the Power BI service. The script connects to the Power BI service, retrieves detailed information about workspaces, dashboards, reports, datasets, and more. Furthermore, it exports the collected data to CSV files for in-depth analysis.

## Code Overview

Below is a snippet of the script's main code:

```powershell
# Output Path Configuration
$output_path = "C:\Users\ted\Downloads"

# Function to Ensure Module Existence
function Assert-ModuleExists {
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
Assert-ModuleExists -ModuleName "MicrosoftPowerBIMgmt"

# Connect to Power BI Service Account
Connect-PowerBIServiceAccount

# Initialize Arrays
$PBI_Dashboards = @()
$PBI_DashboardsReports = @()
$PBI_Reports = @()
$PBI_ErrorLog = @()
$PBI_DataSources = @()
$PBI_ReportsParameters = @()

# Iterate through Power BI Workspaces
ForEach ($workspace in (Get-PowerBIWorkspace)) {
    $workspaceId = $workspace.Id

    # Iterate through Dashboards in the Workspace
    foreach ($dashboard in (Get-PowerBIDashboard -WorkspaceId $workspaceId)) {
        $dashboardInfo = [PSCustomObject]@{
            "Workspace"     = $workspace.Name
            "Dashboard"     = $dashboard.Name
            "WorkspaceId"   = $workspaceId
            "DashboardId"   = $dashboard.Id
        }

        $PBI_Dashboards += $dashboardInfo

        # Iterate through Tiles in the Dashboard
        foreach ($tile in (Get-PowerBITile -DashboardId $dashboard.Id -WorkspaceId $workspaceId)) {
            if ($tile.ReportId -ne $null) {
                # Iterate through Reports linked to the Tile
                foreach ($report in (Get-PowerBIReport -ReportId $tile.ReportId -WorkspaceId $workspaceId)) {
                    $reportInfo = [PSCustomObject]@{
                        "Workspace"    = $workspace.Name
                        "Dashboard"    = $dashboard.Name
                        "Report"       = $report.Name
                        "WorkspaceId"  = $workspaceId 
                        "DashboardId"  = $dashboard.Id
                        "ReportId"     = $report.Id 
                    }

                    $PBI_DashboardsReports += $reportInfo
                }
            }
        }
    }

    # Iterate through Reports in the Workspace
    foreach ($report in (Get-PowerBIReport -WorkspaceId $workspaceId)) {
        $reportInfo = [PSCustomObject]@{
            "ReportId"     = $report.Id
            "Report"       = $report.Name
            "WorkspaceId"  = $workspaceId
            "Workspace"    = $workspace.Name
        }

        $PBI_Reports += $reportInfo

        # Continue if DatasetId is null
        if ($report.DatasetId -eq $null) {
            continue
        }

        # Iterate through Datasets linked to the Report
        foreach ($dataset in (Get-PowerBIDataset -Id $report.DatasetId -WorkspaceId $workspaceId)) {
            Write-Host $dataset.Name

            # Retrieve Refresh History for the Dataset
            $refreshUri = 'groups/{0}/datasets/{1}/refreshes?$top=10' -f $workspaceId, $dataset.Id
            $refreshHistory = Invoke-PowerBIRestMethod -Url $refreshUri -Method Get | ConvertFrom-Json

            # Process Refresh History
            foreach ($refreshEntry in $refreshHistory.value) {
                $errorJson = $refreshEntry.serviceExceptionJson -replace "'", ''

                $errorLogInfo = [PSCustomObject]@{
                    "Workspace"    = $workspace.Name
                    "Report"       = $report.Name
                    "Dataset"      = $dataset.Name
                    "RefreshType"  = $refreshEntry.RefreshType
                    "StartTime"    = $refreshEntry.StartTime
                    "EndTime"      = $refreshEntry.EndTime
                    "Status"       = $refreshEntry.Status
                    "ErrorMessage" = $errorJson
                    "WorkspaceId"  = $workspaceId
                    "ReportId"     = $report.Id
                    "DatasetId"    = $dataset.Id
                }

                $PBI_ErrorLog += $errorLogInfo
            }
            
            # Continue if DatasetId is null
            if ($dataset.Id -eq $null) {
                continue
            }

            # Iterate through Data Sources linked to the Dataset
            foreach ($datasource in (Get-PowerBIDatasource -DatasetId $dataset.Id -WorkspaceId $workspaceId)) {
                $dataSourceInfo = [PSCustomObject]@{
                    "Workspace"    = $workspace.Name
                    "Report"       = $report.Name
                    "Dataset"      = $dataset.Name
                    "Datasource"   = $datasource.DatasourceId
                    "Type"         = $datasource.datasourceType
                    "Server"       = $datasource.connectionDetails.Server
                    "DatabaseName" = $datasource.connectionDetails.Database
                    "URL"          = $datasource.connectionDetails.Url
                    "WorkspaceId"  = $workspaceId
                    "ReportId"     = $report.Id
                    "DatasetId"    = $dataset.Id
                }

                $PBI_DataSources += $dataSourceInfo         
            }

            # Retrieve Parameters for the Dataset
            $parametersUri = 'groups/{0}/datasets/{1}/parameters' -f $workspaceId, $dataset.Id 
            $parameters = Invoke-PowerBIRestMethod -Url $parametersUri -Method Get | ConvertFrom-Json

            # Process Parameters
            foreach ($param in $parameters.value) {
                foreach ($paramEntry in $param.value) {
                    $parameterInfo = [PSCustomObject]@{
                        "ReportId"      = $report.Id
                        "Report"        = $report.Name
                        "WorkspaceId"   = $workspaceId
                        "Workspace"     = $workspace.Name
                        "ParameterName" = $paramEntry.Name
                        "ParameterValue"= $paramEntry.CurrentValue
                    }

                    $PBI_ReportsParameters += $parameterInfo
                }
            }
        }
    }
}

# Export Data to CSV
$PBI_Dashboards | Export-Csv -Path "$output_path\PBI_Dashboards.csv" -NoTypeInformation
$PBI_DashboardsReports | Export-Csv -Path "$output_path\PBI_DashboardsReports.csv" -NoTypeInformation
$PBI_Reports | Export-Csv -Path "$output_path\PBI_Reports.csv" -NoTypeInformation
$PBI_ErrorLog | Export-Csv -Path "$output_path\PBI_ErrorLog.csv" -NoTypeInformation
$PBI_DataSources | Export-Csv -Path "$output_path\PBI_DataSources.csv" -NoTypeInformation
$PBI_ReportsParameters | Export-Csv -Path "$output_path\PBI_ReportsParameters.csv" -NoTypeInformation

Write-Host "Completed"
```

## Issue and Resolution

During the script implementation, a significant issue was encountered: some workspaces in Power BI had refresh history, while others did not. Through in-depth research, it was discovered that the root cause was the inability of Power BI's data to connect to Microsoft SQL Server Management, hindering automatic refresh. Manually triggering a "refresh" resolved the issue and updated the refresh history data.

## Conclusion

Through this PowerShell script, we successfully automated interactions with the Power BI service, extracting metadata about workspaces. Despite facing a challenge with data refresh, a manual workaround was identified. This script serves as a foundation for further customization and optimization of Power BI-related tasks.



