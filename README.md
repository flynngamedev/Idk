bash ‘$w=(Get-NetAdapter|?{$_.Name -like "*Wi-Fi*"}); $m="0"+"26AE"[(Get-Random -M 4)]+(-join(1..10|%{"0123456789ABCDEF"[(Get-Random -M 16)]})); gci "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002BE10318}" | %{ if((gp $_.PSPath -Name DriverDesc -EA 0).DriverDesc -eq $w.InterfaceDescription){sp $_.PSPath NetworkAddress $m -Force} }; Disable-NetAdapter -Name $w.Name -Confirm:$false; Sleep 2; Enable-NetAdapter -Name $w.Name; gci "$env:APPDATA\OpenCode\", "$env:LOCALAPPDATA\OpenCode\" -Filter *.json -Recurse | ri -Force -EA 0; Write-Host "Done! Fresh MAC $m and IP active." -FG Green’

# 1. Generate a brand new valid random MAC (second character must be 2, 6, A, or E)
$chars = "0123456789ABCDEF"
$secChars = "26AE"
$newMac = "0" + $secChars[(Get-Random -Max 4)] + $(1..10 | ForEach-Object { $chars[(Get-Random -Max 16)] }) -join ""

# 2. Get the active Wi-Fi adapter's registry interface index
$wifi = Get-NetAdapter | Where-Object {$_.MediaType -eq "802.3" -and $_.Name -like "*Wi-Fi*"}
$driverDesc = $wifi.InterfaceDescription
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002BE10318}"

$subKeys = Get-ChildItem -Path $regPath
foreach ($key in $subKeys) {
    $desc = Get-ItemProperty -Path $key.PSPath -Name "DriverDesc" -ErrorAction SilentlyContinue
    if ($desc -and $desc.DriverDesc -eq $driverDesc) {
        # Inject the new random MAC directly into the system hardware adapter configuration
        Set-ItemProperty -Path $key.PSPath -Name "NetworkAddress" -Value $newMac -Force
        Write-Host "Injected Fresh MAC Profile: $newMac" -ForegroundColor Green
    }
}

# 3. Quick-restart the network card interface to lease the new IP address immediately
Disable-NetAdapter -Name $wifi.Name -Confirm:$false
Start-Sleep -Seconds 2
Enable-NetAdapter -Name $wifi.Name
Write-Host "Network Adapter Reloaded with a completely fresh IP layout!" -ForegroundColor Cyan
