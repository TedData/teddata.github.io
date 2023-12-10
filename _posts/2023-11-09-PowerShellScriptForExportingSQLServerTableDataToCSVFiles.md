---
layout: post
title: "Export SQL Server Management to Csv"
subtitle: 'SQL Server to CSV'
date: 2023-11-09
author: "Ted"
header-style: text
tags:
  - SQL Server
  - PowerShell
---

# PowerShell Script for Exporting SQL Server Table Data to CSV Files

In everyday data management tasks, exporting data from specific SQL Server tables to CSV files is a common requirement. To simplify this process, we can use the following PowerShell script. The script connects to a SQL Server database and exports data from specified tables to CSV files, allowing for the export of data from specific tables or all tables in the database based on provided parameters.

## Script Overview

### Functions

- `Get-SqlConnection`: Function to establish a connection with SQL Server.
- `Export-TableData`: Function to export data from a table to a CSV file.

### Execution Process

1. Establish a connection with SQL Server using provided parameters.
2. If the `$DownloadAllTable` flag is set, retrieve the list of all tables in the database.
3. Build the folder path for exported CSV files and create the directory if it doesn't exist.
4. Iterate through each specified table and export data using the `Export-TableData` function.

## Code

```powershell
<#
.SYNOPSIS
    This PowerShell script exports data from specified SQL Server tables to CSV files.
.DESCRIPTION
    The script connects to a SQL Server database and exports data from specified tables to CSV files.
    The script can export data from specific tables or all tables in the database based on the provided parameters.
#>

# Define script parameters with default values
param (
    [string]$sqlUsername = "SuperAdmin",                        
    [string]$sqlPassword = "**********",                        
    [string]$serverName = "DESKTOP-*******",                    
    [string]$databaseName = "PBI_Inventory",                   
    [string]$outputPath = "C:\Users\ted\Downloads",        
    [String[]]$tables = @("Users", "Refreshes"),               
    [bool]$DownloadAllTable = $false                             
)

# Function to establish a SQL Server connection
function Get-SqlConnection {
    param (
        [string]$server,        
        [string]$database,      
        [string]$username,      
        [string]$password       
    )

    $connStr = "Server=$server;Database=$database;User Id=$username;Password=$password"
    $connection = New-Object System.Data.SqlClient.SqlConnection($connStr)
    $connection.Open()
    
    return $connection
}

# Function to export data from a table to a CSV file
function Export-TableData {
    param (
        [System.Data.SqlClient.SqlConnection]$connection,  
        [string]$table,                                    
        [string]$outputPath                                
    )

    $outputFile = Join-Path $outputPath "$($table).csv"
    $query = "SELECT * FROM $table"
    
    $command = $connection.CreateCommand()
    $command.CommandText = $query
    
    $dataAdapter = New-Object System.Data.SqlClient.SqlDataAdapter $command
    $dataTable = New-Object System.Data.DataTable
    $dataAdapter.Fill($dataTable)
    
    $dataTable | Export-Csv -Path $outputFile -NoTypeInformation
    Write-Host "Exported table $table to $outputFile."
}

$TodaysDate = Get-Date -Format "yyyyMMdd"

try {
    $connection = Get-SqlConnection -server $serverName -database $databaseName -username $sqlUsername -password $sqlPassword

    if ($DownloadAllTable) {
        $queryTable = "SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE'"
        $tables = Invoke-Sqlcmd -ServerInstance $serverName -Database $databaseName -Username $sqlUsername -Password $sqlPassword -Query $queryTable -TrustServerCertificate | Select-Object -ExpandProperty TABLE_NAME
    }

    $folderPath = Join-Path $outputPath "$($databaseName)_$TodaysDate"

    if (-not (Test-Path $folderPath)) {
        New-Item -ItemType Directory -Path $folderPath
    }

    foreach ($table in $tables) {
        Export-TableData -connection $connection -table $table -outputPath $folderPath
    }
}
catch {
    Write-Host "Error: $_"
}
finally {
    if ($connection.State -eq 'Open') {
        $connection.Close()
    }
}
```

## Conclusion

With this PowerShell script, users can easily export data from specified SQL Server tables to CSV files, providing a quick backup and sharing solution. The script's flexibility allows users to selectively export data from specific tables or download all tables at once, enhancing its usability in various scenarios.
