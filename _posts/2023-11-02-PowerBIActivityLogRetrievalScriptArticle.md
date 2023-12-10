---
layout: post
title: "PowerBI ActivityLog Retrieval Script Article"
subtitle: 'PBI Activity Logs to CSV'
date: 2023-11-02
author: "Ted"
header-style: text
tags:
  - PowerShell
  - Power BI
---

# Retrieving and Exporting Power BI Activity Logs with PowerShell

PowerShell scripts are powerful tools for automating tasks and managing Windows environments. In the realm of data analysis and business intelligence, Power BI serves as a popular visualization tool, and its activity logs are crucial for monitoring and analyzing user operations. This article introduces a PowerShell script designed to retrieve Power BI activity log events within a specified date range and export the results to a CSV file.

## Script Overview

```powershell
<#
    Description: This PowerShell script retrieves Power BI activity log events for 
                 a specified date range and exports the results to a CSV file. 
                 The script collects activity logs for each day within the 
                 specified start and end dates.

    Parameters:
    - outputPath: The path where the CSV file will be saved. 
    - startDate: The start date for retrieving activity logs. 
    - endDate: The end date for retrieving activity logs. 
#>

param (
    [string]$outputPath = "C:\Users\ted\Downloads",
    [string]$startDate = "2023-10-07",
    [string]$endDate = "2023-10-30"
)

# Connect to the Power BI service account
Connect-PowerBIServiceAccount

# Get the current date and time for reference
$retrieveDate = Get-Date 

# Construct the path for the CSV file
$activityLogsPath = Join-Path -Path $outputPath -ChildPath "ActivityLogs.csv"

# Convert start and end date strings to DateTime objects
$startDate = Get-Date $startDate
$endDate = Get-Date $endDate

# Initialize the loop with the start date
$currentDate = $startDate
$activityLog = @()

# Loop through each day in the specified date range
while ($currentDate -le $endDate) {
    # Format the current date to create the start and end datetime strings
    $dateStr = $currentDate.ToString("yyyy-MM-dd")
    $startDt = $dateStr + 'T00:00:00.000'
    $endDt = $dateStr + 'T23:59:59.999'

    # Define parameters for retrieving Power BI activity logs
    $activityLogsParams = @{
        StartDateTime = $startDt
        EndDateTime   = $endDt
    }

    # Retrieve and convert Power BI activity logs from JSON
    $activityLogs = Get-PowerBIActivityEvent @activityLogsParams | ConvertFrom-Json

    # Select relevant properties and add a 'RetrieveDate' property
    $activityLogSchema = $activityLogs | Select-Object Id, RecordType, CreationTime, Operation, OrganizationId, UserType, UserKey, Workload, `
        UserId, ClientIP, UserAgent, Activity, ItemName, WorkspaceName, DatasetName, ReportName, `
        WorkspaceId, CapacityId, CapacityName, AppName, ObjectId, DatasetId, ReportId, IsSuccess, `
        ReportType, RequestId, ActivityId, AppReportId, DistributionMethod, ConsumptionMethod, `
        @{Name="RetrieveDate"; Expression={$retrieveDate}}

    # Add the activity log data to the array
    $activityLog += $activityLogSchema

    # Move to the next day
    $currentDate = $currentDate.AddDays(1)
}

# Export the combined activity log data to a CSV file
$activityLog | Export-Csv -Path $activityLogsPath -NoTypeInformation
```

## Script Breakdown

- *Description:* The script allows users to define output path, start date, and end date as parameters.
- *Connect to Power BI Service Account:* Utilizes `Connect-PowerBIServiceAccount` to establish a connection to the Power BI service account, ensuring access to the required activity log data.
- *Date Range Loop:* Iterates through each day within the specified date range.
- *Log Retrieval:* For each day, constructs start and end datetime strings, then uses `Get-PowerBIActivityEvent` to fetch Power BI activity logs.
- *Data Processing:* Selects relevant properties and adds a 'RetrieveDate' property for each log entry, indicating the retrieval date.
- *Export to CSV:* Exports processed activity log data to a CSV file at the specified path.

## Conclusion

This PowerShell script provides an automated way to collect and export Power BI activity logs, offering users a comprehensive overview of Power BI activities within a specific date range. By adjusting parameters, users can easily perform log retrieval and export operations for different date ranges, providing convenience for monitoring and analysis.
