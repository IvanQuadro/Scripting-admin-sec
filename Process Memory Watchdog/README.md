param([int]$MemMB = 500)

$MemBytes = [long]$MemMB * 1MB
Write-Output "$MemMB MB = $MemBytes bytes"

$whitelist = Get-Content .\config\whitelist.txt
$timestamp  = Get-Date -Format 'yyyyMMdd-HHmm'

🔹 Create output folder
New-Item -ItemType Directory -Path .\output -Force | Out-Null
$outputFile = ".\output\ProcessReport-$timestamp.csv"

🔹 Get filtered processes
$offenders = Get-Process |
  Where-Object { $_.WorkingSet -gt $MemBytes -and $_.ProcessName -notin $whitelist } |
  Sort-Object -Property WorkingSet -Descending |
  Select-Object -First 10 Name, Id, WorkingSet

🔹 Export results
$offenders | Export-Csv -Path $outputFile -NoTypeInformation

Write-Output "Report saved in $outputFile"

🔹 Check result
if ($offenders.Count -gt 0) {
    Write-Warning "Found $($offenders.Count) processes above threshold!"
    exit 1
} else {
    Write-Host "No processes above threshold ✅"
    exit 0
}
