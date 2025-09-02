```bash
#!/usr/bin/env bash
# Process Memory Watchdog (Bash)
# Monitors processes that exceed a memory threshold, supports a whitelist, exports CSV, and sets an exit code.
# Default threshold: 500 MB (override with first arg).

set -euo pipefail

# -----------------------------
# Step 1 — Parameterize threshold
# -----------------------------
MemMB="${1:-500}"                    # threshold in MB (default 500)
MemKB=$(( MemMB * 1024 ))            # convert MB to KB (ps RSS is in KB)
echo "${MemMB} MB = $((MemKB * 1024)) bytes"

# -----------------------------
# Step 2 — Load whitelist
# -----------------------------
# File: ./config/whitelist.txt (one process name per line; no .exe)
whitelist_file="./config/whitelist.txt"
whitelist=()
if [[ -f "$whitelist_file" ]]; then
  # strip CR if file edited on Windows
  mapfile -t whitelist < <(sed 's/\r$//' "$whitelist_file" | awk 'NF')
else
  echo "No whitelist file found at $whitelist_file — continuing without it." >&2
fi

# Build an AWK-ready newline-joined list
wl_joined="$(printf '%s\n' "${whitelist[@]-}")"

# -----------------------------
# Step 3 — Prepare output folder & filename
# -----------------------------
mkdir -p ./output
timestamp="$(date +'%Y%m%d-%H%M')"
outputfile="./output/ProcessReport-${timestamp}.csv"

# -----------------------------
# Step 4 — Collect, filter, sort, select top 10
# -----------------------------
# ps: RSS in KB; fields: PID, COMMAND, RSS
# Filter: RSSKB > MemKB and COMMAND not in whitelist
# Export CSV columns: Name,Id,WorkingSet (bytes)
{
  echo "Name,Id,WorkingSet"
  ps -e -o pid=,comm=,rss= | \
  awk -v memkb="$MemKB" -v list="$wl_joined" '
    BEGIN {
      n = split(list, a, /\n/);
      for (i = 1; i <= n; i++) { if (length(a[i])) wl[a[i]] = 1 }
    }
    {
      pid = $1; name = $2; rsskb = $3;
      if (name == "" || rsskb == "") next;
      # filter by threshold and not in whitelist
      if (rsskb > memkb && !(name in wl)) {
        # WorkingSet in BYTES to match the PowerShell version
        printf "%s,%s,%d\n", name, pid, rsskb * 1024;
      }
    }
  ' | sort -t, -k3,3nr | head -n 10
} > "$outputfile"

echo "Report saved in $outputfile"

# -----------------------------
# Step 5 — Exit code for CI
# -----------------------------
# offenders = number of data rows (minus header)
offenders=$(( $(wc -l < "$outputfile") - 1 ))
if (( offenders > 0 )); then
  echo "Found ${offenders} processes above threshold." >&2
  exit 1
else
  echo "No process above threshold."
```
  exit 0
fi
