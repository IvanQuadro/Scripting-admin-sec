🛡️ EventLog Sentinel
A simple PowerShell script to sweep recent Windows Event Logs, filter noise via whitelist, and export a timestamped CSV.

📦 How to use
Clone the repo:
```powershell

git clone https://github.com/EventLogSentinel/EventLogSentinel.git
cd EventLogSentinel
.\EventLogSentinel.ps1
```
```powershell

param([int]$MinutesBack = 60)

$startTime = (Get-Date).AddMinutes(-$MinutesBack)

$patterns = Get-Content .\config\Event_Whitelist.txt -ErrorAction SilentlyContinue
$regex    = ($patterns -join '|')

if (-not $patterns -or ($patterns -join '' -eq '')) { $regex = $null }

$events = Get-WinEvent -FilterHashtable @{
  LogName   = 'Application'
  Level     = 2              # 2 = Error
  StartTime = $startTime
} |
Where-Object {
  -not $regex -or (
    $_.ProviderName -notmatch $regex -and
    $_.Id           -notmatch $regex -and
    $_.Message      -notmatch $regex
  )
} |
Select-Object `
  TimeCreated,
  LogName,
  Id,
  LevelDisplayName,
  ProviderName,
  Message |
Sort-Object -Property TimeCreated -Descending |
Select-Object -First 500

New-Item -ItemType Directory -Path .\output -Force | Out-Null

$timestamp  = Get-Date -Format 'yyyyMMdd-HHmm'
$outputFile = ".\output\sentinel-$timestamp.csv"

$events | Export-Csv -Path $outputFile -NoTypeInformation
Write-Host "Scanned since $startTime. Report saved in $outputFile"

if ($events.Count -gt 0) {
  Write-Warning "Found $($events.Count) events"
  exit 1
} else {

```
  Write-Host "No relevant events ✅"
  exit 0
}
