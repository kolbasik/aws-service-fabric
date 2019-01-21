## Introduction

* [Service Fabric Tutorial](https://azure.microsoft.com/en-us/documentation/learning-paths/service-fabric/)
* [Service Fabric SDK](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-get-started)
* [Describing a service fabric cluster](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-resource-manager-cluster-description)
* [Party Cluster](https://blogs.msdn.microsoft.com/azureservicefabric/2016/01/03/party-until-you-drop-with-azure-service-fabric/)

If you forget a command in powershell :

```powershell
Get-Command -Module *Fabric*
```

You need to define the user name and password in order to connect to the service fabric cluster using windows credentials with:

* [Credentials Manager](http://windowsitpro.com/windows-81/managing-account-credentials-windows-and-web-credential-manager)
* Command line: `cmdkey.exe /add:service-fabric.address /user:"domain\user_name" /pass:"password"`

[Connect to the service fabric cluster](https://docs.microsoft.com/en-us/powershell/module/servicefabric/connect-servicefabriccluster?view=azureservicefabricps) using windows credentials:

```powershell
Connect-ServiceFabricCluster -ConnectionEndpoint 192.168.99.99:19000 -WindowsCredential
Test-ServiceFabricClusterConnection
```

```powershell
$ServiceFabricClusterConnection = @{ ConnectionEndpoint = 'service-fabric-949735929.eu-west-1.elb.amazonaws.com:8000' ; WindowsCredential = $True }
Connect-ServiceFabricCluster $ServiceFabricClusterConnection

Get-ServiceFabricApplication
Get-ServiceFabricApplication | Test-ServiceFabricApplication
Get-ServiceFabricApplication | Get-ServiceFabricService
Get-ServiceFabricApplication | Get-ServiceFabricService | Test-ServiceFabricService
Get-ServiceFabricApplication | Get-ServiceFabricService | Get-ServiceFabricPartition
```

## Development

Service Fabric runs applications by default under the LOCAL_AUTHORITY user so

### Setup Global Environment Variables

```env
ASPNETCORE_ENVIRONMENT=Development
```

### Setup User Secrets

* [Application Secrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets)
* [Keep your ASP.NET Core application’s secrets safe during development](https://jonhilton.net/2017/06/07/keep-your-asp-dot-net-application-secrets-safe/)

The secrets are stored at the following path  
`%APPDATA%\microsoft\UserSecrets\<userSecretsId>\secrets.json`

Examples:

* user : `C:\Users\{user}\AppData\Roaming\Microsoft\UserSecrets\{SECRET_ID}\secrets.json`
* service fabric (LOCAL_AUTHORITY) : `C:\WINDOWS\system32\config\systemprofile\AppData\Roaming\Microsoft\UserSecrets\{SECRET_ID}\secrets.json`

In order to use your own secrets in service fabric for development purpose you can make a symlink to your secrets:

```cmd
mklink /D /H %WINDIR%\system32\config\systemprofile\AppData\Roaming\Microsoft\UserSecrets %APPDATA%\Microsoft\UserSecrets
```

### Reverse Proxy

* [Service Fabrix Reverse Proxy](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reverseproxy)
* [Windows NAT limitation for Docker](https://github.com/Azure/service-fabric-issues/issues/227)

### ASP.NET Core

* https://docs.microsoft.com/en-gb/azure/service-fabric/service-fabric-reliable-services-communication-aspnetcore
* https://msdn.microsoft.com/en-us/magazine/mt595752.aspx?f=255&MSPPError=-2147217396

### Docker

* http://blogs.recneps.net/post/Custom-ASPNET-MVC-app-running-in-a-Container-on-Service-Fabric
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-deploy-container
* https://github.com/Microsoft/azure-docs/blob/master/articles/service-fabric/service-fabric-deploy-container.md
* https://blogs.msdn.microsoft.com/azureservicefabric/2016/04/25/orchestrating-containers-with-service-fabric/

Service Fabric are not able to redirect requests to the docker container which locates on the same box due to [Windows NAT limitation](https://github.com/Azure/service-fabric-issues/issues/227). There is a workaround for it just to use [placement-constraints](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-resource-manager-cluster-description#placement-constraints-and-node-properties) to deploy docker images on another boxes than reverse proxy.

## Deployment

* [Deployment Strategies](https://blogs.technet.microsoft.com/livedevopsinjapan/2016/11/21/7-deployment-strategies-for-service-fabric/)
* [Blue/Green Deployment](https://blogs.msdn.microsoft.com/cloud_solution_architect/2016/10/17/bluegreen-deployments-in-service-fabric/)
* [Application Upgrade](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-application-upgrade)
* [Octopus: Deploying a package to a Azure Service Fabric cluster](https://octopus.com/docs/deploying-applications/deploying-to-service-fabric/deploying-a-package-to-a-service-fabric-cluster)

### Simple Deployment Pipeline

```powershell
Connect-ServiceFabricCluster -ConnectionEndpoint 'idealcluster.westus.cloudapp.azure.com:19000'
Copy-ServiceFabricApplicationPackage -ApplicationPackagePath . -ImageStoreConnectionString "fabric:ImageStore" -ApplicationPackagePathInImageStore "ActorTicTacToeApplicationType"
Register-ServiceFabricApplicationType -ApplicationPathInImageStore "ActorTicTacToeApplicationType"
New-ServiceFabricApplication "fabric:/ActorTicTacToeApplication" "ActorTicTacToeApplicationType" "1.0.2"
#Remove-ServiceFabricApplication "fabric:/ActorTicTacToeApplication"
```

### Continuous Integration

`build.bat` file

```bash
call "C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\Common7\Tools\VsMSBuildCmd.bat"

for /f %%f in ('dir /s/b *.sln') do MSBuild.exe -t:Clean,Restore,Build /p:Configuration=Release /p:Platform="x64" %%f
for /f %%f in ('dir /s/b *.sfproj') do MSBuild.exe -t:Package /p:Configuration=Release /p:Platform="x64" %%f

pushd %CD%
for /f %%f in ('dir /s/b pkg') do (
    cd %%f\..
    xcopy /I /F /V /Y PublishProfiles\*.xml pkg\Release\PublishProfiles
    xcopy /I /F /V /Y ApplicationParameters\*.xml pkg\Release\ApplicationParameters
)
popd
```

#### Continuous Delivery

`deploy.ps1` file

```powershell
<#
.SYNOPSIS
Deploys a Service Fabric application type to a cluster.

.EXAMPLE

Connect-ServiceFabricCluster -ConnectionEndpoint service-fabric-949735929.eu-west-1.elb.amazonaws.com:8000 -WindowsCredential
.\build.ps1 -UseExistingClusterConnection
#>
param (
    [Switch]
    $UseExistingClusterConnection = $true,

    [String][ValidateSet('Release', 'Debug')]
    $Configuration = 'Release',

    [String]
    $PublishProfileFile = "Cloud.xml",

    [Hashtable]
    $ApplicationParameters = @{},

    [String][ValidateSet('None', 'ForceUpgrade', 'VetoUpgrade')]
    $OverrideUpgradeBehavior = 'None',

    [String][ValidateSet('Never', 'Always', 'SameAppTypeAndVersion')]
    $OverwriteBehavior = 'Never',

    [Switch]
    $DeployOnly = $false,

    [Switch]
    $SkipPackageValidation = $false,

    [Switch]
    $UnregisterUnusedApplicationVersionsAfterUpgrade = $false
)

Get-ChildItem -Recurse -Directory -Filter pkg | ForEach-Object {
    $DIR = $_.Parent.FullName
    Write-Host "Deploying $DIR"
    & "$DIR\Scripts\Deploy-FabricApplication.ps1" `
        -UseExistingClusterConnection:$UseExistingClusterConnection `
        -ApplicationPackagePath "$DIR\pkg\$Configuration" `
        -PublishProfileFile "$DIR\pkg\$Configuration\PublishProfiles\$PublishProfileFile" `
        -ApplicationParameter:$ApplicationParameters `
        -OverrideUpgradeBehavior $OverrideUpgradeBehavior `
        -OverwriteBehavior $OverwriteBehavior `
        -DeployOnly:$DeployOnly `
        -SkipPackageValidation:$SkipPackageValidation `
        -UnregisterUnusedApplicationVersionsAfterUpgrade $UnregisterUnusedApplicationVersionsAfterUpgrade `
        -ErrorAction Stop
}
```

```powershell
Connect-ServiceFabricCluster -ConnectionEndpoint 'service-fabric-949735929.eu-west-1.elb.amazonaws.com:8000' -WindowsCredential
$ApplicationParameters = @{ AppSettings_ProjectKey="zxcasd" ; AppSettings_ClientId="gF47ep" ; AppSettings_ClientSecret="qwerty123" }
.\deploy.ps1 -UseExistingClusterConnection -PublishProfileFile "Cloud.xml" -ApplicationParameters $ApplicationParameters
```

## Infrastructure

* [Describing a service fabric cluster](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-resource-manager-cluster-description)
* [How to create a Service Fabric standalone cluster with AWS EC2 instances](https://blogs.msdn.microsoft.com/azureservicefabric/2017/05/18/tutorial-how-to-create-a-service-fabric-standalone-cluster-with-aws-ec2-instances/)
* [Customize Service Fabric cluster settings and Fabric Upgrade policy](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-fabric-settings)
* [Secure a standalone cluster on Windows by using Windows security](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-windows-cluster-windows-security)
* [Connect to the service fabric cluster](https://docs.microsoft.com/en-us/powershell/module/servicefabric/connect-servicefabriccluster?view=azureservicefabricps)
* https://docs.aws.amazon.com/directoryservice/latest/admin-guide/microsoftadbasestep3.html

### Provisioning Script

```powershell
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes

New-NetFirewallRule -DisplayName "Allow inbound ICMPv4" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -RemoteAddress Any -Action Allow
New-NetFirewallRule -DisplayName "Allow inbound Service Fabric Int" -Direction Inbound -Protocol TCP -LocalPort 135,137-139,445 -RemoteAddress Any -Action Allow
New-NetFirewallRule -DisplayName "Allow inbound Serfice Fabric Ext" -Direction Inbound -Protocol TCP -LocalPort 19000-19003,19080-19081,20001-21000 -RemoteAddress Any -Action Allow

Set-Service -Name RemoteRegistry -StartUpType Automatic
Start-Service -Name RemoteRegistry
Get-Service -Name RemoteRegistry

Invoke-WebRequest "http://go.microsoft.com/fwlink/?LinkId=730690" -OutFile c:\sf.zip
#Start-BitsTransfer -Source "http://go.microsoft.com/fwlink/?LinkId=730690" -Destination c:\SF.zip
Expand-Archive c:\sf.zip c:\sf
```

### Create Cluster

1. Make a copy of the `ClusterConfig.Windows.MultiMachine.json` example configuration file.
    ```powershell
    xcopy .\ClusterConfig.Windows.MultiMachine.json .\ClusterConfig.AWS.json*
    ```
2. Replace the `$.nodes.*.iPAddress` fields with the machine names
3. Add the following object to the `$.properties.fabricSettings` array:
    ```json
    {
        "name": "FileStoreService",
        "parameters": [
            {
                "name": "AnonymousAccessEnabled",
                "value": "true"
            }
        ]
    }
    ```
4. Add the users who have permissions to work with the cluster
5. The last step to create a cluster is to execute the powershell script
    ```powershell
    .\CreateServiceFabricCluster.ps1 -ClusterConfigFilePath .\ClusterConfig.AWS.json -AcceptEULA
    ```

## Maintenance

* https://docs.microsoft.com/en-gb/azure/service-fabric/service-fabric-deploy-remove-applications
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-controlled-chaos

### Add a new node to the cluster


```powershell
.\AddNode.ps1 -NodeName 'vm3' -NodeType 'NodeType0' -NodeIpAddressOrFQDN "$Env:COMPUTERNAME" -UpgradeDomain 'UD3' -FaultDomain 'fd:/dc1/r3' -ExistingClientConnectionEndpoint '192.168.5.46:19000' -AcceptEULA
# or using Add-ServiceFabricNode directly
```

### Remove the existing node from the cluster

```powershell
.\RemoveNode.ps1 -ExistingClientConnectionEndpoint 192.168.5.251:19000
# or using Remove-ServiceFabricNode directly
```

### Upgrade the cluster configuration

``` powershell
Get-ServiceFabricClusterConfiguration
Start-ServiceFabricClusterConfigurationUpgrade -ClusterConfigPath .\ClusterConfig.AWS.json
```

### Restart the node

```powershell
Restart-ServiceFabricNode -NodeName vm3 -CommandCompletionMode Verify
```

### Scale the service

``` powershell
Update-ServiceFabricService -Stateful 'fabric:/System/ImageStoreService' -MinReplicaSetSize 3 -TargetReplicaSetSize 3 -Force
Repair-ServiceFabricPartition -All -Force
```

### Sample of rollback an application

* https://stackoverflow.com/questions/43265395/azure-service-fabric-rollback : 

```powershell
$app = Get-ServiceFabricApplication -ApplicationName "fabric:/xxx"
$table = @{}
$app.ApplicationParameters | ForEach-Object { $table.Add($_.Name, $_.Value) }
Start-ServiceFabricApplicationUpgrade -ApplicationName "fabric:/xxx" -ApplicationTypeVersion 1.0.0 -ApplicationParameter $table -HealthCheckStableDurationSec 60 -UpgradeDomainTimeoutSec 1200 -UpgradeTimeout 3000 -FailureAction Rollback -Monitored 
```

### Sample of how to add a new user to a cluster:

```powershell
function Add-ServiceFabricUser([bool]$isAdmin=$False, [Parameter(Mandatory=$True)]$identity) {
    $configPath = ".\ClusterConfig.$((Get-Date).Ticks).json"
    $config = Get-ServiceFabricClusterConfiguration | ConvertFrom-Json
    $config.Properties.Security.WindowsIdentities.ClientIdentities = $config.Properties.Security.WindowsIdentities.ClientIdentities + [PsCustomObject]@{ IsAdmin = $isAdmin; Identity = $identity }
    $config.ClusterConfigurationVersion = [version]$config.ClusterConfigurationVersion | foreach { (New-Object -TypeName System.Version -ArgumentList $_.Major, $_.Minor, $_.Build, ($_.Revision + 2)).ToString() }
    $config | ConvertTo-Json -Depth 50 | Set-Content -Path $configPath
    Start-ServiceFabricClusterConfigurationUpgrade -ClusterConfigPath $configPath
    return Get-ServiceFabricClusterConfigurationUpgradeStatus
}
```

### Find a mistake to fix configuration on a live

```powershell
cd C:\ProgramData\SF
Get-ChildItem -Path . -Include *.json,*.xml -Recurse | Select-String -Pattern '[CONTENT]' -AllMatches | group path | select name
```