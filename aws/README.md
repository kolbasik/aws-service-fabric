# AWS

* `C:\ProgramData\Amazon\EC2-Windows\Launch\Log`
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-deploy-anywhere
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-tutorial-standalone-create-infrastructure
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-diagnostics-eventstore
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-cluster-fabric-settings
* https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-docker-compose

```powershell
net user Administrator "6gfVW8Jqv48zE848fqtsjJ9k"
Write-Host $env:COMPUTERNAME
```

```powershell
.\TestConfiguration.ps1 -ClusterConfigFilePath .\sf.json
.\CreateServiceFabricCluster.ps1 -ClusterConfigFilePath .\sf.json -AcceptEULA -Verbose
.\RemoveServiceFabricCluster.ps1 -ClusterConfigFilePath .\sf.json -Verbose
```

```cmd
add-computer –domainname cloud.ad -restart –force
cmdkey.exe /add:EC2AMAZ-CPD62UR /user:"cloud.ad\SF" /pass:"SECURE"
.\AddNode.ps1 -NodeName 'vm3' -NodeType 'NodeType0' -NodeIpAddressOrFQDN "$Env:COMPUTERNAME" -UpgradeDomain 'UD3' -FaultDomain 'fd:/aws/r3' -ExistingClientConnectionEndpoint 'EC2AMAZ-CPD62UR:19000' -WindowsCredential -AcceptEULA
Import-Module .\DeploymentComponents\ServiceFabric.psd1
Connect-ServiceFabricCluster EC2AMAZ-CPD62UR:19000 -WindowsCredential
```

## Linux

* https://www.microsoftpressstore.com/articles/article.aspx?p=2923217
* https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/service-fabric/service-fabric-get-started-linux.md
* https://medium.com/@vivekteega/how-to-setup-an-xrdp-server-on-ubuntu-18-04-89f7e205bd4e

```bash
echo "127.0.0.1 $(hostname)" | sudo tee -a /etc/hosts
sudo /opt/microsoft/sdk/servicefabric/common/clustersetup/devclustersetup.sh
sfctl cluster select --endpoint "http://$(hostname):19080"
```

```bash
adduser username
usermod -aG sudo username
```