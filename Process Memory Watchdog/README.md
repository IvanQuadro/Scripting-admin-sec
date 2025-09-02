# Parameter for memory threshold in MB (default: 500 MB)
param([int]$MemMB = 500)

# Convert the threshold from MB to bytes (WorkingSet is measured in bytes)
$MemBytes = [long]$MemMB * 1MB
Write-Output "$MemMB MB = $MemBytes bytes"

# (Standalone line, not required) – left for debug/testing
Where-Object {$_.WorkingSet -gt $MemBytes}

# Load process names to ignore from the whitelist file
$whitelist = Get-Content .\config\whitelist.txt

# Generate a timestamp for the CSV filename
$timestamp = Get-Date -Format "yyyyMMdd-HHmm" 

# Create the output folder if it doesn’t exist (suppress output)
New-Item -ItemType Directory -Path .\output -Force | Out-Null

# Full path of the CSV file with a timestamped name
$outputfile = ".\output\ProcessReport-$timestamp.csv"

# Get processes that:
# - Exceed the memory threshold ($MemBytes)
# - Are NOT in the whitelist
# Sort by WorkingSet in descending order
# Take only the top 10 processes
$offenders = Get-Process | 
    Where-Object {$_.WorkingSet -gt $MemBytes -and $_.ProcessName -notin $whitelist} |
    Sort-Object -Property WorkingSet -Descending |
    Select-Object -First 10 name,id,WorkingSet

# Export the selected processes to a CSV file
$offenders | Export-Csv -Path $outputfile -NoTypeInformation

# Print confirmation message
Write-Output "Report saved in $outputfile"

# If there are processes above the threshold → exit code 1 (warning)
# Otherwise → exit code 0
if($offenders.Count -gt 0) {
    Write-Warning "Found $($offenders.Count) above treshold"
    exit 1
} else {
    Write-Host "No process about treshold found"
    exit 0
}
