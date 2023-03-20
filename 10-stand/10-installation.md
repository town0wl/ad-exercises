### Установка домена

На контроллере домена (будущем):

```
$AdminKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}"
$UserKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}"
Set-ItemProperty -Path $AdminKey -Name "IsInstalled" -Value 0
Set-ItemProperty -Path $UserKey -Name "IsInstalled" -Value 0
Stop-Process -Name Explorer
Enable-PSRemoting -Force
set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server'-name "fDenyTS Connections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 1
Rename-Computer -ComputerName (hostname) -newname "TEMP-DC"
Set-TimeZone -Name "Eastern Standard Time"
Import-Module ServerManager
Install-windowsfeature -name AD-Domain-Services –IncludeManagementTools
Install-WindowsFeature –Name GPMC
```
```
shutdown /f /r /t 1
```
```
$domainname = "org.local"
$NTDPath = "C:\Windows\ntds"
$logPath = "C:\Windows\ntds"
$sysvolPath = "C:\Windows\Sysvol"
$domainmode = "win2012R2"
$forestmode = "win2012R2"

Install-ADDSForest -CreateDnsDelegation:$false -DatabasePath $NTDPath -DomainMode $domainmode -DomainName $domainname -ForestMode $forestmode -InstallDns:$true -LogPath $logPath -NoRebootOnCompletion:$false -SysvolPath $sysvolPath -Force:$true

Confirm SafeModeAdministratorPassword: ********************
```

### Добавление member-сервера в домен

Сначала нужно установить DNS-сервер на будущем member в адрес контроллера домена. Пример:
```
Get-DnsClientServerAddress
Set-DNSClientServerAddress "eth0" –ServerAddresses ("10.128.0.8", "10.128.0.2")
nslookup org.local
```
```
Add-Computer –domainname "org.local" -Restart
```

### Создание админа и пользователя в домене

```
New-ADUser -Name "RDP User" -SamAccountName "user" -AccountPassword(Read-Host -AsSecureString "Input Password") -Enabled $true
Add-ADGroupMember -Identity "S-1-5-32-555" -Members user
New-ADUser -Name "Admin" -SamAccountName "adm" -AccountPassword(Read-Host -AsSecureString "Input Password") -Enabled $true
Add-ADGroupMember -Identity $((Get-ADDomain).DomainSID.Value+'-512') -Members adm
```
Для добавления пользователя в группу RDP-пользователей member-сервера можно воспользоваться групповой политикой, привязанной к специально созданному для этого OU.  
Просмотр эффективной политики на хосте:
```
secedit /export /cfg C:\Windows\security\security_policy.inf
Get-Content -Raw "C:\Windows\security\security_policy.inf"
```

### Наполнение домена

На контроллере домена:
```
mkdir c:\badblood
cd c:\badblood
$url = "https://github.com/davidprowe/BadBlood/archive/master.zip"
$output = "C:\badblood\master.zip"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri $url -OutFile $output
Expand-Archive -Path .\master.zip -DestinationPath C:\badblood
```
В скрипте указать необходимые количества сущностей:
```
    User line to edit : 61
    Group line to edit : 75
    Computer line to edit : 90
```
Запутстить.

### Включение AD CS

Через Server Manager активировать роли AD CS с web-сервисом. Автоматически включается LDAPS.
certlm.msc


