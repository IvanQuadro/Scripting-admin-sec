

[CmdletBinding()]
param(
    [Parameter(Mandatory = $true)]
    [string]$Path,

    [int]$TopFiles     = 10,
    [int]$ThresholdMB  = 100,
    [string]$OutputDir = ".\output"
)

if (-not (Test-Path -Path $Path)) {
    Write-Error "$Path does not exist" -ErrorAction Stop
}

if ($TopFiles -lt 1 -or $ThresholdMB -lt 1) {
    Write-Error "Error! TopFiles=$TopFiles or ThresholdMB=$ThresholdMB must be positive" -ErrorAction Stop
}

$sw = [System.Diagnostics.Stopwatch]::StartNew()
Write-Verbose "Starting scan on '$Path' with TopFiles=$TopFiles, ThresholdMB=$ThresholdMB, OutputDir='$OutputDir'"

$results = foreach ($file in (Get-ChildItem -Path $Path -Recurse -File -ErrorAction Stop)) {
    [PSCustomObject]@{
        Path          = $file.FullName
        SizeBytes     = $file.Length
        SizeMB        = [Math]::Round($file.Length / 1MB, 2)
        LastWriteTime = $file.LastWriteTime
    }
}
Write-Verbose "Discovered $($results.Count) file(s)"

$topResults = $results |
    Sort-Object -Property SizeBytes -Descending |
    Select-Object -First $TopFiles |
    Select-Object *, @{
        Name       = 'IsOverThreshold'
        Expression = { $_.SizeMB -ge $ThresholdMB }
    }

if (-not (Test-Path -Path $OutputDir)) {
    New-Item -ItemType Directory -Path $OutputDir | Out-Null
    Write-Verbose "Created output directory '$OutputDir'"
}

$timeStamp = Get-Date -Format "yyyyMMdd-HHmm"
$OutFile   = Join-Path -Path $OutputDir -ChildPath "report-$timeStamp.csv"
Write-Verbose "Report path will be: $OutFile"

$topResults | Export-Csv -Path $OutFile -NoTypeInformation -Encoding UTF8
Write-Verbose "CSV saved"

$sw.Stop()
Write-Verbose "Completed in $([Math]::Round($sw.Elapsed.TotalSeconds, 2))s"

$topResults
