# AntiGravity Connection Issues - Troubleshooting Guide

## Problem Description

AntiGravity often fails to connect to the agent on startup, displaying the message "One moment, the agent is currently loading..." indefinitely. Multiple restarts or even a full PC reboot may be required before the agent connects successfully.

## Root Cause

The issue stems from multiple zombie processes that remain running after AntiGravity is closed. These orphaned `Antigravity.exe` processes (often 10-15+ instances) prevent clean startup by:

- Blocking ports and network sockets
- Causing resource conflicts
- Interfering with proper agent initialization

## Solution: Clean Startup Script

Create a launcher script that kills all existing AntiGravity processes before starting a fresh instance.

### Step 1: Create the Batch Script

Save the following as `AntigravityLauncher.bat`:

```batch
@echo off
echo ================================================
echo  AntiGravity Cleanup and Restart Script
echo ================================================
echo.

echo [1/3] Terminating all Antigravity processes...
taskkill /F /IM "Antigravity.exe" /T 2>nul
if %errorlevel% equ 0 (
    echo Successfully killed Antigravity processes
) else (
    echo No Antigravity processes found running
)

echo.
echo [2/3] Waiting for system cleanup...
timeout /t 5 /nobreak >nul
echo Cleanup complete.

echo.
echo [3/3] Starting Antigravity...
start "" "D:\Antigravity\Antigravity.exe"

echo.
echo ================================================
echo  Script complete! Antigravity should be starting...
echo ================================================
timeout /t 3
exit
```

**Important:** Adjust the path `D:\Antigravity\Antigravity.exe` to match your actual installation location.

### Step 2: Create a Desktop Shortcut

1. Right-click the `AntigravityLauncher.bat` file
2. Select **"Create shortcut"**
3. Move the shortcut to your Desktop
4. Right-click the shortcut → **Properties**
5. Click **"Advanced..."** → Check **"Run as administrator"** → Click OK
6. Click **"Change Icon..."** → Browse to your `Antigravity.exe` file to use the official icon
7. In the **"Run:"** dropdown, select **"Minimized"** (hides the command window)
8. Click **OK** to save

### Step 3: Use the New Launcher

**Always use this shortcut to launch AntiGravity** instead of the original executable. This ensures:

- All zombie processes are killed before startup
- Network sockets are properly released
- Each launch starts with a clean system state

## How It Works

1. **Process Cleanup**: Forces termination of all `Antigravity.exe` processes
2. **Wait Period**: 5-second delay allows Windows to release resources
3. **Fresh Start**: Launches a new AntiGravity instance with clean state

## Expected Results

- Significantly reduced connection failures
- Faster agent initialization
- More reliable startup experience
- No need for multiple restarts or PC reboots

## Additional Troubleshooting

If you continue to experience issues after using the cleanup script:

1. **Check Internet Connection**: Ensure stable connectivity when launching
2. **Firewall Settings**: Verify AntiGravity is allowed through Windows Firewall
3. **Monitor Process Count**: Open Task Manager after closing AntiGravity to verify all processes terminate
4. **Report to Google**: Use AntiGravity's feedback mechanism to report persistent issues

## Technical Details

**Confirmed Working Environment:**
- Installation Path: `D:\Antigravity\Antigravity.exe`
- OS: Windows 10/11
- Multiple zombie processes observed (13+ instances)
- No external Node.js dependency required

**Log Indicators of Successful Startup:**
```
AntigravityAuthMainService initialized
Browser onboarding server started on http://localhost:59578
Monitoring server started on http://localhost:9101/antigravity
```

---

*Last Updated: January 2026*
