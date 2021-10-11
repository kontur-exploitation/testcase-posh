# Задание 1
Напиши функцию, преобразующую строку с именами серверов ```vm-test[1-3],vm-test3,vm-test[4-7],vm-test8```
в список вида ```vm-test1 vm-test2 vm-test3 vm-test3 vm-test4 vm-test5 vm-test6 vm-test7 vm-test8```.

# Задание 2
Разберись, что делает этот скрипт (построчно). Будь готов рассказать о механике скрипта на собеседовании.
```powershell
$GLPath = "$env:SystemDrive\gitlab-runner"
$GLBin  = "$GLPath\gitlab-runner.exe"
$Token  = $using:Token
$User   = $using:User
$Pass   = $using:Pass
$GitUri = 'https://github.com/git-for-windows/git/releases/download/v2.25.0.windows.1/Git-2.25.0-64-bit.exe'
$Uri    = 'https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe'

$progressPreference = 'silentlyContinue'
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]'Ssl3,Tls,Tls11,Tls12'

& git --version | Out-Null
if ($LASTEXITCODE -ne 0) {
    Set-Location "$env:windir\temp"
    Invoke-WebRequest -UseBasicParsing -Uri $GitUri -OutFile ".\git-installer.exe"
    Start-Process '.\git-installer.exe' -ArgumentList '/SILENT' -NoNewWindow -Wait
    Remove-Item '.\git-installer.exe' -Force
    $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
}

if(Get-Service "gitlab-runner" -ea 0){
    $bin = (Get-WmiObject win32_service | Where-Object{$_.Name -like 'gitlab-runner'} | Select-Object PathName).PathName -split ' ' | Select-Object -First 1
    $GLPath = Split-Path $bin -Parent
    Set-Location $GLPath
    & $bin --version | Where-Object {$_ -match '^Version:\s*(\d+\.\d+\.\d+)'} | Out-Null
    $VersionOld = [version]$Matches[1]
    if(!$VersionOld) {
        throw "Error"
        exit 1
    }

    $GLBinNew = "$GLPath\gitlab-runner_new.exe"
    Invoke-WebRequest -UseBasicParsing -Uri $Uri -OutFile $GLBinNew

    & $GLBinNew --version | Where-Object {$_ -match '^Version:\s*(\d+\.\d+\.\d+)'} | Out-Null
    $VersionNew = [version]$Matches[1]

    if($VersionOld -ge $VersionNew){
        Remove-Item $GLBinNew
        exit 0
    }

    Write-Host "`nУстановка версии $VersionNew"
    & $bin stop      2>&1 | Write-Host
    & $bin uninstall 2>&1 | Write-Host
    Remove-Item $bin -Force

    Rename-Item $GLBinNew -NewName $GLBin
    & $GLBin install --user "$User" --password "$Pass" 2>&1 | Write-Host
    & $GLBin start 2>&1 | Write-Host

    Write-Host "`nDone."
}
else{
    New-Item -ItemType Directory -Path $GLPath -ea 0
    Set-Location $GLPath
    $Tags = @{$true='Evrika-dev';$false='Evrika-prod'}[([System.Net.Dns]::GetHostByName("localhost").HostName) -match 'dev.kontur']
    Invoke-WebRequest -UseBasicParsing -Uri $Uri -OutFile $GLBin

    & $GLBin install --user "$User" --password "$Pass" 2>&1 | Write-Host

    $tmp = New-TemporaryFile
    secedit /export /cfg "$tmp.inf" | Out-Null
    (gc -Encoding ascii "$tmp.inf") -replace '^SeServiceLogonRight .+', "`$0,$User" | sc -Encoding ascii "$tmp.inf"
    secedit /import /cfg "$tmp.inf" /db "$tmp.sdb" | Out-Null
    secedit /configure /db "$tmp.sdb" /cfg "$tmp.inf" | Out-Null
    rm $tmp* -ea 0

    & $GLBin register                     `
        --non-interactive                 `
        --url 'https://git.skbkontur.ru/' `
        --registration-token $Token       `
        --executor "shell"                `
        --description "Evrika"            `
        --tag-list $Tags 2>&1 | Write-Host

    Write-Host "`nStart gitlab-runner"
    & $GLBin start 2>&1 | Write-Host
}
``

# Задание 3
Напиши скрипт, который:
- поднимет на ОС Windows Server веб-сервер IIS;
- удалит дефолтный сайт;
- создаст сайт с путем ```C:\mysite``` на порту ```4321```; 
- отключит дефолтный ресайкл (```"recycle"```) и настроит ресайкл пула на ```3:00 каждый день```.

Бонусное задание - сделать скрипт **идемпотентным** (повторный прогон не упадет с ошибкой и не внесет изменений).