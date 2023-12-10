---
layout: post
title: "PowerBI Inventory Management Script"
subtitle: 'PBI Workspace Summary to SQL Server'
date: 2023-11-16
author: "Ted"
header-style: text
header-mask: 0.3
mathjax: true
tags:
  - PowerShell
  - Power BI
  - SQL Server
---

# Automating Power BI Inventory Management with PowerShell and SQL Server

Power BI is a powerful tool for data visualization and business intelligence, allowing users to create interactive reports and dashboards. As organizations grow, managing Power BI workspaces and reports becomes crucial. In this article, we'll explore a PowerShell script designed to gather Power BI workspace and report information and store it in a SQL Server database. The script also includes the option to clear old data from the database.

The PowerShell script begins with a detailed comment-based help section, providing a synopsis and description of the script's purpose. It includes a parameter section allowing customization of SQL Server credentials, server instance name, database name, and a flag to determine whether to clear old data in the database.

```powershell
<#
.SYNOPSIS
    PowerShell script to gather and store Power BI workspace and report information 
    in a SQL Server database.

.DESCRIPTION
    This script connects to the Power BI service, retrieves information about 
    workspaces and reports, and stores the data in a SQL Server database. 
    It can install or update the required PowerShell module for Power BI management.

.PARAMETER clearOldData
    A boolean flag indicating whether to clear old data in the database.

#>

param (
    [string]$sqlUsername = "SuperAdmin",            # SQL Server username
    [string]$sqlPassword = "**********",            # SQL Server password
    [string]$serverName = "DESKTOP-*******",        # SQL Server instance name
    [string]$databaseName = "PBI_Inventory",        # Database name
    [bool]$clearOldData = $true                     # Flag to determine whether to clear old data
)

# Function to install or update a PowerShell module
function Install-OrUpdate-Module([string]$ModuleName) {
    $module = Get-Module $ModuleName -ListAvailable -ErrorAction SilentlyContinue

    if (!$module -or ($module.Version -ne '1.0.0' -and $module.Version -le '1.0.410')) {
        $action = if (!$module) { "Installing" } else { "Updating" }
        Write-Host "$action module $ModuleName ..."
        Install-Module -Name $ModuleName -Force -Scope CurrentUser
        Write-Host "Module $ModuleName $action complete"
    }
}

# Function to execute a SQL command with parameters
function Execute-DB-Command-With-Params([System.Data.SqlClient.SqlConnection]$connection, [String]$command, [Hashtable]$parameters) {
    $cmd = $connection.CreateCommand()
    $cmd.CommandText = $command
    foreach ($key in $parameters.Keys) {
        $cmd.Parameters.AddWithValue($key, $parameters[$key]) | Out-Null
    }
    $cmd.ExecuteNonQuery()
}

# Function to add a table to the database
function Add-Table([System.Data.SqlClient.SqlConnection]$connection, [String]$tableName, [String[]]$feature) {
    $createTableScript = @"
        IF OBJECT_ID('$tableName', 'U') IS NOT NULL
            DROP TABLE $tableName;
        CREATE TABLE $tableName (
            $($feature -join " VARCHAR(MAX),$([Environment]::NewLine)")
            VARCHAR(MAX)
        );
        ALTER TABLE $tableName
            ADD InsertDataTime VARCHAR(MAX);
"@
    Execute-DB-Command-With-Params $connection $createTableScript @{}
}

# Function to insert data into a table
function Insert-Data([System.Data.SqlClient.SqlConnection]$connection, [String]$tableName, [String[]]$feature, [String[]]$insertData) {
    $today = Get-Date -Format "yyyy-MM-dd_HH:mm:ss"
    $columns = $feature -join " VARCHAR(MAX),$([Environment]::NewLine)"

    $createQuery = @"
    IF OBJECT_ID('$tableName', 'U') IS NULL
    BEGIN
        CREATE TABLE $tableName (
            $columns VARCHAR(MAX),
            InsertDataTime VARCHAR(MAX)
        );
    END
"@

    $insertQuery = @"
    INSERT INTO $tableName ($($feature -join ', '), InsertDataTime)
    VALUES ('$($insertData -join "', '")', '$($today)');
"@

    $combinedScript = $createQuery + $insertQuery
    Execute-DB-Command-With-Params $connection $combinedScript @{}
}

# Install or update the required PowerShell module for Power BI management
Install-OrUpdate-Module -ModuleName "MicrosoftPowerBIMgmt"
Connect-PowerBIServiceAccount 

# Create and open a SQL connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

# Define table and feature information for workspace summary
$tableName_workspaceSummary = "dbo.workspaceSummary"
$feature_workspaceSummary = @("WorkspaceName", "workspaceId", "ReportName", "ReportId", "DatasourceName", "DatasourceId", "Type", "State", "IsReadOnly", "IsOrphaned", "IsOnDedicatedCapacity", "CapacityId", "WebURL", "EmbedUrl", "UsersCount", "UsersNames")

# If clearOldData is true, add a table to the database
if ($clearOldData) {
    Add-Table $connection $tableName_workspaceSummary $feature_workspaceSummary
}

# Retrieve all workspaces in the organization
$allWorkspaces = Get-PowerBIWorkspace -Scope Organization -Include All

# Iterate through each workspace
foreach ($workspace in $allWorkspaces) {
    # Extract workspace information
    $workspaceId = $workspace.Id
    $WorkspaceName = $workspace.Name
    $Type = $workspace.Type
    $State = $workspace.State
    $IsReadOnly = $workspace.IsReadOnly
    $IsOrphaned = $workspace.IsOrphaned
    $IsOnDedicatedCapacity = $workspace.IsOnDedicatedCapacity
    $CapacityId = $workspace.CapacityId
    $UsersCount = ($workspace.Users.Count | Measure-Object -Sum).Sum
    $UserNames = ($workspace.Users.UserPrincipalName -replace '@.*\.com') -join ', '
    $ReportCount = ($workspace.Reports | Measure-Object).Count

    # If the workspace has no reports, insert summary data
    if ($ReportCount -eq 0) {
        $data_workspaceSummary = @($WorkspaceName, $workspaceId, "", "", "", "", $Type, $State, $IsReadOnly, $IsOrphaned, $IsOnDedicatedCapacity, $CapacityId, "", "", $UsersCount, $UserNames)
        Insert-Data $connection $tableName_workspaceSummary $feature_workspaceSummary $data_workspaceSummary
    } else {
        # Iterate through each report in the workspace
        for ($j = 0; $j -lt $ReportCount; $j++) {
            # Extract report information
            if ($ReportCount -eq 1) {
                $ReportId = $workspace.Reports.Id
                $ReportName = $workspace.Reports.Name
                $WebURL = $workspace.Reports.WebUrl
                $EmbedUrl = $workspace.Reports.EmbedUrl
                $DatasetId = $workspace.Reports.DatasetId
            } else {
                $ReportId = $workspace.Reports.Id[$j]
                $ReportName = $workspace.Reports.Name[$j]
                $WebURL = $workspace.Reports.WebUrl[$j]
                $EmbedUrl = $workspace.Reports.EmbedUrl[$j]
                $DatasetId = $workspace.Reports.DatasetId[$j]
            }

            # Extract datasource information
            $DatasourceId = ""
            $DatasourceName = ""
            foreach ($datasource in (Get-PowerBIDatasource -DatasetId $DatasetId -WorkspaceId $workspaceId)) {
                $DatasourceId = $datasource.DatasourceId
                $DatasourceName = $datasource.Name
            }
            # Create data array and insert into the database
            $data_workspaceSummary = @($WorkspaceName, $workspaceId, $ReportName, $ReportId, $DatasourceName, $DatasourceId, $Type, $State, $IsReadOnly, $IsOrphaned, $IsOnDedicatedCapacity, $CapacityId, $WebURL, $EmbedUrl, $UsersCount, $UserNames)
            Insert-Data $connection $tableName_workspaceSummary $feature_workspaceSummary $data_workspaceSummary
        }
    }
}

# Close the database connection
$connection.Close()

# Display completion message
Write-Host "Completed"
```

This PowerShell script serves as a valuable tool for organizations utilizing Power BI, automating the gathering and storage of workspace and report information. By leveraging the Power BI management module and interacting with a SQL Server database, administrators can maintain an up-to-date inventory of Power BI assets, aiding in effective management and decision-making.
