# PC Freezing Fix - Complete Installation Documentation

**Date:** January 7, 2026  
**Problem:** PC freezes multiple times daily  
**Cause:** Antigravity (Google IDE) memory leak + unlimited WSL memory usage  
**Solution:** WSL memory limitation + automatic Antigravity monitoring

---

## Table of Contents

1. [Problem Diagnosis](#problem-diagnosis)
2. [Solution Overview](#solution-overview)
3. [Installation Steps](#installation-steps)
4. [Configuration Details](#configuration-details)
5. [User Guide](#user-guide)
6. [Maintenance](#maintenance)
7. [Troubleshooting](#troubleshooting)
8. [References](#references)

---

## Problem Diagnosis

### Symptoms
- PC freezes completely (mouse and keyboard unresponsive)
- Occurs multiple times daily
- Typically when working in Antigravity/Python
- Must hold power button to shut down

### Original System Status
```
Total RAM: 16GB
WSL could use: Unlimited (up to ~15GB)
Antigravity memory usage: Could grow uncontrolled
CPU: 2% (not the problem)
Disk: 2% active time (not the problem)
```

### Diagnosed Issues

**1. Antigravity Memory Leak (Documented)**
- Virtual Memory Bloat: ~1.4TB VSZ with only 70MB RSS
- VSZ/RSS ratio: 20,000:1 (normal is 2:1 to 10:1)
- AI diff overlays accumulate in RAM
- 50-100x higher than normal Electron applications

**2. Unlimited WSL Memory Usage**
- WSL2 could use all available RAM
- No .wslconfig limits
- VmmemWSL process could grow uncontrolled

**3. Combined Effect**
- Antigravity in WSL + trading programs = memory collision
- When memory is full → Windows uses swap → everything freezes

---

## Solution Overview

### Implemented Measures

| Measure | Purpose | Effect |
|---------|---------|--------|
| `.wslconfig` | Limit WSL to 6GB | Guarantees WSL cannot take all memory |
| Antigravity Guardian | Monitor and restart Antigravity | Prevents memory leak from growing |
| Memory Health Checker | Generic memory monitoring | Can be used for trading programs |
| Quick Check Script | Fast memory status | Easy diagnostics |
| Emergency Cleanup | Emergency stop | Quick recovery when problems occur |

---

## Installation Steps

### STEP 1: Create .wslconfig (Critical)

**Location:** `C:\Users\[username]\.wslconfig`

**On Windows (NOT in WSL):**

1. Open File Explorer
2. Navigate to: `%USERPROFILE%`
3. Create new file: `.wslconfig`
4. Open with Notepad
5. Paste:

```ini
[wsl2]
# Limit memory to 6GB (adjust based on your total memory)
memory=6GB

# Limit CPU to 4 cores
processors=4

# Limit swap to 2GB
swap=2GB

# Enable memory reclaim after inactivity
autoMemoryReclaim=gradual

# Don't lock all memory
lockedMemory=false

# Disable nested virtualization
nestedVirtualization=false
```

6. Save file

**Restart WSL:**

```powershell
# In PowerShell as Administrator
wsl --shutdown
Start-Sleep -Seconds 10
wsl
```

**Verify:**

```bash
# In WSL
free -h
```

Expected output:
```
              total        used        free      shared  buff/cache   available
Mem:           5.8Gi       784Mi       4.1Gi       4.7Mi       1.1Gi       5.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

✅ Total should be ~5.8GB (not 15GB+)

---

### STEP 2: Find Antigravity Installation

```bash
which antigravity
```

**Result for this installation:**
```
/mnt/d/Antigravity/bin/antigravity
```

---

### STEP 3: Install Bash Scripts

#### 3.1 Create Scripts Directory

```bash
cd ~
mkdir -p scripts
cd scripts
pwd  # Should show: /home/rune/scripts
```

#### 3.2 Create antigravity_memory_guardian.sh

```bash
nano antigravity_memory_guardian.sh
```

**Content:**

```bash
#!/bin/bash

##############################################################################
# Antigravity Memory Guardian
# Monitors Antigravity processes and restarts on high memory usage
##############################################################################

# Configuration
MAX_MEMORY_MB=1500              # Max memory per Antigravity instance
CHECK_INTERVAL_SECONDS=30       # How often to check
LOG_FILE="$HOME/antigravity_guardian.log"
RESTART_COOLDOWN=300            # Wait 5 min between restarts
ANTIGRAVITY_PATH="/mnt/d/Antigravity/bin/antigravity"

# Variables
last_restart_time=0

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Check if Antigravity is running
is_antigravity_running() {
    pgrep -f "antigravity" > /dev/null 2>&1
    return $?
}

# Get total memory usage for all Antigravity processes
get_antigravity_memory() {
    # Sum RSS (Resident Set Size) for all antigravity processes
    local mem=$(ps aux | grep -i antigravity | grep -v grep | awk '{sum+=$6} END {print sum/1024}')
    echo "${mem:-0}"
}

# Get number of Antigravity processes
get_antigravity_process_count() {
    pgrep -f "antigravity" | wc -l
}

# List top Antigravity processes
list_top_antigravity_processes() {
    log "Top 5 Antigravity processes:"
    ps aux | grep -i antigravity | grep -v grep | sort -k4 -rn | head -5 | while read line; do
        log "  $line"
    done
}

# Kill Antigravity processes
kill_antigravity() {
    log "⚠️  Killing Antigravity processes..."
    
    # List processes before killing them
    list_top_antigravity_processes
    
    # Try graceful shutdown first
    pkill -15 -f "antigravity"
    sleep 5
    
    # If still running, force kill
    if is_antigravity_running; then
        log "Force killing remaining processes..."
        pkill -9 -f "antigravity"
        sleep 2
    fi
    
    log "✓ Antigravity processes killed"
}

# Start Antigravity
start_antigravity() {
    log "🚀 Starting Antigravity..."
    
    # Start Antigravity in background
    nohup "$ANTIGRAVITY_PATH" > /dev/null 2>&1 &
    
    sleep 3
    
    if is_antigravity_running; then
        log "✓ Antigravity started"
    else
        log "❌ Failed to start Antigravity"
    fi
}

# Check if we can restart (cooldown)
can_restart() {
    current_time=$(date +%s)
    time_since_last_restart=$((current_time - last_restart_time))
    
    if [ $time_since_last_restart -lt $RESTART_COOLDOWN ]; then
        remaining=$((RESTART_COOLDOWN - time_since_last_restart))
        log "⏳ Cooldown active. Waiting $remaining seconds before next restart."
        return 1
    fi
    return 0
}

# Main monitoring loop
main_loop() {
    log "========================================"
    log "Antigravity Memory Guardian started"
    log "Max Memory: ${MAX_MEMORY_MB}MB"
    log "Check Interval: ${CHECK_INTERVAL_SECONDS}s"
    log "Antigravity Path: ${ANTIGRAVITY_PATH}"
    log "========================================"
    
    while true; do
        if is_antigravity_running; then
            memory_mb=$(get_antigravity_memory)
            process_count=$(get_antigravity_process_count)
            
            # Round to integer
            memory_mb=${memory_mb%.*}
            
            log "Antigravity: ${memory_mb}MB across ${process_count} processes"
            
            # Check if memory is over limit
            if [ "$memory_mb" -gt "$MAX_MEMORY_MB" ]; then
                log "🚨 ALERT: Antigravity memory usage over limit! (${memory_mb}MB > ${MAX_MEMORY_MB}MB)"
                
                if can_restart; then
                    kill_antigravity
                    
                    # Wait a bit before restart
                    log "Waiting 10 seconds before restart..."
                    sleep 10
                    
                    start_antigravity
                    
                    last_restart_time=$(date +%s)
                else
                    log "⚠️  Would restart, but cooldown is active"
                fi
            fi
        else
            log "ℹ️  Antigravity not running"
        fi
        
        sleep $CHECK_INTERVAL_SECONDS
    done
}

# Handle SIGINT (Ctrl+C)
trap 'log "Guardian stopped by user"; exit 0' INT TERM

# Start guardian
main_loop
```

Save: `Ctrl+X`, `Y`, `Enter`

#### 3.3 Create check_antigravity_memory.sh

```bash
nano check_antigravity_memory.sh
```

**Content:**

```bash
#!/bin/bash

##############################################################################
# Quick Antigravity Memory Check
# Display detailed memory information for Antigravity
##############################################################################

echo "========================================"
echo "Antigravity Memory Report"
echo "========================================"
echo ""

# Check if Antigravity is running
if ! pgrep -f "antigravity" > /dev/null 2>&1; then
    echo "❌ Antigravity not running"
    exit 0
fi

# Total memory usage
total_mem=$(ps aux | grep -i antigravity | grep -v grep | awk '{sum+=$6} END {print sum/1024}')
echo "Total Memory: ${total_mem%.*} MB"
echo ""

# Number of processes
process_count=$(pgrep -f "antigravity" | wc -l)
echo "Total Processes: $process_count"
echo ""

# Top 10 processes
echo "Top 10 Memory Consumers:"
echo "----------------------------------------"
printf "%-10s %-10s %-10s %s\n" "PID" "MEM%" "RSS(MB)" "COMMAND"
echo "----------------------------------------"

ps aux | grep -i antigravity | grep -v grep | sort -k4 -rn | head -10 | while read line; do
    pid=$(echo $line | awk '{print $2}')
    mem_pct=$(echo $line | awk '{print $4}')
    rss_kb=$(echo $line | awk '{print $6}')
    rss_mb=$((rss_kb / 1024))
    cmd=$(echo $line | awk '{for(i=11;i<=NF;i++) printf "%s ", $i; print ""}' | cut -c1-50)
    
    printf "%-10s %-10s %-10s %s\n" "$pid" "$mem_pct%" "${rss_mb}MB" "$cmd"
done

echo "----------------------------------------"
echo ""

# Process types
echo "Process Types:"
echo "----------------------------------------"
ps aux | grep -i antigravity | grep -v grep | awk '{print $11}' | sort | uniq -c | sort -rn
echo ""

# WSL total memory
echo "========================================"
echo "WSL Memory Status"
echo "========================================"
free -h
echo ""

# Disk usage
echo "========================================"
echo "Disk Usage"
echo "========================================"
df -h | grep -E "Filesystem|/mnt|/$"
```

Save: `Ctrl+X`, `Y`, `Enter`

#### 3.4 Create emergency_cleanup.sh

```bash
nano emergency_cleanup.sh
```

**Content:**

```bash
#!/bin/bash

##############################################################################
# Emergency Cleanup
# Kill all Antigravity processes and free memory
##############################################################################

echo "🚨 EMERGENCY CLEANUP"
echo "========================================"

# Kill Antigravity
echo "Stopping Antigravity processes..."
pkill -9 -f "antigravity"
sleep 2

# Sync disk
echo "Syncing disk..."
sync

# Drop caches (requires sudo)
if [ "$EUID" -eq 0 ]; then
    echo "Dropping caches..."
    echo 3 > /proc/sys/vm/drop_caches
else
    echo "⚠️  Run with sudo to drop caches: sudo $0"
fi

# Show results
echo ""
echo "========================================"
echo "Memory After Cleanup:"
free -h
echo ""
echo "Remaining Antigravity processes:"
pgrep -f "antigravity" | wc -l
echo "========================================"
```

Save: `Ctrl+X`, `Y`, `Enter`

#### 3.5 Make Scripts Executable

```bash
chmod +x *.sh
ls -lah
```

Expected output:
```
-rwxr-xr-x 1 rune rune 4.3K antigravity_memory_guardian.sh
-rwxr-xr-x 1 rune rune 2.0K check_antigravity_memory.sh
-rwxr-xr-x 1 rune rune  892 emergency_cleanup.sh
```

---

### STEP 4: Create Aliases

```bash
nano ~/.bashrc
```

Go to end (`Alt + /`) and add:

```bash

# Antigravity Memory Management
alias ag-guard="~/scripts/antigravity_memory_guardian.sh"
alias ag-check="~/scripts/check_antigravity_memory.sh"
alias ag-emergency="~/scripts/emergency_cleanup.sh"
```

Save: `Ctrl+X`, `Y`, `Enter`

**Load:**

```bash
source ~/.bashrc
```

**Test:**

```bash
type ag-check
```

Expected: `ag-check is aliased to '~/scripts/check_antigravity_memory.sh'`

---

### STEP 5: Test System

#### Test 1: Quick Check

```bash
ag-check
```

Expected output if Antigravity is running:
```
========================================
Antigravity Memory Report
========================================

Total Memory: 7 MB
Total Processes: 2

[... details ...]
```

#### Test 2: Guardian Test

```bash
~/scripts/antigravity_memory_guardian.sh
```

Let it run for 60 seconds, then `Ctrl+C`.

Expected output:
```
[2026-01-07 11:46:44] ========================================
[2026-01-07 11:46:44] Antigravity Memory Guardian started
[2026-01-07 11:46:44] Max Memory: 1500MB
[2026-01-07 11:46:44] Check Interval: 30s
[2026-01-07 11:46:44] Antigravity Path: /mnt/d/Antigravity/bin/antigravity
[2026-01-07 11:46:44] ========================================
[2026-01-07 11:46:44] Antigravity: 6MB across 2 processes
[... logs every 30 seconds ...]
```

#### Test 3: Log File

```bash
cat ~/antigravity_guardian.log
```

Should show all logs from guardian.

---

### STEP 6: Start Guardian Permanently

#### Start in Background:

```bash
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

#### Verify:

```bash
ps aux | grep antigravity_memory_guardian | grep -v grep
```

Expected output:
```
rune      1959  0.0  0.0   4888  3328 pts/0    S    11:55   0:00 /bin/bash /home/rune/scripts/antigravity_memory_guardian.sh
```

#### Add Auto-start:

```bash
cat >> ~/.bashrc << 'EOF'

# Start Antigravity Guardian automatically
if ! pgrep -f "antigravity_memory_guardian.sh" > /dev/null; then
    nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
fi
EOF
```

#### Verify Auto-start:

```bash
tail -10 ~/.bashrc
```

Should show:
```bash
# Antigravity Memory Management
alias ag-guard="~/scripts/antigravity_memory_guardian.sh"
alias ag-check="~/scripts/check_antigravity_memory.sh"
alias ag-emergency="~/scripts/emergency_cleanup.sh"

# Start Antigravity Guardian automatically
if ! pgrep -f "antigravity_memory_guardian.sh" > /dev/null; then
    nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
fi
```

---

## Configuration Details

### .wslconfig Parameters

| Parameter | Value | Explanation |
|-----------|-------|-------------|
| memory | 6GB | Max RAM WSL can use |
| processors | 4 | Max CPU cores |
| swap | 2GB | Max swap space |
| autoMemoryReclaim | gradual | Release memory gradually when inactive |
| lockedMemory | false | Don't lock memory (allow swapping) |
| nestedVirtualization | false | Disable nested VM (not needed) |

### Guardian Configuration

**In `antigravity_memory_guardian.sh`:**

```bash
MAX_MEMORY_MB=1500              # Restart if over 1500MB
CHECK_INTERVAL_SECONDS=30       # Check every 30 seconds
RESTART_COOLDOWN=300            # Wait 5 min between restarts
ANTIGRAVITY_PATH="/mnt/d/Antigravity/bin/antigravity"
```

**Adjusting Parameters:**

- **MAX_MEMORY_MB:** Increase if Antigravity legitimately needs more memory
- **CHECK_INTERVAL_SECONDS:** Reduce for more frequent checks (costs more CPU)
- **RESTART_COOLDOWN:** Increase if getting too many restarts

---

## User Guide

### Daily Use

The system runs automatically in the background. You don't need to do anything.

### Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `ag-check` | Quick memory status | When curious about memory usage |
| `ag-emergency` | Emergency stop Antigravity | If PC starts to freeze |
| `tail -f ~/antigravity_guardian.log` | Follow logs live | For debugging |
| `ps aux \| grep antigravity_memory_guardian` | Check if guardian is running | When troubleshooting |

### Monitoring the System

**View live logs:**

```bash
tail -f ~/antigravity_guardian.log
```

**Example of normal operation:**
```
[2026-01-07 11:55:51] Antigravity: 7MB across 2 processes
[2026-01-07 11:56:20] Antigravity: 7MB across 2 processes
[2026-01-07 11:56:50] Antigravity: 7MB across 2 processes
```

**Example of alert and restart:**
```
[2026-01-07 12:15:30] Antigravity: 1623MB across 15 processes
[2026-01-07 12:15:30] 🚨 ALERT: Antigravity memory usage over limit! (1623MB > 1500MB)
[2026-01-07 12:15:30] ⚠️  Killing Antigravity processes...
[2026-01-07 12:15:30] Top 5 Antigravity processes:
[... details ...]
[2026-01-07 12:15:35] ✓ Antigravity processes killed
[2026-01-07 12:15:45] 🚀 Starting Antigravity...
[2026-01-07 12:15:48] ✓ Antigravity started
```

### If PC Still Freezes

1. **Immediately after freezing, open WSL:**

```bash
ag-check
```

2. **Send output for analysis**

3. **Check logs:**

```bash
tail -50 ~/antigravity_guardian.log
```

4. **Check WSL memory usage from PowerShell:**

```powershell
Get-Process VmmemWSL | Select-Object @{Name="Memory(MB)";Expression={[math]::Round($_.WS/1MB,2)}}
```

---

## Maintenance

### Weekly

**Rotate logs (if they get large):**

```bash
# Check size
ls -lh ~/antigravity_guardian.log

# If over 10MB, archive
mv ~/antigravity_guardian.log ~/antigravity_guardian.log.$(date +%Y%m%d)

# Keep only last 4 weeks
find ~/ -name "antigravity_guardian.log.*" -mtime +28 -delete
```

### Monthly

**Review logs for patterns:**

```bash
# See how often Antigravity exceeded limit
grep "🚨 ALERT" ~/antigravity_guardian.log | wc -l

# See average memory usage
grep "Antigravity:" ~/antigravity_guardian.log | tail -100
```

**Adjust MAX_MEMORY_MB if needed:**

```bash
nano ~/scripts/antigravity_memory_guardian.sh
# Change MAX_MEMORY_MB=1500 to desired value
```

### On WSL Updates

**.wslconfig can be overwritten. Verify:**

```bash
cat ~/.wslconfig  # If this exists in WSL (wrong!)
```

**Correct location is on Windows:**
```
C:\Users\[username]\.wslconfig
```

---

## Troubleshooting

### Problem: Guardian Not Running

**Check:**

```bash
ps aux | grep antigravity_memory_guardian | grep -v grep
```

**If no output:**

```bash
# Start manually
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

**Check for errors:**

```bash
~/scripts/antigravity_memory_guardian.sh
# Let it run for 30 sec to see error messages
```

### Problem: WSL Still Using Too Much Memory

**Verify .wslconfig:**

On Windows, check that file exists:
```
C:\Users\[username]\.wslconfig
```

**Restart WSL:**

```powershell
wsl --shutdown
wsl
```

**Verify in WSL:**

```bash
free -h  # Total should be ~5.8GB
```

### Problem: Antigravity Restarts Too Often

**Increase MAX_MEMORY_MB:**

```bash
nano ~/scripts/antigravity_memory_guardian.sh
# Change MAX_MEMORY_MB=1500 to e.g. MAX_MEMORY_MB=2000
```

**Restart guardian:**

```bash
pkill -f antigravity_memory_guardian
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

### Problem: PC Still Freezes

**Possible causes:**

1. **Other programs consuming memory:**

```bash
# See top memory consumers
ps aux --sort=-%mem | head -20
```

2. **Disk problems (HDD on D:):**

```bash
# Check disk I/O
iostat -x 1 5
```

3. **Overheating:**

```bash
# Install sensors
sudo apt install lm-sensors
sensors
```

4. **RAM failure:**

Run Windows Memory Diagnostic:
- Windows + R → `mdsched.exe`

---

## References

### Antigravity Memory Issues (Documented)

1. **GitHub Gist - Virtual Memory Bloat:**
   - VSZ: 1.4TB per process (normal: 500MB-2GB)
   - VSZ/RSS ratio: 20,000:1 (normal: 2:1 to 10:1)
   - [Source](https://gist.github.com/ecast162/d41d18addf1307350092787e135b36df)

2. **PyRefly Issue #1661:**
   - PyRefly (language server) uses 7GB on macOS
   - Related to Antigravity
   - [Source](https://github.com/facebook/pyrefly/issues/1661)

3. **Medium - Linux RAM Fix:**
   - Antigravity eats ALL memory on Linux
   - Users must manually limit memory
   - [Source](https://aakash4dev.medium.com/stop-antigravity-from-eating-your-ram-a-simple-linux-fix-9e548d3a4481)

4. **AI 505 - macOS Performance Issues:**
   - AI agent diff overlays accumulate in RAM
   - GPU rendering issues
   - [Source](https://ai505.com/antigravity-on-macos-is-running-in-slow-motion-heres-the-fix/)

### WSL2 Memory Management

1. **Microsoft WSL Documentation:**
   - [WSL2 .wslconfig settings](https://docs.microsoft.com/en-us/windows/wsl/wsl-config)

2. **WSL Memory Reclaim:**
   - autoMemoryReclaim feature (WSL 2.0+)

---

## Installed Files Overview

### Windows Side

```
C:\Users\[username]\.wslconfig         # WSL memory limitation
```

### WSL Side

```
/home/rune/scripts/
├── antigravity_memory_guardian.sh       # Main monitoring (4.3K)
├── check_antigravity_memory.sh          # Quick check (2.0K)
└── emergency_cleanup.sh                 # Emergency stop (892 bytes)

/home/rune/
├── .bashrc                              # Aliases and auto-start
└── antigravity_guardian.log             # Log file (grows over time)
```

---

## System Overview After Installation

### Before

```
┌─────────────────────────────────────────┐
│           Physical RAM: 16GB            │
├─────────────────────────────────────────┤
│  Windows: 2-4GB                         │
│  WSL: Unlimited (can take 12GB+)       │
│    ├─ Antigravity: Grows uncontrolled  │
│    ├─ Trading programs: 2-3GB          │
│    └─ Other: 1-2GB                      │
└─────────────────────────────────────────┘
        ↓
   SYSTEM FREEZES
```

### After

```
┌─────────────────────────────────────────┐
│           Physical RAM: 16GB            │
├─────────────────────────────────────────┤
│  Windows: 4-6GB                         │
│  WSL: MAX 6GB (protected)              │
│    ├─ Antigravity: Max 1.5GB (monitored)│
│    ├─ Trading programs: 2-3GB          │
│    └─ Other: 1-2GB                      │
│  Free: 4-6GB (buffer)                  │
└─────────────────────────────────────────┘
        ↓
   STABLE SYSTEM ✓
```

---

## Tips for Future Use

### During Normal Operation

- System runs automatically
- No action required
- Guardian logs silently in background

### When Suspecting Issues

```bash
# Quick status
ag-check

# See recent logs
tail -20 ~/antigravity_guardian.log

# Full system status
free -h && df -h && ps aux --sort=-%mem | head -10
```

### In Critical Situations

```bash
# Emergency stop Antigravity
ag-emergency

# If WSL is completely stuck
# From PowerShell on Windows:
wsl --shutdown
```

### Optimization

**If Antigravity legitimately needs more memory:**

1. Increase MAX_MEMORY_MB in guardian
2. Optionally increase WSL memory in .wslconfig (but not over 8GB)
3. Monitor for a week to see effect

**If you want tighter control:**

1. Reduce MAX_MEMORY_MB to 1000MB
2. Reduce CHECK_INTERVAL_SECONDS to 15
3. Monitor logs for restart frequency

---

## Conclusion

### What We Achieved

✅ WSL can never use more than 6GB memory
✅ Antigravity restarts automatically at >1500MB usage
✅ Guardian starts automatically at WSL boot
✅ Complete logging of all activity
✅ Emergency cleanup available
✅ Quick status check available

### Expected Result

- **PC should no longer freeze** during normal use
- If memory issues occur, they are caught automatically
- System remains responsive even under high load
- Logs provide insight into memory usage over time

### Next Steps (Optional)

1. **Windows PowerShell Monitor:**
   - Alerts on Windows side
   - Toast notifications for high memory usage

2. **Memory Health Checker for trading programs:**
   - Integrate same type of monitoring in RobotTrader
   - Prevent memory leaks in Python code

3. **Automated reporting:**
   - Daily/weekly summary of memory usage
   - Alerts via email/Slack

---

## Support and Contact

When problems or questions arise:

1. **Check logs first:**
   ```bash
   tail -50 ~/antigravity_guardian.log
   ```

2. **Run diagnostics:**
   ```bash
   ag-check
   free -h
   ```

3. **Send information:**
   - Output from commands above
   - Description of problem
   - When problem occurred

---

**Documentation created:** January 7, 2026  
**Version:** 1.0  
**System:** Windows 11 + WSL2 (Ubuntu 24.04.3)  
**Status:** ✅ Installed and tested

---

## Appendix: Complete Command List

### For Copy-Paste During Reinstallation

**Windows side (.wslconfig):**
```ini
[wsl2]
memory=6GB
processors=4
swap=2GB
autoMemoryReclaim=gradual
lockedMemory=false
nestedVirtualization=false
```

**WSL restart:**
```powershell
wsl --shutdown
wsl
```

**Install scripts:**
```bash
cd ~
mkdir -p scripts
cd scripts

# Copy the three scripts (see details earlier)
# antigravity_memory_guardian.sh
# check_antigravity_memory.sh
# emergency_cleanup.sh

chmod +x *.sh
```

**Add aliases and auto-start:**
```bash
cat >> ~/.bashrc << 'EOF'

# Antigravity Memory Management
alias ag-guard="~/scripts/antigravity_memory_guardian.sh"
alias ag-check="~/scripts/check_antigravity_memory.sh"
alias ag-emergency="~/scripts/emergency_cleanup.sh"

# Start Antigravity Guardian automatically
if ! pgrep -f "antigravity_memory_guardian.sh" > /dev/null; then
    nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
fi
EOF

source ~/.bashrc
```

**Start guardian:**
```bash
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

**Verify:**
```bash
free -h
ps aux | grep antigravity_memory_guardian | grep -v grep
ag-check
```

✅ Done!

---

## Quick Reference Card

### Essential Commands

| What | Command |
|------|---------|
| Check memory status | `ag-check` |
| View live logs | `tail -f ~/antigravity_guardian.log` |
| Emergency cleanup | `ag-emergency` |
| Check if guardian runs | `ps aux \| grep antigravity_memory_guardian \| grep -v grep` |
| Restart guardian | `pkill -f antigravity_memory_guardian && nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &` |
| Check WSL memory | `free -h` |
| Check from Windows | `Get-Process VmmemWSL` in PowerShell |

### File Locations

| File | Location | Purpose |
|------|----------|---------|
| WSL config | `C:\Users\[user]\.wslconfig` | Limit WSL memory |
| Guardian script | `~/scripts/antigravity_memory_guardian.sh` | Main monitoring |
| Check script | `~/scripts/check_antigravity_memory.sh` | Quick status |
| Emergency script | `~/scripts/emergency_cleanup.sh` | Emergency stop |
| Log file | `~/antigravity_guardian.log` | All guardian logs |
| Aliases | `~/.bashrc` | Command shortcuts |

### Configuration Values

| Setting | Value | Location |
|---------|-------|----------|
| WSL max memory | 6GB | `.wslconfig` |
| WSL max swap | 2GB | `.wslconfig` |
| Antigravity max memory | 1500MB | `antigravity_memory_guardian.sh` |
| Check interval | 30s | `antigravity_memory_guardian.sh` |
| Restart cooldown | 300s (5min) | `antigravity_memory_guardian.sh` |

---

**End of Documentation**
