# AntiGravity Tilkoblingsproblemer - Feilsøkingsguide

## Problembeskrivelse

AntiGravity klarer ofte ikke å koble til agenten ved oppstart, og viser meldingen "One moment, the agent is currently loading..." på ubestemt tid. Flere omstarter eller til og med en full PC-omstart kan være nødvendig før agenten kobler til.

## Årsak

Problemet skyldes flere zombie-prosesser som fortsetter å kjøre etter at AntiGravity er lukket. Disse foreldreløse `Antigravity.exe`-prosessene (ofte 10-15+ instanser) forhindrer ren oppstart ved å:

- Blokkere porter og nettverks-sockets
- Forårsake ressurskonflikter
- Forstyrre korrekt agent-initialisering

## Løsning: Oppstartsskript med Opprydding

Lag et oppstartsskript som dreper alle eksisterende AntiGravity-prosesser før det starter en ny instans.

### Steg 1: Opprett Batch-skriptet

Lagre følgende som `AntigravityLauncher.bat`:

```batch
@echo off
echo ================================================
echo  AntiGravity Opprydding og Omstart
echo ================================================
echo.

echo [1/3] Avslutter alle Antigravity-prosesser...
taskkill /F /IM "Antigravity.exe" /T 2>nul
if %errorlevel% equ 0 (
    echo Antigravity-prosesser ble avsluttet
) else (
    echo Ingen Antigravity-prosesser kjører
)

echo.
echo [2/3] Venter på systemopprydding...
timeout /t 5 /nobreak >nul
echo Opprydding fullført.

echo.
echo [3/3] Starter Antigravity...
start "" "D:\Antigravity\Antigravity.exe"

echo.
echo ================================================
echo  Skript fullført! Antigravity starter nå...
echo ================================================
timeout /t 3
exit
```

**Viktig:** Juster stien `D:\Antigravity\Antigravity.exe` til din faktiske installasjonsplassering.

### Steg 2: Opprett en Skrivebords-snarvei

1. Høyreklikk på `AntigravityLauncher.bat`-filen
2. Velg **"Opprett snarvei"**
3. Flytt snarveien til skrivebordet
4. Høyreklikk på snarveien → **Egenskaper**
5. Klikk **"Avansert..."** → Huk av **"Kjør som administrator"** → Klikk OK
6. Klikk **"Endre ikon..."** → Bla til din `Antigravity.exe`-fil for å bruke det offisielle ikonet
7. I **"Kjør:"**-rullegardinmenyen, velg **"Minimert"** (skjuler kommandovinduet)
8. Klikk **OK** for å lagre

### Steg 3: Bruk den Nye Oppstarteren

**Bruk alltid denne snarveien til å starte AntiGravity** i stedet for den originale kjørbare filen. Dette sikrer:

- Alle zombie-prosesser blir drept før oppstart
- Nettverks-sockets blir frigjort ordentlig
- Hver oppstart begynner med ren systemtilstand

## Hvordan Det Fungerer

1. **Prosessopprydding**: Tvangslukker alle `Antigravity.exe`-prosesser
2. **Venteperiode**: 5 sekunders forsinkelse lar Windows frigjøre ressurser
3. **Ny Start**: Starter en ny AntiGravity-instans med ren tilstand

## Forventede Resultater

- Betydelig reduserte tilkoblingsfeil
- Raskere agent-initialisering
- Mer pålitelig oppstartsopplevelse
- Ikke behov for flere omstarter eller PC-omstart

## Ytterligere Feilsøking

Hvis du fortsatt opplever problemer etter å ha brukt oppryddingsskriptet:

1. **Sjekk Internettforbindelse**: Sørg for stabil tilkobling ved oppstart
2. **Brannmurinnstillinger**: Verifiser at AntiGravity er tillatt gjennom Windows Brannmur
3. **Overvåk Prosessantall**: Åpne Oppgavebehandling etter at du lukker AntiGravity for å verifisere at alle prosesser avsluttes
4. **Rapporter til Google**: Bruk AntiGravitys tilbakemeldingsmekanisme for å rapportere vedvarende problemer

## Tekniske Detaljer

**Bekreftet Fungerende Miljø:**
- Installasjonssti: `D:\Antigravity\Antigravity.exe`
- OS: Windows 10/11
- Flere zombie-prosesser observert (13+ instanser)
- Ingen ekstern Node.js-avhengighet påkrevd

**Loggindikatorer på Vellykket Oppstart:**
```
AntigravityAuthMainService initialized
Browser onboarding server started on http://localhost:59578
Monitoring server started on http://localhost:9101/antigravity
```

---

*Sist Oppdatert: Januar 2026*
