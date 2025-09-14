🗂️ What the project does

It scans a given directory recursively and reports the largest files.
The goal is to quickly identify files consuming the most disk space and to check whether they exceed a defined size threshold.
```powershell

git clone https://github.com/YourUser/DiskUsageReporter.git
cd DiskUsageReporter
.\'Disk Usage Reporter.ps1' -Path "C:\Users\Example" -TopFiles 10 -ThresholdMB 100
```
```powershell
param(
    [parameter(mandatory=$true)]
    [string]$Path,
    [int]$TopFiles           = 10, 
    [int]$ThresHold          = 100,
    [string]$outPutDir       = ".\output"

)

if (-not (Test-Path $Path)) {
    throw "the path $Path does not exist"
}

if ($topFiles -lt 1) {
    throw "Top file must be at least 1"
}

if ($ThresHold -lt 1) {
    throw "Threshold must be positive"
}


if (-not(Test-Path $outPutDir)) {
    New-Item -ItemType Directory -Path $outPutDir | Out-Null
}

$TimeStamp = Get-Date -Format "yyyyMMdd-HHmm"

try {
    $files = Get-ChildItem -Path $Path -Recurse -File -ErrorAction Stop 
    
    $results = foreach ($file in $files) {
     [PSCustomObject]@{
        FilePath  = $file.fullname
        SizeBytes = $file.length
        SizeMB    = [math]::Round($file.length / 1MB, 2)
        LastWriteTime = $file.LastWriteTime
    }
  }
  $TopResults = $results | Sort-Object -Property SizeBytes -Descending | Select-Object -First $TopFiles 
}
catch {
    Write-Warning "Could not enumerate files in $Path : $($_
    .Exception.Message)"
    exit 1
} 
```
