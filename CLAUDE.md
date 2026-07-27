# Guitar Chord Navigator

Single-file static site: everything (HTML, CSS, JS) lives in `index.html`. No build step, no bundler, no dependencies to install — open the file or serve the directory with any static server.

```bash
python3 -m http.server 8791
```

## Structure

- `index.html` — the entire app.
  - `<head>` — CSS in a single `<style>` block, plus the AlphaTab CDN `<script>` tag.
  - `<body>` — sidebar (key selector, melody explorer button, tabs button) + main chord grid, followed by modals (`chordDetail`, `melodyModal`, `tabsModal`) and one big `<script>` block at the end with all app logic.
- `tabs/` — real Guitar Pro files (`.gp3`, `.gp5`, `.gpx`) used by the Tabs section. Never hand-transcribed data.

## The Tabs section

Sidebar button `.tabs-btn` → `openTabs()` opens a fullscreen modal (`.tabs-modal`, same pattern as `.melody-modal`): level filter chips (Fácil/Medio/Difícil) + a card grid (`renderTabsGrid`) + a detail pane (`renderTabDetail`).

**Data model** — the `TABS` array (search `const TABS = [`) holds one entry per song:

```js
{ id, title, subtitle, level, key, file: 'tabs/some-file.gp5' }
```

`file: null` means "not added yet" — the detail pane shows a placeholder instead of a player.

**Rendering — always via alphaTab, never hand-built.** `renderTabDetail()` creates an `alphaTab.AlphaTabApi` instance pointed at the song's real Guitar Pro file (fetched as an `ArrayBuffer`, passed to `api.load()`). AlphaTab (MIT, loaded from `cdn.jsdelivr.net/npm/@coderline/alphatab`) renders standard notation + tab and provides real MIDI audio playback, a following cursor, and speed control (`api.playbackSpeed`) — all for free. **Do not** reinvent a tab renderer, ASCII-art tab, or SVG fretboard for song playback: it was tried twice in this project and produced results with no real audio and, worse, silently-wrong note data (see below). AlphaTab's cursor color needs explicit CSS overrides (`.at-cursor-bar`, `.at-cursor-beat`, `.at-highlight` under `.tab-alphatab-wrap` in the stylesheet) — the CDN build ships with transparent defaults.

## Adding a new song to Tabs

1. **Get a real Guitar Pro file. Do not transcribe the tab by hand from a photo, PDF, or ASCII text.** Hand transcription of guitar tabs (reading fret numbers off an image, or parsing ASCII tab text into JSON) is unreliable — even careful column-by-column parsing produced a melody that didn't sound like the actual song when tested. A real `.gp3/.gp4/.gp5/.gpx` file already encodes correct pitch, rhythm, and multi-voice arrangement (melody + bass) from a human arranger.
   - Search e.g. `<song> guitar pro tab download gp5`.
   - Sites like `gtptabs.com` serve the real binary directly from a `/tabs/download/<id>.html` link found in the song's page HTML (`grep -oE 'href="/tabs/download/[0-9]+\.html"'`). `curl -sL -A "Mozilla/5.0" <url>` downloads it — no JS rendering needed, unlike Songsterr/Ultimate-Guitar which require a live browser.
   - Identify the real format by inspecting the file header before naming it — don't guess the extension:
     - `BCFZ` → Guitar Pro 6/7, save as `.gpx`
     - `FICHIER GUITAR PRO vX.XX` → save as `.gp3`/`.gp4`/`.gp5` matching the version string right after the header
   - `xxd file.bin | head -2` shows both signature and version string.
2. Save the file under `tabs/<song-id>.<ext>`.
3. Add/update its entry in the `TABS` array with the correct `file` path.
4. Verify by actually opening the Tabs modal, selecting the song, and clicking Reproducir — check the rendered notation looks sane (key signature, tempo, title/composer if present in the file) **and** listen to confirm it matches the real song. Visual inspection alone is not enough — a wrong transcription can still render a plausible-looking (but wrong-sounding) tab.

## Verifying UI changes

No test suite exists. Verify visually:

```bash
python3 -m http.server 8791 --directory /path/to/guitar
```

then drive it with Playwright (installed ad hoc via `npm install playwright` + `npx playwright install chromium` in the scratchpad dir — not a project dependency) or open it in a real browser. Screenshot before/after for any visual change; for Tabs specifically, always test actual playback (click Reproducir, wait, check `alphaTabApi.playerState === 1`), not just that the page loads without console errors.
