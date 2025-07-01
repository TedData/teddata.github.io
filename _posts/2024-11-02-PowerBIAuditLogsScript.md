---
layout: post
title: "Power BI Audit Logs Script"
subtitle: 'PowerShell Script to Ingest Power BI Logs into Azure SQL'
date: 2024-11-02
author: "Ted"
header-style: text
tags:
  - PowerShell
  - Power BI
---

## 🚀 Automating Power BI Activity Log Ingestion into Azure SQL Database with PowerShell

When managing Power BI at scale, monitoring user activities is crucial for governance, security, and usage analytics. However, manually exporting and managing audit logs is inefficient and error-prone. This blog post introduces a **PowerShell-based automation** that retrieves Power BI activity events, processes them, and stores them directly into an Azure SQL Database—providing a robust foundation for building enterprise-grade Power BI monitoring dashboards.

---

### 🎯 What Does This Script Do?

This PowerShell script automates:
✅ Connecting to the Power BI Service via `Connect-PowerBIServiceAccount`
✅ Iterating through a date range (last 27 days) to collect daily Power BI activity events
✅ Formatting and standardizing activity data, including adding calculated columns like `RetrieveDate` and `CreateDate`
✅ Creating or resetting a SQL table (optional, controlled by the `$clearOldData` parameter)
✅ Inserting activity log data into an Azure SQL Database table named `tech.PbiInsightMonitor`

---

## 🛠️ The Full PowerShell Script

```powershell
param (
    [string]$sqlUsername = "********", 
    [string]$sqlPassword = "********", 
    [string]$serverName = "********.database.windows.net", 
    [string]$databaseName = "****", 
    [bool]$clearOldData = $false
)

# Connect to the Power BI service account
Connect-PowerBIServiceAccount 

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

    $combinedScript = @"
        IF NOT EXISTS (SELECT 1 FROM $tableName WHERE Id = '$($insertData[0])')
        BEGIN
            INSERT INTO $tableName ($($feature -join ', '))
            VALUES ('$($insertData -join "', '")');
        END
"@
    Execute-DB-Command-With-Params $connection $combinedScript @{}
}

# Create a new SQL connection
$connection = New-Object System.Data.SqlClient.SqlConnection
$connection.ConnectionString = "Data Source=$serverName;Initial Catalog=$databaseName;User Id=$sqlUsername;Password=$sqlPassword;"
$connection.Open()

$tableName = "tech.PbiInsightMonitor"

$feature = @("Id", "RecordType", "CreationTime", "Operation", "OrganizationId", "UserType", "UserKey", "Workload", "UserId", "ClientIP", "UserAgent", "Activity", "ItemName", "WorkspaceName", "DatasetName", "ReportName", "WorkspaceId", "CapacityId", "CapacityName", "AppName", "ObjectId", "DatasetId", "ReportId", "IsSuccess", "ReportType", "RequestId", "ActivityId", "AppReportId", "DistributionMethod", "ConsumptionMethod", "RetrieveDate", "CreateDate")

if ($clearOldData) {
    Add-Table -connection $connection -tableName $tableName -feature $feature
}

# Get the current date and time for reference
$retrieveDate = Get-Date 

# Convert start and end date strings to DateTime objects
$startDate = $retrieveDate.AddDays(-27)
$endDate = $retrieveDate

# Initialize the loop with the start date
$currentDate = $startDate
$activityLog = @()

# Loop through each day in the specified date range
while ($currentDate -le $endDate) {
    # Format the current date to create the start and end datetime strings
    $dateStr = $currentDate.ToString("yyyy-MM-dd")
    Write-Host $dateStr
    $startDt = $dateStr + 'T00:00:00.000'
    $endDt = $dateStr + 'T23:59:59.999'

    # Define parameters for retrieving Power BI activity logs
    $activityLogsParams = @{
        StartDateTime = $startDt
        EndDateTime   = $endDt
    }

    $activityLogs = Get-PowerBIActivityEvent @activityLogsParams | ConvertFrom-Json
    $activityLogSchema = $activityLogs | Select-Object Id, RecordType, CreationTime, Operation, OrganizationId, UserType, UserKey, Workload, UserId, ClientIP, UserAgent, Activity, ItemName, WorkspaceName, DatasetName, ReportName, WorkspaceId, CapacityId, CapacityName, AppName, ObjectId, DatasetId, ReportId, IsSuccess, ReportType, RequestId, ActivityId, AppReportId, DistributionMethod, ConsumptionMethod, `
        @{Name="RetrieveDate"; Expression={$retrieveDate.ToString("yyyy-MM-ddThh:mm:ss")}}, `
        @{Name="CreateDate"; Expression = { ([datetime]$_.CreationTime).ToString("MM/dd/yyyy") }}

    # Add the activity log data to the array
    $activityLog += $activityLogSchema
     
    # Move to the next day
    $currentDate = $currentDate.AddDays(1)
}

foreach ($row in $activityLog) {
    $insertData = $row.PSObject.Properties.Value
    Insert-Data -connection $connection -tableName $tableName -feature $feature -insertData $insertData
}
```

---

### 📊 Why Use This Automation?

* **Scalability**: Handles large volumes of daily activity logs automatically.
* **Consistency**: Avoids manual errors by standardizing ingestion and schema.
* **Integration**: Outputs data directly to Azure SQL Database, making it easy to build Power BI or other BI dashboards on top of the ingested logs.
* **Governance & Compliance**: Provides a central, queryable store of all Power BI activities for security audits and user adoption tracking.

---

### ✅ Example Use Cases

🔹 Build a Power BI dashboard to visualize trends in report usage, dataset refreshes, or suspicious activities
🔹 Generate automated compliance reports for IT or data governance teams
🔹 Identify underutilized workspaces, datasets, or reports to optimize licensing and reduce costs

---

### 🚨 Final Thoughts

This script bridges the gap between Power BI audit logs and enterprise data monitoring. By directly pushing activity events into a SQL database, organizations can unlock powerful insights and maintain control over their Power BI environment. Feel free to adapt this script for your organization’s needs—whether by customizing the table schema, adding error handling, or integrating alerting mechanisms.
