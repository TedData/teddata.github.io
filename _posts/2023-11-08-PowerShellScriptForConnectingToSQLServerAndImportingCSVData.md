---
layout: post
title: "Import CSV To SQL"
date: 2023-11-08
author: "Ted"
header-style: text
tags:
  - PowerShell
  - SQL Server
---


# PowerShell Script for Connecting to SQL Server and Importing CSV Data

In this article, we'll explore a PowerShell script designed to connect to a SQL Server database, read data from a CSV file, and insert or update the data in a corresponding table. The script provides flexibility through various parameters, allowing users to customize SQL Server connection details, file paths, and options to clear old data.

## Script Overview

The script begins with a detailed comment-based help section, including a synopsis and description of its functionality. The main functionalities of the script are as follows:

1. *Functions:*
   - `Execute-DB-Command-With-Params`: Executes a SQL command with parameters.
   - `Add-Table`: Adds a table to the database based on specified features.
   - `Insert-Data`: Inserts data into a specified table.

2. *Main Script Execution:*
   - Establishes a connection to the SQL Server using provided credentials.
   - Determines the table name from the CSV file name.
   - Constructs the full file path for the CSV file.
   - Imports CSV data using the `Import-Csv` cmdlet.
   - Extracts column names from the first row of the CSV data.

3. *Table Management:*
   - If `$clearOldData` is specified, it adds a new table to clear old data.
   - Inserts each row of data into the database table.

4. *Completion Message:*
   - Displays a completion message once the process is finished.

## Code

```powershell
<#
.SYNOPSIS
    This PowerShell script connects to a SQL Server database, 
    reads data from a CSV file, and inserts or updates the data 
    in a corresponding table. It provides flexibility through 
    parameters such as SQL server credentials, file paths, 
    and options to clear old data.

.DESCRIPTION
    The script begins by defining parameters that allow customization 
    of SQL Server connection details, CSV file paths, and more. 
    It then includes functions for executing SQL commands 
    and managing database tables.

    The main functionalities include:
    1. Creating a new table or updating an existing one based on CSV file data.
    2. Importing CSV data and inserting it into the database table.
    3. Optionally clearing old data by creating a new table.
    4. Displaying a completion message when the process is finished.

#>

param (
    [string]$sqlUsername = "SuperAdmin",
    [string]$sqlPassword = "**********",
    [string]$serverName = "DESKTOP-*******",
    [string]$databaseName = "PBI_Inventory",
    [string]$csvFilePath = "C:\Users\ted\Downloads\",
    [string]$csvFileName = "workspaceSummary.csv",
    [bool]$clearOldData = $false
)

# Function to execute a SQL command with parameters
function Execute-DB-Command-With-Params {
    param(
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$command,
        [Hashtable]$parameters
    )

    $cmd = $connection.CreateCommand()
    $cmd.CommandText = $command
    foreach ($key in $parameters.Keys) {
        $cmd.Parameters.AddWithValue($key, $parameters[$key]) | Out-Null
    }
    $cmd.ExecuteNonQuery()
}

# Function to add a table to the database
function Add-Table {
    param(
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$tableName,
        [String[]]$feature
    )

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
function Insert-Data {
    param(
        [System.Data.SqlClient.SqlConnection]$connection,
        [String]$tableName,
        [String[]]$feature,
        [String[]]$insertData
    )

    $today = Get-Date -Format "yyyy-MM-dd_HH:mm:ss"
    $columns = $feature -join " VARCHAR(MAX),$([Environment]::NewLine)"

    $combinedScript = @"
        IF OBJECT_ID('$tableName', 'U') IS NULL
        BEGIN
            CREATE TABLE $tableName (
                $columns VARCHAR(MAX),
                InsertDataTime VARCHAR(MAX)
            );
        END

        INSERT INTO $tableName ($($feature -join ', '), InsertDataTime)
        VALUES ('$($insertData -join "', '")', '$($today)');
"@
    Execute-DB-Command-With-Params $connection $combinedScript @{}
}

# Create a new SQL connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

# Extract table name from CSV file name
$tableName = $csvFileName -replace '(?s)(.*)\.csv.*', '$1'

# Construct full file path
$csvFilePath = Join-Path $csvFilePath $csvFileName

# Import CSV data
$data = Import-Csv -Path $csvFilePath

# Extract column names from the first row of the CSV data
$feature = $data[0].PSObject.Properties.Name

# If specified, clear old data by adding a new table
if ($clearOldData) {
    Add-Table -connection $connection -tableName $tableName -feature $feature
}

# Insert each row of data into the database
foreach ($row in $data) {
    $insertData = $row.PSObject.Properties.Value
    Insert-Data -connection $connection -tableName $tableName -feature $feature -insertData $insertData
}

# Close the database connection
$connection.Close()

# Display completion message
Write-Host "Completed"
```

This PowerShell script combines clarity and flexibility, making it a valuable tool for managing SQL Server data from CSV files. Users can easily adapt it to their specific needs by adjusting the provided parameters.
