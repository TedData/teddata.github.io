---
layout: post
title: "Automating Power BI Refresh Performance with PowerShell and SQL Server"
subtitle: 'PBI Workspace Metadata to SQL Server'
date: 2023-11-15
author: "Ted"
header-style: text
tags:
  - Power BI
  - PowerShell
  - SQL Server
---


# Automating Power BI Refresh Performance Data Collection and Analysis with PowerShell and SQL Server

Power BI is a powerful business intelligence tool that allows organizations to visualize and analyze their data. One critical aspect of maintaining a healthy Power BI environment is monitoring and analyzing the performance of dataset refreshes. In this article, we will explore a PowerShell script designed to automate the collection of Power BI refresh performance data and store it in a SQL Server database for further analysis.

## Script Overview

The PowerShell script provided here serves the purpose of collecting and storing Power BI refresh performance data. Let's break down its key components and functionality:

### code

```powershell
<#
.SYNOPSIS
    PowerShell script for collecting and storing Power BI refresh 
    Performance data into a SQL Server database.

.DESCRIPTION
    This script connects to the Power BI service, retrieves workspace, 
    report, and dataset information, and captures refresh history details.
    The collected data is then stored in a SQL Server database table 
    for analysis and reporting.

#>

param (
    [string]$sqlUsername = "SuperAdmin",            # SQL Server username
    [string]$sqlPassword = "**********",            # SQL Server password
    [string]$serverName = "DESKTOP-*******",       # SQL Server instance 46A7LA5
    [string]$databaseName = "PBI_Inventory",       # Database name
    [bool]$clearOldData = $false                    # Flag to determine whether to clear old data
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
            $($feature -join " VARCHAR(MAX),`n")
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
    $columns = $feature -join " VARCHAR(MAX),`n"

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

# Set up the SQL Server connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

# Define Power BI table and features
$tableName_PBI_Performance = "dbo.PBI_Performance"
$feature_PBI_Performance = @("Workspace", "Report", "Dataset", "StartTime", "EndTime", "Status", "WorkspaceId", "ReportId", "DatasetId", "CreationTime","LastTimeRefresh","RefreshCostTime")

# Create table if clearOldData flag is set
if ($clearOldData) {
    Add-Table $connection $tableName_PBI_Performance $feature_PBI_Performance
}

# Process Power BI workspaces and refresh history
ForEach ($workspace in (Get-PowerBIWorkspace)) {
    $workspaceId = $workspace.Id
    $WorkspaceName = $workspace.Name

    foreach ($report in (Get-PowerBIReport -WorkspaceId $workspaceId)) {
        $ReportId = $report.Id
        $ReportName = $report.Name

        foreach ($dataset in (Get-PowerBIDataset -Id $report.DatasetId -WorkspaceId $workspaceId)) {
            $DatasetName = $dataset.Name
            Write-Host $DatasetName

            $DatasetId = $dataset.Id
            $refreshUri = "groups/$workspaceId/datasets/$($dataset.Id)/refreshes"
            $refreshHistory = Invoke-PowerBIRestMethod -Url $refreshUri -Method Get | ConvertFrom-Json
            $CreateTime = $refreshHistory.value[0].StartTime
            $LastTimeRefresh = $refreshHistory.value[$refreshHistory.value.Length-1].EndTime

            foreach ($refreshEntry in $refreshHistory.value) {
                $Status = $refreshEntry.Status
                $startTime = [DateTime]::Parse($refreshEntry.StartTime)
                $endTime = [DateTime]::Parse($refreshEntry.EndTime)
                $timeDifferenceInSeconds = [math]::Round(($endTime - $startTime).TotalSeconds)
                $formattedTimeDifference = "$($timeDifferenceInSeconds)s"
                
                $data_PBI_Performance = @($WorkspaceName, $ReportName, $DatasetName, $refreshEntry.StartTime, $refreshEntry.EndTime, $Status, $workspaceId, $ReportId, $DatasetId, $CreateTime, $LastTimeRefresh, $formattedTimeDifference)
                Insert-Data $connection $tableName_PBI_Performance $feature_PBI_Performance $data_PBI_Performance
            }
        }
    }
}

# Close the database connection
$connection.Close()

Write-Host "Completed"
```

### Parameters

The script utilizes parameters to allow customization of the SQL Server connection details and control whether old data should be cleared before inserting new data. These parameters include SQL Server credentials, server name, database name, and a flag for clearing old data.

### Module Installation

The script checks for the presence of the required PowerShell module, `MicrosoftPowerBIMgmt`, and either installs or updates it if necessary. This module is essential for interacting with the Power BI service.

### SQL Server Connection

The script establishes a connection to a SQL Server database using the provided credentials. It then defines functions for executing SQL commands and managing the database tables.

### Power BI Performance Table

The script defines a table structure (`$tableName_PBI_performance`) and features (`$feature_PBI_performance`) to store Power BI refresh performance data. It includes workspace, report, and dataset information, along with details like start time, end time, status, and refresh duration.

### Main Processing Loop

The script iterates through Power BI workspaces, reports, and datasets, querying refresh history details for each dataset. It calculates refresh duration and formats the data for insertion into the SQL Server table.

### Data Insertion

The script uses functions to create the table if the `clearOldData` flag is set and inserts the collected data into the SQL Server table.

### Conclusion

After processing all Power BI workspaces, reports, and datasets, the script closes the database connection and concludes. The result is a SQL Server table populated with valuable Power BI refresh performance data.

## Conclusion

Automating the collection of Power BI refresh performance data is crucial for monitoring the health of your Power BI environment. This PowerShell script provides a scalable and efficient solution to gather, store, and analyze refresh performance data in a SQL Server database. By leveraging the Power BI management module and SQL Server connectivity, organizations can gain insights into refresh trends, identify potential issues, and make informed decisions to optimize their Power BI workflows.
