🛡️ What the project does

It scans the Windows Security Event Log for failed RDP (Remote Desktop Protocol) logon attempts.
The goal is to detect when an IP address is trying repeatedly to break in — a common sign of brute-force attacks.

```powershell
git clone https://github.com/YourUser/RDPBurstDetector.git
cd RDPBurstDetector
.\RDP Failed Logons.ps1
```
```powershell
param(
  [int]$MinutesBack = 60,
  [int]$Threshold = 5,
  [int[]]$LogonTypes = @(10),
  [string]$IpWhitelistPath = ".\config\ip_whitelist.txt",
  [string]$UserWhitelistPath = ".\config\user_whitelist.txt",
  [switch]$ExcludeLocalRanges,
  [string]$OutDir = ".\output"
)

$start = (Get-Date).AddMinutes(-1 * $MinutesBack)
New-Item -ItemType Directory -Path $OutDir -Force | Out-Null
$stamp  = Get-Date -Format 'yyyyMMdd-HHmm'
$outCsv = Join-Path $OutDir "rdp-bursts-$stamp.csv"

$ipWL   = if (Test-Path $IpWhitelistPath)   { Get-Content $IpWhitelistPath }   else { @() }
$userWL = if (Test-Path $UserWhitelistPath) { Get-Content $UserWhitelistPath } else { @() }

$raw = Get-WinEvent -FilterHashtable @{
  LogName   = 'Security'
  Id        = 4625
  StartTime = $start
}

$records = foreach ($e in $raw) {
  $xml = [xml]$e.ToXml()
  $data = @{}
  foreach ($d in $xml.Event.EventData.Data) { $data[$d.Name] = $d.'#text' }
  $logonType = [int]($data['LogonType'])
  $user      = $data['TargetUserName']
  $ip        = $data['IpAddress']
  if ($LogonTypes -contains $logonType) {
    [pscustomobject]@{
      TimeCreated   = $e.TimeCreated
      LogonType     = $logonType
      TargetUser    = $user
      IpAddress     = $ip
      Status        = $data['Status']
      SubStatus     = $data['SubStatus']
      FailureReason = $data['FailureReason']
    }
  }
}

$filtered = $records | Where-Object {
  $_.IpAddress -and $_.IpAddress -ne '-' -and
  ($_.IpAddress -notin $ipWL) -and
  ($_.TargetUser -notin $userWL)
}

function Test-PrivateIP {
  param([string]$ip)
  if (-not $ip) { return $false }
  return  $ip -match '^(10\.)' `
       -or $ip -match '^(192\.168\.)' `
       -or $ip -match '^(172\.(1[6-9]|2[0-9]|3[0-1])\.)' `
       -or $ip -match '^(127\.)' `
       -or $ip -match '^(169\.254\.)'
}

if ($ExcludeLocalRanges) {
  $filtered = $filtered | Where-Object { -not (Test-PrivateIP $_.IpAddress) }
}

$groups = $filtered | Group-Object IpAddress

$findings = foreach ($g in $groups) {
  $count = $g.Count
  if ($count -ge $Threshold) {
    $ordered = $g.Group | Sort-Object TimeCreated
    [pscustomobject]@{
      IpAddress  = $g.Name
      Count      = $count
      FirstSeen  = $ordered[0].TimeCreated
      LastSeen   = $ordered[-1].TimeCreated
      Users      = ($g.Group | Select-Object -Expand TargetUser -Unique) -join ','
      LogonTypes = ($g.Group | Select-Object -Expand LogonType -Unique) -join ','
      Statuses   = ($g.Group | Select-Object -Expand Status -Unique) -join ','
    }
  }
}

$findings | Sort-Object Count -Descending | Export-Csv $outCsv -NoTypeInformation -Force

if ($findings.Count -gt 0) {
  $summary = ($findings | ForEach-Object { "$($_.IpAddress):$($_.Count)" }) -join " | "
  Write-Warning "RDP failed-logon bursts: $($findings.Count) | $summary | Report: $outCsv"
  exit 1
} else {
  Write-Host "Clean: no bursts >= $Threshold in last $MinutesBack minutes."
  exit 0
}
```
