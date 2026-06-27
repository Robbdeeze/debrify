# RobbdeezeNutz_Debrify — Complete Change History

## Project Context
Debrify is a Flutter-based media streaming app (Polyform NC license) that aggregates torrents from multiple YAML-configured engines, Stremio addons. Supports Real-Debrid, Torbox, Premiumize, and All-Debrid for cached torrent playback. The codebase is cloned fresh from `https://github.com/Robbdeeze/debrify.git`.

---

## Session 1 (Jun 25) — Reverted to Clean Slate

**What happened:** Ported WebStreamr engine (73 files), added IPTV alive checker + list/grid toggle, added Source Scope selector, wired WebStreamr into TorrentService, applied scope filtering in Quick Play and list view, fixed home-tab crash. Hit a `!_doingMountOrUpdate` assertion. Decided to revert everything and start fresh.

**Action taken:**
- Backed up `RobbdeezeNutz_Debriyf_Versions.md`
- Deleted entire `/Users/robbdeeze/Documents/projects/debrify/`
- Re-cloned from `https://github.com/Robbdeeze/debrify.git`
- Restored `RobbdeezeNutz_Debriyf_Versions.md`
- Clean GitHub HEAD, no uncommitted changes

---

## Session 2 (Jun 25) — Quick Play respects Direct Streaming Mode toggle

**What was done:**
- Wired the existing "Direct Streaming Mode" toggle (Settings → Streaming) into the Quick Play auto-play decision tree at `torrent_search_screen.dart:6266-6279`
- **Previously**: Quick Play ignored the toggle. It always preferred debrid torrents, only using direct streams as a fallback when no debrid was configured.
- **Now**:
  - **Toggle ON**: Quick Play tries direct streams first. Only falls back to debrid torrents if no direct streams exist.
  - **Toggle OFF**: Original behaviour — prefers debrid torrents; falls back to direct streams if no debrid configured.
- The toggle already controlled the manual tap path (`_handleTorrentCardActivated` → routes to `StreamingDetailsScreen` when ON). Now it also controls auto-play consistently.
- 0 analyzer errors, debug APK builds clean.

**Files modified:**
- `lib/screens/torrent_search_screen.dart` — Quick Play decision tree (`_checkQuickPlayAfterSearch`)

---

## Next Session — Planned Features

### 1. IPTV Stream Validator (Alive Checker)
- Add alive/badge next to M3U channel entries (HTTP HEAD/GET)
- Batch check mode with live progress (`x/total · alive`)
- Stop control for in-progress checks
- Alive-only filter toggle
- **Files:** `lib/widgets/iptv/iptv_results_view.dart`

### 2. IPTV List / Grid View Toggle
- New `IptvChannelListTile` compact row widget
- Toggle button switching grid view (poster cards) ↔ list view (compact rows)
- **Files:** `lib/widgets/iptv/iptv_results_view.dart`, `lib/widgets/iptv/iptv_channel_tile.dart` (new)

### 3. WebStreamr Engine Port (~73 files)
- Port from [PlayTorrioV2](https://github.com/ayman708-UX/PlayTorrioV2) (GPL-2.0)
- 20 sources, 25 extractors, resolver, utils (cache, cookie jar, fetcher, unpacker, semaphore, MFP/Flare)
- New dep: `html: ^0.15.4`
- Omit `LocalServerService` (shelf HLS proxy) — only needed for 1shows.app PNG-stripping
- **Files:** `lib/webstreamr/` (new directory)

### 4. WebStreamr Adapter & Settings
- `WebStreamrService()` singleton, boot init in `main.dart`
- `getStreams(imdbId, isMovie, season, episode)` → `List<StreamSource>`
- Settings: country toggles, extractor exclusions, resolution filters, MFP/FlareSolverr URLs, TMDB token
- **Files:** `lib/services/webstreamr_service.dart`, `lib/services/webstreamr_settings.dart`, `lib/models/stream_source.dart`, `lib/screens/settings/webstreamr_settings_page.dart`

### 5. Source Scope Selector (Direct / Debrids / All)
- `stream_scope` pref in `StorageService` (default `'all'`)
- Segmented control in Settings > Streaming
- **Files:** `lib/services/storage_service.dart`, `lib/screens/settings_screen.dart`

### 6. WebStreamr → Torrent Integration
- `_searchWebStreamr()` runs in parallel with engine + Stremio addon search in `TorrentService.searchByImdbWithStremio()`
- `StreamSource` → `Torrent(streamType: StreamType.directUrl, source: 'webstreamr')`
- Infohash keyed as `ws:<url_hash>` for dedup safety
- **Files:** `lib/services/torrent_service.dart`

### 7. Stream Scope in Quick Play & List View
- `_streamScope` field loaded in `_searchTorrents()`
- Quick Play: `'all'` (prefer torrents) / `'direct'` (direct only) / `'debrids'` (torrents only)
- List filter: `_applyFiltersToList()` pre-filters by scope before provider/quality chain
- **Files:** `lib/screens/torrent_search_screen.dart`

### 8. Home-Tab Crash Fix (if it reappears)
- `_setHomeSectionInitialLoading()` calls `setState()` synchronously from child `onInitialLoadStateChanged` callbacks during viewport build phase
- Fix: wrap in `WidgetsBinding.instance.addPostFrameCallback`
- **Files:** `lib/screens/torrent_search_screen.dart` (line ~22806)

### 9. Settings UI Cleanup
- Remove temporary WebStreamr card from Connections grid
- Restore original 5-row Connections grid layout
- Add Source Scope + Direct Streams tiles under Settings > Streaming
- **Files:** `lib/screens/settings_screen.dart`

---

## Change 1: IPTV Stream Validator (Alive Checker)
**Files modified:**
- `lib/widgets/iptv/iptv_results_view.dart` (M3U results view — the portal view already had it)

**What was done:**
- Added an **alive-check button** next to each IPTV M3U channel entry
- The validator attempts an HTTP HEAD/GET on the channel's stream URL
- Shows status badges: green "Alive" or red "Dead" after check completes
- **Batch check mode:** a "Check All" button iterates over all channels
- **Live progress indicator:** updates in real-time — `"Checking 5/12 · 3 alive"`
- **Stop control:** cancels in-progress batch checks
- **Alive-only filter toggle:** hides dead channels after a batch check
- **UI elements:** uses existing Material Design progress indicators and chips

**Why:**
- IPTV streams go stale frequently; users needed a way to verify which channels still work without manually testing each one.

---

## Change 2: IPTV List/Grid View Toggle
**Files modified:**
- `lib/widgets/iptv/iptv_results_view.dart`
- `lib/widgets/iptv/iptv_channel_tile.dart` (new widget created)

**What was done:**
- Created `IptvChannelListTile` — a compact single-row widget showing channel name, logo, and alive status
- Added a **view toggle button** (grid icon / list icon) in the IPTV results toolbar
- **Grid view:** existing poster-card layout (unchanged)
- **List view:** dense vertical list using `IptvChannelListTile`, better for channels with long names or many entries
- State stored in the widget's state (not persisted across app restarts)

**Why:**
- M3U playlists often have hundreds of channels; the grid view wastes space. List view shows more channels per screen.

---

## Change 3: WebStreamr Engine Port (73 files)
**Source project:** [PlayTorrioV2](https://github.com/ayman708-UX/PlayTorrioV2) (GPL-2.0)
**License note:** GPL-2.0 code ported into Polyform NC project. User accepted this. LICENSE/NOTICE updates pending.

### Architecture
The engine follows a **source → resolver → extractor** pipeline:
1. **Source** scrapes a website for video page links (e.g., Cuevana, StreamKiste)
2. **Resolver** manages the crawl, calling sources in sequence based on user config
3. **Extractor** downloads the video page, unpacks embedded JavaScript (if needed), and extracts the playable video URL

All state is managed via a `Context` object (config flags, proxy URLs, etc.) and `WsEnv` (environment variables).

### Files created under `lib/webstreamr/`

#### Sources (20 files)
| File | Source | Country |
|------|--------|---------|
| `lib/webstreamr/source/cinehdplus.dart` | CineHDPlus | ES/MX |
| `lib/webstreamr/source/cuevana.dart` | Cuevana | ES/MX |
| `lib/webstreamr/source/einschalten.dart` | Einschalten | DE |
| `lib/webstreamr/source/eurostreaming.dart` | Eurostreaming | IT |
| `lib/webstreamr/source/fourkhdhub.dart` | FourKHDHub | multi |
| `lib/webstreamr/source/frembed.dart` | Frembed | FR |
| `lib/webstreamr/source/frenchcloud.dart` | FrenchCloud | FR |
| `lib/webstreamr/source/hdhub4u.dart` | HDHub4u | multi |
| `lib/webstreamr/source/homecine.dart` | HomeCine | ES/MX |
| `lib/webstreamr/source/kinoger.dart` | KinoGer | DE |
| `lib/webstreamr/source/kokoshka.dart` | Kokoshka | AL |
| `lib/webstreamr/source/megakino.dart` | MegaKino | DE |
| `lib/webstreamr/source/meinecloud.dart` | MeineCloud | DE |
| `lib/webstreamr/source/mostraguarda.dart` | MostraGuarda | IT |
| `lib/webstreamr/source/movix.dart` | Movix | FR |
| `lib/webstreamr/source/rgshows.dart` | RgShows | multi |
| `lib/webstreamr/source/source.dart` | Base Source class | — |
| `lib/webstreamr/source/streamkiste.dart` | StreamKiste | DE |
| `lib/webstreamr/source/vegamovies.dart` | VegaMovies | multi |
| `lib/webstreamr/source/verhdlink.dart` | VerHdLink | ES/MX |
| `lib/webstreamr/source/vidsrc.dart` | VidSrc | multi |
| `lib/webstreamr/source/vixsrc.dart` | VixSrc | multi |

#### Extractors (25 files)
| File | Extractor |
|------|-----------|
| `lib/webstreamr/extractor/doodstream.dart` | DoodStream |
| `lib/webstreamr/extractor/dropload.dart` | Dropload |
| `lib/webstreamr/extractor/external_url.dart` | ExternalUrl |
| `lib/webstreamr/extractor/extractor.dart` | Base Extractor class |
| `lib/webstreamr/extractor/extractor_registry.dart` | Registry |
| `lib/webstreamr/extractor/fastream.dart` | Fastream |
| `lib/webstreamr/extractor/filelions.dart` | FileLions |
| `lib/webstreamr/extractor/filemoon.dart` | FileMoon |
| `lib/webstreamr/extractor/fsst.dart` | Fsst |
| `lib/webstreamr/extractor/hubcloud.dart` | HubCloud |
| `lib/webstreamr/extractor/hubdrive.dart` | HubDrive |
| `lib/webstreamr/extractor/kinoger.dart` | KinoGer |
| `lib/webstreamr/extractor/lulustream.dart` | LuluStream |
| `lib/webstreamr/extractor/mixdrop.dart` | Mixdrop |
| `lib/webstreamr/extractor/rgshows.dart` | RgShows |
| `lib/webstreamr/extractor/savefiles.dart` | SaveFiles |
| `lib/webstreamr/extractor/streamembed.dart` | StreamEmbed |
| `lib/webstreamr/extractor/streamtape.dart` | Streamtape |
| `lib/webstreamr/extractor/supervideo.dart` | SuperVideo |
| `lib/webstreamr/extractor/uqload.dart` | Uqload |
| `lib/webstreamr/extractor/vidora.dart` | Vidora |
| `lib/webstreamr/extractor/vidsrc.dart` | VidSrc |
| `lib/webstreamr/extractor/vixsrc.dart` | VixSrc |
| `lib/webstreamr/extractor/voe.dart` | Voe |
| `lib/webstreamr/extractor/youtube.dart` | YouTube |

#### Utilities (10 files)
| File | Purpose |
|------|---------|
| `lib/webstreamr/utils/cache.dart` | In-memory cache for resolved URLs |
| `lib/webstreamr/utils/config.dart` | `disableExtractorConfigKey()` / `excludeResolutionConfigKey()` helpers |
| `lib/webstreamr/utils/cookie_jar.dart` | Cookie jar for HTTP requests |
| `lib/webstreamr/utils/env.dart` | `WsEnv` — environment variable management |
| `lib/webstreamr/utils/fetcher.dart` | HTTP fetcher with cookie support |
| `lib/webstreamr/utils/id.dart` | IMDb/TMDB ID parsing (`ImdbId`, `TmdbId`) |
| `lib/webstreamr/utils/semaphore.dart` | Concurrency limiter |
| `lib/webstreamr/utils/unpacker.dart` | JavaScript unpacker (eval-based packed JS) |
| `lib/webstreamr/utils/mediaflow_proxy.dart` | MFP support (stubbed) |
| `lib/webstreamr/utils/resolver.dart` -> `lib/webstreamr/stream_resolver.dart` | Moved to top-level |

#### Core engine files
| File | Purpose |
|------|---------|
| `lib/webstreamr/stream_resolver.dart` | `StreamResolver` — orchestrates sources→extractors |
| `lib/webstreamr/types.dart` | `Context`, `Stream`, result types |

### Key implementation details
- **`StreamResolver.resolve()`** iterates active sources, collects their results, and runs extractors on each video page URL
- **Sources** implement `Source` abstract class with `countryCodes`, `isMovie()`, `isSeries()`, and `fetchStreams()` methods
- **Extractors** implement `Extractor` abstract class with `matchDomain()` and `extract()` methods
- **User config** controls which country-code sources are active, which extractors are disabled, and resolution exclusions
- **`ExternalUrl`** extractor must be last in the registry (catch-all for URLs that match no other extractor)

### What was NOT ported
- `LocalServerService` — a shelf-based HLS proxy for stripping PNG wrappers. Only needed for 1shows.app (RG Shows). All other sources return standard video URLs directly. Stubbed to return upstream URL unchanged.

---

## Change 4: WebStreamr Adapter Service
**Files created:**
- `lib/services/webstreamr_service.dart` (276 lines)
- `lib/services/webstreamr_settings.dart` (preferences wrapper)
- `lib/models/stream_source.dart` (StreamSource, StreamResult models)

### `WebStreamrService` API
```dart
class WebStreamrService {
  static Future<void> init();  // Call once at app start

  Future<List<StreamSource>> getStreams({
    required String imdbId,
    bool isMovie = true,
    int? season,
    int? episode,
    int? tmdbId,
  });
}
```

### Boot initialization (`lib/main.dart`)
```dart
// Non-fatal catch — if WebStreamr fails to init, the rest of the app still works
try {
  await WebStreamrService.init();
} catch (e) {
  debugPrint('WebStreamr init failed: $e');
}
```

### Settings prefs (`webstreamr_settings.dart`)
- `getEnabledCountryCodes()` / `setEnabledCountryCodes()` — which country sources to activate
- `getDisabledExtractors()` / `setDisabledExtractors()` — which extractors to skip
- `getExcludedResolutions()` / `setExcludedResolutions()` — resolution filters
- `getMediaFlowProxyUrl()` / `getMediaFlowProxyPassword()` — MFP config
- `getFlareSolverrUrl()` / `setFlareSolverrUrl()` — FlareSolverr config
- `getTmdbAccessToken()` / `setTmdbAccessToken()` — TMDB API token (for TMDB ID lookups)

### Model (`stream_source.dart`)
```dart
class StreamSource {
  final String url;
  final String title;
  final String type;
  final Map<String, String>? headers;
}

class StreamResult {
  final List<StreamSource> sources;
  final String provider;
  final bool isRateLimited;
  final String? primaryUrl;
  final Map<String, String>? headers;
}
```

---

## Change 5: WebStreamr Settings Page
**File created:** `lib/screens/settings/webstreamr_settings_page.dart`

A full settings screen with:
- **Country source toggles:** multi/country-group collapsible sections
- **Extractor exclusions:** checkboxes to disable specific extractors
- **Resolution filters:** exclude 4K, 1080p, 720p, etc.
- **MediaFlow Proxy:** URL + password fields
- **FlareSolverr:** URL field
- **TMDB Access Token:** text field (required for TMDB-based lookups)
- All settings persisted via `WebStreamrSettings` → SharedPreferences

Accessible from **Settings > Streaming > Direct Streams (WebStreamr)**.

---

## Change 6: Source Scope Selector (Settings UI)
**File modified:** `lib/screens/settings_screen.dart`
**File modified:** `lib/services/storage_service.dart`

### Storage layer
Added to `StorageService`:
```dart
static const String _streamScopeKey = 'stream_scope';
static const String defaultStreamScope = 'all';

static Future<String> getStreamScope() async { /* reads SharedPreferences */ }
static Future<void> setStreamScope(String scope) async { /* writes SharedPreferences */ }
```
Valid values: `'all'`, `'direct'`, `'debrids'`. Default: `'all'`.

### Settings UI (`_StreamModeSection` in settings_screen.dart)
Added `_SettingsTile` items:
1. **Source Scope** — segmented selector (`Direct | Debrids | All`) in **Settings > Streaming** section. Uses new `_StreamScopeSelector` widget built on `SegmentedButton<String>`.
2. **Direct Streams (WebStreamr)** — opens `WebStreamrSettingsScreen` via Navigator push.

### Reverted Connections grid
- Removed the tentative WebStreamr card that was previously placed in the Connections grid
- Restored the original 5-row Connections grid layout

### New widget: `_StreamScopeSelector`
```dart
class _StreamScopeSelector extends StatelessWidget {
  final String value;
  final ValueChanged<String> onChanged;
  // Uses SegmentedButton<String> with compact style
}
```

---

## Change 7: WebStreamr → Torrent Integration
**File modified:** `lib/services/torrent_service.dart`

### Import added
```dart
import '../models/stream_source.dart';
import 'webstreamr_service.dart';
```

### New method `_searchWebStreamr()`
```dart
static Future<Map<String, dynamic>> _searchWebStreamr({
  required String imdbId,
  required bool isMovie,
  int? season,
  int? episode,
}) async {
  try {
    final ws = WebStreamrService();
    final result = await ws.getStreams(
      imdbId: imdbId, isMovie: isMovie,
      season: season, episode: episode,
    );
    final torrents = result.map((s) => Torrent(
      rowid: 0,
      infohash: 'ws:${s.url.hashCode.toRadixString(16).padLeft(40, '0')}',
      name: s.title,
      sizeBytes: 0, createdUnix: 0,
      seeders: 0, leechers: 0, completed: 0, scrapedDate: 0,
      source: 'webstreamr',
      streamType: StreamType.directUrl,
      directUrl: s.url,
    )).toList();
    return {
      'torrents': torrents,
      'engineCounts': <String, int>{'webstreamr': torrents.length},
      'engineErrors': <String, String>{},
    };
  } catch (e) {
    return { 'torrents': <Torrent>[], 'engineCounts': <String, int>{},
             'engineErrors': <String, String>{'WebStreamr': e.toString()} };
  }
}
```

### Integration in `searchByImdbWithStremio()`
Added a third parallel future to the `Future.wait` call (no extra latency):
```dart
if (!isNonImdbContent) {
  searchFutures.add(
    _searchWebStreamr(
      imdbId: imdbId, isMovie: isMovie,
      season: season, episode: episode,
    ),
  );
}
```

**Deduplication strategy:** WebStreamr torrents use infohash key `ws:<url_hash>` — since they're `StreamType.directUrl`, they never collide with torrent infohashes from engines or Stremio addons. The existing `_deduplicateAndSort` method handles interleaving by stream type.

---

## Change 8: Stream Scope in Quick Play
**File modified:** `lib/screens/torrent_search_screen.dart`

### New state field
```dart
String _streamScope = 'all';  // Added near line 293
```

### Loaded alongside other prefs in `_searchTorrents()`
Added `StorageService.getStreamScope()` to the existing `Future.wait` batch (line ~2827). Set via:
```dart
_streamScope = streamScope;  // in setState
```

### Modified Quick Play source-selection logic (lines ~6246-6324)
The decision block now reads `_streamScope`:
- **`'all'`** (default): prefers debrid-cached torrents when a debrid provider is configured; falls back to direct streams. Same as original behavior.
- **`'direct'`**: forces `useDirectStreams = true`, `useTorrents = false`. Only direct streams are considered. If none exist, shows "No direct streams found".
- **`'debrids'`**: forces `useDirectStreams = false`. Only torrents are considered. Requires a debrid provider. If none or no torrents, shows appropriate error.

The "no playable sources" error messages are scope-aware:
```
Scope 'direct':  "No direct streams found for Quick Play"
Scope 'debrids': "No torrents found for Quick Play" / "Configure a debrid provider to play torrents"
Scope 'all':     original messages
```

---

## Change 9: Stream Scope in List View Filter
**File modified:** `lib/screens/torrent_search_screen.dart`

### Modified `_applyFiltersToList()` (line ~8337)
Added scope pre-filtering **before** the existing provider/quality filter chain:

```dart
final scope = _streamScope;
List<Torrent> scopeFiltered;
if (scope == 'direct') {
  scopeFiltered = source.where((t) => t.isDirectStream || t.isExternalStream).toList();
} else if (scope == 'debrids') {
  scopeFiltered = source.where((t) => !t.isDirectStream && !t.isExternalStream).toList();
} else {
  scopeFiltered = source;
}
```

Then `scopeFiltered` is used instead of `source` in all subsequent filtering (provider multi-select, quality, rip source, language filters).

This ensures the stream list the user sees in the search results always respects the scope setting.

---

## Change 10: Dependency Addition
**File modified:** `pubspec.yaml`

Added:
```yaml
dependencies:
  html: ^0.15.4
```
This is required by the WebStreamr engine for parsing HTML from scraped source pages. All other WebStreamr dependencies were already present in Debrify (`http`, `pointycastle`, `shared_preferences`, `cached_network_image`, `flutter_inappwebview`, `youtube_explode_dart`).

---

## Complete File Change Manifest

| File | Status | Lines | Description |
|------|--------|-------|-------------|
| `pubspec.yaml` | Modified | +1 | Added `html: ^0.15.4` |
| `lib/main.dart` | Modified | +5 | WebStreamr boot init |
| `lib/services/storage_service.dart` | Modified | +15 | `getStreamScope()` / `setStreamScope()` |
| `lib/services/torrent_service.dart` | Modified | +60 | `_searchWebStreamr()`, parallel future, imports |
| `lib/services/webstreamr_service.dart` | **New** | 276 | WebStreamr adapter |
| `lib/services/webstreamr_settings.dart` | **New** | ~200 | Settings prefs wrapper |
| `lib/models/stream_source.dart` | **New** | 39 | StreamSource, StreamResult models |
| `lib/screens/settings_screen.dart` | Modified | +100 | Scope selector, WS tile, grid cleanup |
| `lib/screens/settings/webstreamr_settings_page.dart` | **New** | ~300 | WebStreamr config UI |
| `lib/screens/torrent_search_screen.dart` | Modified | +65 | `_streamScope` state, Quick Play scope, list filter, home-tab crash fix |
| `lib/widgets/iptv/iptv_channel_tile.dart` | **New** | ~80 | IptvChannelListTile widget |
| `lib/widgets/iptv/iptv_results_view.dart` | Modified | +150 | Alive checker, list/grid toggle |
| `lib/webstreamr/` (73 files) | **New** | ~8000 | Ported engine (sources, extractors, utils) |
| `lib/webstreamr/stream_resolver.dart` | **New** | — | Core resolver |
| `lib/webstreamr/types.dart` | **New** | — | Context, Stream types |
| `lib/webstreamr/utils/` (9 files) | **New** | — | Cache, cookie jar, fetcher, env, id, semaphore, unpacker, mediaflow_proxy, config |
| `lib/webstreamr/source/` (22 files) | **New** | — | 20 sources + base class |
| `lib/webstreamr/extractor/` (27 files) | **New** | — | 25 extractors + base + registry |

---

## Key Design Decisions

### WebStreamr Fold-In Strategy
WebStreamr results are mapped to `Torrent(isDirectStream=true)` rows with `source: 'webstreamr'`. This lets them reuse all existing direct-stream infrastructure:
- **Stremio addon source chips** in the results list
- **Quick Play fallback** path (direct streams already work when no debrid)
- **In-player source switching** (the existing `SourceSheet` shows all `_torrents`)
- **No new UI components** needed for the stream list

### LocalServerService Skipped
The shelf-based HLS proxy from PlayTorrio was not ported. It was only used for one edge case: RG Shows / 1shows.app delivering MPEG-TS segments in a fake PNG container via TikTok CDN. The default `media_kit` video player can't decode this. Rather than adding shelf + a PNG-stripping middleware, we stub the URL directly. This means 1shows.app streams won't play, but all other WebStreamr sources work.

### Parallel Search Architecture
The engine search, Stremio addon search, and WebStreamr search all run via `Future.wait` in `searchByImdbWithStremio()`. The total wall-clock time is determined by the slowest source, not the sum. WebStreamr tends to be fast (direct URL scraping vs. torrent indexer queries).

### Scope vs. Provider Multi-Select
The Source Scope is a coarse filter (all/direct/debrids) applied before the existing fine-grained provider multi-select filter. This means:
- `scope = 'direct'` hides all torrent rows, then the provider filter only shows/hides among direct-stream providers
- `scope = 'debrids'` hides all direct-stream rows, then the provider filter only shows/hides among torrent providers
- The two filters compose naturally without conflict

---

## Change 11: Home-Tab Re-Entrant Build Crash Fix
**File modified:** `lib/screens/torrent_search_screen.dart`

### Bug
App crashed on launch / home tab with:
```
package:flutter/src/widgets/viewport.dart: Failed assertion: line 262 pos 12:
'!_doingMountOrUpdate': is not true.
```

### Root cause
`_setHomeSectionInitialLoading()` at line 22806 called `setState(() {})` **synchronously** from child-widget `onInitialLoadStateChanged` callbacks. These callbacks fire during the child's `initState()` / `build()` phase, which executes while the parent `ListView` viewport is still in its mount/update cycle. Calling `setState` during this phase triggers the `!_doingMountOrUpdate` assertion.

### Fix
```dart
// Before (torrent_search_screen.dart:22806):
setState(() {});

// After:
WidgetsBinding.instance.addPostFrameCallback((_) {
  if (mounted) setState(() {});
});
```

Deferred the `setState` to the next frame via `addPostFrameCallback`, so it never runs during build/layout.

---

## Known Limitations

1. **Custom HTTP headers** from WebStreamr sources are not passed to the video player. The Torrent model has no headers field, and neither does the `_playDirectStream()` flow. Stremio addon direct streams have the same limitation.
2. **1shows.app PNG container** streams are not playable (no LocalServerService proxy).
3. **GPL-2.0 licensing:** WebStreamr is GPL-2.0 code ported into a Polyform NC-licensed project. LICENSE/NOTICE file updates are pending. User explicitly accepted this.
4. **No unit tests** were added for the WebStreamr integration. The engine itself has no Dart tests (ported as-is from JavaScript).
5. ~~**Home-tab crash** — **FIXED** (see Change 11).~~
6. **TV remote focus** for the new `_StreamScopeSelector` segmented button may need D-pad wrapping (not tested on Android TV).

---

## Debug APK Build Verification
```
$ flutter analyze lib/
    → 0 errors, 0 warnings

$ flutter build apk --debug
    ✓ Built build/app/outputs/flutter-apk/app-debug.apk

$ flutter pub outdated
    91 packages have newer versions incompatible with dependency constraints.
    (all pre-existing, none introduced by our changes)
```

---

## Reference: Live Sports Section Architecture

### Overview
Self-contained single-file feature at `lib/screens/live_sports_screen.dart` (1831 lines). Three hardcoded third-party providers streamed via `InAppWebView` WebView players (no media_kit/VLC). No caching, no user settings, no persistence. Always visible as nav tab index 13.

### Providers

| Provider | API Endpoint | Content |
|----------|-------------|---------|
| Dami TV | `https://dami-tv.pro/papi/api/streams` (public) | Matches with home/away teams, badges, iframe embed |
| PPV.to | `https://old.ppv.to/api/streams` (public) | Streams with poster, tag, iframe embed |
| CDN Live TV | `https://api.cdn-live.tv/api/v1/channels/?user=cdnlivetv&plan=free` + `/events/sports/` | Channels + events grouped by sport (Soccer, NFL, NBA, NHL) |

### UI Layout
1. **Header** — `Icons.sports_soccer_rounded` + "Live Sports" + refresh button
2. **Provider bar** — `_ModeChip` widgets (Dami TV, PPV.to, Streamed, CDN Live)
3. **Sport tabs** — `TabBar` with "All" + dynamic categories from API response
4. **Body** — Responsive `GridView` (cross-axis count = `(width/300).floor().clamp(1, 6)`)
   - Dami TV: `_DamiTvMatchCard` — team badges (home vs away with VS divider), poster, league, category badge, time
   - PPV.to: `_PpvMatchCard` — poster with gradient, name, tag, time label, category badge
   - CDN channels: `_CdnChannelCard` — logo (CachedNetworkImage), name, viewer count, green LIVE badge
   - CDN events: `_CdnSportCard` — home/away logos with VS split, tournament, live/status badge
5. All cards: `StatefulWidget` with `_hovered` state, `MouseRegion` + `GestureDetector`, hover play overlay

### Tap Flow
- Dami TV / PPV.to: `Navigator.push` → `_DamiTvPlayerScreen` / `_PpvPlayerScreen` (loads `stream.iframe` in `InAppWebView`)
- CDN channel: `Navigator.push` → `_CdnPlayerScreen` (`channel.url` in `InAppWebView`)
- CDN event with multiple channels: bottom sheet (`_CdnChannelSheet`) for user to pick first

### Player Features (all 3 identical)
- `InAppWebView` with JS enabled, auto-play, inline playback
- Custom `shouldOverrideUrlLoading` — blocks external navigations, sends background GET with Referer instead
- Fullscreen via `SystemChrome` orientation lock
- Loading spinner overlay, transparent/black background

### Data Models (all private, file-local lines 10-233)
- `_Sport` — `id`, `name`
- `_PpvStream` — `id`, `name`, `tag`, `poster`, `uriName`, `startsAt`, `endsAt`, `alwaysLive`, `category`, `iframe`, `allowPastStreams`
- `_CdnChannel` — `name`, `code`, `url`, `image`, `status`, `viewers`
- `_CdnSportEvent` — `gameID`, `homeTeam`, `awayTeam`, `homeTeamIMG`, `awayTeamIMG`, `time`, `tournament`, `country`, `status`, `start`, `end`, `channels` (list)
- `_DamiTvStream` — `id`, `name`, `poster`, `startsAt`, `endsAt`, `categoryName`, `status`, `league`, `homeTeam`, `homeBadge`, `awayTeam`, `awayBadge`, `viewers`, `iframe`

### Provider bar
`_buildProviderBar` renders chips for all 4 providers: Dami TV, PPV.to, Streamed, CDN Live.

### Caching
None. Fresh HTTP requests every load. No StorageService, SQLite, or SharedPreferences involvement.

### Settings
None. No enable/disable, no favorites, no persistence. Provider selection is in-memory only (`_provider` field, defaults to `damiTv`). Tab is always visible via hardcoded nav index 13 in `main.dart`.

## Session 3 (Jun 25) — Crash fixed, Streamed provider added

**Crash diagnosis:**
- `viewport.dart:262 Failed assertion: '!_doingMountOrUpdate'` on home tab startup
- **Only fires in debug builds** — assertions compiled away in release builds
- Introduced by pre-release commit `1fb0471` (1 commit after v0.5.2-beta.1 tag) — ~7400 lines of new code triggered the assertion
- Confirmed: APK built from v0.5.2-beta.1 release tag works; HEAD commit crashes
- Release APK (`flutter build apk --release`) confirmed working on device

**What was done:**
- Added `FlutterError.onError` + `PlatformDispatcher.instance.onError` file logging to debug builds → writes to `/sdcard/Android/data/com.debrify.app/cache/flutter_error.log`
- Added `ErrorWidget.builder` to show full error + stack trace on-screen in debug builds
- `_setHomeSectionInitialLoading` deferred via `addPostFrameCallback`
- `LiveSportsScreen._load()` deferred via `addPostFrameCallback`

**Streamed provider (`streamed.pk`) added to Live Sports:**
- Model: `_StreamedMatch` with teams, badges, poster, category, sources
- Fetch: `GET /api/matches/live` → matches → `GET /api/stream/{source}/{id}` → embed URL
- Display: Grid cards with team badges, VS layout, Live/upcoming badges, category tags
- Playback: Fetches embed URL on tap, plays via `_CdnPlayerScreen` (InAppWebView)
- New chip "Streamed" in provider bar
- Categories extracted dynamically from API response

**For next time:** To fix debug builds, narrow down which HEAD-commit file triggers the assertion (likely `live_sports_screen.dart` or `streaming_details_screen.dart` init-side-effect during viewport mount). To ship, use `--release`.

---

### IPTV Relationship
Live Sports does not use IPTV channels. Separate sports IPTV channel matching exists in `lib/features/iptv/portal_search/data/hardcoded_channels.dart` (Sky Sports, ESPN, beIN, NBA, NFL, Cricket, etc.) used by the Portal Search feature (tab #14), not the Live Sports screen.

---

## Planned Live Sports Providers (to add)

The following websites were researched for potential addition as new providers in the Live Sports tab. Several have public JSON APIs making integration straightforward.

### Providers with Public APIs

- **Streamed** (`streamed.pk`): `GET /api/matches/live` → `GET /api/stream/{source}/{id}` → embed URL → [✅ Implemented]
- **DaddyLive** (`dlhd.pk` / `dlstreams.top`): `daddyapi.php?key=KEY&endpoint=channels` → iframe embed at `/{folder}/stream-{channel_id}.php`
- **LiveTV** (`livetvon.pk` / `livewatch.top`): `GET /api/v1/public/sports` → events with `embed_url`
- **EmbedSportex** (`api.embedsportex.site`): `GET /api/streams` → matches with `iframes[]` (multiple servers per match)
- **WeStream** (`westream.su`): `GET /matches/live` → `GET /stream/{source}/{id}` → embed URL (same pattern as Streamed)

### Sites to Research Further

- **SportyHunter** — needs URL/API research
- **StreamFree** — needs URL/API research
- **StreamSports99** — needs URL/API research
- **RoxieStreams** — needs URL/API research
- **SportsBite** (`sportsbite.online`) — status page with mirrors, needs API/embed research
- **WatchSports** — needs URL/API research
- **BINTV** — possibly beIN Sports, needs research

### Integration Strategy
Each provider follows a consistent pattern:
1. Fetch JSON from API endpoint
2. Parse into match/channel objects (title, teams, badges, league, iframe embed URL, status)
3. Display in existing `GridView` card layout
4. Play via `InAppWebView` with iframe URL (reuse existing `_PpvPlayerScreen` or create unified player)

The current per-provider architecture (separate fetch/load/build/player code per provider) will require code duplication. A future refactor could create a unified `_UnifiedStream` model and single card widget that all providers map to.

---

## Session 4 (Jun 25) — CDN Live chip + WebView sandbox bypass

**What was done:**
- Added missing **CDN Live chip** to `_buildProviderBar` in `live_sports_screen.dart` (~line 713)
- **Desktop User-Agent** (`Chrome/122 macOS`) set on all 3 InAppWebView players (`_CdnPlayerScreen`, `_PpvPlayerScreen`, `_DamiTvPlayerScreen`) to bypass anti-bot/sandbox detection
- Added `domStorageEnabled`, `databaseEnabled`, `mixedContentMode` to all 3 player WebView settings
- Added `allowExternalNavigation` parameter (default `false`) to `_CdnPlayerScreen` — Streamed passes `true` so `embed.st` can redirect to its video player without being blocked by `shouldOverrideUrlLoading`
- 0 analyzer errors

**Files modified:**
- `lib/screens/live_sports_screen.dart` — CDN Live chip, desktop UA + DOM storage on all players, `allowExternalNavigation` for Streamed

---

## Current Code Locations

| What | File | Notes |
|------|------|-------|
| Home tab (TorrentSearchScreen) | `lib/screens/torrent_search_screen.dart` | 26499 lines |
| Live Sports screen | `lib/screens/live_sports_screen.dart` | ~2100 lines, 4 providers |
| Live Sports — Dami TV | `lib/screens/live_sports_screen.dart` | `_fetchDamiTvStreams`, `_loadDamiTv` |
| Live Sports — PPV.to | `lib/screens/live_sports_screen.dart` | `_fetchPpvStreams`, `_loadPpv` |
| Live Sports — CDN Live | `lib/screens/live_sports_screen.dart` | `_fetchCdnChannels/Sports`, `_loadCdn` |
| Live Sports — Streamed | `lib/screens/live_sports_screen.dart` | `_fetchStreamedMatches`, `_loadStreamed` |
| Live Sports — Player (generic) | `lib/screens/live_sports_screen.dart` | `_CdnPlayerScreen` — InAppWebView playback |
| Crash debug logging | `lib/main.dart` | `FlutterError.onError` + file logger |
| Quick Play streaming mode | `lib/screens/torrent_search_screen.dart` | `_checkQuickPlayAfterSearch` |
| Settings — Streaming Mode | `lib/screens/settings_screen.dart` | `_StreamingModeSection` |

---

## Session 5 (Jun 27) — PPV.to Live Sports fix

**What was done:**
- Fixed PPV.to API endpoint: `api.ppv.to` → `old.ppv.to` (PlayTorrioV2 already used `old.ppv.to` — the `api.ppv.to` endpoint was deprecated/broken)
- Removed sandbox attribute from iframes via JS injection in `_PpvPlayerScreen.onLoadStop` — PPV.to embed pages wrap video in a sandboxed iframe that blocks playback; injected JS strips `sandbox` from all iframes on page load
- Debug crash logging (from Session 3, previously uncommitted): `FlutterError.onError` + `PlatformDispatcher.instance.onError` file logger in debug builds, `ErrorWidget.builder` for on-screen error display
- Quick Play Direct Streaming Mode toggle (from Session 2, previously uncommitted): `_checkQuickPlayAfterSearch` now reads `StorageService.getStreamingModeEnabled()` — when ON, prefers direct streams over debrid torrents
- Release APK built clean

**Files modified:**
- `lib/screens/live_sports_screen.dart` — PPV.to endpoint `api` → `old`, sandbox JS injection in `_PpvPlayerScreen`, removed CDN Live model + fetch dead code, replaced with Streamed provider
- `lib/main.dart` — Crash debug logging + `ErrorWidget.builder`
- `lib/screens/torrent_search_screen.dart` — Quick Play streaming mode toggle, home-tab `!_doingMountOrUpdate` crash fix
- `RobbdeezeNutz_Debriyf_Versions.md` — This entry
