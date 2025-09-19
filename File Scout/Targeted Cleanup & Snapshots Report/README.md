
```powershell
[CmdletBinding()]
param(
    [string]$Path,
    [int]$OlderThanDays = 90,
    [int]$MinSizeMB     = 10,
    [int]$TopN          = 20,
    [string]$OutputDir  = ".\output"
)

if (-not (Test-Path -Path $Path)) { throw "'$Path' does not exist" }
if ($OlderThanDays -lt 1 -or $MinSizeMB -lt 1 -or $TopN -lt 1) { Write-Error "OlderThanDays, MinSizeMB, and TopN must be positive" -ErrorAction Stop }

try {
    $files = Get-ChildItem -Path $Path -Recurse -File -ErrorAction Stop
}
catch {
    throw "Failed to enumerate files in $Path. Error: $_"
}

$results = foreach ($file in $files) {
    [PSCustomObject]@{
        Path          = $file.FullName
        SizeBytes     = $file.Length
        SizeMB        = [math]::Round($file.Length / 1MB, 2)
        LastWriteTime = $file.LastWriteTime
        Extension     = $file.Extension
        DirectoryName = $file.DirectoryName
    }
}

$cutOff   = (Get-Date).AddDays(-$OlderThanDays)
$filtered = $results | Where-Object { ($_.LastWriteTime -lt $cutOff) -and ($_.SizeMB -ge $MinSizeMB) }

if ($filtered.Count -eq 0) { Write-Output "No files matched the filters - exiting"; exit 1 }

$top = $filtered |
    Sort-Object -Property SizeBytes -Descending |
    Select-Object -First $TopN -Property Path, SizeMB, LastWriteTime, Extension, DirectoryName,
        @{Name='IsArchiveCandidate'; Expression = { $true }}

$TimeStamp = Get-Date -Format "yyyyMMdd-HHmm"
if (-not (Test-Path -Path $OutputDir)) { New-Item -ItemType Directory -Path $OutputDir | Out-Null }
$outFile = Join-Path -Path $OutputDir -ChildPath "report-$TimeStamp.csv"
Write-Verbose "Report path will be: $outFile"
$top | Export-Csv -Path $outFile -NoTypeInformation -Encoding UTF8
Write-Verbose "CSV saved"
```
