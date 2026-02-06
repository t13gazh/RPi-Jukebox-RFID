# Feature Landscape

**Domain:** Embedded media player web interface for children's music box (Phoniebox)
**Researched:** 2026-02-06
**Confidence:** MEDIUM (based on training knowledge of Toniebox/mytonies, Volumio, moOde, Sonos; no live web verification possible)

## Competitor Reference

Before categorizing features, here is what the competitive landscape offers. This informs what parents will expect.

| Feature Area | Toniebox (mytonies) | Volumio | moOde Audio | Sonos App | Phoniebox v2 (current) |
|---|---|---|---|---|---|
| Player controls | Play/pause via physical, basic app controls | Full transport, seek bar, queue | Full transport, seek, EQ | Full transport, seek, crossfade | Play/pause/prev/next/seek/repeat/mute |
| Now playing display | Cover art, title, progress | Cover art, title, artist, progress bar | Cover art, metadata, bitrate, format | Cover art, full metadata, lyrics | Playlist name, track list, no progress bar |
| Library browsing | Simple content tiles by Tonie figure | Grid + list, search, genre/artist/album | List-based, album art, search, filters | Grid, search, browse by artist/album/genre | Folder tree with type filter tabs |
| Content management | Buy from Tonies store, record via Creative-Tonies | Local files, NAS mount | Local files, NAS, USB | Streaming only (no local management) | File upload, folder creation |
| RFID/figure assignment | Automatic (NFC in each Tonie) | N/A | N/A | N/A | Manual: swipe card + select folder/stream |
| Parental controls | Max volume, sleep timer, content curated | Volume limit | Volume limit, output config | Volume limit, room grouping | Max volume, sleep timer, idle shutdown |
| Streaming | Tonies content cloud only | Spotify, Tidal, Qobuz, web radio, Spotify Connect | Spotify Connect, web radio, AirPlay | All major services | Spotify (broken), web radio, podcast URL |
| Settings UI | Minimal (WiFi, volume limit, sleep timer) | Extensive (audio, network, plugins, sources) | Very technical (ALSA, MPD, kernel tweaks) | Moderate (rooms, services, EQ) | Extensive but technical (20+ settings panels) |
| Mobile optimization | Native iOS/Android app | Responsive web, native app | Responsive web | Native app | Responsive Bootstrap 3, dated look |
| Offline capability | Full offline playback | Yes (local files) | Yes (local files) | No (streaming only) | Yes (local files) |
| Auth/roles | Account-based (Tonies cloud) | None by default | None | Account-based | None |

**Key insight:** The Toniebox/mytonies app is the closest competitor in terms of use case (parents managing children's music box). Volumio and moOde are audiophile-oriented -- too technical. Sonos is the gold standard for UX simplicity. The sweet spot is: **Sonos-level simplicity applied to Phoniebox's use case.**

---

## Table Stakes

Features users expect. Missing = product feels incomplete or frustrating. Parents coming from Toniebox or any music app will expect these.

### Player Controls & Now Playing

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Play / Pause / Stop | Universal music control | Low | Already exists in v2, needs visual refresh |
| Previous / Next track | Universal music control | Low | Already exists in v2 |
| Volume slider (not just +/-) | Every music app has this | Low | v2 has +/- buttons only, needs proper slider |
| Now Playing: cover art | Visual identification of content | Medium | v2 has optional cover display; needs reliable, prominent display |
| Now Playing: title + artist | Basic "what's playing" info | Low | v2 shows playlist, not clean track info |
| Progress bar with seek | Parents need to skip ahead in long audiobooks | Medium | v2 has seek buttons but no visual progress bar |
| Track position in playlist | "Track 3 of 12" context | Low | v2 shows track list but not prominent position indicator |
| Repeat modes (off / playlist / single) | Standard for audiobooks: repeat off to stop after story | Low | Already in v2 |

### Library Browsing

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Browse by folder/album with cover art grid | Visual browsing is standard in every music app | Medium | v2 has text-only folder tree; grid with covers is expected |
| Content type filter (music/audiobook/radio/podcast) | v2 already has this concept (filter tabs) | Low | v2 has filter buttons, keep but improve UX |
| Search (text search across library) | Universal expectation from any content app | Medium | v2 has separate search page; should be inline/always available |
| Sort options (alphabetical, recently added, recently played) | Basic organization | Low | v2 sorts alphabetically only |
| Play directly from library | Tap an album/folder to start playing | Low | v2 has this via folder play buttons |

### Content Management

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Upload audio files via browser | Core feature -- parents need to add content | Medium | v2 has this but limited (PHP upload size issues) |
| Create/rename/delete folders | Basic file organization | Low | v2 has folder creation; rename and delete missing |
| Multi-file upload with progress indicator | Uploading 20 audiobook chapters needs progress | Medium | v2 has basic multi-upload, no progress indication |
| Folder/album cover art assignment | Visual identity for each album/audiobook | Medium | v2 has cover support but manual |

### RFID Card Management

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Assign card to folder (audio content) | Core Phoniebox feature | Low | v2 has this; needs UX improvement |
| Assign card to stream URL (radio/podcast) | Core Phoniebox feature | Low | v2 has this |
| Assign card to system command (volume, shutdown, etc.) | Power user feature but expected in Phoniebox ecosystem | Low | v2 has this |
| View all card assignments (card overview) | Parents need to see what each card does | Low | v2 has this implicitly via folder view shortcuts |
| Edit existing card assignment | Change what a card plays | Low | v2 has cardEdit.php |
| Delete card assignment | Remove unused cards | Low | v2 has this |
| "Last scanned card" display during registration | Real-time feedback: "Put card on reader, then assign" | Low | v2 has this via AJAX polling |

### Parental Controls

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Maximum volume limit | Child ear protection -- Toniebox has this | Low | v2 has this setting |
| Sleep timer (stop after X minutes) | Bedtime listening -- Toniebox has this | Low | v2 has sleep timer and stop timer |
| Idle shutdown (shutdown after X min idle) | Battery/energy saving | Low | v2 has this |
| Startup volume (fixed volume on boot) | Prevent blasting after restart | Low | v2 has this |

### Settings & Configuration

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| WiFi configuration | Network setup without SSH | Medium | v2 has this |
| Volume step size configuration | Customize how much each button press changes volume | Low | v2 has this |
| Language selection | Multi-language support | Low | v2 supports DE, EN, FR, NL |
| System info (IP address, disk space, uptime) | Basic diagnostics | Low | v2 has systemInfo.php |
| Shutdown / Reboot buttons | Remote power control | Low | v2 has these in navigation |
| RFID reader behavior (second swipe action) | Phoniebox-specific: what happens when card is swiped again | Low | v2 has this |

### Mobile Optimization

| Feature | Why Expected | Complexity | Notes |
|---------|-------------|------------|-------|
| Responsive design (works on phone) | Parents primarily access from smartphone | Medium | v2 has Bootstrap 3 responsive, but dated |
| Touch-friendly controls (large tap targets) | Phone usage requires 44x44px minimum touch targets | Low | Design concern, not feature; built into modern frameworks |
| Fast initial load (< 3 seconds) | Slow load = frustration, especially on WiFi to RPi | Medium | v2 loads each page server-side; SPA would be faster after initial load |

---

## Differentiators

Features that set this product apart from v2 and from competitors. Not expected, but highly valued -- the "wow" moments that make parents love the interface.

### UX & Accessibility

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Three-tier access model (open player / PIN-parent / expert) | Unique: kids see only player, parents manage, experts configure | Medium | Not in v2 or any competitor. Toniebox has app-only (no open web). Core differentiator for Phoniebox. |
| "Guided" card registration wizard | Step-by-step: 1. Scan card 2. Choose content 3. Done | Medium | v2 shows all options at once (folder, stream, command, YouTube) which overwhelms. A wizard simplifies. |
| Visual card-to-content mapping ("card wall") | Grid of cards with cover art showing what each card plays | Medium | No competitor has this. Parents can see all cards at a glance. |
| Resume position memory (per card) | Pick up audiobook where child stopped | Medium | v2 has basic MPD resume. Making this visible and reliable is a differentiator (Toniebox does this well). |
| PWA / Add to Home Screen | Access like a native app, no app store needed | Low | Simply adding a manifest.json and service worker. Big UX win for parents. |
| Drag-and-drop playlist editing | Reorder tracks within a folder/playlist by dragging | Medium | v2 has move up/down buttons. Drag-and-drop is modern and intuitive. |
| Real-time player state via WebSocket | Instant UI updates when child presses physical button or scans card | Medium | v2 polls every second. WebSocket gives instant feedback. Parents see exactly what the box is doing. |

### Content Acquisition

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Pre-configured stream catalog (ARD/NDR Audiothek, Deutschlandfunk, etc.) | Browse and add German children's radio stations with one tap | Medium | v2 requires manual URL entry. A curated catalog of kid-friendly streams is hugely valuable for German market. |
| Podcast RSS subscription | Enter podcast URL, auto-download new episodes | High | Not in v2. Would make Phoniebox a podcast player for bedtime stories. |
| YouTube/URL audio download via yt-dlp | Download audio from URL, save to library | Medium | v2 had this (broken). Restoring and improving it is valuable. |
| Content source browser (Librivox, Freie Hoerspiele, etc.) | Browse free audiobook libraries directly in the UI | High | No competitor does this. Curated links to free children's audiobook sources. |

### Gyro Sensor Integration

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Gyro configuration in Web UI | Configure tilt gestures without SSH | Low | phonie-gyro has config file; exposing in UI is straightforward |
| Gyro sensitivity profile selection | Choose from 3 pre-configured profiles | Low | phonie-gyro already supports this |
| Gyro calibration trigger from UI | Start calibration process from web interface | Medium | Currently requires SSH; web trigger would be convenient |
| Gyro gesture visualization | Show which direction does what (visual diagram) | Low | Static informational display with current mapping |

### Smart Automation

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Scheduled playback (alarm clock mode) | "Play Einschlafgeschichten at 19:00 every day" | High | No competitor in this space does this. Parents would love automated bedtime stories. |
| Volume schedule (quiet hours) | Auto-limit volume during naptime/bedtime hours | Medium | Extension of existing max volume feature with time awareness. |
| Content rotation | "Play a different audiobook each day from this set" | High | Advanced but unique for children's content management. |

---

## Anti-Features

Features to explicitly NOT build. Common mistakes in this domain.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Full-featured music library database (artist/album/genre metadata) | Phoniebox content is folders of audiobooks and children's music, not a music collection. Metadata parsing (ID3 tags, MusicBrainz) adds massive complexity for little value. | Use folder names as primary organization. Show cover art from folder. Do NOT build a metadata indexer. |
| Equalizer / DSP / Audio format settings | Parents don't care about FLAC vs MP3, sample rates, or EQ curves. moOde goes deep here and it scares normal users. | Fixed "good enough" audio settings. One optional "bass boost" toggle at most. |
| Multi-room / multi-box sync | Sonos does this, but Phoniebox is one box for one child. Building multi-room adds enormous complexity (clock sync, network discovery). | If someone has two boxes, they manage them independently. |
| Social features (sharing playlists, ratings) | This is a local family device, not a social network. | Keep it simple and private. |
| Streaming service integration UI (Spotify browse/search) | Spotify's API is unstable for OSS, and building a Spotify browser duplicates their app. Same for Tidal, Apple Music. | Support Spotify Connect (box appears as speaker in Spotify app). Let parents use official apps to cast. |
| Complex user management (multiple accounts, permissions matrix) | Overkill for a family device. Three tiers (open/parent/expert) is enough. | Simple PIN protection, not account management. |
| Theme engine / custom CSS editor | Developers love this, parents don't. One clean theme is better than a customizable ugly one. | One well-designed responsive theme. Light/dark mode toggle is acceptable. |
| Real-time audio visualization (spectrum analyzer, waveforms) | Looks cool, wastes RPi resources, adds no value for parents or children (children interact physically, not via screen). | Static cover art is better than animated distractions. |
| Auto-tagging / metadata scraping from online databases | Unreliable, slow, requires internet, often wrong for children's content (German audiobooks poorly covered in MusicBrainz). | Let parents name folders clearly. The folder name IS the metadata. |
| In-app audio editor (trim, merge, split tracks) | Way out of scope. Complex, buggy, and parents can use Audacity if needed. | Upload pre-prepared files. Maybe offer "split by chapter" for single-file audiobooks as a future differentiator. |

---

## Feature Dependencies

```
Core Infrastructure (must exist first):
  API Layer ──> Player Controls
  API Layer ──> Library Browsing
  API Layer ──> Settings UI

Player Controls (foundation):
  Player Controls ──> Now Playing Display
  Player Controls ──> Volume Controls
  Player Controls ──> Progress Bar + Seek

Library Browsing (builds on API):
  Library Browsing ──> Content Type Filters
  Library Browsing ──> Search
  Library Browsing ──> Play from Library

Content Management:
  Library Browsing ──> File Upload
  File Upload ──> Folder Management
  Folder Management ──> Cover Art Assignment

RFID Card Management (builds on Library):
  Library Browsing ──> Card Assignment (need to browse to select content)
  Card Assignment ──> Card Overview ("card wall")
  Card Assignment ──> Card Edit / Delete

Access Control:
  All Features ──> Three-Tier Access (wraps everything)
  Settings UI ──> PIN Protection (parent area gate)

Streaming Integration:
  Library Browsing ──> Stream Catalog
  Stream Catalog ──> Podcast Subscription
  Content Management ──> YouTube/URL Download

Gyro Sensor:
  Settings UI ──> Gyro Configuration
  Gyro Configuration ──> Gyro Calibration
  Gyro Configuration ──> Sensitivity Profiles

Advanced (Phase 2+):
  Player Controls + Card Management ──> Resume Position
  Settings UI ──> Scheduled Playback
  Settings UI ──> Volume Schedule
```

---

## MVP Recommendation

For MVP (Phase 1), prioritize table stakes that fix the biggest v2 pain points. The goal: **parents can manage the box from their phone without reading a manual.**

### MVP Must-Have (Phase 1 Web Interface)

1. **Player controls with now-playing display** -- Play/pause, prev/next, volume slider, progress bar, cover art, track info. This is the "home screen" parents see.
2. **Library browsing with cover art grid** -- Visual browsing replaces v2's text-only folder tree. Search included.
3. **RFID card registration (simplified wizard)** -- The core Phoniebox workflow, but guided instead of overwhelming.
4. **Card overview ("card wall")** -- See all assigned cards at a glance.
5. **Basic settings** -- Max volume, sleep timer, idle shutdown, WiFi, language. PIN-protected parent area.
6. **File upload** -- Upload audio files to folders.
7. **Mobile-first responsive design** -- Entire UI must work perfectly on phone.

### Defer to Phase 2

- **Stream catalog** (ARD/NDR pre-configured): Valuable but not blocking; parents can still enter URLs manually.
- **Gyro sensor configuration**: Works without UI (existing config file). UI is convenience.
- **Podcast RSS subscription**: Complex, can be manually downloaded and uploaded for now.
- **YouTube/URL download**: Was broken in v2 anyway, not expected.
- **Three-tier access model**: Important for polish but MVP can ship with simple PIN for settings.
- **PWA / Add to Home Screen**: Quick win but not MVP-blocking.
- **Real-time WebSocket updates**: Can use polling initially, upgrade later.

### Defer to Phase 3+

- **Resume position per card** (requires backend changes to track position per RFID card)
- **Scheduled playback / volume schedule** (requires backend timer service)
- **Content source browser** (Librivox etc.) (requires external API integration)
- **Drag-and-drop playlist editing** (polish feature)
- **Content rotation** (advanced automation)

---

## Existing v2 Features to Preserve

These features exist in v2 and must NOT be lost in the new web interface:

| v2 Feature | Location | Must Keep |
|---|---|---|
| Player transport controls | inc.loadControls.php | Yes -- rebuild modern |
| Cover art display | inc.loadCover.php | Yes -- make prominent |
| Loaded playlist view | inc.loadedPlaylist.php | Yes -- improve UX |
| Volume control | inc.setVolume.php | Yes -- add slider |
| Audio folder browser with type filters | index.php filter buttons | Yes -- add grid view |
| Card registration (interactive) | cardRegisterNew.php + inc.formCardEdit.php | Yes -- wizard UX |
| Card export/import CSV | rfidExportCsv.php + cardRegisterNew.php upload | Yes -- keep for backup |
| Card edit | cardEdit.php | Yes -- simplify |
| File/folder upload and creation | manageFilesFolders.php | Yes -- improve UX |
| Search | search.php | Yes -- make inline |
| Settings: language, volume, max volume, startup volume, volume step, boot volume | settings.php sections | Yes |
| Settings: sleep timer, stop timer, idle shutdown, shutdown volume reduction | settings.php sections | Yes |
| Settings: WiFi, WLAN IP display | settings.php sections | Yes |
| Settings: second swipe behavior, second swipe pause | settings.php sections | Yes |
| Settings: web UI config, input devices, debug logging | settings.php sections | Yes -- move to expert tier |
| System info | systemInfo.php | Yes -- move to expert tier |
| Shutdown / Reboot | inc.navigation.php | Yes |
| Multi-language (DE, EN, FR, NL) | lang/*.php | Yes -- use i18n framework |
| Bluetooth audio sink toggle | inc.bluetooth.php | Yes -- if enabled |
| RSS/podcast feed | rss-mp3.php | Yes -- improve |

---

## Complexity Estimates (Relative)

| Complexity | Definition | Examples |
|---|---|---|
| Low | < 1 day, standard patterns, no backend changes | Volume slider, sort options, shutdown button, language selector |
| Medium | 1-3 days, some state management or backend integration | Cover art grid, progress bar with seek, file upload with progress, card wizard |
| High | 3-7 days, new backend capability or external integration | Podcast RSS subscription, scheduled playback, content source browser, WebSocket real-time |

---

## Sources

- **Toniebox/mytonies:** Based on training knowledge of the mytonies app (iOS/Android), Toniebox product documentation, and community reviews. Confidence: MEDIUM -- product is well-established but specific feature details may have changed.
- **Volumio:** Based on training knowledge of Volumio 3 web interface and plugin system. Confidence: MEDIUM -- core features stable for years.
- **moOde Audio:** Based on training knowledge of moOde Audio Player 8.x web interface. Confidence: MEDIUM -- niche product, less likely to have changed dramatically.
- **Sonos:** Based on training knowledge of Sonos S2 app. Confidence: MEDIUM -- redesigned in 2024, specific current features may differ.
- **Phoniebox v2:** Verified directly from codebase analysis. Confidence: HIGH -- first-party source code examined.
- **phonie-gyro:** Verified from project memory and ANALYSIS.md. Confidence: HIGH -- user's own project.

**Note:** WebSearch and WebFetch were unavailable during research. All competitor feature claims are based on training data (cutoff: early 2025) and should be treated as MEDIUM confidence. The Phoniebox v2 analysis is HIGH confidence as it was verified directly from the codebase.
