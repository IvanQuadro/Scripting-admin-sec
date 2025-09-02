# 🖥️ Process Memory Watchdog

A simple PowerShell script to monitor and export processes above a memory threshold.

## 📦 How to use

Clone the repo:
```powershell
git clone https://github.com/YOUR_USERNAME/ProcessMemoryWatchdog.git
cd ProcessMemoryWatchdog
.\Variables.ps1
```
```powershell

param([int]$MemMB = 500)

$MemBytes = [long]$MemMB * 1MB
Write-Output "$MemMB MB = $MemBytes bytes"

$whitelist = Get-Content .\config\whitelist.txt
$timestamp  = Get-Date -Format 'yyyyMMdd-HHmm'

New-Item -ItemType Directory -Path .\output -Force | Out-Null
$outputFile = ".\output\ProcessReport-$timestamp.csv"

$offenders = Get-Process |
  Where-Object { $_.WorkingSet -gt $MemBytes -and $_.ProcessName -notin $whitelist } |
  Sort-Object -Property WorkingSet -Descending |
  Select-Object -First 10 Name, Id, WorkingSet

$offenders | Export-Csv -Path $outputFile -NoTypeInformation
Write-Output "Report saved in $outputFile"    

if ($offenders.Count -gt 0) {      
    Write-Warning "Found $($offenders.Count) processes above threshold!"
    exit 1
} else {
    Write-Host "No processes above threshold ✅"
    exit 0
}
```
- Le triple backticks + `powershell` danno **colorazione PowerShell**.
- Gli utenti possono **selezionare tutto e copiare** facilmente.

---

### 🔹 4. Carica i file su GitHub
Puoi usare:
1. **Git da terminale**:
   ```bash
   git add .
   git commit -m "Add Process Memory Watchdog script"
   git push
```
