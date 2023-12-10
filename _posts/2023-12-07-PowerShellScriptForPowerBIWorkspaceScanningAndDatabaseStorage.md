---
layout: post
title: "Power BI Scanner"
subtitle: 'Power BI Scanner to SQL Server'
date: 2023-12-07
author: "Ted"
header-style: text
mathjax: true
tags:
  - PowerShell
  - SQL Server
---

# PowerShell Script for Power BI Workspace Scanning and Database Storage

The following PowerShell script serves as a robust solution for scanning Power BI workspaces, extracting details about dashboards, reports, datasets, tables, columns, and measures, and efficiently storing this information in a SQL Server database.

```powershell
<#
.SYNOPSIS
    PowerShell script for scanning Power BI workspaces, dashboards, reports, 
    datasets, tables, columns, and measures. Retrieves details about these 
    elements and stores the information in a SQL Server database.

.PARAMETER sqlUsername
    Specifies the SQL Server username.

.PARAMETER sqlPassword
    Specifies the SQL Server password.

.PARAMETER serverName
    Specifies the SQL Server name.

.PARAMETER databaseName
    Specifies the name of the SQL Database.

.PARAMETER clearOldData
    Specifies whether to clear existing data in tables before inserting new data.

#>

param (
    [string]$sqlUsername = "SuperAdmin",
    [string]$sqlPassword = "**********",
    [string]$serverName = "DESKTOP-*******",
    [string]$databaseName = "PBI_Inventory",
    [bool]$clearOldData = $true
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

# Function to execute a SQL command
function Execute-DB-Command([System.Data.SqlClient.SqlConnection]$connection, [String]$command) {
    $cmd = $connection.CreateCommand()
    $cmd.CommandText = $command
    $cmd.ExecuteNonQuery()
}

# Function to add a table to the database
function Add-Table {
    param (
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$tableName,
        [String[]]$feature
    )

    $createTableScript = @"
        IF OBJECT_ID('$tableName', 'U') IS NOT NULL
            DROP TABLE $tableName;
        CREATE TABLE $tableName (
            $($feature -join " VARCHAR(MAX),`n")
            VARCHAR(MAX)
        );
"@
    Execute-DB-Command $connection $createTableScript
}

# Function to insert data into a table
function Insert-Data {
    param (
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$tableName,
        [String[]]$feature,
        [String[]]$insertData
    )

    $columns = $feature -join " VARCHAR(MAX),`n"

    $insertQuery = @"
    INSERT INTO $tableName ($($feature -join ', '))
    VALUES ('$($insertData -join "', '")');
"@

    Execute-DB-Command $connection $insertQuery
}

# Install or update the required PowerShell module for Power BI management
Install-OrUpdate-Module -ModuleName "MicrosoftPowerBIMgmt"
Connect-PowerBIServiceAccount

# Set up the SQL Server connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

$dataset_unique = @() 

# Define table names and features for database
$tableName_Dashboards = "dbo.PBI_ScannerDashboards"
$feature_Dashboards = @("workspaceId","dashboardsId","displayName","isReadOnly","tiles_id","tiles_title","tiles_reportId","tiles_datasetId","dashboard_sensitivityLabel")

$tableName_ReportsAndDatasets = "dbo.PBI_ScannerReportsAndDatasets"
$feature_ReportsAndDatasets = @("workspaceId","workspaceName","reportId","reportName","datasetsId","datasetsName","datasourceId","gatewayId","createdBy","createdDate_Dataset","createdDateTime","createdById","modifiedBy","modifiedDateTime","modifiedById","type","state","isOnDedicatedCapacity","isEffectiveIdentityRequired","isEffectiveIdentityRolesRequired","reportType","datasourceType","targetStorageMode","configuredBy","configuredById","contentProviderType","connectionDetails","endorsement","endorsement_certifiedBy","report_sensitivityLabel")

$tableName_TablesAndColumns = "dbo.PBI_ScannerTablesAndColumns"
$feature_TablesAndColumns =@("datasetsId","datasetsName","tablesName","tableDescription","columns_name","columns_dataType","isHidden","columns_type","source")

$tableName_Measures = "dbo.PBI_ScannerMeasures"
$feature_Measures =@("datasetsId","datasetsName","tablesName","measureName","measureDescription","measureExpression")

# Create tables in the database if clearOldData is true
if ($clearOldData) {
    Add-Table $connection $tableName_Dashboards $feature_Dashboards
    Add-Table $connection $tableName_ReportsAndDatasets $feature_ReportsAndDatasets
    Add-Table $connection $tableName_TablesAndColumns $feature_TablesAndColumns
    Add-Table $connection $tableName_Measures $feature_Measures
}

# Get all workspaces in the organization
$allWorkspaces = Get-PowerBIWorkspace -Scope Organization

# Loop through each workspace
ForEach ($workspace in ($allWorkspaces)) {
    # Prepare the request body for scanning workspace details
    $body = "{'workspaces': ['$($workspace.Id)']}"
    $url_getscanId = "https://api.powerbi.com/v1.0/myorg/admin/workspaces/getInfo?lineage=True&datasourceDetails=True&datasetSchema=True&datasetExpressions=True"
    $GetScanId = Invoke-PowerBIRestMethod -Url $url_getscanId -Method Post -Body $body | ConvertFrom-Json

    # Get the scan results for the workspace
    $url = "https://api.powerbi.com/v1.0/myorg/admin/workspaces/scanResult/$($GetScanId.id)"
    $GetScanResult = Invoke-PowerBIRestMethod -Url $url -Method Get | ConvertFrom-Json

    # Extract workspace information
    $workspaceId = $GetScanResult.workspaces.id
    $workspaceName = $GetScanResult.workspaces.name 
    $type = $GetScanResult.workspaces.type 
    $state = $GetScanResult.workspaces.state 
    $isOnDedicatedCapacity = $GetScanResult.workspaces.isOnDedicatedCapacity

    Write-Host $workspaceName

    # Process dashboards in the workspace
    if ($GetScanResult.workspaces.dashboards.Length -ne 0){
        foreach ($dashboard in $GetScanResult.workspaces.dashboards){
            $dashboardsId = $dashboard.id
            $displayName = $dashboard.displayName
            $isReadOnly = $dashboard.isReadOnly
            $dashboard_sensitivityLabel_labelId = $dashboard.sensitivityLabel.labelId

            if ($dashboard.tiles.Length -ne 0){
                # Process tiles in the dashboard
                foreach ($tile in $dashboard.tiles){
                    $tiles_id = $tile.id
                    $tiles_title = $tile.title
                    $tiles_reportId = $tile.reportId
                    $tiles_datasetId = $tile.datasetId 

                    # Create an array to store dashboard data
                    $data_Dashboards = @($workspaceId,$dashboardsId,$displayName,$isReadOnly,$tiles_id,$tiles_title,$tiles_reportId,$tiles_datasetId,$dashboard_sensitivityLabel_labelId)
                    # Insert dashboard data into the database
                    Insert-Data $connection $tableName_Dashboards $feature_Dashboards $data_Dashboards
                }
            } 
            else {
                # If no tiles in the dashboard
                $ScannerDashboardsInfo = [PSCustomObject]@{
                    "workspaceId" = $workspaceId
                    "dashboardsId" = $dashboardsId
                    "displayName" = $displayName
                    "isReadOnly" = $isReadOnly
                    "tiles_id" = ""
                    "tiles_title" = ""
                    "tiles_reportId" = ""
                    "tiles_datasetId" = ""
                    "dashboard_sensitivityLabel" = $dashboard_sensitivityLabel_labelId
                }

                $data_Dashboards = @($workspaceId,$dashboardsId,$displayName,$isReadOnly,"","","","",$dashboard_sensitivityLabel_labelId)
                # Insert dashboard data into the database
                Insert-Data $connection $tableName_Dashboards $feature_Dashboards $data_Dashboards
            }
        }
    }

    # Process reports in the workspace
    foreach ($report in $GetScanResult.workspaces.reports){
        $reportId = $report.id
        $reportName = $report.name
        $report_datasetsId = $report.datasetId
        $createdDateTime = $report.createdDateTime
        $modifiedDateTime = $report.modifiedDateTime
        $modifiedBy = $report.modifiedBy
        $reportType = $report.reportType
        $createdBy = $report.createdBy
        $modifiedById = $report.modifiedById
        $createdById = $report.createdById
        $endorsement = $report.endorsementDetails.endorsement
        $endorsement_certifiedBy = $report.endorsementDetails.certifiedBy
        $report_sensitivityLabel = $report.sensitivityLabel.labelId

        # Handle the case where there are no datasets associated with the report
        if ($report_datasetsId.Count -eq 0){
            $report_datasetsId = "wrong_report_dataset_Id"
        }

        # Loop through each dataset associated with the report
        foreach ($report_datasetId in $report_datasetsId){
            $getDatasets = $GetScanResult.workspaces.datasets | Where-Object { $_.id -eq $report_datasetId}

            # Handle the case where there are no datasets found
            if ($getDatasets.count -eq 0){
                $getDatasets = "wrong_dataset"
            }

            # Loop through each dataset
            foreach ($dataset in $getDatasets){
                $datasetsId = $dataset.id
                $datasetsName = $dataset.name
                $configuredBy = $dataset.configuredBy
                $configuredById = $dataset.configuredById
                $isEffectiveIdentityRequired = $dataset.isEffectiveIdentityRequired
                $isEffectiveIdentityRolesRequired = $dataset.isEffectiveIdentityRolesRequired
                $targetStorageMode = $dataset.targetStorageMode
                $createdDate_Dataset = $dataset.createdDate
                $contentProviderType = $dataset.contentProviderType
                $datasourceUsages_datasourceInstanceId = $dataset.datasourceUsages.datasourceInstanceId

                # Handle the case where there are no datasource instances associated with the dataset
                if ($datasourceUsages_datasourceInstanceId.Count -eq 0){
                    $datasourceUsages_datasourceInstanceId = "wrong_datasourceInstance_Id"
                }

                # Loop through each datasource instance
                foreach ($datasourceInstanceId in $datasourceUsages_datasourceInstanceId) {
                    $getDatasources = $GetScanResult.datasourceInstances | Where-Object { $_.datasourceId -eq $datasourceInstanceId }

                    # Handle the case where there are no datasource instances found
                    if ($getDatasources.count -eq 0){
                        $getDatasources = "wrong_datasource"
                    }

                    # Loop through each datasource
                    foreach ($datasource in $getDatasources){
                        $datasourceId = $datasource.datasourceId
                        $datasourceType = $datasource.datasourceType
                        $connectionDetails = $datasource.connectionDetails
                        $gatewayId = $datasource.gatewayId

                        $data_ReportsAndDatasets = @($workspaceId,$workspaceName,$reportId,$reportName, $datasetsId,$datasetsName,$datasourceId,$gatewayId,$createdBy,$createdDate_Dataset,$createdDateTime,$createdById,$modifiedBy,$modifiedDateTime,$modifiedById,$type,$state,$isOnDedicatedCapacity,$isEffectiveIdentityRequired,$isEffectiveIdentityRolesRequired,$reportType,$datasourceType,$targetStorageMode,$configuredBy,$configuredById,$contentProviderType,$connectionDetails,$endorsement,$endorsement_certifiedBy,$report_sensitivityLabel)
                        # Insert report and dataset information into the database
                        Insert-Data $connection $tableName_ReportsAndDatasets $feature_ReportsAndDatasets $data_ReportsAndDatasets
                    }
                }

                if ($dataset_unique -notcontains $datasetsId) {
                    $dataset_unique += $datasetsId

                    # Process tables in the dataset
                    if ($dataset.tables.Length -ne 0){
                        foreach ($table in $dataset.tables){
                            $tablesName = $table.name
                            $source = $table.source.expression -replace '\s{2,}', ' ' -replace '"', "``" -replace "'", "``"
                            $isHidden = $table.isHidden
                            $tableDescription = $table.description

                            # Process measures in the table
                            if ($table.measures.Length -ne 0){
                                foreach ($measure in $table.measures){
                                    $measureName = $measure.name
                                    $measureExpression = $measure.expression -replace '\s{2,}', ' ' -replace '"', "``" -replace "'", "``"
                                    $measureDescription = $measure.description

                                    $data_Measures = @($datasetsId,$datasetsName,$tablesName,$measureName, $measureDescription,$measureExpression)
                                    # Insert measure information into the database
                                    Insert-Data $connection $tableName_Measures $feature_Measures $data_Measures
                                }
                            }

                            # Process columns in the table
                            if ($table.columns.Length -ne 0){
                                foreach ($column in $table.columns){
                                    $columnsName = $column.name
                                    $columnsDataType = $column.dataType
                                    $columnsIsHidden = $column.isHidden
                                    $columnsType = $column.columnType

                                    $data_TablesAndColumns = @($datasetsId,$datasetsName,$tablesName,$tableDescription,$columnsName,$columnsDataType,$columnsIsHidden,$columnsType,$source)
                                    # Insert table and column information into the database
                                    Insert-Data $connection $tableName_TablesAndColumns $feature_TablesAndColumns $data_TablesAndColumns
                                }
                            } 
                            # If no columns in the table
                            else {
                                $data_TablesAndColumns = @($datasetsId,$datasetsName,$tablesName,$tableDescription,"","",$isHidden,"",$source)
                                # Insert table and column information into the database
                                Insert-Data $connection $tableName_TablesAndColumns $feature_TablesAndColumns $data_TablesAndColumns
                            }
                        }
                    }
                }
            }
        }
    }
}

# Close the database connection
$connection.Close()
Write-Host "Completed"
```

## Script Parameters

The script is highly customizable through parameters:

- sqlUsername: Specifies the SQL Server username.
- sqlPassword: Specifies the SQL Server password.
- serverName: Specifies the SQL Server name.
- databaseName: Specifies the name of the SQL Database.
- clearOldData: Specifies whether to clear existing data in tables before inserting new data.

## Essential Functions

### Install-OrUpdate-Module

This function ensures the presence of the required PowerShell module, "MicrosoftPowerBIMgmt," by either installing or updating it.

### Execute-DB-Command

Executes a specified SQL command on the provided SQL connection.

### Add-Table

Adds a table to the SQL Server database, handling table creation and dropping if it already exists.

### Insert-Data

Inserts data into a specified table in the SQL Server database, utilizing dynamic SQL queries.

## Code Execution Flow

1. *Module Installation and Connection Setup:* The script begins by checking and installing/updating the necessary PowerShell module for Power BI management. It then establishes a connection to the Power BI service using Connect-PowerBIServiceAccount.

2. *SQL Server Connection:* The script creates a connection to the SQL Server using the provided parameters, including username, password, server name, and database name.

3. *Table Initialization:* If clearOldData is set to true, the script creates tables in the SQL Server database for dashboards, reports and datasets, tables and columns, and measures.

4. *Workspace Iteration:* The script retrieves all Power BI workspaces within the organization and iterates through each.

5. *Workspace Details Retrieval:* For each workspace, the script constructs a request to retrieve detailed information about dashboards, reports, and datasets. It then processes the obtained data.

6. *Data Extraction and Storage:* The script extracts workspace, dashboard, report, dataset, table, column, and measure details, organizing the data into arrays. These arrays are then inserted into their respective SQL Server tables using the defined functions.

7. *Efficient Dataset Handling:* The script employs a mechanism ($dataset_unique) to uniquely identify datasets, preventing redundant processing and enhancing overall efficiency.

8. *Database Connection Closure:* Once all workspaces are processed, the script closes the connection to the SQL Server database.

9. *Completion Message:* The script concludes by printing a completion message.

This PowerShell script provides organizations with a powerful tool for maintaining an up-to-date inventory of their Power BI assets, facilitating effective management and analysis of Power BI workspaces.
