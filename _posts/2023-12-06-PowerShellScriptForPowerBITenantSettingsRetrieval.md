---
layout: post
title: "PowerBI Tenant Settings Retrieval"
subtitle: 'Using markdown to make a post'
date: 2023-12-06
author: "Ted"
header-style: text
mathjax: true
tags:
  - PowerShell
  - SQL Server
---

# PowerShell Script for Power BI Tenant Settings Retrieval

The following PowerShell script is designed to retrieve Power BI tenant settings from a specified API endpoint and store the information in a SQL Server database. This script makes use of functions to handle tasks such as installing or updating required PowerShell modules, executing SQL commands, adding tables to the database, and inserting data into those tables.

```powershell
<#
.SYNOPSIS
This PowerShell script retrieves Power BI tenant settings from the specified API 
endpoint and stores the information in a SQL Server database. It utilizes functions 
to install or update required PowerShell modules, execute SQL commands, add tables 
to the database, and insert data into tables.

.DESCRIPTION
The script connects to the Power BI service using the MicrosoftPowerBIMgmt module, 
retrieves tenant settings from the API endpoint "https://api.fabric.microsoft.com/
v1/admin/tenantsettings," and stores the data in a SQL Server database. 
The script includes functions for module management, database interaction, 
and table operations.

#>

param (
    [string]$sqlUsername = "SuperAdmin",
    [string]$sqlPassword = "**********",
    [string]$serverName = "DESKTOP-*******",
    [string]$databaseName = "PBI_Inventory",
    [bool]$clearOldData = $true
)

# Function to install or update a PowerShell module
function Install-OrUpdate-Module {
    param (
        [string]$ModuleName
    )

    $module = Get-Module $ModuleName -ListAvailable -ErrorAction SilentlyContinue
    if (!$module -or ($module.Version -ne '1.0.0' -and $module.Version -le '1.0.410')) {
        Write-Host "Installing or updating module $ModuleName ..."
        Install-Module -Name $ModuleName -Force -Scope CurrentUser
        Write-Host "Module $ModuleName installation or update complete"
    }
}

# Function to execute a SQL command on the provided connection
function Execute-DB-Command {
    param (
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$command
    )

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
Login-PowerBI

# Set up the SQL Server connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

# Define tables and their features
$tables = @(
    @{
        Name = "dbo.PBI_TenantSettings"
        Features = @("settingName", "title", "enabled", "canSpecifySecurityGroups", "tenantSettingGroup")
    },
    @{
        Name = "dbo.PBI_TenantSettingEnabledSecurityGroups"
        Features = @("settingName", "enabledSecurityGroups_graphId", "enabledSecurityGroups_name")
    },
    @{
        Name = "dbo.PBI_TenantSettingProperties"
        Features = @("settingName", "name", "value", "type")
    }
)

# If specified, clear old data by dropping and recreating tables
if ($clearOldData) {
    foreach ($table in $tables) {
        Add-Table -connection $connection -tableName $table.Name -feature $table.Features
    }
}

# Retrieve Power BI tenant settings from the specified API endpoint
$tenantsettings = Invoke-PowerBIRestMethod -Url https://api.fabric.microsoft.com/v1/admin/tenantsettings -Method GET | ConvertFrom-Json

# Iterate through each tenant setting and insert data into corresponding tables
foreach ($tenantsetting in $tenantsettings.tenantSettings) {
    $settingName = $tenantsetting.settingName
    Write-Host "Loading $($settingName)"

    # Insert data into PBI_TenantSettings table
    $data_tenantSetting = @($tenantsetting.settingName, $tenantsetting.title, $tenantsetting.enabled, $tenantsetting.canSpecifySecurityGroups, $tenantsetting.tenantSettingGroup)
    Insert-Data -connection $connection -tableName $tables[0].Name -feature $tables[0].Features -insertData $data_tenantSetting

    # Insert data into PBI_TenantSettingEnabledSecurityGroups table for each enabled security group
    foreach ($enabledSecurityGroup in $tenantsetting.enabledSecurityGroups) {
        $data_enabledSecurityGroups = @($settingName, $enabledSecurityGroup.graphId, $enabledSecurityGroup.name)
        Insert-Data -connection $connection -tableName $tables[1].Name -feature $tables[1].Features -insertData $data_enabledSecurityGroups
    }

    # Insert data into PBI_TenantSettingProperties table for each property
    foreach ($property in $tenantsetting.properties) {
        $data_properties = @($settingName, $property.name, $property.value, $property.type)
        Insert-Data -connection $connection -tableName $tables[2].Name -feature $tables[2].Features -insertData $data_properties
    }
}

# Close the database connection
$connection.Close()

# Display completion message
Write-Host "Complete"
```

This PowerShell script serves the purpose of seamlessly retrieving Power BI tenant settings and storing them in a SQL Server database. The modular structure enhances its readability and maintainability, making it a valuable tool for Power BI administrators.
