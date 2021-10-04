#Connect-AzAccount
#########################Variables##############################
param(
    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
	[string] $TenantID = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
	[string] $AppID = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
	[string] $Secret = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [string] $ReportName = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $ReportPath = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $WorkSpaceName = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $DbName = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $DbServer = "",

    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $sqlUserName = "",
	
	[Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
	[string] $ParamDbName = "",
	
	[Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
	[string] $ParamDbServerName = "",
	
	[Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String] $sqlUserPassword = ""

)
<#
$TenantID = ""
$AppID = ""
$Secret = ""
$ReportName = ""
$ReportPath = "" #local path of pbix file
$WorkSpaceName = ""
$DbName = ""
$DbServer = ""
$sqlUserName = ""
sqlUserPassword = ""
$ParamDbName = ""
$ParamDbServerName = ""
#>
#######################################################################

try
{
Write-Host "`n##[info]-->Importing Power BI Modules.`n"
Get-Module -ListAvailable MicrosoftPowerBI* | Import-Module
Write-Host "##[section]-->Power BI Modules import Successful.`n"

#Connect to PBI
Write-Host "##[info]-->Connecting to PBI.`n"
$Password = ConvertTo-SecureString $Secret -AsPlainText -Force
$creds = New-Object PSCredential $AppID, $Password

$connect = Connect-PowerBIServiceAccount -ServicePrincipal -credential $creds -Tenant $TenantID

if($? -eq $false)
        {
            Write-Host "##[error]-->Connection Failure to PBI Service Account.`n"
Resolve-PowerBIError -Last
throw
        }
else
        {
            Write-Host "##[section]-->Successfully connected to PBI Service Account.`n"
        }

#Get PBI Access Token
#$headers = Get-PowerBIAccessToken

#Get Workspace
Write-Host "##[info]-->Getting workspace.`n"
$Workspace = Get-PowerBIWorkspace -Name $WorkSpaceName

if($Workspace -eq $null)
        {
            Write-Host "##[error]-->Unable to find --> $WorkSpaceName <-- in available workspace.`n"
Resolve-PowerBIError -Last
throw
        }
        
else
        {
            Write-Host "##[section]-->Successfully found --> $WorkSpaceName <-- in available workspaces.`n"
        }

#Get existing Report.

Write-Host "##[info]-->Checking if $ReportName already exists.`n"

$OldReport = Get-PowerBIReport -WorkspaceId $Workspace.Id -Name $ReportName


if($OldReport -eq $null)
        {
            Write-Host "##[info]-->Report named $ReportName does not exists in workspace $WorkSpaceName.`nA new report will be created with name $ReportName.`n"
            Write-Host "##[info]-->Publishing New Report.`n"
            $NewReport = New-PowerBIReport -Path $ReportPath -Name $ReportName -WorkspaceId $Workspace.Id -ConflictAction CreateOrOverwrite
            Start-Sleep -Seconds 30
            $Report = Get-PowerBIReport -WorkspaceId $Workspace.Id -Id $NewReport.Id
            $NewReportId = $Report.Id
            $NewDatasetId = $Report.DatasetId
            #Write-Host "##[Warning]-->New Report Created. The Report ID and Dataset ID need to be entered in DB`n`nReport ID = $NewReportId`n`nDataset ID = $NewDatasetId`n"
Write-Host "##[error]-->New Report Created. The Report ID and Dataset ID need to be entered in DB.`n"
        }
        
else
        {
            Write-Host "##[info]-->Taking Over Report.`n"

            $TakeOver = Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($OldReport.DatasetId)/Default.TakeOver -WarningAction Ignore

            if ($? -eq $true)
                        {
                            Write-Host "##[section]-->TakeOver Successful.`n"
                        }
            else
                        {
                            Write-Host "##[error]-->TakeOver NOT Successful.`n"
                        Resolve-PowerBIError -Last
throw
                        }

            Write-Host "##[info]-->Disabling Refresh Schedule.`n"

$ScheduleJsonDisable = '{
"value":
{
"enabled":false
}
}'

            Invoke-PowerBIRestMethod -Method Patch -Url groups/$($Workspace.Id.Guid)/datasets/$($OldReport.DatasetId)/refreshSchedule -Body $ScheduleJsonDisable
            
            if ($? -eq $true)
                    {
                        Write-Host "##[section]-->Refresh Schedule Disabled Successfully.`n"
                    }
            else
                    {
                        Write-Host "##[error]-->Refresh Schedule Disable NOT Successful.`n"
                    Resolve-PowerBIError -Last
throw
                    }

            Write-Host "##[info]-->$ReportName will be replace by new PBIX as`n$ReportName already exists in workspace $WorkSpaceName.`n"
            Write-Host "##[info]-->Replacing with New Report.`n"
            $NewReport = New-PowerBIReport -Path $ReportPath -Name $ReportName -WorkspaceId $Workspace.Id -ConflictAction CreateOrOverwrite
            Start-Sleep -Seconds 30
            $Report = Get-PowerBIReport -WorkspaceId $Workspace.Id -Id $NewReport.Id
            Write-Host "##[section]--> "Report Replaced Successfully.`n"

            $OldReportid = $OldReport.Id
            $OldDatasetId = $OldReport.DatasetId
            $NewReportId = $Report.Id
            $NewDatasetId = $Report.DatasetId

            if ($OldReportid -eq $NewReportId)
            {
            Write-Host "##[section]-->Old Report & New Report ID's are identical.`nNo action Required.`n"
            }
            else
            {
            #Write-Host "##[warning]-->Old Report & New Report ID's are NOT Identical.`nDB entry required for Report ID = $NewReportId`n"
			Write-Host "##[warning]-->Old Report & New Report ID's are NOT Identical.`nDB entry required for Report ID.`n"
            }
            if ($OldDatasetid -eq $NewDatasetId)
            {
            Write-Host "##[section]-->Old Dataset & New Dataset ID's are identical.`nNo action Required.`n"
            }
            else
            {
            #Write-Host "##[warning]-->Old Datset & New Dataset ID's are NOT Identical.`nDB entry required for Dataset ID = $NewDatasetId`n"
			Write-Host "##[warning]-->Old Datset & New Dataset ID's are NOT Identical.`nDB entry required for Dataset ID.`n"
            }
        }


#Get Report details
Write-Host "##[info]-->Getting Report Details.`n"
$Report = Get-PowerBIReport -WorkspaceId $Workspace.Id -Id $NewReport.Id
Write-Host "##[section]-->Report Details Acquired.`n"

#Get paramaters
<#Write-Host "##[info]-->Getting Existing Parameters.`n"
$Response = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/parameters | ConvertFrom-Json
$ShowParam = $Response.value | Select-Object name, currentValue

Write-Host "##[section]-->Existing Parameters are:`n"
Write-Output $ShowParam > .\param.txt
Get-Content .\param.txt#>


#TakeOver Report

Write-Host "##[info]-->Taking Over Report.`n"
$TakeOver = Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/Default.TakeOver -WarningAction Ignore

if ($? -eq $true)
    {
    Write-Host "##[section]-->TakeOver Successful.`n"
    }
else
    {
    Write-Host "##[error]-->TakeOver NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }


#Update DataSet Parameters

Write-Host "##[info]-->Updating DataSet Parameters.`n";
Start-Sleep -Seconds 20
$Parameters = @{
        "updateDetails"= @(
            @{
                "name"="$($ParamDbName)";
                "newValue"="$($DbName)";
             },
            @{
                "name"="$($ParamDbServerName)";
                "newValue"="$($DbServer)";
             }            
        )
}
Start-Sleep -Seconds 20
$ParametersJson = $Parameters | ConvertTo-Json -Compress
Start-Sleep -Seconds 20
$UpdateParam = Invoke-PowerBIRestMethod -Method Post -Body $ParametersJson -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/Default.UpdateParameters

if ($? -eq $true)
    {
    Write-Host "##[section]-->Paramaeter Update Successful.`n"
    }
else
    {
    Write-Host "##[error]-->Paramameter Update NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }

# Get paramaters- Not required
<#Write-Host "##[info]-->Getting New Parameters.`n"
$Response = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/parameters | ConvertFrom-Json
$ShowParam = $Response.value | Select-Object name, currentValue
Write-Host "##[section]-->New Parameters are:`n"

Write-Output $ShowParam > .\param.txt
Get-Content .\param.txt#>


#Update SQL Credentials

Write-Host "##[info]-->Updating SQL Credentials.`n"

$Dataset = Get-PowerBIDataset -WorkspaceId $Workspace.Id -Id $Report.DatasetId
$WorkspaceId = $Workspace.Id
$DatasetId = $Dataset.Id
$Datasources = Get-PowerBIDatasource -WorkspaceId $WorkspaceId -DatasetId $DatasetId

foreach($Datasource in $Datasources) {
  
  $GatewayId = $Datasource.GatewayId
  $DatasourceId = $Datasource.DatasourceId
  $DatasourePatchUrl = "gateways/$GatewayId/datasources/$DatasourceId"
  
  Write-Host "##[info]-->Patching credentials.`n"

  # HTTP request body to patch datasource credentials
  Start-Sleep -Seconds 10
  $userNameJson = "{""name"":""username"",""value"":""$sqlUserName""}"
  Start-Sleep -Seconds 10
  $passwordJson = "{""name"":""password"",""value"":""$sqlUserPassword""}"

  Start-Sleep -Seconds 20
  $patchBody = @{
    "credentialDetails" = @{
      "credentials" = "{""credentialData"":[ $userNameJson, $passwordJson ]}"
      "credentialType" = "Basic"
      "encryptedConnection" =  "Encrypted"
      "encryptionAlgorithm" = "None"
      "privacyLevel" = "None"
    }
  }

  # Convert body contents to JSON
  Start-Sleep -Seconds 20
  $patchBodyJson = ConvertTo-Json -InputObject $patchBody -Depth 6 -Compress

  # Execute PATCH operation to set datasource credentials
  Start-Sleep -Seconds 20
  $CredUpdate = Invoke-PowerBIRestMethod -Method Patch -Url $DatasourePatchUrl -Body $patchBodyJson

  if ($? -eq $true)
    {
    Write-Host "##[section]-->Credential Update Successful.`n"
    }
else
    {
    Write-Host "##[error]-->Credential Update NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }
}

#Refresh DataSet
Write-Host "##[info]-->Refreshing DataSet.`n"

Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes -WarningAction Ignore

if ($? -eq $true)
    {
    Start-Sleep -Seconds 30
    Write-Host "##[section]-->Refresh Successful.`n"
    }
else
    {
    Write-Host "##[error]-->Refresh NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }

#Get last 2 refresh values
Write-Host "##[info]-->Getting last 2 refresh Status.`n"
$refresh = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes | ConvertFrom-Json
$ShowRefresh =  $refresh.value[0,1]
Write-Host "##[section]-->Last 2 Refresh Status:`n"

Write-Output $ShowRefresh > ./refresh.txt
Get-Content ./refresh.txt


#Setup Schedule Refresh

Write-Host "##[info]-->Enabling & Creating a Refresh Schedule.`n"

$ScheduleJsonEnable = '{
        "value":
            {
                "enabled":true,
                "notifyOption":"NoNotification",
                "days":["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"],
                "times":["00:00","01:00","02:00","03:00","04:00","05:00","06:00","07:00","08:00","09:00","10:00","11:00","12:00","13:00","14:00","15:00","16:00","17:00","18:00","19:00","20:00","21:00","22:00","23:00"],
                "localTimeZoneId":"UTC"
        }
}'

Invoke-PowerBIRestMethod -Method Patch -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshSchedule -Body $ScheduleJsonEnable
if ($? -eq $true)
    {
    Start-Sleep -Seconds 10
    Write-Host "##[section]-->Refresh Schedule Created Successfully.`n"
    }
else
    {
    Write-Host "##[error]-->Refresh Schedule creation NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }


#Refresh DataSet
Write-Host "##[info]-->Refreshing DataSet.`n"

Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes -WarningAction Ignore

if ($? -eq $true)
    {
    Start-Sleep -Seconds 30
    Write-Host "##[section]-->Refresh Successful.`n"
    }
else
    {
    Write-Host "##[error]-->Refresh NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }

#Get last 2 refresh values
Write-Host "##[info]-->Getting last 2 refresh Status.`n"
$refresh = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes | ConvertFrom-Json
$ShowRefresh =  $refresh.value[0,1]
Write-Host "##[section]-->Last 2 Refresh Status:`n"

Write-Output $ShowRefresh > ./refresh.txt
Get-Content ./refresh.txt


#delete dataset - Not Required
<# Invoke-PowerBIRestMethod -Method Delete -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId) #>

#Disconnet PBI
Write-Host "##[info]-->Disconnecting PBI Service Account.`n"

Disconnect-PowerBIServiceAccount
if ($? -eq $true)
    {
    Write-Host "##[section]-->Disconnect Successful.`n"
    }
else
    {
    Write-Host "##[error]-->Disconnect NOT Successful.`n"
Resolve-PowerBIError -Last
throw
    }
}
catch
{
    Write-Host "##[error]-->PBI Deployment Script Failed.`n"
    Resolve-PowerBIError -Last
throw;
}
