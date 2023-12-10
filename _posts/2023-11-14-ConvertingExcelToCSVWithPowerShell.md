---
layout: post
title: "Streamlining Excel to CSV"
subtitle: 'Get all sheet from Excel to CSV'
date: 2023-11-14
author: "Ted"
header-style: text
tags:
  - PowerShell
---

# Converting Excel to CSV with PowerShell

PowerShell is a powerful scripting language that allows for automation and task automation on Windows systems. In this article, we will explore a PowerShell script designed to convert an Excel file to CSV format and save the resulting CSV files in a specified directory.

The script begins with a set of comments, including a synopsis and description of its functionality, as well as parameters it accepts.

```powershell
<#
.SYNOPSIS
    This PowerShell script converts an Excel file to CSV format 
    and saves the CSV files in a specified directory.
.DESCRIPTION
    The script takes input parameters for the file path and name, 
    checks for the ImportExcel module, installs it if needed,
    opens the Excel workbook, creates a folder for CSV files, 
    iterates through each worksheet, and saves them as CSV files.
.PARAMETER filePath
    Specifies the path where the Excel file is located.
.PARAMETER fileName
    Specifies the name of the Excel file.
#>

param (
    [string]$filePath = "C:\Users\ted\Downloads\", 
    [string]$fileName = "WPP2022_Locations_notes.xls" 
)

# Get all files in the specified directory
$fileList = Get-ChildItem -Path $filePath -File

# Output the list of files
foreach ($file in $fileList) {
    Write-Host $($file.FullName -replace '^.*\\')
}
Write-Host "**************"

# Check if ImportExcel module is installed, if not, install it
if (-not (Get-Module -ListAvailable -Name ImportExcel)) {
    Import-Module ImportExcel
    Write-Host "The ImportExcel module has been installed."
}

# Build the full paths for Excel file and CSV output directory
$excelFilePath = Join-Path $filePath $fileName
$csvOutputDirectory = Join-Path $filePath ($fileName -replace '\..*$', '')

# Create Excel application object
$excel = New-Object -ComObject Excel.Application
$excel.Visible = $false

# Open the Excel workbook
$workbook = $excel.Workbooks.Open($excelFilePath)

# Create a folder for CSV files if it doesn't exist
if (-not (Test-Path -Path $csvOutputDirectory -PathType Container)) {
    New-Item -Path $csvOutputDirectory -ItemType Directory
}

# Iterate through each worksheet and save it as a CSV file
foreach ($worksheet in $workbook.Sheets) {
    $csvFileName = Join-Path $csvOutputDirectory "$($fileName -replace '\..*$', '')_$($worksheet.Name).csv"
    $worksheet.SaveAs($csvFileName, 6)  # 6 represents CSV format
}

# Close Excel application
$excel.Quit()
[System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null

Write-Host "CSV files saved to $csvOutputDirectory"
```

In conclusion, this PowerShell script provides a convenient way to convert Excel files to CSV format, offering flexibility through customizable input parameters. The script is well-commented, making it easy to understand and modify according to specific requirements.
