Automating Power BI Deployments.

This article contains API references to achieve the various tasks for PBI report deployment. Also, you will see some tips to troubleshoot issues you might face as I did. 

I'm aware that there exists an ADO task PBI Actions to achieve most of the tasks. However, it was a requirement to do everything using PBI native APIs.

Context
Pre-Requisites.
Importing modules.
Login to PBI.
Publish PBIX.
Take Over Report.
Update Dataset Parameters.
Update Datasource (SQL Credentials).
Refresh Dataset.
Create Refresh Schedule.
Bonus Script.

Pre-Requisites.
Service Principal to connect to PBI.
You'll need an SPN that will be used to connect & perform various tasks in PBI. Make sure you give relevant access to SPN. Probably Contributor/Owner access.

Install PowerShell modules.
Click here to view & install the PowerShell modules required for executing various commands.

If you plan on using ADO MS Hosted Agent to execute Script make sure to include module installation steps first thing in the script.

  3. PBIX file.

Importing modules.
If you have a shared location where all the modules get installed, it's a good practice to import the used modules in your script so that system does not eat time looking for modules in a shared location when it can't find one locally. 

Get-Module -ListAvailable MicrosoftPowerBI* | Import-Module
Login to PBI.
Next, you need to login to PBI using SPN you created initially.

Tip: If you are using the local system for executing the script. You'll first need to install the AZ Powershell module and login to Azure using the below command.

Connect-AzAccount
Use below code to login PBI.

$Password = ConvertTo-SecureString $Secret -AsPlainText -Force
$creds = New-Object PSCredential $AppID, $Password


Connect-PowerBIServiceAccount -ServicePrincipal -credential $Creds -Tenant $TenantID
If, during the execution of the script or any PBI command you get below error. Worry not, I've a workaround.

Get-PowerBIWorkspace : Could not load file or assembly 'Newtonsoft.Json, Version=11.0.0.0, Culture=neutral, PublicKeyToken=30ad4fe6b2a6aeed' or one of its dependencies. The system cannot find the file specified.
Once you encounter the error, execute the below command and note down the location (FullyQualifiedName) of Newtonsoft.Json.

[Newtonsoft.Json.JsonConvert].Module
You'll also see the Version under Assembly key. What happen here is that your connection to PBI requires the Newtonsoft.Json file version 11.0.0.0. However, your system has another version. We need to get v11 on the system. To do that, you need to download the Newtonsoft.Json package from Here.

Once, you have the package extract it using 7zip or a similar tool and copy the DLL file from the lib/net45 directory. Then go to the Newtonsoft.Json location you got in the previous step. Rename the existing dll and paste new one and restart your powershell. Now, the error should be resolved. 

Publish PBIX.
Once you are logged in try to Get the existing Workspace.

Get-PowerBIWorkspace -Name $WorkSpaceName
If, you can get successful output then try to Publish a new report.

New-PowerBIReport -Path $ReportPath -Name $ReportName -WorkspaceId $Workspace.Id -ConflictAction CreateOrOverwrite
Tip: Using, -ConflictAction CreateOrOverwrite will either create a new report if it does not already exist with the same name or will overwrite the existing one. Thus you will not have multiple reports or datasets with the same name.

Take Over Report.
There is a possibility that some user might have logged in to PBI UI portal and takes over the dataset to make any change or review something. So we could get an error while making any update in the dataset/datasource. So we need to Take Over via API to make sure the next steps run smoothly.

Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/Default.TakeOver -WarningAction Ignore
Update Dataset Parameters.
Your report might have N number of parameters that might differ for each Environment For Example - DB Server and DB Name will be different for Dev, QA & PROD.

$Parameters = @{
        "updateDetails"= @(
            @{
                "name"="$($ParamName1)";
                "newValue"="$($DbName)";
             },
            @{
                "name"="$($ParamName2)";
                "newValue"="$($DbServer)";
             }            
        )
}

$ParametersJson = $Parameters | ConvertTo-Json -Compress
$UpdateParam = Invoke-PowerBIRestMethod -Method Post -Body $ParametersJson -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/Default.UpdateParameters
You can check parameters before and after making changes to make sure it's working.

$Response = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/parameters | ConvertFrom-Json
$Response.value | Select-Object name, currentValue
Update Datasource (SQL Credentials).
To get the data from DB you need to provide credentials, which will be different for each environment. Using below code you can set these credentials.

$Dataset = Get-PowerBIDataset -WorkspaceId $Workspace.Id -Id $Report.DatasetId
$WorkspaceId = $Workspace.Id
$DatasetId = $Dataset.Id
$Datasources = Get-PowerBIDatasource -WorkspaceId $WorkspaceId -DatasetId $DatasetId


foreach($Datasource in $Datasources) {
  
  $GatewayId = $Datasource.GatewayId
  $DatasourceId = $Datasource.DatasourceId
  $DatasourePatchUrl = "gateways/$GatewayId/datasources/$DatasourceId"
  
  Write-Host -BackgroundColor Yellow -ForegroundColor Black "Patching credentials.`n"


# HTTP request body to patch datasource credentials
  Start-Sleep -Seconds 10
  $userNameJson = "{""name"":""username"",""value"":""$sqlUserName""}"
  Start-Sleep -Seconds 10
  $passwordJson = "{""name"":""password"",""value"":""$sqlUserPassword""}"


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
  $patchBodyJson = ConvertTo-Json -InputObject $patchBody -Depth 6 -Compress


# Execute PATCH operation to set datasource credentials
  $CredUpdate = Invoke-PowerBIRestMethod -Method Patch -Url $DatasourePatchUrl -Body $patchBodyJson
}
Tip: Make sure that DB Password does not contain \ or $

Refresh Dataset.
Make an On-Demand Dataset refresh. To check if Credentials in the previous step were updated successfully and Dataset can connect to DB.

Invoke-PowerBIRestMethod -Method Post -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes -WarningAction Ignore
Give it a couple of minutes before you fetch the refresh results.

#Get last 2 refresh values

$refresh = Invoke-PowerBIRestMethod -Method Get -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshes | ConvertFrom-Json

$refresh.value[0,1]
Create Refresh Schedule.
You might need a refresh schedule to make sure the Report shows the latest data from DB and Dataset fetches it on periodic intervals.

$ScheduleJsonEnable = '{
        "value":
            {
                "enabled":true,
                "notifyOption":"NoNotification",
                "days":["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"],
                "times":["00:00","01:00","02:00","03:00","04:00","05:00","06:00","07:00","08:00","09:00","10:00","11:00","12:00"],
                "localTimeZoneId":"UTC"
        }
	}'


Invoke-PowerBIRestMethod -Method Patch -Url groups/$($Workspace.Id.Guid)/datasets/$($Report.DatasetId)/refreshSchedule -Body $ScheduleJsonEnable
Tip: Currently API doesnot respond well to two parameters:

TimeZone other than UTC
notifyOption other than NoNotification
Bonus Tip:  While execution I found that as soon as I publish Report Refresh Schedule Kicks in and the Script tends to fail as Dataset starts the referesh and becomes unresponsive. To deal with this, at beginning of script I disabled the Refresh Schedule.

Bonus Script.
I will provide two scripts one is ADO specific and one for execution from PowerShell. Both scripts can be executed on any platform. However, ADO does not supports the usual Background and Foreground colors as PowerShell. Hence some modifications to the script were done to show colors in ADO logs.

Reference:
I lost track of Forums and Articles I went through to find the right commands for different tasks. Therefore, I thank the Authors and members for their contributions. If you see a part of your contribution in my script, let me know so that I can give credit to you with your Article URL.

If you want to check more features from Power BI API. Check below MS Documentation.

https://docs.microsoft.com/en-us/rest/api/power-bi/
https://docs.microsoft.com/en-us/powershell/power-bi/overview?view=powerbi-ps
Download Scripts.
You can download the scripts from GitHub using below URL.

https://github.com/hkhanna09/PowerBI
