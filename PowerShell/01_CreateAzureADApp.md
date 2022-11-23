# Create Azure AD Application with Microsoft.Graph PowerShell Module

## Install Microsoft.Graph PowerShell Module
Start PowerShell as Administrator

```
Install-Module Microsoft.Graph.Authentication
Install-Module Microsoft.Graph.Applications
```
Close PowerShell

## Import Module
Start PowerShell as User

```
Import-Module Microsoft.Graph.Authentication
Import-Module Microsoft.Graph.Applications
```
## Connect-MgGraph
Connect to your Azure Active Directory with "Application Adminstrator" or "Global Administrator" Role

```
Connect-MgGraph -Scopes "Application.Read.All","Application.ReadWrite.All","User.Read.All"
Get-MgContext
```

## Create Application
Check your App Registration in AzureAD
https://aad.portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps

```
$AppName =  "DemoApp"
$App = New-MgApplication -DisplayName $AppName 
$APPObjectID = $App.Id
```

## Add additional Owner
The User who created the Application is automatically the Owner

```
$User = Get-MgUser -UserId "m.muster@domain.tld"
$ObjectId = $User.ID
$NewOwner = @{
	"@odata.id"= "https://graph.microsoft.com/v1.0/directoryObjects/{$ObjectId}"
	}
New-MgApplicationOwnerByRef -ApplicationId $APPObjectID -BodyParameter $NewOwner
```

## Add a ClientSecret
Now we add a Client Secret. Keep In mind, that Certificates are BestPractice.

```
#ClientSecret
$passwordCred = @{
    "displayName" = "DemoClientSecret"
    "endDateTime" = (Get-Date).AddMonths(+12)
}
$ClientSecret2 = Add-MgApplicationPassword -ApplicationId $APPObjectID -PasswordCredential $passwordCred
$ClientSecret2
$ClientSecret2.SecretText

#Show ClientSecrets
$App = Get-MgApplication -ApplicationId $APPObjectID
$App.PasswordCredentials
```

## Create SelfSignedCertificate
Now we create a self signed Certificate in the Current User Cert Store

```
$Subject = "DemoCert"
$NotAfter = (Get-Date).AddMonths(+24)
$Cert = New-SelfSignedCertificate -Subject $Subject -CertStoreLocation "Cert:\CurrentUser\My" -KeySpec Signature -NotAfter $Notafter -KeyExportPolicy Exportable
$ThumbPrint = $Cert.ThumbPrint
```

Start > Run > certmgr.msc
Check the Certificate

Check Certificates with PowerShell

```
Get-ChildItem -Path cert:\CurrentUser\my | Format-Table
```

## Export Certificate as Base64 (PEM Format)
```
$CurrentLocation = (Get-Location).path
$Base64 = [convert]::tobase64string((get-item cert:\currentuser\my\$ThumbPrint).RawData)
$Base64Block = $Base64 |
ForEach-Object {
    $line = $_

    for ($i = 0; $i -lt $Base64.Length; $i += 64)
    {
        $length = [Math]::Min(64, $line.Length - $i)
        $line.SubString($i, $length)
    }
}
$base64Block2 = $Base64Block | Out-String

$Value = "-----BEGIN CERTIFICATE-----`r`n"
$Value += "$Base64Block2"
$Value += "-----END CERTIFICATE-----"
$Value
Set-Content -Path "$CurrentLocation\$Subject-BASE64.cer" -Value $Value
```

## Add Certificate to AzureAD App
Now we add the Certificate to the Azure AD App

```
$CurrentLocation = (Get-Location).path
$CertPath = $CurrentLocation + "\" + $Subject + "-BASE64.cer"
$Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate($CertPath)

#Get Certificate with Thumbprint from UserCertStore
$ThumbPrint = "07EFF3918F47995EB53B91848F69B5C0E78622FD"
$Cert = Get-ChildItem -Path cert:\CurrentUser\my\$ThumbPrint 

#Get Certificate with SubjectName from UserCertStore
$Subject = "CN=DemoCert"
$Cert = Get-ChildItem -Path cert:\CurrentUser\my\ | Where-Object {$_.Subject -eq "$Subject"}  


# Create a keyCredential (Certificate) for App
$keyCreds = @{ 
    Type = "AsymmetricX509Cert";
    Usage = "Verify";
    key = $cert.RawData
}
 
try {
   Update-MgApplication -ApplicationId $APPObjectID  -KeyCredentials $keyCreds
} catch {
   Write-Error $Error[0]
}

#Show Certificate
$App = Get-MgApplication -ApplicationId $APPObjectID
$App.KeyCredentials
```

## Add Permissions

All permissions and IDs
https://learn.microsoft.com/en-us/graph/permissions-reference#all-permissions-and-ids

Add Delegated Permission

```
#Delegated Permission "email"
$params = @{
	RequiredResourceAccess = @(
		@{
			ResourceAppId = "00000003-0000-0000-c000-000000000000"
			ResourceAccess = @(
				@{
					Id = "64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0"
					Type = "Scope"
				}
			)
		}
	)
}
Update-MgApplication -ApplicationId $APPObjectID -BodyParameter $params
```

Add Application Permission

```
#Application Permission OrgContact.Read.All
$params = @{
	RequiredResourceAccess = @(
		@{
			ResourceAppId = "00000003-0000-0000-c000-000000000000"
			ResourceAccess = @(
				@{
					Id = "e1a88a34-94c4-4418-be12-c87b00e26bea"
					Type = "Role"
				}
			)
		}
	)
}
Update-MgApplication -ApplicationId $APPObjectID -BodyParameter $params
```

Show the App Permissions

```
$App = Get-MgApplication -ApplicationId $APPObjectID -Property * 
$App.RequiredResourceAccess | Format-List
$app.RequiredResourceAccess.resourceaccess
```

## Grant Admin Consent

## Redirect URI
If you need to add Redirect URI's.

```
#Redirect URI
$App = Get-MgApplication -ApplicationId $APPObjectID -Property * 
$AppId = $App.AppId
$RedirectURI = @()
$RedirectURI += "msal" + $AppId + "://auth"
$RedirectURI += "https://localhost:3000"

$params = @{
	RedirectUris = @($RedirectURI)
}

Update-MgApplication -ApplicationId $APPObjectID -IsFallbackPublicClient -PublicClient $params
```

## Remove the Application
Remove the Application

```
Remove-MgApplication -ApplicationId $APPObjectID
```