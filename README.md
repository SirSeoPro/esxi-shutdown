# PS скрипт для выключения гипервизора
Для удобства скрипт запускается заббиксом. 
#### Настройки заббикс агента
Заббикс аген должен быть установлен.
Дописываем в:  <br>
C:\Program Files\Zabbix Agent\zabbix_agentd.conf
```
EnableRemoteCommands=1
```

#### Настройки заббикс сервера
Должен быть добавлен хост, привязан шаблон, настроены триггеры. <br> 
В alerts -> scripts создана скрипт, на забикс агенте: <br>
Type: Script <br>
Execute on: Zabbix agent <br>
Commands:
```
powershell.exe -ExecutionPolicy Bypass -File "C:\Scripts\shutdown_esxi.ps1"
```

В alerts -> actions -> trigger action: <br>
Conditions: привязываем наш триггер <br>
Operations: наш скрипт. <br>


#### Скрипт:
```
$ESXiIP       = "192.168.1.1"
$ESXiUser     = "root"
$ESXiPassword = "ПАРОЛЬ"

Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false -Scope Session | Out-Null
Set-PowerCLIConfiguration -ParticipateInCEIP $false -Confirm:$false -Scope Session | Out-Null

Connect-VIServer -Server $ESXiIP -User $ESXiUser -Password $ESXiPassword -Force -ErrorAction Stop | Out-Null

Write-Host "=== Starting shutdown sequence ==="

$poweredOnVMs = Get-VM | Where-Object PowerState -eq "PoweredOn"

foreach ($vm in $poweredOnVMs) {

    Write-Host ""
    Write-Host "===== Processing VM: $($vm.Name) ====="

    # 1. Graceful shutdown via VMware Tools
    $toolsRunning = ($vm.ExtensionData.Guest.ToolsRunningStatus -eq "guestToolsRunning")
    if ($toolsRunning) {
        Write-Host "Graceful shutdown via VMware Tools..."
        Shutdown-VMGuest -VM $vm -Confirm:$false | Out-Null

        # Wait up to 2 minutes
        for ($i = 0; $i -lt 24; $i++) {
            if ((Get-VM -Name $vm.Name).PowerState -eq "PoweredOff") {
                Write-Host "VM is powered off gracefully"
                break
            }
            Start-Sleep -Seconds 5
        }
    }

    # 2. ACPI Shutdown if still running
    if ((Get-VM -Name $vm.Name).PowerState -eq "PoweredOn") {
        Write-Host "Trying ACPI shutdown..."
        Stop-VMGuest -VM $vm -Confirm:$false | Out-Null

        # Wait up to 1 minute
        for ($i = 0; $i -lt 12; $i++) {
            if ((Get-VM -Name $vm.Name).PowerState -eq "PoweredOff") {
                Write-Host "VM powered off via ACPI"
                break
            }
            Start-Sleep -Seconds 5
        }
    }

    # 3. Force power-off if still on
    if ((Get-VM -Name $vm.Name).PowerState -eq "PoweredOn") {
        Write-Host "FORCE power-off..."
        Stop-VM -VM $vm -Kill -Confirm:$false | Out-Null
        Write-Host "VM forced off"
    }
}

Write-Host ""
Write-Host "=== All VMs processed. Shutting down ESXi host ==="

Stop-VMHost -VMHost (Get-VMHost) -Force -Confirm:$false | Out-Null

Disconnect-VIServer -Server * -Force -Confirm:$false | Out-Null

exit 0

```
