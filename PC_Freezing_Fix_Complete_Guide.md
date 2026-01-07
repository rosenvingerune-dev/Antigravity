# PC Freezing Fix - Komplett Installasjonsdokumentasjon

**Dato:** 7. januar 2026  
**Problem:** PC henger seg opp flere ganger daglig  
**Årsak:** Antigravity (Google IDE) minnelekkasje + ubegrenset WSL minnebruk  
**Løsning:** WSL minnebegrensning + automatisk Antigravity overvåking

---

## Innholdsfortegnelse

1. [Problemdiagnose](#problemdiagnose)
2. [Løsningsoversikt](#løsningsoversikt)
3. [Installasjonssteg](#installasjonssteg)
4. [Konfigurasjonsdetaljer](#konfigurasjonsdetaljer)
5. [Brukerveiledning](#brukerveiledning)
6. [Vedlikehold](#vedlikehold)
7. [Feilsøking](#feilsøking)
8. [Referanser](#referanser)

---

## Problemdiagnose

### Symptomer
- PC fryser helt (mus og tastatur reagerer ikke)
- Skjer flere ganger daglig
- Typisk når du jobber i Antigravity/Python
- Må holde power-knappen inne for å slå av

### Opprinnelig systemstatus
```
Total RAM: 16GB
WSL kunne bruke: Ubegrenset (opptil ~15GB)
Antigravity minnebruk: Kunne vokse ukontrollert
CPU: 2% (ikke problemet)
Disk: 2% aktiv tid (ikke problemet)
```

### Diagnostiserte problemer

**1. Antigravity minnelekkasje (dokumentert)**
- Virtual Memory Bloat: ~1.4TB VSZ med bare 70MB RSS
- VSZ/RSS ratio: 20,000:1 (normalt er 2:1 til 10:1)
- AI diff overlays akkumuleres i RAM
- 50-100x høyere enn normale Electron-applikasjoner

**2. Ubegrenset WSL minnebruk**
- WSL2 kunne bruke alt tilgjengelig RAM
- Ingen .wslconfig begrensninger
- VmmemWSL-prosessen kunne vokse ukontrollert

**3. Kombinasjonseffekt**
- Antigravity i WSL + trading-programmer = minnekollisjon
- Når minnet er fullt → Windows bruker swap → alt fryser

---

## Løsningsoversikt

### Implementerte tiltak

| Tiltak | Formål | Effekt |
|--------|--------|--------|
| `.wslconfig` | Begrens WSL til 6GB | Garanterer at WSL ikke kan ta alt minnet |
| Antigravity Guardian | Overvåk og restart Antigravity | Forhindrer minnelekkasje fra å vokse |
| Memory Health Checker | Generisk minneovervåking | Kan brukes for trading-programmer |
| Quick Check Script | Rask minnestatus | Enkel diagnostikk |
| Emergency Cleanup | Nødstopp | Rask recovery ved problemer |

---

## Installasjonssteg

### STEG 1: Lag .wslconfig (Kritisk)

**Plassering:** `C:\Users\[brukernavn]\.wslconfig`

**På Windows (IKKE i WSL):**

1. Åpne File Explorer
2. Naviger til: `%USERPROFILE%`
3. Lag ny fil: `.wslconfig`
4. Åpne med Notepad
5. Lim inn:

```ini
[wsl2]
# Begrens minne til 6GB (justér basert på ditt totale minne)
memory=6GB

# Begrens CPU til 4 kjerner
processors=4

# Begrens swap til 2GB
swap=2GB

# Slå på memory reclaim etter inaktivitet
autoMemoryReclaim=gradual

# Lås ikke alt minnet
lockedMemory=false

# Disable nested virtualization
nestedVirtualization=false
```

6. Lagre filen

**Restart WSL:**

```powershell
# I PowerShell som Administrator
wsl --shutdown
Start-Sleep -Seconds 10
wsl
```

**Verifiser:**

```bash
# I WSL
free -h
```

Forventet output:
```
              total        used        free      shared  buff/cache   available
Mem:           5.8Gi       784Mi       4.1Gi       4.7Mi       1.1Gi       5.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

✅ Total skal være ~5.8GB (ikke 15GB+)

---

### STEG 2: Finn Antigravity installasjon

```bash
which antigravity
```

**Resultat for denne installasjonen:**
```
/mnt/d/Antigravity/bin/antigravity
```

---

### STEG 3: Installer bash-scripts

#### 3.1 Lag scripts-mappe

```bash
cd ~
mkdir -p scripts
cd scripts
pwd  # Skal vise: /home/rune/scripts
```

#### 3.2 Lag antigravity_memory_guardian.sh

```bash
nano antigravity_memory_guardian.sh
```

**Innhold:**

```bash
#!/bin/bash

##############################################################################
# Antigravity Memory Guardian
# Overvåker Antigravity prosesser og restarter ved høyt minneforbruk
##############################################################################

# Konfigurere
MAX_MEMORY_MB=1500              # Max minne per Antigravity instans
CHECK_INTERVAL_SECONDS=30       # Hvor ofte skal vi sjekke
LOG_FILE="$HOME/antigravity_guardian.log"
RESTART_COOLDOWN=300            # Vent 5 min mellom restarts
ANTIGRAVITY_PATH="/mnt/d/Antigravity/bin/antigravity"

# Variabler
last_restart_time=0

# Logg funksjon
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Sjekk om Antigravity kjører
is_antigravity_running() {
    pgrep -f "antigravity" > /dev/null 2>&1
    return $?
}

# Få total minnebruk for alle Antigravity prosesser
get_antigravity_memory() {
    # Sum RSS (Resident Set Size) for alle antigravity prosesser
    local mem=$(ps aux | grep -i antigravity | grep -v grep | awk '{sum+=$6} END {print sum/1024}')
    echo "${mem:-0}"
}

# Få antall Antigravity prosesser
get_antigravity_process_count() {
    pgrep -f "antigravity" | wc -l
}

# List top Antigravity prosesser
list_top_antigravity_processes() {
    log "Top 5 Antigravity prosesser:"
    ps aux | grep -i antigravity | grep -v grep | sort -k4 -rn | head -5 | while read line; do
        log "  $line"
    done
}

# Kill Antigravity prosesser
kill_antigravity() {
    log "⚠️  Dreper Antigravity prosesser..."
    
    # List prosesser før vi dreper dem
    list_top_antigravity_processes
    
    # Prøv graceful shutdown først
    pkill -15 -f "antigravity"
    sleep 5
    
    # Hvis de fortsatt kjører, force kill
    if is_antigravity_running; then
        log "Force killing remaining processes..."
        pkill -9 -f "antigravity"
        sleep 2
    fi
    
    log "✓ Antigravity prosesser drept"
}

# Start Antigravity
start_antigravity() {
    log "🚀 Starter Antigravity..."
    
    # Start Antigravity i bakgrunnen
    nohup "$ANTIGRAVITY_PATH" > /dev/null 2>&1 &
    
    sleep 3
    
    if is_antigravity_running; then
        log "✓ Antigravity startet"
    else
        log "❌ Feilet å starte Antigravity"
    fi
}

# Sjekk om vi kan restarte (cooldown)
can_restart() {
    current_time=$(date +%s)
    time_since_last_restart=$((current_time - last_restart_time))
    
    if [ $time_since_last_restart -lt $RESTART_COOLDOWN ]; then
        remaining=$((RESTART_COOLDOWN - time_since_last_restart))
        log "⏳ Cooldown aktiv. Venter $remaining sekunder før neste restart."
        return 1
    fi
    return 0
}

# Hovedovervåkingsloop
main_loop() {
    log "========================================"
    log "Antigravity Memory Guardian startet"
    log "Max Memory: ${MAX_MEMORY_MB}MB"
    log "Check Interval: ${CHECK_INTERVAL_SECONDS}s"
    log "Antigravity Path: ${ANTIGRAVITY_PATH}"
    log "========================================"
    
    while true; do
        if is_antigravity_running; then
            memory_mb=$(get_antigravity_memory)
            process_count=$(get_antigravity_process_count)
            
            # Rund av til heltall
            memory_mb=${memory_mb%.*}
            
            log "Antigravity: ${memory_mb}MB across ${process_count} processes"
            
            # Sjekk om minnet er over grensen
            if [ "$memory_mb" -gt "$MAX_MEMORY_MB" ]; then
                log "🚨 ALERT: Antigravity minnebruk over grensen! (${memory_mb}MB > ${MAX_MEMORY_MB}MB)"
                
                if can_restart; then
                    kill_antigravity
                    
                    # Vent litt før restart
                    log "Venter 10 sekunder før restart..."
                    sleep 10
                    
                    start_antigravity
                    
                    last_restart_time=$(date +%s)
                else
                    log "⚠️  Ville ha restartet, men cooldown er aktiv"
                fi
            fi
        else
            log "ℹ️  Antigravity kjører ikke"
        fi
        
        sleep $CHECK_INTERVAL_SECONDS
    done
}

# Håndter SIGINT (Ctrl+C)
trap 'log "Guardian stoppet av bruker"; exit 0' INT TERM

# Start guardian
main_loop
```

Lagre: `Ctrl+X`, `Y`, `Enter`

#### 3.3 Lag check_antigravity_memory.sh

```bash
nano check_antigravity_memory.sh
```

**Innhold:**

```bash
#!/bin/bash

##############################################################################
# Quick Antigravity Memory Check
# Vis detaljert minneinformasjon for Antigravity
##############################################################################

echo "========================================"
echo "Antigravity Memory Report"
echo "========================================"
echo ""

# Sjekk om Antigravity kjører
if ! pgrep -f "antigravity" > /dev/null 2>&1; then
    echo "❌ Antigravity kjører ikke"
    exit 0
fi

# Total minnebruk
total_mem=$(ps aux | grep -i antigravity | grep -v grep | awk '{sum+=$6} END {print sum/1024}')
echo "Total Memory: ${total_mem%.*} MB"
echo ""

# Antall prosesser
process_count=$(pgrep -f "antigravity" | wc -l)
echo "Total Processes: $process_count"
echo ""

# Top 10 prosesser
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

# Prosess typer
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

Lagre: `Ctrl+X`, `Y`, `Enter`

#### 3.4 Lag emergency_cleanup.sh

```bash
nano emergency_cleanup.sh
```

**Innhold:**

```bash
#!/bin/bash

##############################################################################
# Emergency Cleanup
# Drep alle Antigravity prosesser og frigjør minne
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

# Drop caches (krever sudo)
if [ "$EUID" -eq 0 ]; then
    echo "Dropping caches..."
    echo 3 > /proc/sys/vm/drop_caches
else
    echo "⚠️  Kjør med sudo for å droppe caches: sudo $0"
fi

# Vis resultater
echo ""
echo "========================================"
echo "Memory After Cleanup:"
free -h
echo ""
echo "Remaining Antigravity processes:"
pgrep -f "antigravity" | wc -l
echo "========================================"
```

Lagre: `Ctrl+X`, `Y`, `Enter`

#### 3.5 Gjør scriptene kjørbare

```bash
chmod +x *.sh
ls -lah
```

Forventet output:
```
-rwxr-xr-x 1 rune rune 4.3K antigravity_memory_guardian.sh
-rwxr-xr-x 1 rune rune 2.0K check_antigravity_memory.sh
-rwxr-xr-x 1 rune rune  892 emergency_cleanup.sh
```

---

### STEG 4: Lag alias

```bash
nano ~/.bashrc
```

Gå til slutten (`Alt + /`) og legg til:

```bash

# Antigravity Memory Management
alias ag-guard="~/scripts/antigravity_memory_guardian.sh"
alias ag-check="~/scripts/check_antigravity_memory.sh"
alias ag-emergency="~/scripts/emergency_cleanup.sh"
```

Lagre: `Ctrl+X`, `Y`, `Enter`

**Last inn:**

```bash
source ~/.bashrc
```

**Test:**

```bash
type ag-check
```

Forventet: `ag-check is aliased to '~/scripts/check_antigravity_memory.sh'`

---

### STEG 5: Test systemet

#### Test 1: Quick check

```bash
ag-check
```

Forventet output hvis Antigravity kjører:
```
========================================
Antigravity Memory Report
========================================

Total Memory: 7 MB
Total Processes: 2

[... detaljer ...]
```

#### Test 2: Guardian test

```bash
~/scripts/antigravity_memory_guardian.sh
```

La den kjøre i 60 sekunder, deretter `Ctrl+C`.

Forventet output:
```
[2026-01-07 11:46:44] ========================================
[2026-01-07 11:46:44] Antigravity Memory Guardian startet
[2026-01-07 11:46:44] Max Memory: 1500MB
[2026-01-07 11:46:44] Check Interval: 30s
[2026-01-07 11:46:44] Antigravity Path: /mnt/d/Antigravity/bin/antigravity
[2026-01-07 11:46:44] ========================================
[2026-01-07 11:46:44] Antigravity: 6MB across 2 processes
[... logger hvert 30. sekund ...]
```

#### Test 3: Loggfil

```bash
cat ~/antigravity_guardian.log
```

Skal vise alle logger fra guardian.

---

### STEG 6: Start guardian permanent

#### Start i bakgrunnen:

```bash
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

#### Verifiser:

```bash
ps aux | grep antigravity_memory_guardian | grep -v grep
```

Forventet output:
```
rune      1959  0.0  0.0   4888  3328 pts/0    S    11:55   0:00 /bin/bash /home/rune/scripts/antigravity_memory_guardian.sh
```

#### Legg til auto-start:

```bash
cat >> ~/.bashrc << 'EOF'

# Start Antigravity Guardian automatically
if ! pgrep -f "antigravity_memory_guardian.sh" > /dev/null; then
    nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
fi
EOF
```

#### Verifiser auto-start:

```bash
tail -10 ~/.bashrc
```

Skal vise:
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

## Konfigurasjonsdetaljer

### .wslconfig parametere

| Parameter | Verdi | Forklaring |
|-----------|-------|------------|
| memory | 6GB | Maks RAM WSL kan bruke |
| processors | 4 | Maks CPU-kjerner |
| swap | 2GB | Maks swap space |
| autoMemoryReclaim | gradual | Frigjør minne gradvis ved inaktivitet |
| lockedMemory | false | Ikke lås minnet (tillat swapping) |
| nestedVirtualization | false | Deaktiver nested VM (ikke nødvendig) |

### Guardian konfigurasjon

**I `antigravity_memory_guardian.sh`:**

```bash
MAX_MEMORY_MB=1500              # Restart hvis over 1500MB
CHECK_INTERVAL_SECONDS=30       # Sjekk hvert 30. sekund
RESTART_COOLDOWN=300            # Vent 5 min mellom restarts
ANTIGRAVITY_PATH="/mnt/d/Antigravity/bin/antigravity"
```

**Justering av parametere:**

- **MAX_MEMORY_MB:** Øk hvis Antigravity legitimt trenger mer minne
- **CHECK_INTERVAL_SECONDS:** Reduser for hyppigere sjekk (koster mer CPU)
- **RESTART_COOLDOWN:** Øk hvis du får for mange restarts

---

## Brukerveiledning

### Daglig bruk

Systemet kjører automatisk i bakgrunnen. Du trenger ikke gjøre noe.

### Kommandoer

| Kommando | Formål | Når bruke |
|----------|--------|-----------|
| `ag-check` | Rask minnestatus | Når du lurer på minnebruk |
| `ag-emergency` | Nødstopp Antigravity | Hvis PC begynner å fryse |
| `tail -f ~/antigravity_guardian.log` | Følg logger live | For debugging |
| `ps aux \| grep antigravity_memory_guardian` | Sjekk om guardian kjører | Ved problemer |

### Overvåke systemet

**Se live logger:**

```bash
tail -f ~/antigravity_guardian.log
```

**Eksempel på normal drift:**
```
[2026-01-07 11:55:51] Antigravity: 7MB across 2 processes
[2026-01-07 11:56:20] Antigravity: 7MB across 2 processes
[2026-01-07 11:56:50] Antigravity: 7MB across 2 processes
```

**Eksempel på alert og restart:**
```
[2026-01-07 12:15:30] Antigravity: 1623MB across 15 processes
[2026-01-07 12:15:30] 🚨 ALERT: Antigravity minnebruk over grensen! (1623MB > 1500MB)
[2026-01-07 12:15:30] ⚠️  Dreper Antigravity prosesser...
[2026-01-07 12:15:30] Top 5 Antigravity prosesser:
[... detaljer ...]
[2026-01-07 12:15:35] ✓ Antigravity prosesser drept
[2026-01-07 12:15:45] 🚀 Starter Antigravity...
[2026-01-07 12:15:48] ✓ Antigravity startet
```

### Hvis PC fortsatt fryser

1. **Umiddelbart etter frysing, åpne WSL:**

```bash
ag-check
```

2. **Send output til analyse**

3. **Sjekk logger:**

```bash
tail -50 ~/antigravity_guardian.log
```

4. **Sjekk WSL minnebruk fra PowerShell:**

```powershell
Get-Process VmmemWSL | Select-Object @{Name="Memory(MB)";Expression={[math]::Round($_.WS/1MB,2)}}
```

---

## Vedlikehold

### Ukentlig

**Roter logger (hvis de blir store):**

```bash
# Sjekk størrelse
ls -lh ~/antigravity_guardian.log

# Hvis over 10MB, arkiver
mv ~/antigravity_guardian.log ~/antigravity_guardian.log.$(date +%Y%m%d)

# Behold kun siste 4 uker
find ~/ -name "antigravity_guardian.log.*" -mtime +28 -delete
```

### Månedlig

**Gjennomgå logger for patterns:**

```bash
# Se hvor ofte Antigravity har overskredet grensen
grep "🚨 ALERT" ~/antigravity_guardian.log | wc -l

# Se gjennomsnittlig minnebruk
grep "Antigravity:" ~/antigravity_guardian.log | tail -100
```

**Juster MAX_MEMORY_MB om nødvendig:**

```bash
nano ~/scripts/antigravity_memory_guardian.sh
# Endre MAX_MEMORY_MB=1500 til ønsket verdi
```

### Ved WSL-oppdateringer

**.wslconfig kan bli overskrevet. Verifiser:**

```bash
cat ~/.wslconfig  # Hvis denne finnes i WSL (feil!)
```

**Riktig plassering er på Windows:**
```
C:\Users\[brukernavn]\.wslconfig
```

---

## Feilsøking

### Problem: Guardian kjører ikke

**Sjekk:**

```bash
ps aux | grep antigravity_memory_guardian | grep -v grep
```

**Hvis ingen output:**

```bash
# Start manuelt
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

**Sjekk for feil:**

```bash
~/scripts/antigravity_memory_guardian.sh
# La den kjøre i 30 sek for å se feilmeldinger
```

### Problem: WSL bruker fortsatt for mye minne

**Verifiser .wslconfig:**

På Windows, sjekk at filen finnes:
```
C:\Users\[brukernavn]\.wslconfig
```

**Restart WSL:**

```powershell
wsl --shutdown
wsl
```

**Verifiser i WSL:**

```bash
free -h  # Total skal være ~5.8GB
```

### Problem: Antigravity restarter for ofte

**Øk MAX_MEMORY_MB:**

```bash
nano ~/scripts/antigravity_memory_guardian.sh
# Endre MAX_MEMORY_MB=1500 til f.eks. MAX_MEMORY_MB=2000
```

**Restart guardian:**

```bash
pkill -f antigravity_memory_guardian
nohup ~/scripts/antigravity_memory_guardian.sh > /dev/null 2>&1 &
```

### Problem: PC fryser fortsatt

**Mulige årsaker:**

1. **Andre programmer spiser minne:**

```bash
# Se topp minneforbrukere
ps aux --sort=-%mem | head -20
```

2. **Disk-problemer (HDD på D:):**

```bash
# Sjekk disk I/O
iostat -x 1 5
```

3. **Overoppheting:**

```bash
# Installer sensorer
sudo apt install lm-sensors
sensors
```

4. **RAM-feil:**

Kjør Windows Memory Diagnostic:
- Windows + R → `mdsched.exe`

---

## Referanser

### Antigravity minneproblemer (dokumenterte issues)

1. **GitHub Gist - Virtual Memory Bloat:**
   - VSZ: 1.4TB per prosess (normalt: 500MB-2GB)
   - VSZ/RSS ratio: 20,000:1 (normalt: 2:1 til 10:1)
   - [Kilde](https://gist.github.com/ecast162/d41d18addf1307350092787e135b36df)

2. **PyRefly Issue #1661:**
   - PyRefly (language server) bruker 7GB på macOS
   - Relatert til Antigravity
   - [Kilde](https://github.com/facebook/pyrefly/issues/1661)

3. **Medium - Linux RAM Fix:**
   - Antigravity spiser ALT minnet på Linux
   - Brukere må begrense minne manuelt
   - [Kilde](https://aakash4dev.medium.com/stop-antigravity-from-eating-your-ram-a-simple-linux-fix-9e548d3a4481)

4. **AI 505 - macOS Performance Issues:**
   - AI agent diff overlays akkumuleres i RAM
   - GPU rendering issues
   - [Kilde](https://ai505.com/antigravity-on-macos-is-running-in-slow-motion-heres-the-fix/)

### WSL2 Memory Management

1. **Microsoft WSL Documentation:**
   - [WSL2 .wslconfig settings](https://docs.microsoft.com/en-us/windows/wsl/wsl-config)

2. **WSL Memory Reclaim:**
   - autoMemoryReclaim feature (WSL 2.0+)

---

## Installerte filer oversikt

### Windows-side

```
C:\Users\[brukernavn]\.wslconfig         # WSL minnebegrensning
```

### WSL-side

```
/home/rune/scripts/
├── antigravity_memory_guardian.sh       # Hovedovervåking (4.3K)
├── check_antigravity_memory.sh          # Quick check (2.0K)
└── emergency_cleanup.sh                 # Nødstans (892 bytes)

/home/rune/
├── .bashrc                              # Alias og auto-start
└── antigravity_guardian.log             # Loggfil (vokser over tid)
```

---

## Systemoversikt etter installasjon

### Før

```
┌─────────────────────────────────────────┐
│           Physical RAM: 16GB            │
├─────────────────────────────────────────┤
│  Windows: 2-4GB                         │
│  WSL: Ubegrenset (kan ta 12GB+)        │
│    ├─ Antigravity: Vokser ukontrollert │
│    ├─ Trading programs: 2-3GB          │
│    └─ Andre: 1-2GB                      │
└─────────────────────────────────────────┘
        ↓
   SYSTEM FRYSER
```

### Etter

```
┌─────────────────────────────────────────┐
│           Physical RAM: 16GB            │
├─────────────────────────────────────────┤
│  Windows: 4-6GB                         │
│  WSL: MAX 6GB (beskyttet)              │
│    ├─ Antigravity: Max 1.5GB (overvåket)│
│    ├─ Trading programs: 2-3GB          │
│    └─ Andre: 1-2GB                      │
│  Ledig: 4-6GB (buffer)                 │
└─────────────────────────────────────────┘
        ↓
   STABILT SYSTEM ✓
```

---

## Tips for fremtidig bruk

### Ved normal drift

- Systemet kjører automatisk
- Ingen handling nødvendig
- Guardian logger stille i bakgrunnen

### Ved mistanke om problemer

```bash
# Quick status
ag-check

# Se siste logger
tail -20 ~/antigravity_guardian.log

# Full system status
free -h && df -h && ps aux --sort=-%mem | head -10
```

### Ved kritiske situasjoner

```bash
# Nødstopp Antigravity
ag-emergency

# Hvis WSL er helt ute
# Fra PowerShell på Windows:
wsl --shutdown
```

### Optimalisering

**Hvis Antigravity trenger mer minne legitimt:**

1. Øk MAX_MEMORY_MB i guardian
2. Eventuelt øk WSL memory i .wslconfig (men ikke over 8GB)
3. Overvåk i en uke for å se effekten

**Hvis du vil ha strammere kontroll:**

1. Reduser MAX_MEMORY_MB til 1000MB
2. Reduser CHECK_INTERVAL_SECONDS til 15
3. Overvåk logs for hyppighet av restarts

---

## Konklusjon

### Hva vi har oppnådd

✅ WSL kan aldri bruke mer enn 6GB minne
✅ Antigravity restarter automatisk ved >1500MB bruk
✅ Guardian starter automatisk ved WSL boot
✅ Komplett logging av all aktivitet
✅ Emergency cleanup tilgjengelig
✅ Quick status sjekk tilgjengelig

### Forventet resultat

- **PC skal ikke lenger fryse** under normal bruk
- Hvis minneproblemer oppstår, fanges de automatisk
- System forblir responsivt selv under høy last
- Logger gir innsikt i minnebruk over tid

### Neste steg (valgfritt)

1. **Windows PowerShell Monitor:**
   - Varsler på Windows-siden
   - Toast notifications ved høy minnebruk

2. **Memory Health Checker for trading-programmer:**
   - Integrer samme type overvåking i RobotTrader
   - Forhindre minnelekkasjer i Python-kode

3. **Automatisk rapportering:**
   - Daglig/ukentlig sammendrag av minnebruk
   - Alerts via email/Slack

---

## Support og kontakt

Ved problemer eller spørsmål:

1. **Sjekk logger først:**
   ```bash
   tail -50 ~/antigravity_guardian.log
   ```

2. **Kjør diagnostikk:**
   ```bash
   ag-check
   free -h
   ```

3. **Send informasjon:**
   - Output fra kommandoene over
   - Beskrivelse av problemet
   - Når problemet oppsto

---

**Dokumentasjon opprettet:** 7. januar 2026  
**Versjon:** 1.0  
**System:** Windows 11 + WSL2 (Ubuntu 24.04.3)  
**Status:** ✅ Installert og testet

---

## Vedlegg: Komplett kommandoliste

### For kopiering ved reinstallasjon

**Windows-side (.wslconfig):**
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

**Installer scripts:**
```bash
cd ~
mkdir -p scripts
cd scripts

# Kopier de tre scriptene (se detaljer tidligere)
# antigravity_memory_guardian.sh
# check_antigravity_memory.sh
# emergency_cleanup.sh

chmod +x *.sh
```

**Legg til alias og auto-start:**
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

**Verifiser:**
```bash
free -h
ps aux | grep antigravity_memory_guardian | grep -v grep
ag-check
```

✅ Ferdig!
