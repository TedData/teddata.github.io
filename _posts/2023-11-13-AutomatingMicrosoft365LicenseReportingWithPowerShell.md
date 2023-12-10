---
layout: post
title: "Automating M365 License Reporting"
date: 2023-11-13
author: "Ted"
header-style: text
tags:
  - PowerShell
---

# Automating Microsoft 365 License Reporting with PowerShell

## Introduction

Managing licenses for users in a Microsoft 365 environment can be a complex task, especially in large organizations. Keeping track of who has access to which services and ensuring compliance with licensing agreements is crucial. In this article, we'll explore a PowerShell script that automates the retrieval of detailed license information for users in Microsoft 365, using the Microsoft Graph Beta module.

## Script Overview

The provided PowerShell script connects to the Microsoft Graph Beta module and fetches detailed license information for users. It generates two types of reports – a detailed report and a simplified report – both in CSV format. Let's break down the key components of the script.

```powershell
<#
.SYNOPSIS
This PowerShell script connects to the Microsoft Graph Beta module and 
retrieves detailed license information for users in Microsoft 365. 
It generates both a detailed and a simplified report in CSV format.

.DESCRIPTION
This script requires parameters for user-specific information 
such as UserNamesFile, TenantId, ClientId, CertificateThumbprint, 
and outputPath. The Microsoft Graph Beta module is checked and 
installed if necessary. The script then connects to Microsoft Graph 
using either a certificate or username/password. License information 
for each user is retrieved, and two CSV reports are generated - a 
detailed report and a simplified report.

.PARAMETER outputPath
Specifies the path where the output CSV files will be saved.

.NOTES
- Lines 186 and 192 require modification to set the correct file input paths ("LicenseFriendlyName.txt", "ServiceFriendlyName.csv").
- If Microsoft Graph Beta module is not installed, the script prompts for installation.
- Ensure that the necessary permissions are granted for the specified Client ID in Microsoft 365.

#>

Param
(
    [Parameter(Mandatory = $false)]
    [string]$UserNamesFile,
    [string]$TenantId,
    [string]$ClientId,
    [string]$CertificateThumbprint,
    [string]$outputPath = "C:\Users\ted\Downloads"
)

# Function to connect to Microsoft Graph module
Function ConnectMgGraphModule 
{
    # Check if Microsoft Graph Beta module is available, install if not
    $MsGraphBetaModule =  Get-Module Microsoft.Graph.Beta -ListAvailable
    if($MsGraphBetaModule -eq $null)
    { 
        Write-host "Important: Microsoft Graph Beta module is unavailable. It is mandatory to have this module installed in the system to run the script successfully." 
        $confirm = Read-Host "Are you sure you want to install Microsoft Graph Beta module? [Y] Yes [N] No"
        if($confirm -match "[yY]") 
        { 
            Write-host "Installing Microsoft Graph Beta module..."
            Install-Module Microsoft.Graph.Beta -Scope CurrentUser -AllowClobber
            Write-host "Microsoft Graph Beta module is installed in the machine successfully" -ForegroundColor Magenta 
        } 
        else
        { 
            Write-host "Exiting. `nNote: Microsoft Graph Beta module must be available in your system to run the script" -ForegroundColor Red
            Exit 
        } 
    }
    # Connect to Microsoft Graph using certificate if provided, else use username and password
    if(($TenantId -ne "") -and ($ClientId -ne "") -and ($CertificateThumbprint -ne ""))  
    {  
        Connect-MgGraph  -TenantId $TenantId -AppId $ClientId -CertificateThumbprint $CertificateThumbprint -ErrorAction SilentlyContinue -ErrorVariable ConnectionError|Out-Null
        if($ConnectionError -ne $null)
        {    
            Write-Host $ConnectionError -Foregroundcolor Red
            Exit
        }
    }
    else
    {
        Connect-MgGraph -Scopes "Directory.Read.All"  -ErrorAction SilentlyContinue -Errorvariable ConnectionError |Out-Null
        if($ConnectionError -ne $null)
        {
            Write-Host $ConnectionError -Foregroundcolor Red
            Exit
        }
    }
    Write-Host "Microsoft Graph Beta PowerShell module is connected successfully" -ForegroundColor Green
}

# Function to retrieve users' license information
Function Get_UsersLicenseInfo
{
    $LicensePlanWithEnabledService=""
    $FriendlyNameOfLicensePlanWithService=""
    $UPN = $_.UserPrincipalName
    $Country = $_.Country
    if($Country -eq "")
    {
        $Country="-"
    }
    Write-Progress -Activity "`n     Exported user count:$LicensedUserCount "`n"Currently Processing:$upn"
    $SKUs = Get-MgBetaUserLicenseDetail -UserId $UPN -ErrorAction SilentlyContinue
    $LicenseCount = $SKUs.count
    $count = 0
    foreach($Sku in $SKUs)  # License loop
    {
        if($FriendlyNameHash[$Sku.SkuPartNumber])
        {
            $NamePrint = $FriendlyNameHash[$Sku.SkuPartNumber]
        }
        else
        {
            $NamePrint = $Sku.SkuPartNumber
        }
        # Get all services for current SKUId
        $Services = $Sku.ServicePlans
        if(($Count -gt 0) -and ($count -lt $LicenseCount))
        {
            $LicensePlanWithEnabledService = $LicensePlanWithEnabledService+","
            $FriendlyNameOfLicensePlanWithService = $FriendlyNameOfLicensePlanWithService+","
        }
        $DisabledServiceCount = 0
        $EnabledServiceCount = 0
        $serviceExceptDisabled = ""
        $FriendlyNameOfServiceExceptDisabled = ""
        foreach($Service in $Services) # Service loop
        {
            $flag = 0
            $ServiceName = $Service.ServicePlanName
            if($service.ProvisioningStatus -eq "Disabled")
            {
                $DisabledServiceCount++
            }
            else
            {
                $EnabledServiceCount++
                if($EnabledServiceCount -ne 1)
                {
                    $serviceExceptDisabled = $serviceExceptDisabled+","
                }
                $serviceExceptDisabled = $serviceExceptDisabled+$ServiceName
                $flag = 1
            }
            # Convert ServiceName to friendly name
            $ServiceFriendlyName = $ServiceArray | Where-Object{$_.Service_Plan_Name -eq $ServiceName}
            if($ServiceFriendlyName -ne $Null)
            {
                $ServiceFriendlyName = $ServiceFriendlyName[0].ServiceFriendlyNames
            }
            else
            {
                $ServiceFriendlyName = $ServiceName
            }
            if($flag -eq 1)
            {
                if($EnabledServiceCount -ne 1)
                {
                    $FriendlyNameOfServiceExceptDisabled = $FriendlyNameOfServiceExceptDisabled+","
                }
                $FriendlyNameOfServiceExceptDisabled = $FriendlyNameOfServiceExceptDisabled+$ServiceFriendlyName
            }
            $Result = [PSCustomObject]@{'DisplayName'=$_.Displayname;'UserPrinciPalName'=$UPN;'LicensePlan'=$Sku.SkuPartNumber;'FriendlyNameofLicensePlan'=$nameprint;'ServiceName'=$ServiceName;'FriendlyNameofServiceName'=$serviceFriendlyName;'ProvisioningStatus'=$service.ProvisioningStatus}
            $Result  | Export-Csv -Path $ExportCSV -Notype -Append
        }
        if($Disabledservicecount -eq 0)
        {
            $serviceExceptDisabled = "All services"
            $FriendlyNameOfServiceExceptDisabled = "All services"
        }
        $LicensePlanWithEnabledService = $LicensePlanWithEnabledService + $Sku.SkuPartNumber +"[" +$serviceExceptDisabled +"]"
        $FriendlyNameOfLicensePlanWithService = $FriendlyNameOfLicensePlanWithService+ $NamePrint + "[" + $FriendlyNameOfServiceExceptDisabled +"]"
        # Increment SKUid count
        $count++
     }
     $Output=[PSCustomObject]@{'Displayname'=$_.Displayname;'UserPrincipalName'=$UPN;Country=$Country;'LicensePlanWithEnabledService'=$LicensePlanWithEnabledService;'FriendlyNameOfLicensePlanAndEnabledService'=$FriendlyNameOfLicensePlanWithService}
     $Output | Export-Csv -path $ExportSimpleCSV -NoTypeInformation -Append
}

# Function to close the connection
function CloseConnection
{
    Disconnect-MgGraph |Out-Null
    Exit
}

# Main function
Function main()
{
    ConnectMgGraphModule
    Write-Host "`nNote: If you encounter module-related conflicts, run the script in a fresh PowerShell window." -ForegroundColor Yellow
    # Set output file paths
    $ExportCSV = "$($outputPath)\DetailedO365UserLicenseReport.csv"
    $ExportSimpleCSV = "$($outputPath)\SimpleO365UserLicenseReport.csv"
    # FriendlyName list for license plan and service
    try
    {
        $FriendlyNameHash = Get-Content -Raw -Path C:\Users\Public\Downloads\LicenseFriendlyName.txt -ErrorAction SilentlyContinue -ErrorVariable ERR | ConvertFrom-StringData
        if($ERR -ne $null)
        {
            Write-Host $ERR -ForegroundColor Red
            CloseConnection
        }
        $ServiceArray =  Import-Csv -Path C:\Users\Public\Downloads\ServiceFriendlyName.csv 
    }
    catch
    {
        Write-Host $_.Exception.message -ForegroundColor Red
        CloseConnection
    }
    # Get licensed user
    $LicensedUserCount = 0
    # Check for input file/Get users from the input file
    if($UserNamesFile -ne "")
    {
        # We have an input file, read it into memory
        $UserNames = @()
        $UserNames = Import-Csv -Header "UserPrincipalName" $UserNamesFile
        foreach($item in $UserNames)
        {
            Get-MgBetaUser -UserId $item.UserPrincipalName -ErrorAction SilentlyContinue | Where-Object{$_.AssignedLicenses -ne $null} | ForEach-Object{
                Get_UsersLicenseInfo
                $LicensedUserCount++
            }
        }
    }
    # Get all licensed users
    else
    {
        Get-MgBetaUser -All  | Where-Object{$_.AssignedLicenses -ne $null} | ForEach-Object{
            Get_UsersLicenseInfo
            $LicensedUserCount++
        }
    }
    # Open output file after execution
    if((Test-Path -Path $ExportCSV) -eq "True") 
    {   
        Write-Host `n "Detailed report available in:" -NoNewline -ForegroundColor Yellow; Write-Host "$ExportCSV" 
        Write-Host `n "Simple report available in:" -NoNewline -ForegroundColor Yellow; Write-Host "$ExportSimpleCSV" `n 
        $Prompt = New-Object -ComObject wscript.shell
        $UserInput = $Prompt.popup("Do you want to open output files?",` 0,"Open Files",4)
        if($UserInput -eq 6)
        {
            Invoke-Item $ExportCSV
            Invoke-Item $ExportSimpleCSV
        }
    }
    else
    {
        Write-Host "No data found" 
    }
    Write-Host `n~~ Script prepared by AdminDroid Community ~~`n -ForegroundColor Green
    Write-Host "~ Check out " -NoNewline -ForegroundColor Green; Write-Host "admindroid.com" -ForegroundColor Yellow -NoNewline; Write-Host " to get access to 1800+ Microsoft 365 reports. ~" -ForegroundColor Green `n`n
    CloseConnection
}

# Execute the main function
. main
```

## Conclusion

Automating Microsoft 365 license reporting with PowerShell streamlines the process of gathering detailed information about user licenses. The script provided in this article offers flexibility by supporting both username/password and certificate-based authentication. By regularly running this script, administrators can ensure they have up-to-date insights into the licensing status of users in their Microsoft 365 environment, aiding in compliance and resource optimization.

Note: Before running the script, administrators should review and customize certain sections, such as file paths and permissions, to align with their specific environment.

This script not only enhances efficiency but also serves as a practical example of leveraging PowerShell and Microsoft Graph for Microsoft 365 management. Feel free to adapt and extend the script based on your organization's specific needs and reporting requirements.

## Appendices

### Appendix A: License Friendly Names

The script relies on a file (`LicenseFriendlyName.txt`) containing friendly names for license plans. Ensure this file is correctly configured with the appropriate mappings.

```txt
AAD_BASIC= Azure Active Directory Basic
	AAD_PREMIUM= Azure Active Directory Premium
	AAD_PREMIUM_P1= Azure Active Directory Premium P1
	AAD_PREMIUM_P2= Azure Active Directory Premium P2
	ADALLOM_O365= Office 365 Advanced Security Management
	ADALLOM_STANDALONE= Microsoft Cloud App Security
	ADALLOM_S_O365= POWER BI STANDALONE
	ADALLOM_S_STANDALONE= Microsoft Cloud App Security
	ATA= Azure Advanced Threat Protection for Users
	ATP_ENTERPRISE= Exchange Online Advanced Threat Protection
	ATP_ENTERPRISE_FACULTY= Exchange Online Advanced Threat Protection
	BI_AZURE_P0= Power BI (free)
	BI_AZURE_P1= Power BI Reporting and Analytics
	BI_AZURE_P2= Power BI Pro
	CCIBOTS_PRIVPREV_VIRAL= Dynamics 365 AI for Customer Service Virtual Agents Viral SKU
	CRMINSTANCE= Microsoft Dynamics CRM Online Additional Production Instance (Government Pricing)
	CRMIUR= CRM for Partners
	CRMPLAN1= Microsoft Dynamics CRM Online Essential (Government Pricing)
	CRMPLAN2= Dynamics CRM Online Plan 2
	CRMSTANDARD= CRM Online
	CRMSTORAGE= Microsoft Dynamics CRM Online Additional Storage
	CRMTESTINSTANCE= CRM Test Instance
	DESKLESS= Microsoft StaffHub
	DESKLESSPACK= Office 365 (Plan K1)
	DESKLESSPACK_GOV= Microsoft Office 365 (Plan K1) for Government
	DESKLESSPACK_YAMMER= Office 365 Enterprise K1 with Yammer
	DESKLESSWOFFPACK= Office 365 (Plan K2)
	DESKLESSWOFFPACK_GOV= Microsoft Office 365 (Plan K2) for Government
	DEVELOPERPACK= Office 365 Enterprise E3 Developer
	DEVELOPERPACK_E5= Microsoft 365 E5 Developer(without Windows and Audio Conferencing)
	DMENTERPRISE= Microsoft Dynamics Marketing Online Enterprise
	DYN365_ENTERPRISE_CUSTOMER_SERVICE= Dynamics 365 for Customer Service Enterprise Edition
	DYN365_ENTERPRISE_P1_IW= Dynamics 365 P1 Trial for Information Workers
	DYN365_ENTERPRISE_PLAN1= Dynamics 365 Plan 1 Enterprise Edition
	DYN365_ENTERPRISE_SALES= Dynamics 365 for Sales Enterprise Edition
	DYN365_ENTERPRISE_SALES_CUSTOMERSERVICE= Dynamics 365 for Sales and Customer Service Enterprise Edition
	DYN365_ENTERPRISE_TEAM_MEMBERS= Dynamics 365 for Team Members Enterprise Edition
	DYN365_FINANCIALS_BUSINESS_SKU= Dynamics 365 for Financials Business Edition
  DYN365_MARKETING_USER= Dynamics 365 for Marketing USL
  DYN365_MARKETING_APP=  Dynamics 365 Marketing
	DYN365_SALES_INSIGHTS= Dynamics 365 AI for Sales
	D365_SALES_PRO= Dynamics 365 for Sales Professional
	Dynamics_365_for_Operations= Dynamics 365 Unf Ops Plan Ent Edition
	ECAL_SERVICES= ECAL
	EMS= Enterprise Mobility + Security E3
	EMSPREMIUM= Enterprise Mobility + Security E5
	ENTERPRISEPACK= Office 365 Enterprise E3
	ENTERPRISEPACKLRG= Office 365 Enterprise E3 LRG
	ENTERPRISEPACKWITHOUTPROPLUS= Office 365 Enterprise E3 without ProPlus Add-on
	ENTERPRISEPACK_B_PILOT= Office 365 (Enterprise Preview)
	ENTERPRISEPACK_FACULTY= Office 365 (Plan A3) for Faculty
	ENTERPRISEPACK_GOV= Microsoft Office 365 (Plan G3) for Government
	ENTERPRISEPACK_STUDENT= Office 365 (Plan A3) for Students
	ENTERPRISEPREMIUM= Enterprise E5 (with Audio Conferencing)
	ENTERPRISEPREMIUM_NOPSTNCONF= Enterprise E5 (without Audio Conferencing)
	ENTERPRISEWITHSCAL= Office 365 Enterprise E4
	ENTERPRISEWITHSCAL_FACULTY= Office 365 (Plan A4) for Faculty
	ENTERPRISEWITHSCAL_GOV= Microsoft Office 365 (Plan G4) for Government
	ENTERPRISEWITHSCAL_STUDENT= Office 365 (Plan A4) for Students
	EOP_ENTERPRISE= Exchange Online Protection
	EOP_ENTERPRISE_FACULTY= Exchange Online Protection for Faculty
	EQUIVIO_ANALYTICS= Office 365 Advanced Compliance
	EQUIVIO_ANALYTICS_FACULTY= Office 365 Advanced Compliance for Faculty
	ESKLESSWOFFPACK_GOV= Microsoft Office 365 (Plan K2) for Government
	EXCHANGEARCHIVE= Exchange Online Archiving
	EXCHANGEARCHIVE_ADDON= Exchange Online Archiving for Exchange Online
	EXCHANGEDESKLESS= Exchange Online Kiosk
	EXCHANGEENTERPRISE= Exchange Online Plan 2
	EXCHANGEENTERPRISE_FACULTY= Exch Online Plan 2 for Faculty
	EXCHANGEENTERPRISE_GOV= Microsoft Office 365 Exchange Online (Plan 2) only for Government
	EXCHANGEESSENTIALS= Exchange Online Essentials
	EXCHANGESTANDARD= Office 365 Exchange Online Only
	EXCHANGESTANDARD_GOV= Microsoft Office 365 Exchange Online (Plan 1) only for Government
	EXCHANGESTANDARD_STUDENT= Exchange Online (Plan 1) for Students
	EXCHANGETELCO= Exchange Online POP
	EXCHANGE_ANALYTICS= Microsoft MyAnalytics
	EXCHANGE_L_STANDARD= Exchange Online (Plan 1)
	EXCHANGE_S_ARCHIVE_ADDON_GOV= Exchange Online Archiving
	EXCHANGE_S_DESKLESS= Exchange Online Kiosk
	EXCHANGE_S_DESKLESS_GOV= Exchange Kiosk
	EXCHANGE_S_ENTERPRISE= Exchange Online (Plan 2) Ent
	EXCHANGE_S_ENTERPRISE_GOV= Exchange Plan 2G
	EXCHANGE_S_ESSENTIALS= Exchange Online Essentials
	EXCHANGE_S_FOUNDATION= Exchange Foundation for certain SKUs
	EXCHANGE_S_STANDARD= Exchange Online (Plan 2)
	EXCHANGE_S_STANDARD_MIDMARKET= Exchange Online (Plan 1)
	FLOW_FREE= Microsoft Flow (Free)
	FLOW_O365_P2= Flow for Office 365
	FLOW_O365_P3= Flow for Office 365
	FLOW_P1= Microsoft Flow Plan 1
	FLOW_P2= Microsoft Flow Plan 2
	FORMS_PLAN_E3= Microsoft Forms (Plan E3)
	FORMS_PLAN_E5= Microsoft Forms (Plan E5)
	INFOPROTECTION_P2= Azure Information Protection Premium P2
	INTUNE_A= Windows Intune Plan A
	INTUNE_A_VL= Intune (Volume License)
	INTUNE_O365= Mobile Device Management for Office 365
	INTUNE_STORAGE= Intune Extra Storage
	IT_ACADEMY_AD= Microsoft Imagine Academy
	LITEPACK= Office 365 (Plan P1)
	LITEPACK_P2= Office 365 Small Business Premium
	LOCKBOX= Customer Lockbox
	LOCKBOX_ENTERPRISE= Customer Lockbox
	MCOCAP= Command Area Phone
	MCOEV= Skype for Business Cloud PBX
	MCOIMP= Skype for Business Online (Plan 1)
	MCOLITE= Lync Online (Plan 1)
	MCOMEETADV= PSTN conferencing
	MCOPLUSCAL= Skype for Business Plus CAL
	MCOPSTN1= Skype for Business Pstn Domestic Calling
	MCOPSTN2= Skype for Business Pstn Domestic and International Calling
	MCOSTANDARD= Skype for Business Online Standalone Plan 2
	MCOSTANDARD_GOV= Lync Plan 2G
	MCOSTANDARD_MIDMARKET= Lync Online (Plan 1)
	MCVOICECONF= Lync Online (Plan 3)
	MDM_SALES_COLLABORATION= Microsoft Dynamics Marketing Sales Collaboration
	MEE_FACULTY= Minecraft Education Edition Faculty
  MEE_STUDENT= Minecraft Education Edition Student
  MEETING_ROOM=  Meeting Room
	MFA_PREMIUM= Azure Multi-Factor Authentication
  MICROSOFT_BUSINESS_CENTER= Microsoft Business Center
  MICROSOFT_REMOTE_ASSIST= Dynamics 365 Remote Assist
	MIDSIZEPACK= Office 365 Midsize Business
	MINECRAFT_EDUCATION_EDITION= Minecraft Education Edition Faculty
	MS-AZR-0145P= Azure
	MS_TEAMS_IW= Microsoft Teams
	NBPOSTS= Microsoft Social Engagement Additional 10k Posts (minimum 100 licenses) (Government Pricing)
	NBPROFESSIONALFORCRM= Microsoft Social Listening Professional
	O365_BUSINESS= Microsoft 365 Apps for business
	O365_BUSINESS_ESSENTIALS= Microsoft 365 Business Basic
  O365_BUSINESS_PREMIUM= Microsoft 365 Business Standard
  OFFICE365_MULTIGEO= Multi-Geo Capabilities in Office 365
	OFFICESUBSCRIPTION= Microsoft 365 Apps for enterprise
	OFFICESUBSCRIPTION_FACULTY= Office 365 ProPlus for Faculty
	OFFICESUBSCRIPTION_GOV= Office ProPlus
	OFFICESUBSCRIPTION_STUDENT= Office ProPlus Student Benefit
	OFFICE_FORMS_PLAN_2= Microsoft Forms (Plan 2)
	OFFICE_PRO_PLUS_SUBSCRIPTION_SMBIZ= Office ProPlus
  ONEDRIVESTANDARD= OneDrive
  PAM_ENTERPRISE = Exchange Primary Active Manager
	PLANNERSTANDALONE= Planner Standalone
	POWERAPPS_INDIVIDUAL_USER= Microsoft PowerApps and Logic flows
	POWERAPPS_O365_P2= PowerApps
	POWERAPPS_O365_P3= PowerApps for Office 365
	POWERAPPS_VIRAL= PowerApps (Free)
	POWERFLOW_P1= Microsoft PowerApps Plan 1
	POWERFLOW_P2= Microsoft PowerApps Plan 2
	POWER_BI_ADDON= Office 365 Power BI Addon
	POWER_BI_INDIVIDUAL_USE= Power BI Individual User
	POWER_BI_INDIVIDUAL_USER= Power BI for Office 365 Individual
	POWER_BI_PRO= Power BI Pro
	POWER_BI_STANDALONE= Power BI Standalone
	POWER_BI_STANDARD= Power-BI Standard
	PREMIUM_ADMINDROID= AdminDroid Office 365 Reporter
	PROJECTCLIENT= Project Professional
	PROJECTESSENTIALS= Project Lite
	PROJECTONLINE_PLAN_1= Project Online (Plan 1)
	PROJECTONLINE_PLAN_1_FACULTY= Project Online for Faculty Plan 1
	PROJECTONLINE_PLAN_1_STUDENT= Project Online for Students Plan 1
	PROJECTONLINE_PLAN_2= Project Online and PRO
	PROJECTONLINE_PLAN_2_FACULTY= Project Online for Faculty Plan 2
	PROJECTONLINE_PLAN_2_STUDENT= Project Online for Students Plan 2
	PROJECTPREMIUM= Project Online Premium
	PROJECTPROFESSIONAL= Project Online Pro
	PROJECTWORKMANAGEMENT= Office 365 Planner Preview
	PROJECT_CLIENT_SUBSCRIPTION= Project Pro for Office 365
	PROJECT_ESSENTIALS= Project Lite
  PROJECT_MADEIRA_PREVIEW_IW_SKU= Dynamics 365 for Financials for IWs
  PROJECT_ONLINE_PRO=  Project Online Plan 3
	RIGHTSMANAGEMENT= Azure Rights Management Premium
	RIGHTSMANAGEMENT_ADHOC= Windows Azure Rights Management
	RIGHTSMANAGEMENT_STANDARD_FACULTY= Azure Rights Management for faculty
	RIGHTSMANAGEMENT_STANDARD_STUDENT= Information Rights Management for Students
	RMS_S_ENTERPRISE= Azure Active Directory Rights Management
	RMS_S_ENTERPRISE_GOV= Windows Azure Active Directory Rights Management
	RMS_S_PREMIUM= Azure Information Protection Plan 1
	RMS_S_PREMIUM2= Azure Information Protection Premium P2
	SCHOOL_DATA_SYNC_P1= School Data Sync (Plan 1)
	SHAREPOINTDESKLESS= SharePoint Online Kiosk
	SHAREPOINTDESKLESS_GOV= SharePoint Online Kiosk
	SHAREPOINTENTERPRISE= SharePoint Online (Plan 2)
	SHAREPOINTENTERPRISE_EDU= SharePoint Plan 2 for EDU
	SHAREPOINTENTERPRISE_GOV= SharePoint Plan 2G
	SHAREPOINTENTERPRISE_MIDMARKET= SharePoint Online (Plan 1)
	SHAREPOINTLITE= SharePoint Online (Plan 1)
	SHAREPOINTPARTNER= SharePoint Online Partner Access
	SHAREPOINTSTANDARD= SharePoint Online Plan 1
	SHAREPOINTSTANDARD_EDU= SharePoint Plan 1 for EDU
	SHAREPOINTSTORAGE= SharePoint Online Storage
	SHAREPOINTWAC= Office Online
	SHAREPOINTWAC_EDU= Office Online for Education
	SHAREPOINTWAC_GOV= Office Online for Government
	SHAREPOINT_PROJECT= SharePoint Online (Plan 2) Project
  SHAREPOINT_PROJECT_EDU= Project Online Service for Education
  SMB_APPS=  Business Apps (free)
	SMB_BUSINESS= Office 365 Business
	SMB_BUSINESS_ESSENTIALS= Office 365 Business Essentials
  SMB_BUSINESS_PREMIUM= Office 365 Business Premium
  SPZA IW= Microsoft PowerApps Plan 2 Trial
	SPB= Microsoft 365 Business
	SPE_E3= Secure Productive Enterprise E3
	SQL_IS_SSIM= Power BI Information Services
	STANDARDPACK= Office 365 (Plan E1)
	STANDARDPACK_FACULTY= Office 365 (Plan A1) for Faculty
	STANDARDPACK_GOV= Microsoft Office 365 (Plan G1) for Government
	STANDARDPACK_STUDENT= Office 365 (Plan A1) for Students
	STANDARDWOFFPACK= Office 365 (Plan E2)
	STANDARDWOFFPACKPACK_FACULTY= Office 365 (Plan A2) for Faculty
	STANDARDWOFFPACKPACK_STUDENT= Office 365 (Plan A2) for Students
	STANDARDWOFFPACK_FACULTY= Office 365 Education E1 for Faculty
	STANDARDWOFFPACK_GOV= Microsoft Office 365 (Plan G2) for Government
	STANDARDWOFFPACK_IW_FACULTY= Office 365 Education for Faculty
	STANDARDWOFFPACK_IW_STUDENT= Office 365 Education for Students
	STANDARDWOFFPACK_STUDENT= Microsoft Office 365 (Plan A2) for Students
	STANDARD_B_PILOT= Office 365 (Small Business Preview)
	STREAM= Microsoft Stream
	STREAM_O365_E3= Microsoft Stream for O365 E3 SKU
	STREAM_O365_E5= Microsoft Stream for O365 E5 SKU
	SWAY= Sway
	TEAMS1= Microsoft Teams
	TEAMS_COMMERCIAL_TRIAL= Microsoft Teams Commercial Cloud Trial
	THREAT_INTELLIGENCE= Office 365 Threat Intelligence
	VIDEO_INTEROP = Skype Meeting Video Interop for Skype for Business
	VISIOCLIENT= Visio Online Plan 2
	VISIOONLINE_PLAN1= Visio Online Plan 1
	VISIO_CLIENT_SUBSCRIPTION= Visio Pro for Office 365
	WACONEDRIVEENTERPRISE= OneDrive for Business (Plan 2)
	WACONEDRIVESTANDARD= OneDrive for Business with Office Online
  WACSHAREPOINTSTD= Office Online STD
  WHITEBOARD_PLAN3=  White Board (Plan 3)
	WIN_DEF_ATP= Windows Defender Advanced Threat Protection
  WIN10_PRO_ENT_SUB= Windows 10 Enterprise E3
  WIN10_VDA_E3= Windows E3
  WIN10_VDA_E5=  Windows E5
	WINDOWS_STORE= Windows Store
	YAMMER_EDU= Yammer for Academic
	YAMMER_ENTERPRISE= Yammer for the Starship Enterprise
	YAMMER_ENTERPRISE_STANDALONE= Yammer Enterprise
	YAMMER_MIDSIZE= Yammer
```

### Appendix B: Service Friendly Names

The script also uses a CSV file (`ServiceFriendlyName.csv`) containing friendly names for services. Verify the accuracy of this file and update it as needed for correct service name conversions.

```csv
Service_Plan_Name,ServiceFriendlyNames
TEAMS_ADVCOMMS,Microsoft 365 Advanced Communications
CDSAICAPACITY,AI Builder capacity add-on
EXCHANGE_S_FOUNDATION,Exchange Foundation
SPZA,APP CONNECT
EXCHANGE_S_FOUNDATION,EXCHANGE FOUNDATION
MCOMEETADV,Microsoft 365 Audio Conferencing
AAD_BASIC,MICROSOFT AZURE ACTIVE DIRECTORY BASIC
AAD_PREMIUM,AZURE ACTIVE DIRECTORY PREMIUM P1
ADALLOM_S_DISCOVERY,CLOUD APP SECURITY DISCOVERY
MFA_PREMIUM,MICROSOFT AZURE MULTI-FACTOR AUTHENTICATION
AAD_PREMIUM,Azure Active Directory Premium P1
MFA_PREMIUM,Microsoft Azure Multi-Factor Authentication
ADALLOM_S_DISCOVERY,Microsoft Defender for Cloud Apps Discovery
AAD_PREMIUM_P2,AZURE ACTIVE DIRECTORY PREMIUM P2
RMS_S_ENTERPRISE,AZURE INFORMATION PROTECTION PREMIUM P1
RMS_S_PREMIUM,MICROSOFT AZURE ACTIVE DIRECTORY RIGHTS
DYN365BC_MS_INVOICING,Microsoft Invoicing
MICROSOFTBOOKINGS,Microsoft Bookings
CDS_DB_CAPACITY,Common Data Service for Apps Database Capacity
CDS_DB_CAPACITY_GOV,Common Data Service for Apps Database Capacity for Government
EXCHANGE_S_FOUNDATION_GOV,Exchange Foundation for Government
CDS_LOG_CAPACITY,Common Data Service for Apps Log Capacity
MCOPSTNC,COMMUNICATIONS CREDITS
COMPLIANCE_MANAGER_PREMIUM_ASSESSMENT_ADDON,Compliance Manager Premium Assessment Add-On
CRMSTORAGE,Microsoft Dynamics CRM Online Storage Add-On
CRMINSTANCE,Microsoft Dynamics CRM Online Instance
CRMTESTINSTANCE,Microsoft Dynamics CRM Online Additional Test Instance
SOCIAL_ENGAGEMENT_APP_USER,Dynamics 365 AI for Market Insights - Free
D365_AssetforSCM,Asset Maintenance Add-in
DYN365_BUSCENTRAL_ENVIRONMENT,Dynamics 365 Business Central Additional Environment Addon
DYN365_BUSCENTRAL_DB_CAPACITY,Dynamics 365 Business Central Database Capacity
DYN365_FINANCIALS_BUSINESS,Dynamics 365 for Business Central Essentials
FLOW_DYN_APPS,Flow for Dynamics 365
POWERAPPS_DYN_APPS,PowerApps for Dynamics 365
DYN365_FINANCIALS_ACCOUNTANT,Dynamics 365 Business Central External Accountant
PROJECT_MADEIRA_PREVIEW_IW,Dynamics 365 Business Central for IWs
DYN365_BUSCENTRAL_PREMIUM,Dynamics 365 Business Central Premium
DYN365_FINANCIALS_TEAM_MEMBERS,Dynamics 365 for Team Members
POWERAPPS_DYN_TEAM,Power Apps for Dynamics 365
FLOW_DYN_TEAM,Power Automate for Dynamics 365
D365_CSI_EMBED_CE,Dynamics 365 Customer Service Insights for CE Plan
DYN365_ENTERPRISE_P1,Dynamics 365 P1
D365_ProjectOperations,Dynamics 365 Project Operations
D365_ProjectOperationsCDS,Dynamics 365 Project Operations CDS
FLOW_DYN_P2,Flow for Dynamics 365
Forms_Pro_CE,Microsoft Dynamics 365 Customer Voice for Customer Engagement Plan
NBENTERPRISE,Microsoft Social Engagement Enterprise
SHAREPOINTWAC,Office for the web
POWERAPPS_DYN_P2,Power Apps for Dynamics 365
PROJECT_FOR_PROJECT_OPERATIONS,Project for Project Operations
PROJECT_CLIENT_SUBSCRIPTION,Project Online Desktop Client
SHAREPOINT_PROJECT,Project Online Service
SHAREPOINTENTERPRISE,SharePoint (Plan 2)
CDS_CUSTOMER_INSIGHTS_TRIAL,Common Data Service for Customer Insights Trial
DYN365_CUSTOMER_INSIGHTS_ENGAGEMENT_INSIGHTS_BASE_TRIAL,Dynamics 365 Customer Insights Engagement Insights Viral
DYN365_CUSTOMER_INSIGHTS_VIRAL,Dynamics 365 Customer Insights Viral Plan
Forms_Pro_Customer_Insights,Microsoft Dynamics 365 Customer Voice for Customer Insights
CUSTOMER_VOICE_DYN365_VIRAL_TRIAL,Customer Voice for Dynamics 365 vTrial
CCIBOTS_PRIVPREV_VIRAL,Dynamics 365 AI for Customer Service Virtual Agents Viral
DYN365_CS_MESSAGING_VIRAL_TRIAL,Dynamics 365 Customer Service Digital Messaging vTrial
DYN365_CS_ENTERPRISE_VIRAL_TRIAL,Dynamics 365 Customer Service Enterprise vTrial
DYNB365_CSI_VIRAL_TRIAL,Dynamics 365 Customer Service Insights vTrial
DYN365_CS_VOICE_VIRAL_TRIAL,Dynamics 365 Customer Service Voice vTrial
POWER_APPS_DYN365_VIRAL_TRIAL,Power Apps for Dynamics 365 vTrial
POWER_AUTOMATE_DYN365_VIRAL_TRIAL,Power Automate for Dynamics 365 vTrial
DYN365_AI_SERVICE_INSIGHTS,Dynamics 365 AI for Customer Service Trial
DYN365_CDS_FORMS_PRO,Common Data Service
FORMS_PRO,Dynamics 365 Customer Voice
FORMS_PLAN_E5,Microsoft Forms (Plan E5)
FLOW_FORMS_PRO,Power Automate for Dynamics 365 Customer Voice
DYN365_CUSTOMER_SERVICE_PRO,Dynamics 365 for Customer Service Pro
POWERAPPS_CUSTOMER_SERVICE_PRO,Power Apps for Customer Service Pro
FLOW_CUSTOMER_SERVICE_PRO,Power Automate for Customer Service Pro
PROJECT_ESSENTIALS,Project Online Essentials
Customer_Voice_Base,Dynamics 365 Customer Voice Base Plan
Forms_Pro_AddOn,Microsoft Dynamics 365 Customer Voice Add-on
CUSTOMER_VOICE_ADDON,Dynamics Customer Voice Add-On
CDS_FORM_PRO_USL,Common Data Service
Forms_Pro_USL,Microsoft Dynamics 365 Customer Voice USL
CRM_ONLINE_PORTAL,Microsoft Dynamics CRM Online - Portal Add-On
DYN365_FS_ENTERPRISE_VIRAL_TRIAL,Dynamics 365 Field Service Enterprise vTrial
DYN365_CDS_FINANCE,Common Data Service for Dynamics 365 Finance
DYN365_REGULATORY_SERVICE,Dynamics 365 for Finance and Operations Enterprise edition - Regulatory Service
D365_Finance,Microsoft Dynamics 365 for Finance
POWERAPPS_DYN_APPS,Power Apps for Dynamics 365
FLOW_DYN_APPS,Power Automate for Dynamics 365
DYN365_ENTERPRISE_CASE_MANAGEMENT,Dynamics 365 for Case Management
NBENTERPRISE,Retired - Microsoft Social Engagement
SHAREPOINTWAC,Office for the Web
D365_CSI_EMBED_CSEnterprise,Dynamics 365 Customer Service Insights for CS Enterprise
DYN365_ENTERPRISE_CUSTOMER_SERVICE,MICROSOFT SOCIAL ENGAGEMENT - SERVICE DISCONTINUATION
Forms_Pro_Service,Microsoft Dynamics 365 Customer Voice for Customer Service Enterprise
FLOW_DYN_APPS,PROJECT ONLINE ESSENTIALS
NBENTERPRISE,SHAREPOINT ONLINE (PLAN 2)
POWERAPPS_DYN_APPS,FLOW FOR DYNAMICS 365
PROJECT_ESSENTIALS,POWERAPPS FOR DYNAMICS 365
SHAREPOINTENTERPRISE,DYNAMICS 365 FOR CUSTOMER SERVICE
SHAREPOINTWAC,OFFICE ONLINE
D365_FIELD_SERVICE_ATTACH,Dynamics 365 for Field Service Attach
DYN365_ENTERPRISE_FIELD_SERVICE,Dynamics 365 for Field Service
Forms_Pro_FS,Microsoft Dynamics 365 Customer Voice for Field Service
DYN365_FINANCIALS_BUSINESS,FLOW FOR DYNAMICS 365
FLOW_DYN_APPS,POWERAPPS FOR DYNAMICS 365
POWERAPPS_DYN_APPS,DYNAMICS 365 FOR FINANCIALS
DYN365_MARKETING_MSE_USER,Dynamics 365 for Marketing MSE User
DYN365_MARKETING_USER,Dynamics 365 for Marketing USL
Forms_Pro_Marketing,Microsoft Dynamics 365 Customer Voice for Marketing
DYN365_ENTERPRISE_P1,DYNAMICS 365 CUSTOMER ENGAGEMENT PLAN
FLOW_DYN_APPS,FLOW FOR DYNAMICS 365
NBENTERPRISE,MICROSOFT SOCIAL ENGAGEMENT - SERVICE DISCONTINUATION
POWERAPPS_DYN_APPS,POWERAPPS FOR DYNAMICS 365
PROJECT_ESSENTIALS,PROJECT ONLINE ESSENTIALS
SHAREPOINTENTERPRISE,SHAREPOINT ONLINE (PLAN 2)
DYN365_ENTERPRISE_SALES,DYNAMICS 365 FOR SALES
DYN365_BUSINESS_Marketing,Dynamics 365 Marketing
DYN365_SALES_ENTERPRISE_VIRAL_TRIAL,Dynamics 365 Sales Enterprise vTrial
DYN365_SALES_INSIGHTS_VIRAL_TRIAL,Dynamics 365 Sales Insights vTrial
DYN365_SALES_PRO,Dynamics 365 for Sales Professional
POWERAPPS_SALES_PRO,Power Apps for Sales Pro
SHAREPOINTENTERPRISE,SharePoint (Plan 2)Dynamics 365 for Sales Pro Attach
D365_SALES_PRO_IW,Dynamics 365 for Sales Professional Trial
D365_SALES_PRO_IW_Trial,Dynamics 365 for Sales Professional Trial
D365_SALES_PRO_ATTACH,Dynamics 365 for Sales Pro Attach
DYN365_CDS_SUPPLYCHAINMANAGEMENT,COMMON DATA SERVICE FOR DYNAMICS 365 SUPPLY CHAIN MANAGEMENT
DYN365_REGULATORY_SERVICE,DYNAMICS 365 FOR FINANCE AND OPERATIONS ENTERPRISE EDITION - REGULATORY SERVICE
D365_SCM,DYNAMICS 365 FOR SUPPLY CHAIN MANAGEMENT
DYN365_CDS_DYN_APPS,Common Data Service
Dynamics_365_Hiring_Free_PLAN,Dynamics 365 for Talent: Attract
Dynamics_365_Onboarding_Free_PLAN,Dynamics 365 for Talent: Onboard
Dynamics_365_for_HCM_Trial,Dynamics 365 for HCM Trial
DYN365_Enterprise_Talent_Attract_TeamMember,DYNAMICS 365 FOR TALENT - ATTRACT EXPERIENCE TEAM MEMBER
DYN365_Enterprise_Talent_Onboard_TeamMember,DYNAMICS 365 FOR TALENT - ONBOARD EXPERIENCE
DYN365_ENTERPRISE_TEAM_MEMBERS,DYNAMICS 365 FOR TEAM MEMBERS
DYNAMICS_365_FOR_OPERATIONS_TEAM_MEMBERS,DYNAMICS 365 FOR OPERATIONS TEAM MEMBERS
Dynamics_365_for_Retail_Team_members,DYNAMICS 365 FOR RETAIL TEAM MEMBERS
Dynamics_365_for_Talent_Team_members,DYNAMICS 365 FOR TALENT TEAM MEMBERS
FLOW_DYN_TEAM,FLOW FOR DYNAMICS 365
POWERAPPS_DYN_TEAM,POWERAPPS FOR DYNAMICS 365
DYN365_CDS_GUIDES,Common Data Service
GUIDES,Dynamics 365 Guides
POWERAPPS_GUIDES,Power Apps for Guides
DYN365_RETAIL_DEVICE,Dynamics 365 for Retail Device
Dynamics_365_for_OperationsDevices,Dynamics 365 for Operations Devices
Dynamics_365_for_Operations_Sandbox_Tier2,Dynamics 365 for Operations non-production multi-box instance for standard acceptance testing (Tier 2)
Dynamics_365_for_Operations_Sandbox_Tier4,Dynamics 365 for Operations Enterprise Edition - Sandbox Tier 4:Standard Performance Testing
DYN365_ENTERPRISE_P1_IW,DYNAMICS 365 P1 TRIAL FOR INFORMATION WORKERS
CDS_REMOTE_ASSIST,Common Data Service for Remote Assist
MICROSOFT_REMOTE_ASSIST,Microsoft Remote Assist
TEAMS1,Microsoft Teams
D365_SALES_ENT_ATTACH,Dynamics 365 for Sales Enterprise Attach
DYN365_CDS_DYN_APPS,COMMON DATA SERVICE
Dynamics_365_Onboarding_Free_PLAN,DYNAMICS 365 FOR TALENT: ONBOARD
Dynamics_365_Talent_Onboard,DYNAMICS 365 FOR TALENT: ONBOARD
DYNAMICS_365_FOR_RETAIL_TEAM_MEMBERS,DYNAMICS 365 FOR RETAIL TEAM MEMBERS
DYN365_ENTERPRISE_TALENT_ATTRACT_TEAMMEMBER,DYNAMICS 365 FOR TALENT - ATTRACT EXPERIENCE TEAM MEMBER
DYN365_ENTERPRISE_TALENT_ONBOARD_TEAMMEMBER,DYNAMICS 365 FOR TALENT - ONBOARD EXPERIENCE
DYNAMICS_365_FOR_TALENT_TEAM_MEMBERS,DYNAMICS 365 FOR TALENT TEAM MEMBERS
DYN365_TEAM_MEMBERS,DYNAMICS 365 TEAM MEMBERS
SHAREPOINTWAC,OFFICE FOR THE WEB
SHAREPOINTENTERPRISE,SHAREPOINT (PLAN 2)
DDYN365_CDS_DYN_P2,COMMON DATA SERVICE
DYN365_TALENT_ENTERPRISE,DYNAMICS 365 FOR TALENT
Dynamics_365_for_Operations,DYNAMICS 365 FOR_OPERATIONS
Dynamics_365_for_Retail,DYNAMICS 365 FOR RETAIL
DYNAMICS_365_HIRING_FREE_PLAN,DYNAMICS 365 HIRING FREE PLAN
FLOW_DYN_P2,FLOW FOR DYNAMICS 36
POWERAPPS_DYN_P2,POWERAPPS FOR DYNAMICS 365
AAD_EDU,Azure Active Directory for Education
RMS_S_PREMIUM,Azure Information Protection Premium P1
RMS_S_ENTERPRISE,Azure Rights Management
INTUNE_A,Microsoft Intune
INTUNE_EDU,Microsoft Intune for Education
WINDOWS_STORE,Windows Store Service
RMS_S_PREMIUM,AZURE INFORMATION PROTECTION PREMIUM P1
RMS_S_ENTERPRISE,MICROSOFT AZURE ACTIVE DIRECTORY RIGHTS
INTUNE_A,MICROSOFT INTUNE
RMS_S_PREMIUM2,AZURE INFORMATION PROTECTION PREMIUM P2
ADALLOM_S_STANDALONE,MICROSOFT CLOUD APP SECURITY
ATA,MICROSOFT DEFENDER FOR IDENTITY
ADALLOM_S_DISCOVERY,Cloud App Security Discovery
RMS_S_ENTERPRISE,Microsoft Azure Active Directory Rights
AAD_PREMIUM_P2,Azure Active Directory Premium P2
RMS_S_PREMIUM2,Azure Information Protection Premium P2
RMS_S_ENTERPRISE),Microsoft Azure Active Directory Rights
ADALLOM_S_STANDALONE,Microsoft Cloud App Security
ATA,Microsoft Defender for Identity
EOP_ENTERPRISE_PREMIUM,Exchange Enterprise CAL Services (EOP DLP)
EXCHANGE_S_STANDARD,Exchange Online (Plan 1)
INTUNE_O365,Mobile Device Management for Office 365
BPOS_S_TODO_1,To-Do (Plan 1)
YAMMER_EDU,Yammer for Academic
RMS_S_BASIC,Microsoft Azure Rights Management Service
EXCHANGE_S_STANDARD_GOV,Exchange Online (Plan 1) for Government
EXCHANGE_S_ENTERPRISE,EXCHANGE ONLINE (PLAN 2)
EXCHANGE_S_ARCHIVE_ADDON,EXCHANGE ONLINE ARCHIVING FOR EXCHANGE ONLINE
EXCHANGE_S_ARCHIVE,EXCHANGE ONLINE ARCHIVING FOR EXCHANGE SERVER
EXCHANGE_S_STANDARD,EXCHANGE ONLINE (PLAN 1)
EXCHANGE_S_ESSENTIALS,EXCHANGE ESSENTIALS
BPOS_S_TODO_1,TO-DO (PLAN 1)
EXCHANGE_S_DESKLESS,EXCHANGE ONLINE KIOSK
EXCHANGE_B_STANDARD,EXCHANGE ONLINE POP
EOP_ENTERPRISE,Exchange Online Protection
ERP_TRIAL_INSTANCE,Dynamics 365 Operations Trial Environment
MTP,Microsoft 365 Defender
ATP_ENTERPRISE,Microsoft Defender for Office 365 (Plan 1)
THREAT_INTELLIGENCE,Microsoft Defender for Office 365 (Plan 2)
INTUNE_EDU,Intune for Education
AAD_BASIC_EDU,Azure Active Directory Basic for Education
CDS_O365_P2,Common Data Service for Teams
EducationAnalyticsP1,Education Analytics
EXCHANGE_S_ENTERPRISE,Exchange Online (Plan 2)
INFORMATION_BARRIERS,Information Barriers
ContentExplorer_Standard,Information Protection and Governance Analytics ? Standard
MIP_S_CLP1,Information Protection for Office 365 - Standard
MYANALYTICS_P2,Insights by MyAnalytics
OFFICESUBSCRIPTION,Microsoft 365 Apps for enterprise
MDE_LITE,Microsoft Defender for Endpoint Plan 1
OFFICE_FORMS_PLAN_2,Microsoft Forms (Plan 2)
KAIZALA_O365_P3,Microsoft Kaizala Pro
PROJECTWORKMANAGEMENT,Microsoft Planner
MICROSOFT_SEARCH,Microsoft Search
Deskless,Microsoft StaffHub
STREAM_O365_E3,Microsoft Stream for Office 365 E3
MINECRAFT_EDUCATION_EDITION,Minecraft Education Edition
Nucleus,Nucleus
ADALLOM_S_O365,Office 365 Cloud App Security
SHAREPOINTWAC_EDU,Office for the Web for Education
PROJECT_O365_P2,Project for Office (Plan E3)
SCHOOL_DATA_SYNC_P2,School Data Sync (Plan 2)
SHAREPOINTENTERPRISE_EDU,SharePoint (Plan 2) for Education
MCOSTANDARD,Skype for Business Online (Plan 2)
SWAY,Sway
BPOS_S_TODO_2,To-Do (Plan 2)
VIVA_LEARNING_SEEDED,Viva Learning Seeded
WHITEBOARD_PLAN2,Whiteboard (Plan 2)
UNIVERSAL_PRINT_01,Universal Print
Virtualization Rights for Windows 10 (E3/E5+VDA),Windows 10/11 Enterprise
WINDOWSUPDATEFORBUSINESS_DEPLOYMENTSERVICE,Windows Update for Business Deployment Service
DYN365_CDS_O365_P2,Common Data Service
POWERAPPS_O365_P2,Power Apps for Office 365
FLOW_O365_P2,Power Automate for Office 365
POWER_VIRTUAL_AGENTS_O365_P2,Power Virtual Agents for Office 365
UNIVERSAL_PRINT_NO_SEEDING,Universal Print Without Seeding
OFFICESUBSCRIPTION_unattended,Microsoft 365 Apps for Enterprise (Unattended)
CDS_O365_P3,Common Data Service for Teams
LOCKBOX_ENTERPRISE,Customer Lockbox
MIP_S_Exchange,Data Classification in Microsoft 365
Content_Explorer,Information Protection and Governance Analytics - Premium
MIP_S_CLP2,Information Protection for Office 365 - Premium
M365_ADVANCED_AUDITING,Microsoft 365 Advanced Auditing
OFFICESUBSCRIPTION,Microsoft 365 Apps for Enterprise
MICROSOFT_COMMUNICATION_COMPLIANCE,Microsoft 365 Communication Compliance
MCOEV,Microsoft 365 Phone System
COMMUNICATIONS_DLP,Microsoft Communications DLP
CUSTOMER_KEY,Microsoft Customer Key
DATA_INVESTIGATIONS,Microsoft Data Investigations
EXCEL_PREMIUM,Microsoft Excel Advanced Analytics
OFFICE_FORMS_PLAN_3,Microsoft Forms (Plan 3)
INFO_GOVERNANCE,Microsoft Information Governance
INSIDER_RISK,Microsoft Insider Risk Management
KAIZALA_STANDALONE,Microsoft Kaizala Pro
ML_CLASSIFICATION,Microsoft ML-Based Classification
EXCHANGE_ANALYTICS,Microsoft MyAnalytics (Full)
RECORDS_MANAGEMENT,Microsoft Records Management
STREAM_O365_E5,Microsoft Stream for Office 365 E5
EQUIVIO_ANALYTICS,Office 365 Advanced eDiscovery
PAM_ENTERPRISE,Office 365 Privileged Access Management
SAFEDOCS,Office 365 SafeDocs
POWERAPPS_O365_P3,Power Apps for Office 365 (Plan 3)
BI_AZURE_P2,Power BI Pro
PREMIUM_ENCRYPTION,Premium Encryption in Office 365
PROJECT_O365_P3,Project for Office (Plan E5)
COMMUNICATIONS_COMPLIANCE,Microsoft Communications Compliance
INSIDER_RISK_MANAGEMENT,Microsoft Insider Risk Management
BPOS_S_TODO_3,To-Do (Plan 3)
WHITEBOARD_PLAN3,Whiteboard (Plan 3)
WINDEFATP,Microsoft Defender for Endpoint
MICROSOFTENDPOINTDLP,Microsoft Endpoint DLP
DYN365_CDS_O365_P3,Common Data Service
ADALLOM_S_STANDALONE,Microsoft Defender for Cloud Apps
FLOW_O365_P3,Power Automate for Office 365
POWER_VIRTUAL_AGENTS_O365_P3,Power Virtual Agents for Office 365
COMMUNICATIONS_COMPLIANCE,RETIRED - Microsoft Communications Compliance
INSIDER_RISK_MANAGEMENT,RETIRED - Microsoft Insider Risk Management
AAD_BASIC_EDU,Azure Active Directory Basic for EDU
DYN365_CDS_O365_P3,Common Data Service - O365 P3
Content_Explorer,Information Protection and Governance Analytics - Premium)
KAIZALA_STANDALONE,Microsoft Kaizala
STREAM_O365_E3,Microsoft Stream for O365 E3 SKU
ADALLOM_S_O365,Office 365 Advanced Security Management
SHAREPOINTWAC_EDU,Office for the web (Education)
SHAREPOINTENTERPRISE_EDU,SharePoint Plan 2 for EDU
"Virtualization 	Rights 	for 	Windows 	10 	(E3/E5+VDA)",Windows 10 Enterprise (New)
FORMS_PLAN_E1,MICROSOFT FORMS (PLAN E1)
OFFICE_BUSINESS,OFFICE 365 BUSINESS
ONEDRIVESTANDARD,ONEDRIVESTANDARD
SWAY,SWAY
FORMS_PLAN_E1,Microsoft Forms (Plan E1)
ONEDRIVESTANDARD,OneDrive for Business (Plan 1)
OFFICE_PROPLUS_DEVICE,Microsoft 365 Apps for Enterprise (Device)
EXCHANGE_FOUNDATION_GOV,EXCHANGE FOUNDATION FOR GOVERNMENT
MCOMEETADV_GOV,MICROSOFT 365 AUDIO CONFERENCING FOR GOVERNMENT
MCOMEETACPEA,Microsoft 365 Audio Conferencing Pay-Per-Minute
FLOW_O365_P1,FLOW FOR OFFICE 365
MCOSTANDARD,SKYPE FOR BUSINESS ONLINE (PLAN 2)
OFFICEMOBILE_SUBSCRIPTION,OFFICEMOBILE_SUBSCRIPTION
POWERAPPS_O365_P1,POWERAPPS FOR OFFICE 365
PROJECTWORKMANAGEMENT,MICROSOFT PLANNE
SHAREPOINTSTANDARD,SHAREPOINTSTANDARD
TEAMS1,TEAMS1
YAMMER_ENTERPRISE,YAMMER_ENTERPRISE
VIVAENGAGE_CORE,Viva Engage Core
YAMMER_MIDSIZE,YAMMER MIDSIZE
OFFICE_BUSINESS,Microsoft 365 Apps for Business
KAIZALA_O365_P2,Microsoft Kaizala Pro
SHAREPOINTSTANDARD,SharePoint (Plan 1)
STREAM_O365_SMB,Stream for Office 365
WHITEBOARD_PLAN1,Whiteboard (Plan 1)
YAMMER_ENTERPRISE,Yammer Enterprise
POWERAPPS_O365_P1,Power Apps for Office 365
FLOW_O365_P1,Power Automate for Office 365
Deskless,MICROSOFT STAFFHUB
MICROSOFTBOOKINGS,MICROSOFTBOOKINGS
O365_SB_Relationship_Management,OUTLOOK CUSTOMER MANAGER
YAMMER_MIDSIZE,YAMMER_MIDSIZE
BPOS_S_DlpAddOn,Data Loss Prevention
EXCHANGE_S_ARCHIVE_ADDON,Exchange Online Archiving
M365_LIGHTHOUSE_CUSTOMER_PLAN1,Microsoft 365 Lighthouse (Plan 1)
M365_LIGHTHOUSE_PARTNER_PLAN1,Microsoft 365 Lighthouse (Plan 2)
MDE_SMB,Microsoft Defender for Business
OFFICE_SHARED_COMPUTER_ACTIVATION,Office Shared Computer Activation
O365_SB_Relationship_Management,RETIRED - Outlook Customer Manager
WINBIZ,Windows 10/11 Business
AAD_SMB,Azure Active Directory
INTUNE_SMBIZ,Microsoft Intune
STREAM_O365_E1,Microsoft Stream for Office 365 E1
MCOPSTN1,Microsoft 365 Domestic Calling Plan
MCOPSTN5,MICROSOFT 365 DOMESTIC CALLING PLAN (120 min)
MCOPSTN1_GOV,Domestic Calling for Government
ContentExplorer_Standard,Information Protection and Governance Analytics - Standard
FORMS_PLAN_E3,Microsoft Forms (Plan E3)
WIN10_PRO_ENT_SUB,Windows 10/11 Enterprise (Original)
Windows_Autopatch,Windows Autopatch
Windows Autopatch,Windows Autopatch
TEAMS_AR_DOD,Microsoft Teams for DOD (AR)
OFFICESUBSCRIPTION,Office 365 ProPlus
SHAREPOINTWAC,Office Online
SHAREPOINTENTERPRISE,SharePoint Online (Plan 2)
RMS_S_PREMIUM,Azure Information Protection Premium P
TEAMS_AR_GCCHIGH,Microsoft Teams for GCCHigh (AR)
GRAPH_CONNECTORS_SEARCH_INDEX,Graph Connectors Search with Index
MCOPSTN8,Microsoft 365 Domestic Calling Plan (120 min) at User Level
RMS_S_ENTERPRISE_GOV,Azure Rights Management
STREAM_O365_K,Microsoft Stream for O365 K SKU
SHAREPOINTDESKLESS,SharePoint Online Kiosk
MCOIMP,Skype for Business Online (Plan 1)
CDS_O365_F1,Common Data Service for Teams
EXCHANGE_S_DESKLESS,Exchange Online Kiosk
FORMS_PLAN_K,Microsoft Forms (Plan F1)
KAIZALA_O365_P1,Microsoft Kaizala Pro
OFFICEMOBILE_SUBSCRIPTION,Office Mobile Apps for Office 365
PROJECT_O365_F3,Project for Office (Plan F)
SHAREPOINTDESKLESS,SharePoint Kiosk
BPOS_S_TODO_FIRSTLINE,To-Do (Firstline)
WHITEBOARD_FIRSTLINE1,Whiteboard (Firstline)
WIN10_ENT_LOC_F1,Windows 10 Enterprise E3 (Local Only)
DYN365_CDS_O365_F1,Common Data Service
STREAM_O365_K,Microsoft Stream for Office 365 F3
POWERAPPS_O365_S1,Power Apps for Office 365 F3
FLOW_O365_S1,Power Automate for Office 365 F3
POWER_VIRTUAL_AGENTS_O365_F1,Power Virtual Agents for Office 365
RMS_S_PREMIUM_GOV,Azure Information Protection Premium P1 for GCC
DYN365_CDS_O365_F1_GCC,Common Data Service - O365 F1
CDS_O365_F1_GCC,Common Data Service for Teams_F1 GCC
EXCHANGE_S_DESKLESS_GOV,Exchange Online (Kiosk) for Government
FORMS_GOV_F1,Forms for Government (Plan F1)
STREAM_O365_K_GOV,Microsoft Stream for O365 for Government (F1)
TEAMS_GOV,Microsoft Teams for Government
PROJECTWORKMANAGEMENT_GOV,Office 365 Planner for Government
SHAREPOINTWAC_GOV,Office for the Web for Government
OFFICEMOBILE_SUBSCRIPTION_GOV,Office Mobile Apps for Office 365 for GCC
POWERAPPS_O365_S1_GOV,Power Apps for Office 365 F3 for Government
FLOW_O365_S1_GOV,Power Automate for Office 365 F3 for Government
SHAREPOINTDESKLESS_GOV,SharePoint KioskG
MCOIMP_GOV,Skype for Business Online (Plan 1) for Government
ADALLOM_S_STANDALONE_DOD,Microsoft Defender for Cloud Apps for DOD
LOCKBOX_ENTERPRISE_GOV,Customer Lockbox for Government
EQUIVIO_ANALYTICS_GOV,Office 365 Advanced eDiscovery for Government
CDS_O365_P3_GCC,Common Data Service for Teams
EXCHANGE_S_ENTERPRISE_GOV,Exchange Online (Plan 2) for Government
OFFICESUBSCRIPTION_GOV,Microsoft 365 Apps for enterprise G
MCOMEETADV_GOV,Microsoft 365 Audio Conferencing for Government
MCOEV_GOV,Microsoft 365 Phone System for Government
ATP_ENTERPRISE_GOV,Microsoft Defender for Office 365 (Plan 1) for Government
THREAT_INTELLIGENCE_GOV,Microsoft Defender for Office 365 (Plan 2) for Government
FORMS_GOV_E5,Microsoft Forms for Government (Plan E5)
EXCHANGE_ANALYTICS_GOV,Microsoft MyAnalytics for Government (Full)
BI_AZURE_P_2_GOV,Power BI Pro for Government
SHAREPOINTENTERPRISE_GOV,SharePoint Plan 2G
MCOSTANDARD_GOV,Skype for Business Online (Plan 2) for Government
STREAM_O365_E5_GOV,Stream for Office 365 for Government (E5)
RMS_S_PREMIUM2_GOV,Azure Information Protection Premium P2 for GCC
DYN365_CDS_O365_P3_GCC,Common Data Service
POWERAPPS_O365_P3_GOV,Power Apps for Office 365 for Government
FLOW_O365_P3_GOV,Power Automate for Office 365 for Government
DYN365_CDS_VIRAL,COMMON DATA SERVICE - VIRAL
FLOW_P2_VIRAL,FLOW FREE
EXCHANGE_S_FOUNDATION_GOV,EXCHANGE FOUNDATION FOR GOVERNMENT
ML_CLASSIFICATION,Microsoft ML-based classification
AAD_PREMIUM,AAD_PREMIUM
RMS_S_PREMIUM,RMS_S_PREMIUM
ADALLOM_S_DISCOVERY,ADALLOM_S_DISCOVERY
DYN365_CDS_O365_F1,DYN365_CDS_O365_F1
EXCHANGE_S_DESKLESS,EXCHANGE_S_DESKLESS
RMS_S_ENTERPRISE,RMS_S_ENTERPRISE
MFA_PREMIUM,MFA_PREMIUM
INTUNE_A,INTUNE_A
PROJECTWORKMANAGEMENT,PROJECTWORKMANAGEMENT
MICROSOFT_SEARCH,MICROSOFT_SEARCH
STREAM_O365_K,STREAM_O365_K
INTUNE_O365,INTUNE_O365
SHAREPOINTDESKLESS,SHAREPOINTDESKLESS
MCOIMP,MCOIMP
RMS_S_ENTERPRISE_GOV,AZURE RIGHTS MANAGEMENT
RMS_S_PREMIUM_GOV,AZURE RIGHTS MANAGEMENT PREMIUM FOR GOVERNMENT
DYN365_CDS_O365_P2_GCC,COMMON DATA SERVICE - O365 P2 GCC
CDS_O365_P2_GCC,COMMON DATA SERVICE FOR TEAMS_P2 GCC
EXCHANGE_S_ENTERPRISE_GOV,EXCHANGE PLAN 2G
FORMS_GOV_E3,FORMS FOR GOVERNMENT (PLAN E3)
CONTENT_EXPLORER,INFORMATION PROTECTION AND GOVERNANCE ANALYTICS - PREMIUM
CONTENTEXPLORER_STANDARD,INFORMATION PROTECTION AND GOVERNANCE ANALYTICS - STANDARD
MIP_S_CLP1,INFORMATION PROTECTION FOR OFFICE 365 - STANDARD
MYANALYTICS_P2_GOV,INSIGHTS BY MYANALYTICS FOR GOVERNMENT
OFFICESUBSCRIPTION_GOV,MICROSOFT 365 APPS FOR ENTERPRISE G
MFA_PREMIUM,MICROSOFT Azure Multi-Factor Authentication
MICROSOFTBOOKINGS,MICROSOFT BOOKINGS
STREAM_O365_E3_GOV,MICROSOFT STREAM FOR O365 FOR GOVERNMENT (E3)
TEAMS_GOV,MICROSOFT TEAMS FOR GOVERNMENT
PROJECTWORKMANAGEMENT_GOV,OFFICE 365 PLANNER FOR GOVERNMENT
SHAREPOINTWAC_GOV,OFFICE FOR THE WEB (GOVERNMENT)
POWERAPPS_O365_P2_GOV,POWER APPS FOR OFFICE 365 FOR GOVERNMENT
FLOW_O365_P2_GOV,POWER AUTOMATE FOR OFFICE 365 FOR GOVERNMENT
SHAREPOINTENTERPRISE_GOV,SHAREPOINT PLAN 2G
MCOSTANDARD_GOV,SKYPE FOR BUSINESS ONLINE (PLAN 2) FOR GOVERNMENT
EXCHANGE_S_ARCHIVE_ADDON,Exchange Online Archiving for Exchange Online
MICROSOFT_COMMUNICATION_COMPLIANCE,M365 Communication Compliance
WINDEFATP,Microsoft Defender For Endpoint
MICROSOFT_BUSINESS_CENTER,MICROSOFT BUSINESS CENTER
WINDEFATP,MICROSOFT DEFENDER FOR ENDPOINT
Intune_Defender,MDE_SecurityManagement
CRMPLAN2,MICROSOFT DYNAMICS CRM ONLINE BASIC
ADALLOM_FOR_AATP,SecOps Investigation for MDI
CRMSTANDARD,MICROSOFT DYNAMICS CRM ONLINE PROFESSIONA
MDM_SALES_COLLABORATION,MICROSOFT DYNAMICS MARKETING SALES COLLABORATION - ELIGIBILITY CRITERIA APPLY
NBPROFESSIONALFORCRM,MICROSOFT SOCIAL ENGAGEMENT PROFESSIONAL - ELIGIBILITY CRITERIA APPLY
IT_ACADEMY_AD,MS IMAGINE ACADEMY
DYN365_CDS_DEV_VIRAL,Common Data Service - DEV VIRAL
FLOW_DEV_VIRAL,Flow for Developer
POWERAPPS_DEV_VIRAL,PowerApps for Developer
DYN365_CDS_VIRAL,Common Data Service - VIRAL
FLOW_P2_VIRAL,Flow Free
FLOW_P2_VIRAL_REAL,Flow P2 Viral
POWERAPPS_P2_VIRAL,PowerApps Trial
DYN365_CDS_P2,Common Data Service - P2
FLOW_P2,Power Automate (Plan 2)
AAD_SMB,AZURE ACTIVE DIRECTORY
INTUNE_SMBIZ,MICROSOFT INTUNE
POWERAPPS_P2,Power Apps (Plan 2)
MICROSOFTSTREAM,MICROSOFT STREAM
STREAM_P2,Microsoft Stream Plan 2
STREAM_STORAGE,Microsoft Stream Storage Add-On
MCOMEETBASIC,Microsoft Teams Audio Conferencing with dial-out to select geographies
MCOFREE,MCO FREE FOR MICROSOFT TEAMS (FREE)
TEAMS_FREE,MICROSOFT TEAMS (FREE)
SHAREPOINTDESKLESS,SHAREPOINT KIOSK
TEAMS_FREE_SERVICE,TEAMS FREE SERVICE
WHITEBOARD_FIRSTLINE1,WHITEBOARD (FIRSTLINE)
TeamsEss,Microsoft Teams Essentials
CDS_O365_P1,COMMON DATA SERVICE FOR TEAMS_P1
MYANALYTICS_P2,INSIGHTS BY MYANALYTICS
PROJECTWORKMANAGEMENT,MICROSOFT PLANNER
MICROSOFT_SEARCH,MICROSOFT SEARCH
DESKLESS,MICROSOFT STAFFHUB
STREAM_O365_E1,MICROSOFT STREAM FOR O365 E1 SKU
TEAMS1,MICROSOFT TEAMS
MCO_TEAMS_IW,MICROSOFT TEAMS
INTUNE_O365,MOBILE DEVICE MANAGEMENT FOR OFFICE 365
OFFICEMOBILE_SUBSCRIPTION,OFFICE MOBILE APPS FOR OFFICE 365
POWERAPPS_O365_P1,POWER APPS FOR OFFICE 365
FLOW_O365_P1,POWER AUTOMATE FOR OFFICE 365
POWER_VIRTUAL_AGENTS_O365_P1,POWER VIRTUAL AGENTS FOR OFFICE 365 P1
SHAREPOINTSTANDARD,SHAREPOINT STANDARD
WHITEBOARD_PLAN1,WHITEBOARD (PLAN 1)
YAMMER_ENTERPRISE,YAMMER ENTERPRIS
MCOEV,MICROSOFT 365 PHONE SYSTEM
MCOEV,MICROSOFT 365 PHONE SYSTE
MCOEV_GOV,MICROSOFT 365 PHONE SYSTEM FOR GOVERNMENT
MCOEVSMB,SKYPE FOR BUSINESS CLOUD PBX FOR SMALL AND MEDIUM BUSINESS
MCOEV_VIRTUALUSER,Microsoft 365 Phone Standard Resource Account
MCOEV_VIRTUALUSER_GOV,Microsoft 365 Phone Standard Resource Account for Government
MICROSOFT_ECDN,Microsoft eCDN
TEAMSPRO_MGMT,Microsoft Teams Premium Intelligent
TEAMSPRO_CUST,Microsoft Teams Premium Personalized
TEAMSPRO_PROTECTION,Microsoft Teams Premium Secure
TEAMSPRO_VIRTUALAPPT,Microsoft Teams Premium Virtual Appointment
MCO_VIRTUAL_APPT,Microsoft Teams Premium Virtual Appointments
TEAMSPRO_WEBINAR,Microsoft Teams Premium Webinar
AAD_PREMIUM,Azure Active Directory Premium Plan 1
Teams_Room_Standard,Teams Room Standard
MCO_TEAMS_IW,Microsoft Teams
EXPERTS_ON_DEMAND,Microsoft Threat Experts - Experts on Demand
EXCHANGEONLINE_MULTIGEO,Exchange Online Multi-Geo
SHAREPOINTONLINE_MULTIGEO,SharePoint Multi-Geo
TEAMSMULTIGEO,Teams Multi-Geo
NONPROFIT_PORTAL,Nonprofit Portal
DYN365_CDS_O365_P1,Common Data Service - O365 P1
KAIZALA_O365_P2,Microsoft Kaizala Pro Plan 2
PROJECT_O365_P1,Project for Office (Plan E1)
SCHOOL_DATA_SYNC_P1,School Data Sync (Plan 1)
SHAREPOINTSTANDARD_EDU,SharePoint (Plan 1) for Education
DYN365_CDS_O365_P2,Common Data Service - O365 P2
CDS_O365_P2,Common Data Service for Teams_P2
KAIZALA_O365_P3,Microsoft Kaizala Pro Plan 3
POWER_VIRTUAL_AGENTS_O365_P2,Power Virtual Agents for Office 365 P2
Content_Explorer,Information Protection and Governance Analytics -Premium
EXCHANGE_S_FOUNDATION_GOV,EXCHANGE_S_FOUNDATION_GOV
SHAREPOINTSTORAGE_GOV,SHAREPOINTSTORAGE_GOV
CDS_O365_P1,Common Data Service for Teams_P1
STREAM_O365_E1,Microsoft Stream for O365 E1 SKU
POWER_VIRTUAL_AGENTS_O365_P1,Power Virtual Agents for Office 365 P1
SHAREPOINTSTORAGE,Office 365 Extra File Storage
CDS_O365_P1,Common Data Service for Teams
DYN365_CDS_O365_P1,Common Data Service
POWER_VIRTUAL_AGENTS_O365_P1,Power Virtual Agents for Office 365
BPOS_S_TODO_1,BPOS_S_TODO_1
BPOS_S_TODO_3,BPOS_S_TODO_3
FLOW_O365_P2,FLOW FOR OFFICE 365
FORMS_PLAN_E5,MICROSOFT FORMS (PLAN E5)
OFFICESUBSCRIPTION,OFFICESUBSCRIPTION
POWERAPPS_O365_P2,POWERAPPS FOR OFFICE 36
SHAREPOINT_S_DEVELOPER,SHAREPOINT FOR DEVELOPER
SHAREPOINTWAC_DEVELOPER,OFFICE ONLINE FOR DEVELOPER
STREAM_O365_E5,MICROSOFT STREAM FOR O365 E5 SKU
BPOS_S_TODO_2,BPOS_S_TODO_2
FORMS_PLAN_E3,MICROSOFT FORMS (PLAN E3)
MCOVOICECONF,SKYPE FOR BUSINESS ONLINE (PLAN 3)
STREAM_O365_E3,MICROSOFT STREAM FOR O365 E3 SKU
CDS_O365_P3,Common Data Service for Teams_P3
STREAM_O365_E5,Microsoft Stream for O365 E5 SKU
POWER_VIRTUAL_AGENTS_O365_P3,Power Virtual Agents for Office 365 P3
POWERAPPS_O365_P3,PowerApps for Office 365 Plan 3
DYN365_CDS_O365_F1,Common Data Service - O365 F1
CDS_O365_F1,Common Data Service for Teams_F1
KAIZALA_O365_P1,Microsoft Kaizala Pro Plan 1
POWER_VIRTUAL_AGENTS_O365_F1,Power Virtual Agents for Office 365 F1
DYN365_CDS_O365_P1_GCC,Common Data Service - O365 P1 GCC
CDS_O365_P1_GCC,Common Data Service for Teams_P1 GCC
FORMS_GOV_E1,Forms for Government (Plan E1)
MYANALYTICS_P2_GOV,Insights by MyAnalytics for Government
STREAM_O365_E1_GOV,Microsoft Stream for O365 for Government (E1)
POWERAPPS_O365_P1_GOV,Power Apps for Office 365 for Government
FLOW_O365_P1_GOV,Power Automate for Office 365 for Government
SharePoint Plan 1G,SharePoint Plan 1G
Content_Explorer,INFORMATION PROTECTION AND GOVERNANCE ANALYTICS - PREMIUM
ContentExplorer_Standard,INFORMATION PROTECTION AND GOVERNANCE ANALYTICS - STANDARD
RMS_S_ENTERPRISE_GOV,RMS_S_ENTERPRISE_GOV
DYN365_CDS_O365_P3_GCC,DYN365_CDS_O365_P3_GCC
CDS_O365_P3_GCC,CDS_O365_P3_GCC
LOCKBOX_ENTERPRISE_GOV,LOCKBOX_ENTERPRISE_GOV
EXCHANGE_S_ENTERPRISE_GOV,EXCHANGE_S_ENTERPRISE_GOV
FORMS_GOV_E5,FORMS_GOV_E5
INFORMATION_BARRIERS,INFORMATION_BARRIERS
Content_Explorer,Content_Explorer
ContentExplorer_Standard,ContentExplorer_Standard
MIP_S_CLP2,MIP_S_CLP2
MIP_S_CLP1,MIP_S_CLP1
MICROSOFT_COMMUNICATION_COMPLIANCE,MICROSOFT_COMMUNICATION_COMPLIANCE
M365_ADVANCED_AUDITING,M365_ADVANCED_AUDITING
OFFICESUBSCRIPTION_GOV,OFFICESUBSCRIPTION_GOV
MCOMEETADV_GOV,MCOMEETADV_GOV
MTP,MTP
MCOEV_GOV,MCOEV_GOV
COMMUNICATIONS_DLP,COMMUNICATIONS_DLP
CUSTOMER_KEY,CUSTOMER_KEY
ATP_ENTERPRISE_GOV,ATP_ENTERPRISE_GOV
THREAT_INTELLIGENCE_GOV,THREAT_INTELLIGENCE_GOV
INFO_GOVERNANCE,INFO_GOVERNANCE
EXCHANGE_ANALYTICS_GOV,EXCHANGE_ANALYTICS_GOV
RECORDS_MANAGEMENT,RECORDS_MANAGEMENT
STREAM_O365_E5_GOV,STREAM_O365_E5_GOV
TEAMS_GOV,TEAMS_GOV
EQUIVIO_ANALYTICS_GOV,EQUIVIO_ANALYTICS_GOV
ADALLOM_S_O365,ADALLOM_S_O365
PROJECTWORKMANAGEMENT_GOV,PROJECTWORKMANAGEMENT_GOV
SHAREPOINTWAC_GOV,SHAREPOINTWAC_GOV
POWERAPPS_O365_P3_GOV,POWERAPPS_O365_P3_GOV
FLOW_O365_P3_GOV,FLOW_O365_P3_GOV
BI_AZURE_P_2_GOV,BI_AZURE_P_2_GOV
PREMIUM_ENCRYPTION,PREMIUM_ENCRYPTION
SHAREPOINTENTERPRISE_GOV,SHAREPOINTENTERPRISE_GOV
MCOSTANDARD_GOV,MCOSTANDARD_GOV
EXCHANGE_S_STANDARD_MIDMARKET,EXCHANGE ONLINE PLAN
MCOSTANDARD_MIDMARKET,SKYPE FOR BUSINESS ONLINE (PLAN 2) FOR MIDSIZ
SHAREPOINTENTERPRISE_MIDMARKET,SHAREPOINT PLAN 1
EXCHANGE_L_STANDARD,EXCHANGE ONLINE (P1)
MCOLITE,SKYPE FOR BUSINESS ONLINE (PLAN P1)
SHAREPOINTLITE,SHAREPOINTLITE
OFFICE_PRO_PLUS_SUBSCRIPTION_SMBIZ,OFFICE 365 SMALL BUSINESS SUBSCRIPTION
ONEDRIVEENTERPRISE,ONEDRIVEENTERPRISE
POWERFLOWSFREE,LOGIC FLOWS
POWERVIDEOSFREE,MICROSOFT POWER VIDEOS BASIC
POWERAPPSFREE,MICROSOFT POWERAPPS
CDS_PER_APP_IWTRIAL,CDS Per app baseline access
Flow_Per_APP_IWTRIAL,Flow per app baseline access
POWERAPPS_PER_APP_IWTRIAL,PowerApps per app baseline access
CDS_PER_APP,CDS PowerApps per app plan
POWERAPPS_PER_APP,Power Apps per App Plan
Flow_Per_APP,Power Automate for Power Apps per App Plan
CDSAICAPACITY_PERAPP,AI Builder capacity Per App add-on
DATAVERSE_POWERAPPS_PER_APP_NEW,Dataverse for Power Apps per app
POWERAPPS_PER_APP_NEW,Power Apps per app
POWERAPPS_PER_USER,Power Apps per User Plan
Flow_PowerApps_PerUser,Power Automate for Power Apps per User Plan
CDSAICAPACITY_PERUSER,AI Builder capacity Per User add-on
CDSAICAPACITY_PERUSER_NEW,AI Builder capacity Per User add-on
DYN365_CDS_P2_GOV,Common Data Service for Government
POWERAPPS_PER_USER_GCC,Power Apps per User Plan for Government
Flow_PowerApps_PerUser_GCC,Power Automate for Power Apps per User Plan for GCC
DYN365_CDS_P1_GOV,Common Data Service for Government
FLOW_P1_GOV,Power Automate (Plan 1) for Government
POWERAPPS_P1_GOV,PowerApps Plan 1 for Government
CDS_POWERAPPS_PORTALS_LOGIN,Common Data Service Power Apps Portals Login Capacity
POWERAPPS_PORTALS_LOGIN,Power Apps Portals Login Capacity Add-On
CDS_POWERAPPS_PORTALS_LOGIN_GCC,Common Data Service Power Apps Portals Login Capacity for GCC
POWERAPPS_PORTALS_LOGIN_GCC,Power Apps Portals Login Capacity Add-On for Government
CDS_POWERAPPS_PORTALS_PAGEVIEW_GCC,CDS PowerApps Portals page view capacity add-on for GCC
POWERAPPS_PORTALS_PAGEVIEW_GCC,Power Apps Portals Page View Capacity Add-On for Government
CDS_Flow_Business_Process,Common data service for Flow per business process plan
FLOW_BUSINESS_PROCESS,Flow per business process plan
FLOW_PER_USER,Flow per user plan
FLOW_PER_USER_GCC,Power Automate per User Plan for Government
CDS_ATTENDED_RPA,Common Data Service Attended RPA
POWER_AUTOMATE_ATTENDED_RPA,Power Automate RPA Attended
CDS_UNATTENDED_RPA,Common Data Service Unattended RPA
POWER_AUTOMATE_UNATTENDED_RPA,Power Automate Unattended RPA add-on
SQL_IS_SSIM,Microsoft Power BI Information Services Plan 1
BI_AZURE_P1,Microsoft Power BI Reporting and Analytics Plan 1
BI_AZURE_P0,Power BI (free)
BI_AZURE_P1,MICROSOFT POWER BI REPORTING AND ANALYTICS PLAN 1
SQL_IS_SSIM,MICROSOFT POWER BI INFORMATION SERVICES PLAN
PBI_PREMIUM_P1_ADDON,Power BI Premium P
BI_AZURE_P3,Power BI Premium Per User
CDS_VIRTUAL_AGENT_BASE,Common Data Service for Virtual Agent Base
FLOW_VIRTUAL_AGENT_BASE,Power Automate for Virtual Agent
VIRTUAL_AGENT_BASE,Virtual Agent Base
DYN365_CDS_CCI_BOTS,Common Data Service for CCI Bots
FLOW_CCI_BOTS,Flow for CCI Bots
PROJECT_CLIENT_SUBSCRIPTION,PROJECT ONLINE DESKTOP CLIENT
PROJECT_ESSENTIALS_GOV,Project Online Essentials for Government
SHAREPOINT_PROJECT,SHAREPOINT_PROJECT
DYN365_CDS_FOR_PROJECT_P1,COMMON DATA SERVICE FOR PROJECT P1
Power_Automate_For_Project_P1,POWER AUTOMATE FOR PROJECT P1
PROJECT_P1,PROJECT P1
SHAREPOINTSTANDARD,SHAREPOINT
DYN365_CDS_FOR_PROJECT_P1,Common Data Service for Project P1
Power_Automate_For_Project_P1,Power Automate for Project P1
PROJECT_P1,Project P1
DYN365_CDS_PROJECT,Common Data Service for Project
FLOW_FOR_PROJECT,Flow for Project
PROJECT_PROFESSIONAL,Project P3
SHAREPOINT_PROJECT_EDU,Project Online Service for Education
PROJECT_PROFESSIONAL_FACULTY,Project P3 for Faculty
FLOW_FOR_PROJECT,Power Automate for Project
SHAREPOINTWAC_GOV,Office for the web (Government)
PROJECT_CLIENT_SUBSCRIPTION_GOV,Project Online Desktop Client for Government
SHAREPOINT_PROJECT_GOV,Project Online Service for Government
RMS_S_ADHOC,Rights Management Adhoc
D365_IOTFORSCM_ADDITIONAL,IoT Intelligence Add-in Additional Machines
D365_IOTFORSCM,Iot Intelligence Add-in for D365 Supply Chain Management
CDS_O365_E5_KM,Common Data Service for SharePoint Syntex
Intelligent_Content_Services,SharePoint Syntex
Intelligent_Content_Services_SPO_type,SharePoint Syntex - SPO type
MCOIMP,SKYPE FOR BUSINESS ONLINE (PLAN 1)
MCOPSTN2,DOMESTIC AND INTERNATIONAL CALLING PLAN
MCOPSTN1,DOMESTIC CALLING PLAN
MCOPSTN5,DOMESTIC CALLING PLAN
MCOPSTN3,MCOPSTN3
MMR_P1,Meeting Room Managed Services
MCOPSTNEAU,AUSTRALIA CALLING PLAN
ONEDRIVE_BASIC,OneDrive for business Basic
VISIOONLINE,Visio web app
ONEDRIVE_BASIC,OneDrive for Business (Basic)
VISIO_CLIENT_SUBSCRIPTION,Visio Desktop App
VISIOONLINE,Visio Web App
ONEDRIVE_BASIC,ONEDRIVE FOR BUSINESS BASIC
VISIOONLINE,VISIO WEB APP
VISIO_CLIENT_SUBSCRIPTION,VISIO DESKTOP APP
ONEDRIVE_BASIC_GOV,ONEDRIVE FOR BUSINESS BASIC FOR GOVERNMENT
VISIO_CLIENT_SUBSCRIPTION_GOV,VISIO DESKTOP APP FOR Government
VISIOONLINE_GOV,VISIO WEB APP FOR GOVERNMENT
GRAPH_CONNECTORS_SEARCH_INDEX_TOPICEXP,Graph Connectors Search with Index (Viva Topics)
CORTEX,Viva Topics
DATAVERSE_FOR_POWERAUTOMATE_DESKTOP,Dataverse for PAD
POWERAUTOMATE_DESKTOP_FOR_WIN,PAD for Windows
WIN10_PRO_ENT_SUB,WINDOWS 10 ENTERPRISE
UNIVERSAL_PRINT_01,UNIVERSAL PRINT
"Virtualization 	Rights 	for 	Windows 	10 	(E3/E5+VDA)",WINDOWS 10 ENTERPRISE (NEW)
WINDOWSUPDATEFORBUSINESS_DEPLOYMENTSERVICE,WINDOWS UPDATE FOR BUSINESS DEPLOYMENT SERVICE
CPC_B_1C_2RAM_64GB,Windows 365 Business 1 vCPU 2 GB 64 GB
CPC_B_2C_4RAM_128GB,Windows 365 Business 2 vCPU 4 GB 128 GB
CPC_B_2C_4RAM_256GB,Windows 365 Business 2 vCPU 4 GB 256 GB
CPC_B_2C_4RAM_64GB,Windows 365 Business 2 vCPU 4 GB 64 GB
CPC_SS_2,"Windows 365 Business 2 vCPU, 8 GB, 128 GB"
CPC_B_2C_8RAM_256GB,Windows 365 Business 2 vCPU 8 GB 256 GB
CPC_B_4C_16RAM_128GB,Windows 365 Business 4 vCPU 16 GB 128 GB
CPC_B_4C_16RAM_256GB,Windows 365 Business 4 vCPU 16 GB 256 GB
CPC_B_4C_16RAM_512GB,Windows 365 Business 4 vCPU 16 GB 512 GB
CPC_B_8C_32RAM_128GB,Windows 365 Business 8 vCPU 32 GB 128 GB
CPC_B_8C_32RAM_256GB,Windows 365 Business 8 vCPU 32 GB 256 GB
CPC_B_8C_32RAM_512GB,Windows 365 Business 8 vCPU 32 GB 512 GB
CPC_E_1C_2GB_64GB,Windows 365 Enterprise 1 vCPU 2 GB 64 GB
CPC_E_2C_4GB_64GB,Windows 365 Enterprise 2 vCPU 4 GB 64 GB
CPC_1,Windows 365 Enterprise 2 vCPU 4 GB 128 GB
CPC_E_2C_4GB_256GB,Windows 365 Enterprise 2 vCPU 4 GB 256 GB
CPC_2,Windows 365 Enterprise 2 vCPU 8 GB 128 GB
CPC_E_2C_8GB_256GB,Windows 365 Enterprise 2 vCPU 8 GB 256 GB
CPC_E_4C_16GB_128GB,Windows 365 Enterprise 4 vCPU 16 GB 128 GB
CPC_E_4C_16GB_256GB,Windows 365 Enterprise 4 vCPU 16 GB 256 GB
CPC_E_4C_16GB_512GB,Windows 365 Enterprise 4 vCPU 16 GB 512 GB
CPC_E_8C_32GB_128GB,Windows 365 Enterprise 8 vCPU 32 GB 128 GB
CPC_E_8C_32GB_256GB,Windows 365 Enterprise 8 vCPU 32 GB 256 GB
CPC_E_8C_32GB_512GB,Windows 365 Enterprise 8 vCPU 32 GB 512 GB
CPC_S_2C_4GB_64GB,Windows 365 Shared Use 2 vCPU 4 GB 64 GB
CPC_S_2C_4GB_128GB,Windows 365 Shared Use 2 vCPU 4 GB 128 GB
CPC_S_2C_4GB_256GB,Windows 365 Shared Use 2 vCPU 4 GB 256 GB
CPC_S_2C_8GB_128GB,Windows 365 Shared Use 2 vCPU 8 GB 128 GB
CPC_S_2C_8GB_256GB,Windows 365 Shared Use 2 vCPU 8 GB 256 GB
CPC_S_4C_16GB_128GB,Windows 365 Shared Use 4 vCPU 16 GB 128 GB
CPC_S_4C_16GB_256GB,Windows 365 Shared Use 4 vCPU 16 GB 256 GB
CPC_S_4C_16GB_512GB,Windows 365 Shared Use 4 vCPU 16 GB 512 GB
CPC_S_8C_32GB_128GB,Windows 365 Shared Use 8 vCPU 32 GB 128 GB
CPC_S_8C_32GB_256GB,Windows 365 Shared Use 8 vCPU 32 GB 256 GB
CPC_S_8C_32GB_512GB,Windows 365 Shared Use 8 vCPU 32 GB 512 GB
WINDOWS_STORE,WINDOWS STORE SERVICE
Windows Store for Business EDU Store_faculty,Windows Store for Business EDU Store_faculty
WORKPLACE_ANALYTICS,Microsoft Workplace Analytics
WORKPLACE_ANALYTICS_INSIGHTS_BACKEND,Microsoft Workplace Analytics Insights Backend
WORKPLACE_ANALYTICS_INSIGHTS_USER,Microsoft Workplace Analytics Insights User

```
