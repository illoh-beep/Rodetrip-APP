# Roadtrip-Spieleapp — Projektkontext für Claude

> Diese Datei gibt jeder neuen (auch Cloud-/Handy-)Session sofort den vollen Überblick.
> Sprache mit dem Nutzer: **Deutsch**.

## Was ist das?
Eine **Roadtrip-Spieleapp** für 4 Freunde (**Gulli, Feli, Nico, Anna**), die gemeinsam
auf **einem iPhone im Safari-Browser** spielen. Gehostet über **GitHub Pages**.

## Harte Constraints (NICHT brechen!)
- **Alles in einer einzigen Datei: `index.html`** — kein React, kein Build-Tool, kein
  Server, kein npm. Nur Vanilla HTML/CSS/JS.
- **Keine externen Abhängigkeiten / kein CDN** (Ausnahme: Spotify-/iTunes-Fetch-APIs zur
  Laufzeit). Tesseract.js (OCR) wurde bewusst wieder entfernt.
- **iPhone-Safari-tauglich**: `touch-action: manipulation` auf Buttons (kein Doppeltipp-Zoom),
  große Tap-Flächen, dunkles Design.
- **Funktioniert offline** nach erstem Laden (für die Nicht-Online-Teile).
- **Punkte** liegen in `localStorage` (Key `roadtrip-scores-v1`).

## Architektur (alles in `index.html`)
- `<head>`: Meta/Viewport + komplettes `<style>` (dunkles Theme, CSS-Variablen in `:root`).
- `<body>`: mehrere `.screen`-Divs (`#screen-home`, `#screen-song-setup`, `#screen-song-dedupe`,
  `#screen-song-battle`, `#screen-song-winner`, `#screen-km-setup`, `#screen-km-tracking`,
  `#screen-km-result`, `#screen-scoreboard`). Navigation via `showScreen(id)`.
- `<script>` Abschnitte:
  - **1. Konstanten**: `PLAYERS`, `SCORES_KEY`, `KM_EVENTS`, `getKmEvent()` (← KI-Swap-Punkt)
  - **5b. SPOTIFY-INTEGRATION**: PKCE-Login + Playlist-Abruf (siehe unten)
  - **2. State**: globales `state`-Objekt (`scores`, `songBattle`, `kmGame`)
  - **3. Storage**, **4. Navigation/Toast**, **6. Home**
  - **7. Song Battle Setup**, **8. Dedupe/Review**, **9. Turnier**
  - **10./11. Km Schätzen** (inkl. GPS), **12. Haversine**, **13. Scoreboard**, **14. Init**

## Spiel 1: Song Battle (mit Spotify)
- Jeder Spieler fügt einen **öffentlichen Spotify-Playlist-Link** ein → App lädt alle Tracks
  (Titel, Künstler, Cover direkt von Spotify).
- Duplikate-Review (Textliste) → **K.O.-Turnier** (zwei Songs als Text+Cover, antippen = Sieger)
  → Gewinner-Song; sein Besitzer bekommt **3 Punkte**.
- Wichtige Funktionen: `loadPlaylist()`, `fetchPlaylistTracks()`, `startTournament()`,
  `buildPairs()`, `showBattleScreen()`, `pickWinner()`, `showWinnerScreen()`.

### Spotify-Setup (PKCE, kein Secret — passt zu statischem Hosting)
- `SPOTIFY_CLIENT_ID = '01154e8461b5413193b17043685672f4'`
- Flow: `spotifyLogin()` (PKCE) → Weiterleitung zu Spotify → Rücksprung wird in
  `handleSpotifyRedirect()` (in `init`) verarbeitet → Token in `localStorage` (`sp_token`).
- Scopes: `playlist-read-private playlist-read-collaborative`.
  **`SPOTIFY_SCOPE_VERSION`** erzwingt Neu-Login, wenn Scopes geändert werden (hochzählen!).
- **Nur EINE Person muss sich einloggen** — öffentliche Playlists sind mit jedem gültigen Token lesbar.
- **Redirect URIs** müssen im Spotify-Dashboard **zeichengenau** eingetragen sein
  (developer.spotify.com → App → Settings → Redirect URIs):
  - `https://illoh-beep.github.io/Rodetrip-APP/`  (Live, mit Schrägstrich!)
  - `http://127.0.0.1:8080/`  (lokal — **nicht** `localhost`, das lehnt Spotify ab)
- **Development Mode**: nur unter *User Management* freigeschaltete Spotify-Konten dürfen die App nutzen.

### ⚠️ OFFENES PROBLEM (Stand zuletzt)
Login klappt, aber **Playlist-Laden schlägt teils fehl**. Fehler werden jetzt im Setup-Screen
**im Klartext** angezeigt (`Spotify 403: …` / `Spotify 404: …`, gespeichert in
`state.songBattle.playerError`). Nächster Schritt: exakten Code vom Nutzer geben lassen.
- `403` → meist Development-Mode/User-Management im Dashboard (Konto freischalten)
- `404` → Playlist nicht wirklich öffentlich

## Spiel 2: Kilometer Schätzen (GPS)
- App zieht ein Ereignis aus `KM_EVENTS` (Auto-/LKW-Farben, Kennzeichen-Länder, Infrastruktur).
- Alle 4 schätzen die km bis zum nächsten Auftreten → GPS trackt die **kumulativ** zurückgelegte
  Strecke (`watchPosition` + `haversine`, Jitter-Filter). „GESEHEN!" → wer am nächsten lag, +1 Punkt.
- GPS-Fallback: bei verweigerter Berechtigung manuelle Distanz-Eingabe.
- Wichtige Funktionen: `startKmSchaetzen()`, `kmStartTracking()`, `kmSeen()`, `kmShowResult()`, `haversine()`.

## Geplante Erweiterung
Es kommen **5 weitere Spiele** dazu. Architektur ist darauf ausgelegt (Spiel-Kacheln auf Home,
Screens nach Muster ergänzen). `getKmEvent()` zeigt das Muster für späteren **Claude-AI-Swap**
(hardcodiert jetzt → später `fetch` zur Anthropic-API, nur die eine Funktion tauschen).

## Entwicklung & Test
- Lokaler Server: `python3 -m http.server 8080` (siehe `.claude/launch.json`), dann
  **`http://127.0.0.1:8080/`** öffnen (wegen Spotify-Redirect, nicht `localhost`).
- **JS-Syntaxcheck**: JS aus `index.html` extrahieren und `node --check` laufen lassen.
- **Nicht im Cloud/Desktop testbar**: echter Spotify-Login-Redirect und GPS — das muss der
  Nutzer auf seinem **echten Handy** prüfen. Cloud/Desktop kann nur Code + Logik verifizieren.

## Deploy
- Push auf **`main`** → GitHub Pages baut automatisch.
- Repo: `illoh-beep/Rodetrip-APP` · Live: `https://illoh-beep.github.io/Rodetrip-APP/`
- Commits beim Bestätigen durch den Nutzer; sinnvolle, knappe Messages.

## Design-Tokens (CSS-Variablen in `:root`)
`--bg #0f0f1a` · `--card #1e1e30` · `--accent #6c63ff` · `--danger #e63946`
`--success #2dc653` · `--text #e8e8f0` · Buttons min-Höhe **64px** · `--radius 16px`.
