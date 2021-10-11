# Задание 1
Напиши функцию, преобразующую строку с именами серверов ```vm-test[1-3],vm-test3,vm-test[4-7],vm-test8```
в список вида ```vm-test1 vm-test2 vm-test3 vm-test3 vm-test4 vm-test5 vm-test6 vm-test7 vm-test8```.

# Задание 2
Разберись, что делает этот скрипт (построчно). Будь готов рассказать о механике скрипта на собеседовании.
```
param (
    [String] $role_id = $null,
    [String] $secret_id = $null,
    [string] $VaultAddr = 'https://vault.kontur.host'
)

if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(`
    [Security.Principal.WindowsBuiltInRole] "Administrator")) {
    $newProcess = New-Object System.Diagnostics.ProcessStartInfo "PowerShell";
    $newProcess.Arguments = "-Command $($myInvocation.line)"
    $newProcess.Verb = "runas";
    [System.Diagnostics.Process]::Start($newProcess) | Out-Null;
    exit
}

[System.Environment]::SetEnvironmentVariable('HOME', $env:USERPROFILE, [System.EnvironmentVariableTarget]::User)
[System.Environment]::SetEnvironmentVariable('VAULT_ADDR', $VaultAddr, [System.EnvironmentVariableTarget]::Machine)

if ($role_id -and $secret_id) {
  $body = @{"role_id" = $role_id; "secret_id" = $secret_id} | ConvertTo-Json
  $Response = Invoke-WebRequest -Method Post -Body $body -Uri "$env:VAULT_ADDR/v1/auth/approle/login" -ErrorAction Stop
  $content = $Response.Content | ConvertFrom-Json | Select-Object -ExpandProperty auth
  $token = $content.client_token
  $token | Out-File "$env:USERPROFILE\.vault-token" -Encoding ascii    
    $token | Out-File "$env:windir\System32\config\systemprofile\.vault-token" -Encoding ascii  
  exit 0
}
try{ & vault --version | Out-Null }catch{}
if ($LASTEXITCODE -ne 0) {
    choco install vault --yes -s kontur -f    
}

$isNeedToken = $false
$current = vault token lookup -format "json" 2>&1
if ($LASTEXITCODE -eq 0) {
    $token = $current | ConvertFrom-Json | Select-Object -ExpandProperty data
    $ExpireDays = New-TimeSpan -Start (Get-Date) -End (Get-Date($token.expire_time)) | Select-Object -ExpandProperty days     
    if ($null -ne $token -and $ExpireDays -lt 10) {
        vault token revoke -self
        $isNeedToken = $true
    } else {
        $token.id | Out-File "$env:windir\System32\config\systemprofile\.vault-token" -Encoding ascii
        Write-Host "Nothing to do, $ExpireDays days left before the token expires" -ForegroundColor Green
    vault read secret/data/evrika/services/git_readonly 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
      Write-Warning "Сannot read the test secret, need to update the token"
      $isNeedToken = $true 
    }
    }
} else {
    $isNeedToken = $true
}

if ($isNeedToken) {    
  while ($isNeedToken) {
    Write-Host "`nIssue token for $env:USERNAME" -ForegroundColor Green
    $token = vault login -token-only -method=ldap username=$env:USERNAME 
    if ($LASTEXITCODE -eq 0) {
      $isNeedToken = $false
    } else {
      Write-Host "`nIncorrect password for $env:USERNAME or network error" -ForegroundColor Red
    }
  }
    
    $token | Out-File "$env:USERPROFILE\.vault-token" -Encoding ascii    
    $token | Out-File "$env:windir\System32\config\systemprofile\.vault-token" -Encoding ascii
}
```

# Задание 3
Напиши скрипт, который:
- поднимет на ОС Windows Server веб-сервер IIS;
- удалит дефолтный сайт;
- создаст сайт с путем ```C:\mysite``` на порту ```4321```; 
- отключит дефолтный ресайкл (```"recycle"```) и настроит ресайкл пула на ```3:00 каждый день```.

Бонусное задание - сделать скрипт **идемпотентным** (повторный прогон не упадет с ошибкой и не внесет изменений).