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

## Spiel 1: Song Battle (mit Spotify, OHNE Login)
- Jeder Spieler fügt einen **öffentlichen Spotify-Playlist-Link** ein → App lädt alle Tracks
  (Titel, Künstler, 30-Sek-Preview). **Kein Login, kein Account, kein Dashboard nötig.**
- Duplikate-Review (Textliste) → **K.O.-Turnier** (zwei Songs antippen = Sieger)
  → Gewinner-Song; sein Besitzer bekommt **3 Punkte**.
- Wichtige Funktionen: `loadPlaylist()`, `fetchPlaylistTracks()`, `_fetchViaProxy()`,
  `_findTrackList()`, `startTournament()`, `buildPairs()`, `showBattleScreen()`,
  `pickWinner()`, `showWinnerScreen()`.

### Spotify-Anbindung (Niklas-Ansatz: Embed-Seite über CORS-Proxy)
- **Kein OAuth/PKCE mehr.** Stattdessen: `fetchPlaylistTracks(id)` lädt die öffentliche
  Embed-Seite `https://open.spotify.com/embed/playlist/{id}` über einen **CORS-Proxy**,
  liest das `<script id="__NEXT_DATA__">`-JSON und sucht rekursiv (`_findTrackList`) das
  erste `trackList`-Array.
- `CORS_PROXIES` (Reihenfolge mit Fallback) in `index.html` (~Zeile 1240):
  1. `https://explosis173--…web.val.run/?url=`  ← funktioniert (Niklas' Instanz)
  2. `/proxy?url=`  ← nur mit lokalem Server, **nicht** auf GitHub Pages
  3. `https://api.allorigins.win/raw?url=`  ← öffentlich, wackelig
- **Pro Track liefert das Embed-JSON:** `title`, `subtitle` (Künstler), `duration`,
  `audioPreview.url` (30-Sek-MP3), `uri` (→ `spotifyId`). **Kein Cover im Embed.**
- **Cover** kommt deshalb über `fetchCover(song)` aus der **Spotify-Track-oEmbed**
  (`open.spotify.com/oembed?url=…/track/{id}` → `thumbnail_url`): kein Key, CORS direkt
  erlaubt, Proxy als Fallback, exakt per `spotifyId`. Lazy beim Battle/Winner geladen.
- **Preview-Playback** (Niklas): `togglePlaySide()`/`toggleSongPlay()` spielen die
  30-Sek-`audioPreview` im Battle (`▶`-Button), `stopSongPlay()` stoppt.
- **„In Spotify öffnen"-Button** (`.spotify-btn`): `spotifyTrackUrl(song)` →
  `open.spotify.com/track/{id}`, öffnet den exakten Song in der Spotify-App
  (Battle beide Seiten + Winner; `event.stopPropagation()` verhindert Sieger-Auswahl).
- **Fragilität (bewusst in Kauf genommen):** hängt an externem Proxy + an Spotifys
  Embed-Format. Ändert Spotify `__NEXT_DATA__`, bricht das Parsen.

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
- **Playlist-Laden** ist Desktop/Cloud testbar, **solange der CORS-Proxy erreichbar ist**
  (val.run-Instanz). **GPS** dagegen nur auf dem **echten Handy** prüfbar.

## Deploy
- Push auf **`main`** → GitHub Pages baut automatisch.
- Repo: `illoh-beep/Rodetrip-APP` · Live: `https://illoh-beep.github.io/Rodetrip-APP/`
- Commits beim Bestätigen durch den Nutzer; sinnvolle, knappe Messages.

## Design-Tokens (CSS-Variablen in `:root`)
`--bg #0f0f1a` · `--card #1e1e30` · `--accent #6c63ff` · `--danger #e63946`
`--success #2dc653` · `--text #e8e8f0` · Buttons min-Höhe **64px** · `--radius 16px`.
