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
- **Genau `SONGS_PER_PLAYER` (=20) Songs pro Spieler**: bei mehr muss aussortiert, bei
  weniger manuell ergänzt werden. `playerSongCount()` zählt (Titel nicht leer); der
  „Weiter"-Button ist gesperrt, bis genau 20 erreicht sind (Zähler `X / 20` im Setup).
- Duplikate-Review (Textliste) → **K.O.-Turnier** (zwei Songs antippen = Sieger)
  → Gewinner-Song; sein Besitzer bekommt **3 Punkte**.
- **Paarung (`buildPairs`)**: greedy nach Besitzer-Gruppengröße → es treten **immer Songs
  zweier verschiedener Personen** an, solange möglich (gleiche nur, wenn unvermeidbar).
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

## Spiel 3: Bingo Karten Generator (Lock-out Bingo)
- **Ein gemeinsames 5×5-Board** für 2 zufällig ausgeloste Zweier-Teams (`drawBingoTeams()`).
  Felder = Dinge auf Kennzeichen/Schildern, feste Verteilung **6/6/8/4/1** aus `BINGO_POOLS`
  (`getBingoBoard()` ← KI-Swap-Punkt, baut+mischt die 25 Felder).
  **Mindestens 2 Zeichen pro Feld** (z. B. „HF", „92") — keine Einzelbuchstaben/-ziffern
  (sonst Streit, wer es zuerst gesehen hat).
- **Lock-out**: `claimCell()` — freies Feld → aktives Team (gesperrt, Teamfarbe); eigenes
  Feld nochmal = freigeben (Verklick-Korrektur); gegnerisches = gesperrt (Toast).
  Aktives Team über die Team-Chips (`setActiveTeam()`).
- **Kein Auto-Ende**: `teamHasBingo()`/`BINGO_LINES` zeigen nur einen Hinweis-Banner.
  „Spiel beenden" → `showBingoResult()` → Gewinner manuell wählen (`pickBingoWinner()`)
  → **beide Team-Mitglieder +1**.
- Screens: `#screen-bingo-setup`, `#screen-bingo-game`, `#screen-bingo-result`.

## Spiel 4: Wer bin ich?
- Jeder bekommt eine geheime Identität (kennt seine eigene nicht). Play-Screen mit
  **4 Namens-Buttons** → Tippen zeigt die Identität dieses Spielers im Vollbild
  (`revealIdentity()`/`hideIdentity()`); vertrauensbasiert (eigenes nicht tippen).
- **2 Modi** (`chooseWhoAmIMode`): **manuell** — Handy reihum, Spieler i tippt die
  Identität für Spieler (i+1) (`renderWhoAmIEntry`/`whoAmIEntryNext`); **random** — aus
  `WHO_AM_I_POOL` ziehen (`getWhoAmINames()` ← KI-Swap-Punkt; Starter-Liste, per .txt ersetzbar).
- **Punkte**: die ersten beiden, die sich selbst erraten, je **+1** (`whoAmIStartScoring`
  → `pickGuesser` 1./2. → Stand inline). Screens: `#screen-whoami-setup/-entry/-play/-result`.

## Spiel 5: Kennzeichen Mathe
- App gibt eine **Zielzahl 1–100** vor (`getKennzeichenTarget()` ← KI-Swap-Punkt).
  Man tippt die **Ziffern eines Kennzeichens** + einen **Rechenweg** (`+ - * /`, Klammern).
- **Cheat-sicher**: `kzCheck()` prüft (a) es werden **genau** die Kennzeichen-Ziffern benutzt
  (Multiset-Abgleich) und (b) das Ergebnis = Zielzahl. Auswertung über `kzSafeEval()`
  (eigener Shunting-Yard-Parser, **kein eval()**; unäres Minus unterstützt).
- Richtig → Löser wählen → **+1**, dann neue Zahl. Screen: `#screen-kzmath`.

## Spiel 6: Reise Parlament
- App schlägt **4 zufällige absurde Gesetze** vor (`getParlamentVorschlaege()` ←
  KI-Swap-Punkt; Pool `PARLAMENT_GESETZE`, je `{text, dauerMinuten}`, ≥45 Einträge).
- Gruppe einigt sich auf EINS (antippen) → **finale Abstimmung** (10-Sek-Timer,
  pro Spieler Ja/Nein). Mehrheit entscheidet, **Gleichstand = nicht in Kraft**.
- Angenommene Gesetze → `localStorage` (`LAWS_KEY='aktive_gesetze'`, je
  `{text, startTime, durationMinutes}`), laufen nach Zeit automatisch ab (`pruneLaws`).
- **App-weites Banner** (`#law-banner`, fix oben) zeigt aktive Gesetze + Countdown auf
  JEDEM Screen; `startParlamentTick()` (setInterval 1s) aktualisiert/entfernt; in `init`
  gestartet. `body.has-laws` schiebt Screens nach unten.
- Screens: `#screen-parlament` (+ `-select/-vote/-result`). Funktionen: `startParlament`,
  `parlamentPropose`, `parlamentChoose`, `parlamentVote`, `parlamentEvaluate`, `renderLawBanner`.

## Geplante Erweiterung
Es kommt **1 weiteres Spiel** dazu. Architektur ist darauf ausgelegt (Spiel-Kacheln auf Home,
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
