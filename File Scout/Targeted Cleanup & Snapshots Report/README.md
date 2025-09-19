
```powershell
param(
    #[parameter(Mandatory=$true)]
    [string]$Path,
    [int]$OlderThanDays = 90,
    [int]$MinSizeMB     = 10,
    [int]$TopN          = 20,
    [string]$OutPutDir  = ".\output"
)

if (-not(Test-Path -Path $Path)) {
    throw "'$Path' does not exist" 
} 

if ($OlderThanDays -lt 1 -or $MinSizeMB -lt 1 -or $TopN -lt 1) {
    Write-Error "'$OlderThanDays', '$MinSizeMB', and '$TopN' must be positive" -ErrorAction Stop
}

try {
    $files = Get-ChildItem -path $Path -Recurse -File -ErrorAction Stop
}
catch {
    throw "Failed to enumerate files in $Path. Error: $_"
}

$results = foreach($file in $files){
    [PSCustomObject]@{
        Path            = $file.fullname
        SizeBytes       = $file.length
        SizeMB          = [math]::Round($file.length /1MB, 2)
        LastWriteTime   = $file.LastWriteTime
        Extension       = $file.Extension
        DirectoryName   = $file.DirectoryName
    }
}

$cutOff = (Get-Date).AddDays(-$OlderThanDays)
    $filtered = $results | Where-Object {
        ($_.LastWriteTime -lt $cutOff) -and ($_.SizeMB -ge $MinSizeMB)
    } 

if ($filtered.Count -eq 0) {
    Write-Output "No files matched the filters - exiting"
    exit 1
} 

$top = $filtered | 
Sort-Object -Property SizeBytes -Descending | 
Select-Object -First $TopN -Property Path, SizeMB, LastWriteTime,
 @{Name='isArchiveCandidate'; Expression={$true}}


$TimeStamp = Get-Date -Format "yyyyMMdd-HHmm"
$outFile   = Join-Path -Path $OutPutDir -ChildPath "report-$timeStamp.csv"
Write-Verbose "Report path will be: $outFile"
```
