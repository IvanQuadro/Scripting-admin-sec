📝 Log Cleaner & Reporter

A simple PowerShell script to scan log files, filter unwanted entries, and export a CSV report with counts of ERROR, WARNING, and CRITICAL events.

📦 How to use

Clone the repo:

```powershell
git clone https://github.com/LogCleanerReporter/LogCleanerReporter.git
cd LogCleanerReporter
```

```powershell

param (
    [Parameter(Mandatory=$true)]
    [string]$LogPath,
    [string]$OutputDir = ".\output",
    [string]$ExcludeFile = ".\config\exclude.txt"
)

if (Test-Path $ExcludeFile) {
    $exclusions = Get-Content $ExcludeFile
} else {
    $exclusions = @()
}

if (-not (Test-Path $OutputDir)) {
    New-Item -ItemType Directory -Path $OutputDir | Out-Null
}

$TimeStamp = Get-Date -Format "yyyyMMdd-HHmm"

$files = Get-ChildItem -Path $LogPath -Filter *.log -File

$exRegex = $null
if ($exclusions -and $exclusions.Count -gt 0) {
    $exRegex = ($exclusions | ForEach-Object { [regex]::Escape($_) }) -join '|'
}

$results = @()

foreach ($file in $files) {
    try {
        $lines = Get-Content -Path $file.FullName -ErrorAction Stop
        if ($exRegex) {
            $lines = $lines | Where-Object { $_ -notmatch $exRegex }
        }
        $errCount  = ($lines | Where-Object { $_ -match '\bERROR\b'    }).Count
        $warnCount = ($lines | Where-Object { $_ -match '\bWARNING\b'  }).Count
        $critCount = ($lines | Where-Object { $_ -match '\bCRITICAL\b' }).Count
        $results += [PSCustomObject]@{
            File     = $file.FullName
            Error    = $errCount
            Warning  = $warnCount
            Critical = $critCount
        }
    }
    catch {
        Write-Warning "Impossibile leggere '$($file.FullName)': $($_.Exception.Message)"
    }
}

$outputFile = Join-Path $OutputDir "LogReport-$TimeStamp.csv"
$results | Export-Csv -Path $outputFile -NoTypeInformation
Write-Output "Report saved in $outputFile"

$totalErrors = ($results | Measure-Object -Property Error -Sum).Sum
if ($totalErrors -gt 0) {
    Write-Warning "⚠️ A total of $totalErrors errors were found in the logs!"
    exit 1
} else {
    Write-Host "✅ No errors found"
    exit 0
}
```
