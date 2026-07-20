# Zeiterfassung – Hubler Metallbau AG

## Übersicht
Single-File SPA (Vanilla JS) zur Arbeitszeiterfassung für ~8 Mitarbeiter. Deployment via GitHub Pages direkt aus `index.html`.

## Tech-Stack
- **Frontend:** Vanilla JS, CSS Custom Properties (Dark/Light Mode), kein Build-Tool
- **Auth:** MSAL.js 2.38.3 (Auth Code Flow + PKCE), Azure AD Tenant `bf52de6b-9185-46fb-998d-989d583b27a2`, Client-ID `19ae5266-ce31-4758-80e5-9cb87be67dfa`
- **Backend:** SharePoint Online via Microsoft Graph API – JSON-Dateien in Dokumentbibliothek
- **SharePoint Site:** `hublermetallbau.sharepoint.com/sites/HublerMetallbauDatenablage`
- **Pfad:** `/Shared Documents/Zeiterfassung/`

## Dateistruktur auf SharePoint
```
/Zeiterfassung/
  projekte.json              # 213 Bexio-Projekte mit {id, name, aktiv, produktiv}
  employees.json             # Mitarbeiterliste
  emp_settings.json          # MA-Einstellungen (Pensum, Ferien, Raucher, etc.)
  Mitarbeiter/
    Vorname_Nachname_2026.json   # Zeiteinträge pro MA/Jahr
  Abwesenheiten/
    alle.json                # Alle Absenzen (Soft-Delete mit _deleted)
  betriebsferien.json
  feiertage.json
  overtime_payouts.json      # Überstunden-Auszahlungen
  timer_HM-001.json          # Timer-State pro MA
  notif_settings.json        # Benachrichtigungs-Settings
```

## Architektur (alles in index.html)
Die gesamte App ist eine einzige `index.html` (~340KB). Struktur:
1. **`<head>`** – MSAL-Script (CDN), Google Fonts (DM Sans, JetBrains Mono)
2. **`<style>`** – ~20KB CSS mit Custom Properties
3. **`<body>`** – HTML Templates für alle Tabs/Modals
4. **`<script>`** – ~250KB JS:
   - Zeile ~1-240: `PROJECTS_DATA[]` (hartcodiert aus Bexio)
   - Zeile ~240-320: Globale Variablen, MSAL-Config, Helper-Funktionen
   - Zeile ~320-460: SP-Objekt (Graph API Wrapper mit `getToken()`, `gf()`, `readFile()`, `writeFile()`, `fullSync()`)
   - Zeile ~460-620: Auth (MSAL login/logout, Email-Matching mit 7-stufiger Gast-Konto-Logik)
   - Zeile ~620-1100: Timer, Pausen-Dialog, Segment-Splitting
   - Zeile ~1100-1500: Kalender, Einträge, Edit-Modal
   - Zeile ~1500-1800: Abwesenheiten, Sync (Read-Merge-Write mit Soft-Delete)
   - Zeile ~1800-2100: Report-Tab (Wochen-Report, Export HTML/CSV)
   - Zeile ~2100-2400: Monatsrapport (Lohnabrechnung), Produktivitäts-Report
   - Zeile ~2400-2800: Admin (MA-Verwaltung, Projekte, Feiertage, Betriebsferien, Vorträge)
   - Zeile ~2800-3000: MSAL Tab-Focus-Sync, Health-Check, Offline-Banner
   - Zeile ~3000+: Abwesenheitskalender, Dashboard, Überstundenkonto

## Wichtige Patterns

### Sync-Architektur (Read-Merge-Write)
```
1. Read: SP-Daten laden
2. Merge: Timestamp-basiert (lastModified) – neuere Version gewinnt
3. Write: Zusammengeführte Daten zurück auf SP
```
- `getRawEntries()` / `getRawAbsences()`: Inkl. `_deleted`-Einträge (für Merge)
- `getAllEntries()` / `getAllAbsences()`: Gefiltert ohne `_deleted` (für UI)
- Soft-Delete: `_deleted: true, lastModified: Date.now()` statt physisches Löschen

### Auth (MSAL)
```js
SP.getToken() → acquireTokenSilent() → bei Fehler: acquireTokenRedirect()
SP.gf(endpoint) → getToken() + fetch(Graph API)
```
- Login: `loginRedirect()` (immer Redirect, kein Popup)
- Gast-Konten: 7-stufiges Email-Matching (normalizeEmail, extractCore, #EXT#, x_-Prefix)

### Rollen
- **Admin** (`user.admin`): Voller Zugriff, MA-Verwaltung, Projekte, Settings
- **Approver** (`user.approver`): Report-Tab, Abwesenheiten freigeben, MA-Edit
- **Normal**: Eigene Einträge, Timer, Abwesenheiten beantragen

### Wochensperre
Normale MA können nur Einträge der **aktuellen Woche** (Mo–So) bearbeiten. Admins/Approver: keine Einschränkung. Betrifft NUR Zeiteinträge, nicht Abwesenheiten.

### Überstundenkonto
```
Guthaben = Vortrag + Saldo(Ist-Soll) - Auszahlungen - Kompensation
```
- `calcOvertimeAccount(empId, year, customFrom, customTo)`
- `calcSollHours()` berücksichtigt: Pensum, Temp-Pensum, Arbeitstage, Feiertage, Betriebsferien, genehmigte Abwesenheiten
- Raucherabzug: 15 Min/Tag vom Ist

### Produktivität
- Projekte haben `produktiv: true/false` Flag
- `isProduktiv(projectName)` matcht auch kombiniertes Format "ID - Name"
- Report im Report-Tab mit MA-Filter (Checkboxen)

## Schweizer Besonderheiten
- **Immer `ss` statt `ß`** (Schweiz)
- **Umlaute: ä, ö, ü** (nie ae, oe, ue)
- Sprache: Deutsch
- Währung: CHF
- Arbeitszeit: 41h/Woche Standard, Mo–Fr

## Deployment
```bash
git add -A
git commit -m "Beschreibung"
git push origin main
```
GitHub Pages deployed automatisch von `main` Branch. Live-URL: `https://dominic8112.github.io/zeiterfassung-hubler/`

## Konventionen
- Kein Framework, kein Build-Tool – alles in einer Datei
- `$('id')` ist Shortcut für `document.getElementById('id')`
- `toast(msg, type)` für User-Feedback
- `dateKey(date)` → `YYYY-MM-DD` String
- `fmtDate(date)` → `DD.MM.YYYY` String
- `fmtH(hours)` → formatierte Stunden
- CSS-Variablen: `--bg`, `--text`, `--accent` (#FF6B35), `--border`, etc.

## Häufige Aufgaben
- **Neues Feature im Timer:** Suche nach `btnStart.onclick` und `showPauseDialog`
- **Neuer Report:** Suche nach `exportReportHTML` oder `buildMonatsrapportPage`
- **Neuer Admin-Bereich:** Suche nach `renderEmpManagement` oder `renderProjectList`
- **SP-Datenstruktur ändern:** Suche nach `fullSync`, `readFile`, `writeFile`
- **Neuer Absenz-Typ:** Suche nach `absTypeMap` und `absTypes`
