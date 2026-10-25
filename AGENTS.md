# AGENTS.md: Chroma Canvas addon development

This file is the complete working contract for an AI agent that creates or reviews a Chroma Canvas addon. Follow it as a normative specification. Do not infer undocumented native APIs, Yandex Music internals, Android hooks, filesystem paths, tokens, or browser capabilities.

Updated for the Canvas development build on 2026-10-21, including the shared lyrics composition API. API version remains 1. Feature-detect `chroma.features?.lyrics === 1`, `chroma.features?.objects`, and `chroma.features?.namedMounts`; published older hosts may not contain these extensions. `minimumHostVersion: "1.0.0"` does not prove they exist. The current DOM matches the default native player, not a custom ChromaSync theme.

## 1. Objective and operating model

Build a ZIP addon for **Chroma Canvas API v1**. The archive contains a root `manifest.json`, one JavaScript entrypoint, optional CSS, optional image assets, and optional `settings.schema.json`.

An addon runs only while the full-screen Canvas is open. It is not an Android service, an LSPosed module, an ES module, or a background worker. It receives the public `chroma` SDK and its own `root` element.

The priorities are:

1. Correct behavior with the declared permissions.
2. Cleanup and stability when Canvas closes or reloads.
3. Honest handling of unavailable player/catalog data.
4. Low frame-time and battery cost.
5. Minimal permission and domain scope.
6. Clear user-visible failure states.

Never solve an addon task by reaching into obfuscated app classes or by installing hooks. That belongs to the host module, not the addon API.

## 2. Deliverables

For a complete addon, produce:

```text
addon-directory/
├── manifest.json
├── index.js
├── style.css              # optional
├── settings.schema.json   # optional
└── images/                # optional PNG/JPEG/WebP assets
```

Also provide:

- a short README explaining purpose, permissions, domains, settings, and known limits;
- the packed ZIP with `manifest.json` at archive root;
- a test checklist or automated browser-contract tests where useful;
- no generated dependency directory such as `node_modules` inside the ZIP.

Do not put the files inside an extra top-level directory in the archive.

## 3. Manifest contract

Example:

```json
{
  "id": "dev.example.now-playing",
  "name": "Now Playing",
  "version": "1.0.0",
  "apiVersion": 1,
  "type": "widget",
  "entrypoint": "index.js",
  "styles": ["style.css"],
  "permissions": ["player.read", "canvas.widget"],
  "networkDomains": [],
  "mounts": ["@player.afterMetadata"],
  "priority": 100,
  "minimumHostVersion": "1.0.0"
}
```

### 3.1 Fields

- `id`: required, stable, at most 80 characters. It must match `[a-z][a-z0-9]*(?:[.-][a-z0-9]+)+`. Use a reverse-domain-like value. Never change it merely to publish an update.
- `name`: required, 1-80 characters.
- `version`: required SemVer matching `major.minor.patch` with optional prerelease, for example `1.3.0` or `1.3.0-beta.2`.
- `apiVersion`: required integer `1`.
- `type`: required; see the type table below.
- `entrypoint`: required relative path to JavaScript. A CSS-only patch still needs a valid, possibly empty, JS entrypoint.
- `styles`: optional list of relative CSS paths.
- `permissions`: required list containing only known permission strings. Request the minimum set.
- `networkDomains`: optional list of exact DNS hostnames. Do not include scheme, path, port, wildcard, IP literal, `.local`, or `.localhost`.
- `remoteScripts`: optional list of at most four immutable remote JavaScript descriptors. Each descriptor must contain exactly `url`, `mimeType`, `sizeBytes`, and `sha256`.
- `mounts`: optional list of up to 10 distinct supported names. The first name selects the initial addon root; additional names authorize `canvas.createMount()` in full-access execution. Declaring names alone does not duplicate the addon or create additional roots. Allowed values depend on type; older hosts can accept only one name.
- `priority`: optional integer-like value used for ascending load order. Default is 100; ID is the tie-breaker.
- `minimumHostVersion`: optional but, in API v1, the only accepted value is exactly `1.0.0`.
- `dependencies`: unsupported. Omit it or use an empty array.

### 3.2 Package paths

Every package path must:

- be non-empty and at most 180 characters;
- be relative and use `/` separators;
- not start with `/`;
- not contain `\`, `:`, NUL, `.` segments, `..` segments, or empty segments;
- be unique case-insensitively inside the ZIP.

Do not rely on symlinks or path traversal. The installer stages and validates the package before activating its immutable content directory.

### 3.3 Addon types

| Type | Purpose | Required role permission | Allowed mounts | Execution realm |
| --- | --- | --- | --- | --- |
| `widget` | Inline player UI | `canvas.widget` | player/widget/lyrics mounts | sandbox iframe unless fullAccess |
| `tool` | Modal tool; no public launcher in the current UI | `canvas.widget` | player/widget/lyrics mounts | sandbox iframe unless fullAccess |
| `page` | Modal page; no public launcher in the current UI | `canvas.widget` | player/widget/lyrics mounts | sandbox iframe unless fullAccess |
| `effect` | Visual background/foreground layer | `canvas.overlay` | foregroundEffects/backgroundEffects | sandbox iframe |
| `style.patch` | Modify current host UI/CSS | `canvas.style` + `canvas.fullAccess` | player/widget/lyrics mounts | shared host DOM |
| `theme.primary` | Complete primary theme | `canvas.style` + `canvas.fullAccess` | player/widget/lyrics mounts | shared host DOM |

Only one `theme.primary` can be enabled. Installing or updating any package leaves it disabled until the user confirms its permissions.

### 3.4 Mounts

- `@player.afterMetadata`: host ID `player-extra-info`.
- `@player.actions`: host ID `canvas-widgets`.
- `@canvas.widgets`: host ID `canvas-widgets`.
- `@canvas.backgroundEffects`: host ID `canvas-background-effects`; effects only.
- `@canvas.foregroundEffects`: host ID `canvas-foreground-effects`; effects only.
- `@lyrics.overlay`: `lyrics-addon-overlay`, overlay in both lyrics presentations.
- `@lyrics.compact`: `lyrics-addon-compact`, overlay in compact lyrics only.
- `@lyrics.fullscreen`: `lyrics-addon-fullscreen`, overlay in fullscreen lyrics only.
- `@lyrics.beforeLines`: `lyrics-addon-before`, inside the scroll container before source rows.
- `@lyrics.afterLines`: `lyrics-addon-after`, inside the scroll container after source rows.
- `@lyrics.controls`: `lyrics-addon-controls`, controls over the bottom of the lyrics region.
- `@lyrics.miniPlayer`: `lyrics-addon-mini`, additional content in the fullscreen mini-player.

Default mount: foregroundEffects for an `effect`, canvas.widgets for other isolated types. A mount is not an arbitrary CSS selector.

Effects remain limited to the two effect mounts. Lyrics mounts are for the other types. An isolated addon can use a lyrics mount as its first mount, but it cannot access the host DOM or create a second root. Empty lyric mounts do not change the default layout. Overlay layers use `pointer-events: none` with interactive direct children; keep your own section transparent outside its actual controls. beforeLines/afterLines scroll with source rows, so use controls/overlay when content must stay fixed. The mini-player mount is visible only in fullscreen lyrics; it shares finite horizontal space with the title, like, and play controls.

### 3.5 Permissions

- `player.read`: `getState`, `getQueue`, player events, and all shared `chroma.lyrics` reads/events.
- `player.control`: player commands.
- `canvas.overlay`: required role permission for `effect`.
- `canvas.widget`: required role permission for `widget`, `tool`, and `page`.
- `canvas.style`: required role permission for `style.patch` and `theme.primary`.
- `canvas.fullAccess`: shared host DOM and trusted JavaScript. Required by style/theme types.
- `storage`: per-addon persistent JSON storage.
- `network`: constrained external HTTPS GET.
- `code.remote`: execution of manifest-pinned remote JavaScript. It is valid only together with `network` and non-empty `remoteScripts`.
- `yandex.read`: constrained native catalog GET and the second permission required by legacy `player.getLyrics()`. The shared `chroma.lyrics` API does not require it.
- `yandex.write`: reserved; all writes are currently rejected.
- `downloads`: external audio export. `network` is also required.

`canvas.fullAccess` is not a stronger sandbox. It removes isolation. Full-access code can manipulate the whole Canvas and may interfere with other enabled addons. Use it only when the requested feature cannot be built in the isolated `root`.

## 4. JavaScript execution and lifecycle

The host invokes the entrypoint as conceptually equivalent to:

```javascript
new Function('chroma', 'root', '"use strict";\n' + source)(sdk, root);
```

Consequences:

- no static `import` or `export`;
- no top-level await;
- bundle third-party code into the entrypoint, or declare immutable dependencies through `remoteScripts`;
- verified remote scripts execute in manifest order before the entrypoint;
- `chroma` and `root` are provided lexical parameters;
- code is strict mode;
- asynchronous initialization must use functions or an async IIFE;
- background execution after Canvas closes is unsupported.

Every allocated resource must be disposed:

```javascript
const controller = new AbortController();
const timer = setInterval(refresh, 10_000);
const observer = new ResizeObserver(layout);
observer.observe(root);
const unsubscribe = chroma.player.on('trackChanged', render);

chroma.canvas.onDispose(() => {
  controller.abort();
  clearInterval(timer);
  observer.disconnect();
  unsubscribe();
  root.replaceChildren();
});
```

Prefer event-driven rendering. Do not poll player state per frame. Use `requestAnimationFrame` only for actual animation, stop it while hidden, and honor `prefers-reduced-motion`.

Unhandled exceptions or unhandled Promise rejections can cause the addon to be removed from the current screen and disabled for the next run. The host can enter conservative safe mode after an unclean Canvas shutdown and disable all addons.

## 5. Public TypeScript-shaped API

This is the public contract of the current development SDK. Permission checks and compatibility rules below still apply; older hosts can lack the lyrics/object extensions. The declarations are synchronized with canvas-sdk/index.d.ts.

```ts
type Permission = 'player.read' | 'player.control' | 'canvas.overlay' | 'canvas.widget' | 'canvas.style' | 'canvas.fullAccess' | 'storage' | 'network' | 'code.remote' | 'yandex.read' | 'yandex.write' | 'downloads';
interface RemoteScript { url: string; mimeType: 'application/javascript' | 'text/javascript'; sizeBytes: number; sha256: string }
type Command = 'PLAY' | 'PAUSE' | 'TOGGLE_PLAYBACK' | 'NEXT' | 'PREVIOUS' | 'SEEK_TO' | 'SET_LIKED' | 'SET_DISLIKED' | 'CYCLE_REPEAT' | 'TOGGLE_SHUFFLE';
interface Manifest {
  id: string; name: string; version: string; apiVersion: 1;
  type: 'theme.primary' | 'style.patch' | 'effect' | 'widget' | 'tool' | 'page';
  entrypoint: string; styles?: string[]; permissions: Permission[];
  networkDomains?: string[]; remoteScripts?: RemoteScript[]; mounts?: string[]; priority?: number; minimumHostVersion?: '1.0.0';
}
/** Optional in older hosts and browser mocks. Availability describes the native action, not a DOM menu. */
interface PlayerHeader {
  revision: number;
  title: string; subtitle: string;
  casting: boolean; queueKind: 'history' | 'queue' | '';
  collapse: boolean; cast: boolean; queue: boolean;
  /** Ephemeral same-origin images. Their URL revision is not the action revision above. */
  collapseIcon?: string; castIcon?: string; queueIcon?: string;
}
interface PlayerState {
  available: boolean; playing: boolean; positionMs: number; durationMs: number; speed: number;
  queueRevision?: number;
  lyricsTrackKey?: string;
  track: { id: string; title: string; artists: string; album: string; cover: string | null };
  liked: boolean | null; disliked: boolean | null; repeatMode: 'None' | 'One' | 'All' | 'Disabled' | null; commands: Command[];
  /** Null when the native queue does not support shuffle. Never infer availability from track ID. */
  shuffleEnabled: boolean | null;
  capabilities: { queue: boolean; yandexAuthorized: boolean; lyrics: boolean; currentTrackDownload: boolean; repeat: boolean; shuffle: boolean; report: boolean; share: boolean; album?: boolean; artist?: boolean; menu?: boolean };
  header?: PlayerHeader;
  nativeBackground?: string;
  safeAreaTopDp?: number;
}
interface LyricLineRef { trackKey: string; revision: number; index: number }
interface LyricLine extends LyricLineRef {
  id: string; text: string;
  /** Original timestamps, before offsetMs. Plain text has null timestamps. */
  timeMs: number | null; endTimeMs: number | null;
}
interface LyricsPosition {
  trackKey: string; revision: number; positionMs: number; lyricPositionMs: number;
  durationMs: number; playing: boolean; activeIndex: number;
}
interface LyricsPresentation { open: boolean; fullscreen: boolean; manual: boolean; suspended: boolean }
interface LyricsActiveLine extends LyricsPosition { previous: LyricLine | null; current: LyricLine | null; next: LyricLine | null }
interface LyricsOptions {
  autoExpand: boolean; autoExpandDelayMs: number; fullFollowDelayMs: number; compactFollowDelayMs: number;
  fullAnchor: number; transitionMs: number; miniTransitionMs: number;
  focusMaxFontPx: number; focusFontWidth: number; focusHeight: number; focusGapPx: number; focusGapRatio: number;
  focusNeighborScale: number; focusNeighborOpacity: number; focusActiveOpacity: number; focusTransitionMs: number;
}
interface LyricsState extends LyricsPosition {
  trackId: string; status: 'idle' | 'checking' | 'available' | 'unavailable' | 'deferred';
  source: 'yandex' | 'lrclib' | null; recordId: number | null; stale: boolean; offsetMs: number;
  choice: 'auto' | 'yandex' | 'lrclib'; priority: 'yandex' | 'lrclib'; writers: string[];
  synced: boolean; lines: LyricLine[]; presentation: LyricsPresentation; options: LyricsOptions;
}
interface LyricsEvents {
  stateChanged: LyricsState; linesChanged: LyricsState; activeLineChanged: LyricsActiveLine;
  presentationChanged: LyricsPresentation; positionChanged: LyricsPosition;
  elementsChanged: { trackKey: string; revision: number; elementRevision: number };
}
type MountName = '@canvas.foregroundEffects' | '@canvas.backgroundEffects' | '@player.afterMetadata' |
  '@player.actions' | '@canvas.widgets' | '@lyrics.overlay' | '@lyrics.compact' | '@lyrics.fullscreen' |
  '@lyrics.beforeLines' | '@lyrics.afterLines' | '@lyrics.controls' | '@lyrics.miniPlayer';
interface CanvasObjects {
  /** Full access only. Includes current dynamic row IDs; query('*') accesses unnamed descendants too. */
  list(): Array<{ id: string; tag: string }>;
  get(id: string): HTMLElement | SVGElement | null;
  query(selector: string): Element[];
  /** CSS property names use kebab-case. Requires canvas.style; disposal restores preceding styles. */
  style(id: string, properties: Record<string, string>): () => void;
  on(id: string, event: string, callback: (event: Event) => void, options?: AddEventListenerOptions): () => void;
}
interface LyricsAPI {
  /** Shared host cache, not a provider request. Every result is detached; requires player.read. */
  getState(): Promise<LyricsState>; getLines(): Promise<LyricLine[]>;
  getLine(index: number): Promise<LyricLine | null>;
  getActiveLine(): Promise<LyricsActiveLine>; getPosition(): Promise<LyricsPosition>;
  /** Future changes only; use getState for initial rendering. Unsubscribe/disposal stops delivery. */
  on<E extends keyof LyricsEvents>(event: E, callback: (payload: LyricsEvents[E]) => void): () => void;
  open(): Promise<boolean>; close(): Promise<boolean>; expand(): Promise<boolean>; collapse(): Promise<boolean>; follow(): Promise<boolean>;
  /** Fullscreen browsing without seeking. References are rejected after a track/text replacement. */
  scrollToLine(line: LyricLineRef): Promise<boolean>;
  /** Requires player.control; applies host offset, does not start paused playback. */
  seekToLine(line: LyricLineRef): Promise<{ accepted: true }>;
  /** Full access only. Returns the source row and assigned compact focus copies; [] when closed. Nodes can be replaced. */
  getLineElements(line: LyricLineRef): HTMLElement[];
  /** Requires player.read + canvas.style + canvas.fullAccess. Last live lease wins per option. */
  configure(options: Partial<LyricsOptions>): () => void;
  /** Refit after changing fonts or CSS without a new track. No provider requests. */
  refreshLayout(): void;
  /** Exclusive renderer; retains host controls, timing and loading. Must render synchronously.
   * Called on state/line changes, not every tick; positionChanged is opt-in for continuous effects.
   * Releasing, unloading or an error restores the default renderer. Host owns the supplied root. */
  registerRenderer(renderer: { render(state: LyricsState, root: HTMLElement): void; dispose?(): void }): () => void;
}
interface ChromaSDK {
  apiVersion: 1;
  features: { lyrics: 1; objects: boolean; namedMounts: boolean };
  lyrics: LyricsAPI;
  player: {
    getState(): Promise<PlayerState>;
    /** Requires player.read + yandex.read. Network failure is not proof lyrics are absent. */
    getLyrics(): Promise<{ trackId: string; lines: Array<{timeMs: number | null; text: string}>; writers: string[] }>;
    getQueue(options?: { offset?: number; limit?: number }): Promise<Array<{ id: string; queueId: string; title: string; artists: string; artworkAvailable: boolean }>>;
    getQueueArtwork(queueId: string): Promise<{ queueId: string; url: string; width: number; height: number }>;
    command(command: Command, args?: { positionMs?: number; value?: boolean }): Promise<{ accepted: true }>;
    on(event: 'stateChanged' | 'trackChanged' | 'playbackChanged' | 'queueChanged', callback: (state: PlayerState) => void): () => void;
  };
  canvas: { root: HTMLElement; mount(): HTMLElement;
    /** Full access only; creates an owned section in a mount declared in manifest.mounts. */
    createMount(name: MountName): HTMLElement; objects: CanvasObjects;
    asset(path: string): Promise<string>; showMessage(text: string): void; onDispose(callback: () => void): void; settings: Record<string, unknown>; addButton(label: string, action: () => void): () => void };
  storage: { get<T = unknown>(key: string): Promise<T | null>; set(key: string, value: unknown): Promise<null> };
  network: { request(options: { url: string; method?: 'GET' }): Promise<{ status: number; contentType: string; body: string }> };
  /** Catalog GET routes only; credentials stay native and must first be observed from the active app session. */
  yandex: { request(options: { path: string; method?: string; query?: Record<string, string> }): Promise<unknown> };
  downloads: { enqueue(options: { url: string; filename: string }): Promise<{ saved: boolean; filename: string; bytes: number }> };
  log(...values: unknown[]): void;
}
```

## 6. Player state and commands

### 6.1 State

Always handle missing or nullable values. The MediaSession published by Yandex Music is authoritative. The host does not invent queue, rating, cover, repeat, or command support.

```javascript
async function renderCurrent() {
  const state = await chroma.player.getState();
  root.dataset.available = String(state.available);
  title.textContent = state.track?.title || 'Нет трека';
  artist.textContent = state.track?.artists || '';
  like.disabled = !state.commands.includes('SET_LIKED');
}
```

Critical identity rule: `state.track.id` is the current **MediaSession item identity**. It can be a UUID, opaque string, local item ID, or numeric-looking value. It is not a guaranteed Yandex catalog track ID. Never use `split(':')[0]`, numeric coercion, or regex success as proof of catalog identity.

`header.title` and `header.subtitle` describe the playback source (Wave, playlist, album, etc.), not the current track. Both can be empty; preserve the distinction from `track.title` and do not substitute a song title as the source. `header.queueKind` chooses native history versus upcoming queue; `header.queue` means the header action is available, whereas `capabilities.queue` means the queue-reading SDK is available. They are not interchangeable.

`header.collapse`, `header.cast`, `header.queue`, and `capabilities.album/artist/menu` describe availability only. They do not introduce callable SDK methods. Treat absent fields as unavailable. Header icon URLs, like cover URLs, are temporary, screen-scoped resources; never persist or construct them. `header.revision` is an opaque action revision, not the revision in an icon URL.

`nativeBackground`, when present, is a native sampled `#RRGGBB` background. The host applies it in the base CSS layer; a theme can override it in its own layer. `safeAreaTopDp` is the host-provided top inset; avoid adding a second status-bar inset on top of the existing header.

### 6.2 Events

- `stateChanged`: any published state update.
- `trackChanged`: `track.id` changed.
- `playbackChanged`: emitted for every SDK state update, not only boolean play/pause transitions. Make rendering idempotent.
- `queueChanged`: queue revision changed.

The callback receives a cloned `PlayerState`. Store only what is needed. Subscribe once and unsubscribe in `onDispose`.

Native callback bursts may be coalesced to the latest state. Events are not a per-frame clock or a complete playback history. `getState()` returns the latest SDK snapshot, not a fresh network request. The host interpolates progress locally and schedules only visible changes; addons must not depend on the host's frame rate. If an animation needs interpolation, anchor its own monotonic clock on new state and honor pause, seek, speed, visibility, and disposal.

### 6.3 Queue

`getQueue({offset, limit})` accepts offset 0-1000 and limit 1-50. Offset is relative to the current queue item. It can fail with `PLAYER_UNAVAILABLE`, `QUEUE_UNAVAILABLE`, or `QUEUE_POSITION_UNKNOWN`. A queue item `id` has the same identity caveat as `track.id`.

Each queue item also contains `artworkAvailable`. If true, call `getQueueArtwork(queueId)` to request the image. The host resolves only an item that still exists in the current MediaSession queue, obtains its published bitmap/URI or, when the addon also has `yandex.read`, a catalog cover through the internally resolved catalog identity, bounds the download and decoded dimensions, scales it to at most 768 px, and returns a same-origin ephemeral URL. The catalog identity and remote source URL are not exposed to addon JavaScript.

The returned URL belongs to the current Canvas screen only. Do not persist it. Catch `QUEUE_ITEM_NOT_FOUND`, `QUEUE_ARTWORK_UNAVAILABLE`, `QUEUE_ARTWORK_URL_DENIED`, `QUEUE_ARTWORK_TYPE`, `QUEUE_ARTWORK_TOO_LARGE`, `QUEUE_ARTWORK_INVALID`, and catalog/network errors. Use a generation counter so an old response cannot update a new queue:

```javascript
let generation = 0;
async function loadNextCover() {
  const request = ++generation;
  const [next] = await chroma.player.getQueue({ offset: 1, limit: 1 });
  if (!next?.artworkAvailable) return null;
  const artwork = await chroma.player.getQueueArtwork(next.queueId);
  return request === generation ? artwork : null;
}
const offTrack = chroma.player.on('trackChanged', () => loadNextCover().catch(() => {}));
const offQueue = chroma.player.on('queueChanged', () => loadNextCover().catch(() => {}));
chroma.canvas.onDispose(() => { generation++; offTrack(); offQueue(); });
```

### 6.4 Commands

Before every command, check `state.commands`. Supported names and args:

- `PLAY`, `PAUSE`, `TOGGLE_PLAYBACK`, `NEXT`, `PREVIOUS`, `CYCLE_REPEAT`, `TOGGLE_SHUFFLE`: no args.
- `SEEK_TO`: `{positionMs}`; value must be non-negative and no greater than known duration.
- `SET_LIKED`, `SET_DISLIKED`: `{value: boolean}`.

`{accepted: true}` confirms transport submission only. Wait for the subsequent state callback to update UI. Do not optimistically claim success unless a reversible visual pending state is used.

Shuffle is queue-dependent, not track-ID-dependent. In Wave it is normally unavailable. Hide a custom shuffle button when `capabilities.shuffle !== true`, disable it when `TOGGLE_SHUFFLE` is absent from `commands`, and set its active state only when `shuffleEnabled === true`. `null` means unknown/unsupported, not "off". Recheck support when pressed and after queue changes:

```javascript
function renderShuffle(state) {
  shuffle.hidden = state.capabilities?.shuffle !== true;
  shuffle.disabled = shuffle.hidden || !state.commands?.includes('TOGGLE_SHUFFLE');
  shuffle.setAttribute('aria-pressed', String(state.shuffleEnabled === true));
}
async function toggleShuffle() {
  try {
    const state = await chroma.player.getState();
    if (!state.capabilities?.shuffle || !state.commands?.includes('TOGGLE_SHUFFLE')) return;
    await chroma.player.command('TOGGLE_SHUFFLE');
  } catch (error) { chroma.log(error.message); }
}
```

The example assumes an addon-owned `shuffle` button and requires `player.read` + `player.control` in addition to the type's role permission. Subscribe once to `stateChanged`, initialize with `getState`, and clean up both the subscription and click listener.

### 6.5 Lyrics

Use `chroma.lyrics` for the lyrics actually selected by Canvas, including Yandex and LRCLIB, provider overrides, user offset, cached results, compact focus, and fullscreen presentation. It reads the existing host cache and uses the existing playback clock. It does not start a second provider request, create its own prefetch queue, change the user's source choice, or require `network`/`yandex.read`. Provider loading and queue prefetch remain host responsibilities.

#### Feature detection and permissions

```javascript
if (chroma.features?.lyrics !== 1 || !chroma.lyrics) {
  root.textContent = 'Update ChromaSync to use this addon';
} else {
  // Initialize once; put asynchronous work inside a function or async IIFE.
}
```

| Operation | Required permissions in addition to the addon type |
| --- | --- |
| getState/getLines/getLine/getActiveLine/getPosition/on | `player.read` |
| open/close/expand/collapse/follow | `player.read` and at least one of `canvas.widget`, `canvas.style`, `canvas.fullAccess` |
| scrollToLine | `player.read` and `canvas.widget` or `canvas.style` |
| seekToLine | `player.read` + `player.control`, plus current native `SEEK_TO` support |
| getLineElements | `player.read` + `canvas.fullAccess` |
| configure/refreshLayout/registerRenderer | `player.read` + `canvas.style` + `canvas.fullAccess` |
| canvas.objects.list/get/query/on | `canvas.fullAccess` |
| canvas.objects.style | `canvas.fullAccess` + `canvas.style` |
| canvas.createMount | `canvas.fullAccess` and the mount listed in this manifest |

Do not request full access just to read lines or draw a separate isolated widget. `features.objects` and `features.namedMounts` describe this execution realm, not permission to invoke every method. A full-access `widget` is supported: keep its `canvas.widget` role permission and add the permissions needed for trusted host modification.

#### Snapshot and timing

`getState()` returns a detached `LyricsState`. Modifying the returned object never edits the host's text or settings. It contains:

| Fields | Meaning |
| --- | --- |
| trackKey, revision | Opaque lyrics-document identity. Keep both with line references; do not persist across documents. |
| trackId | Host-resolved lyrics identity. It can be empty; do not replace it with MediaSession `track.id`, parse it, or infer a catalog ID from it. |
| status | `idle`, `checking`, `available`, `unavailable`, or `deferred`. Only unavailable confirms absence; deferred is temporary. |
| source, recordId | Selected `yandex`/`lrclib` source or null, plus selected provider record ID or null. |
| stale | Cached content marked stale by the host; it can still be displayed while refreshing. |
| choice, priority | User source choice `auto`/`yandex`/`lrclib` and source priority `yandex`/`lrclib`. Read-only. |
| offsetMs | User timing correction, in milliseconds. Positive offset delays a row's activation. |
| writers, synced, lines | Credits, whether any row is timed, and the complete ordered array. |
| positionMs, lyricPositionMs, durationMs, playing, activeIndex | Shared playback snapshot. `lyricPositionMs = positionMs - offsetMs`. `activeIndex` may be -1. |
| presentation | `open`, `fullscreen`, `manual`, `suspended` booleans. Suspended means the host is not actively presenting; stop your animation. |
| options | Effective presentation options after active addon leases, including defaults. |

`getPosition()` returns only identity/timing/playback/index; `getActiveLine()` adds previous/current/next (each nullable). Before the first timed row, current is null and next can be the first row. `getLines()` returns all rows. `getLine(index)` takes a zero-based integer, returns null when the index is outside the array, and rejects non-integers. These reads also work when the lyrics panel is closed; DOM access does not imply that rows are currently mounted.

Every row is `{trackKey, revision, index, id, text, timeMs, endTimeMs}`. `id` is a document-local row key, not an HTML ID. `timeMs` is the original provider timestamp before offset. `endTimeMs` is the next distinct timestamp, or null for the final interval. Duplicate-timestamp rows share the same end. Plain unsynchronized rows have null timeMs/endTimeMs; do not invent timestamps, highlight a fabricated active row, or enable seeking for them. Timed activation uses the host index; do not recompute it from labels or array position.

Track/text/source/record replacement invalidates prior references, even when the visible track name matches. Offset-only updates and equivalent data refreshes preserve document references. `seekToLine`, `scrollToLine`, and `getLineElements` validate trackKey + revision + index and reject stale references with `LYRICS_STALE_LINE`. Never send `{index}` alone. `id` or index alone cannot identify a line across tracks.

#### Events and initialization

`on(event, callback)` returns an unsubscribe function. Events describe future changes, so subscribe once and read the initial state. Each callback receives its own detached payload. Never mutate another listener's state or depend on one callback per frame.

| Event | Payload and trigger |
| --- | --- |
| stateChanged | Full state on document/status/options/presentation/playback-flag/active-line/element changes; not every position tick. |
| linesChanged | Full state when document revision changes; not just an offset change. |
| activeLineChanged | Timing plus previous/current/next when active index or document revision changes. |
| presentationChanged | Only open/fullscreen/manual/suspended flags. |
| positionChanged | Small timing snapshot for opt-in continuous effects; delivery is suspended with the host, frequency is not a contract. |
| elementsChanged | `{trackKey, revision, elementRevision}` after source rows or compact copies change. Reacquire DOM references. |

Avoid an initial read overwriting a later event:

```javascript
const value = document.createElement('p');
root.append(value);
let disposed = false;
let changes = 0;
function render(state) {
  value.textContent = state.status === 'available'
    ? (state.lines[state.activeIndex]?.text || '') : '';
}
if (chroma.features?.lyrics === 1) {
  const off = chroma.lyrics.on('stateChanged', state => {
    changes++;
    render(state);
  });
  const initial = changes;
  chroma.lyrics.getState().then(state => {
    if (!disposed && initial === changes) render(state);
  }).catch(error => chroma.log(error.message));
  chroma.canvas.onDispose(() => { disposed = true; off(); value.remove(); });
}
```

Prefer activeLineChanged for a line counter and stateChanged for complete renderers. Subscribe to positionChanged only when smooth progress/karaoke needs it; do not fetch getState/getLines on each position event. Cache the current lines until linesChanged. Provider diagnostics/auth details are not included in the shared snapshot; do not show raw service errors to end users.

#### Presentation and line actions

- `open()` opens available lyrics without forcing the host button enabled; false is a normal result if unavailable or suspended.
- `close()` closes lyrics and returns to the cover; it does not close Canvas.
- `expand()` / `collapse()` change fullscreen lyrics versus compact lyrics, not the whole player.
- `follow()` ends manual browsing and resumes following the active row.
- `scrollToLine(ref)` browses a mounted row only while lyrics are open and fullscreen. It returns false otherwise; it does not seek, play, or force-open lyrics.
- `seekToLine(ref)` computes `max(0, timeMs + offsetMs)` and clamps to known duration through the native command. It does not start paused playback. `{accepted:true}` means command submission, not confirmed playback position.

Catch Promise failures and synchronous DOM/configuration exceptions separately. Reacquire a line after LYRICS_STALE_LINE; do not retry with its index on a different document. Disable timed actions for `timeMs === null` and check current `player.commands` for SEEK_TO.

#### Reversible configuration

`configure(partialOptions)` is synchronous and returns a release function. Every property is checked atomically; invalid/unknown/non-finite values reject the entire call with LYRICS_OPTIONS. Only supplied properties change. The last live lease wins per property; release restores the preceding lease or built-in default. Closing/unloading cleans owned leases, but explicit release is useful for live toggles. It does not persist user preferences; store your own settings through the normal storage/settings contract.

| Option | Default | Allowed value | Purpose |
| --- | --- | --- | --- |
| autoExpand | true | boolean | Automatic fullscreen lyrics while playing. |
| autoExpandDelayMs | 10000 | 0..120000 | Automatic expansion delay. |
| fullFollowDelayMs | 5000 | 0..120000 | Return to active row after fullscreen manual browsing. |
| compactFollowDelayMs | 8000 | 0..120000 | Return after compact manual browsing. |
| fullAnchor | 0.12 | 0..1 | Active row's vertical anchor within scroll viewport. |
| transitionMs | 320 | 0..5000 | Presentation transition duration. |
| miniTransitionMs | 280 | 0..5000 | Mini-player track transition duration. |
| focusMaxFontPx | 96 | 4..256 | Compact focus font ceiling. |
| focusFontWidth | 0.22 | 0.01..1 | Font-size ratio against compact stage width. |
| focusHeight | 0.62 | 0.1..1 | Height budget for compact focus fitting. |
| focusGapPx | 6 | 0..128 | Minimum compact neighbor gap. |
| focusGapRatio | 0.03 | 0..0.5 | Neighbor gap relative to compact stage height. |
| focusNeighborScale | 0.76 | 0.1..2 | Previous/next compact row scale. |
| focusNeighborOpacity | 0.32 | 0..1 | Previous/next compact row opacity. |
| focusActiveOpacity | 1 | 0..1 | Current compact row opacity. |
| focusTransitionMs | 380 | 0..5000 | Compact line animation duration. |

```javascript
const release = chroma.lyrics.configure({
  autoExpand: false,
  fullFollowDelayMs: 7000,
  focusMaxFontPx: 110,
  focusNeighborOpacity: 0.45
});
chroma.canvas.onDispose(release);
```

Use CSS for fonts, colors, masks, spacing, and mini-player layout (section 8.5). Call synchronous `refreshLayout()` once after font loading or geometry changes. Do not call it on each playback event. A CSS override of focus font size can supersede the measured fit; test long multiline rows and consistent wrapping before/after activation.

#### Source rows and compact copies

`getLineElements(ref)` synchronously returns the current source row plus any compact focus copies currently assigned to this row. It returns [] when lyrics are closed. Nodes can be hidden by the active presentation or custom renderer; presence is not proof of visibility. A compact text node is reused for other rows. The source row HTML ID is `lyrics-line-{index}`, with zero-based data-line-index and provider data-time-ms when timed. Source row classes active/near/unsynced and data-distance are host-owned.

Do not retain returned nodes as permanent line identity. Reacquire on elementsChanged, linesChanged, and presentation changes. Read line identity from the API and use DOM only for styling. Example highlighting the current row/copies with reversible leases:

```javascript
let released = [];
let generation = 0;
let disposed = false;
function clear() { for (const undo of released.splice(0)) undo(); }
async function repaint() {
  const request = ++generation;
  const { current } = await chroma.lyrics.getActiveLine();
  if (disposed || request !== generation) return;
  clear();
  if (!current) return;
  try {
    for (const node of chroma.lyrics.getLineElements(current)) {
      if (node.id) released.push(chroma.canvas.objects.style(node.id, {
        'text-shadow': '0 0 12px var(--canvas-accent)'
      }));
    }
  } catch (error) {
    if (error.message !== 'LYRICS_STALE_LINE') throw error;
  }
}
const refresh = () => { repaint().catch(error => chroma.log(error.message)); };
const off = chroma.lyrics.on('elementsChanged', refresh);
const offLine = chroma.lyrics.on('activeLineChanged', refresh);
const offView = chroma.lyrics.on('presentationChanged', refresh);
refresh();
chroma.canvas.onDispose(() => {
  disposed = true; generation++; off(); offLine(); offView(); clear();
});
```

This needs player.read + canvas.style + canvas.fullAccess. The helper owns only text-shadow, not the row text or active class. Prefer CSS classes/selectors when no row-specific computation is needed.

#### Custom renderer

`registerRenderer({render(state, root), dispose?()})` is synchronous and exclusive. Only one renderer can own lyrics; a second registration throws LYRICS_RENDERER_BUSY. It returns a release function. The host keeps provider selection, loading, shared timing, native controls, mini-player progress, and presentation transitions. The supplied renderer root is host-owned and removed on release. The default scroll/focus content becomes hidden rather than destroyed.

`render` is called for meaningful state changes, not each clock tick. Handle status unavailable/checking/deferred, activeIndex -1, empty lines, untimed rows, compact/fullscreen, and rapid document changes. Build nodes using textContent. Returning a Promise is unsupported (LYRICS_RENDERER_ASYNC); fetch or prepare assets outside render, then update your owned UI. A thrown renderer error restores the default and follows normal addon fault handling. Optional dispose is called during release. Do not replace the host root or call registerRenderer on every state event.

For continuous effects, subscribe to positionChanged separately and keep a small cache keyed by trackKey/revision. Use offset-corrected lyricPositionMs and original line timestamps, handle endTimeMs null, and stop on pause/suspension/disposal. No word-level timestamps are supplied; a line-progress effect must not pretend it has true word timing.

The complete runnable Lyrics Workbench example is listed in section 24. It demonstrates an explicit user toggle, shared lines, source/counter display, timed seeking, a second mini-player mount, and disposal. Its own style is a teaching example; default Canvas lyrics remain unchanged until the addon toggle is used.

#### Legacy direct Yandex API

`chroma.player.getLyrics()` remains supported with player.read + yandex.read. It performs the existing direct Yandex lyrics path and returns `{trackId, lines:[{timeMs,text}], writers}`. It does not select LRCLIB or reflect the shared Canvas document/presentation. Use it only for a deliberate Yandex-specific tool, catch errors, and discard stale results after track changes. Do not call it to redraw shared lyrics or on every playback tick.

`player.capabilities.lyrics` means the host can attempt lyrics resolution; it does not prove text exists. For new addons use `lyrics.getState().status` and events. DOM `#lyrics[data-availability]` values remain idle/checking/available/unavailable/error; shared status maps a temporary error to deferred. Do not force-enable disabled controls or interpret a temporary provider failure as confirmed absence.

## 7. Creating UI objects

For isolated addons, all UI belongs under the supplied `root`.

```javascript
const card = document.createElement('article');
card.className = 'addon-card';

const heading = document.createElement('h3');
const value = document.createElement('p');
const action = document.createElement('button');
action.type = 'button';
action.textContent = 'Обновить';

card.append(heading, value, action);
root.append(card);

action.addEventListener('click', refresh);
chroma.canvas.onDispose(() => {
  action.removeEventListener('click', refresh);
  card.remove();
});
```

Rules:

- use semantic elements and `aria-label`/`aria-pressed` where needed;
- set untrusted strings via `textContent`, never `innerHTML`;
- avoid fixed pixel widths that break on narrow phones;
- support long titles and large font scale;
- do not copy host buttons and expect native handlers to follow;
- do not attach events repeatedly on every state render;
- keep rendering idempotent;
- use CSS classes rather than rewriting large `style.cssText` strings per callback.

`chroma.canvas.addButton(label, action)` is a convenience that appends a standard button to the addon root and returns a removal function:

```javascript
const removeButton = chroma.canvas.addButton('Пауза', async () => {
  try {
    await chroma.player.command('PAUSE');
  } catch (error) {
    chroma.canvas.showMessage(error.message);
  }
});
chroma.canvas.onDispose(removeButton);
```

For `tool` and `page`, the runtime hides the root and retains modal plumbing, but the current UI has no public Open launcher. The old custom track-details popup has been removed. Do not ship a tool/page whose only entrypoint relies on that popup or undocumented runtime functions. Use `widget` for reachable inline UI; wait for a documented launcher API if a separate page is essential.

## 8. Modifying default Canvas objects

Only trusted full-access execution can access the host document. `style.patch` and `theme.primary` require canvas.style + canvas.fullAccess; a widget/tool/page can also explicitly request fullAccess while retaining its role permission. Styling through the object API additionally requires canvas.style. Isolated iframe addons cannot query host IDs even though the names are documented.

Prefer CSS-only modification:

```css
#player-cover-region { border-radius: 28px; }
#play { box-shadow: 0 14px 38px color-mix(in srgb, var(--canvas-accent) 30%, transparent); }
#player-controls { gap: 18px; }
```

If JavaScript modification is required, make it reversible:

```javascript
const controls = document.getElementById('player-controls');
if (!controls) throw new Error('PLAYER_CONTROLS_NOT_FOUND');

const badge = document.createElement('span');
badge.className = 'addon-controls-badge';
badge.textContent = 'CUSTOM';
controls.append(badge);

chroma.canvas.onDispose(() => badge.remove());
```

Do not:

- remove or replace host buttons;
- overwrite host `onclick` properties;
- patch global prototypes;
- replace `document.body` or `#chroma-canvas`;
- move host elements without restoring them;
- read another addon's private iframe DOM;
- assume a selector exists without null checking;
- rely on unlisted internal class names.

### 8.1 Object discovery and reversible helpers

Feature-detect `chroma.features?.objects`. The object API is synchronous and scoped to `#chroma-canvas` and its descendants:

- `chroma.canvas.objects.list()` returns `{id, tag}[]` for the root and every current ID-bearing descendant, including dynamic lyric rows and addon-created objects with IDs.
- `objects.get(id)` takes the exact ID without `#`, returns an HTMLElement/SVGElement or null; objects outside Canvas are not returned.
- `objects.query(selector)` returns all matching descendants, including unnamed nodes; it does not include the root itself. Invalid CSS selectors throw synchronously. Do not assume every returned node is an HTMLElement.
- `objects.style(id, properties)` takes kebab-case CSS property names or custom properties and string values. At most 128 properties, at most 2048 characters per value. It returns an idempotent release function, refreshes lyrics layout, and restores previous property values/priorities after release. Overlapping addon leases restore the last still-live value, even when unloaded out of order.
- `objects.on(id, event, callback, options?)` adds a listener and returns an idempotent unsubscribe function. The host also removes owned listeners during addon disposal. It does not overwrite native handlers.

Missing style/event targets throw CANVAS_OBJECT_MISSING; malformed style input throws CANVAS_STYLE_INVALID. Do not repeatedly acquire style leases on each position event: release the prior lease first or update a small owned node. Raw DOM writes/listeners are still possible in fullAccess but are your cleanup responsibility; the host cannot automatically revert arbitrary mutations, replaced nodes, prototypes, or global state.

```javascript
if (!chroma.features?.objects) throw Error('OBJECT_SDK_REQUIRED');
const catalog = chroma.canvas.objects.list();
const rows = chroma.canvas.objects.query('.lyrics-line');
const play = chroma.canvas.objects.get('play');
let restore = () => {};
let offClick = () => {};
if (play) {
  restore = chroma.canvas.objects.style('play', {
    'border-radius': '18px',
    'box-shadow': '0 0 16px var(--canvas-accent)'
  });
  offClick = chroma.canvas.objects.on('play', 'click', () => {
    // Observe the user's click; keep the original playback handler.
  });
}
chroma.canvas.onDispose(() => { offClick(); restore(); });
```

For multiple placement points, declare them in manifest.mounts, check features.namedMounts, then call `canvas.createMount(name)`. It returns a new owned section at that declared point, not the host mount container. The host removes this section on unload; remove your listeners/observers as usual. The original `root` and `canvas.mount()` remain the initial addon root. A call can throw MOUNT_NOT_DECLARED or MOUNT_UNSUPPORTED. Do not attach ad hoc nodes outside Canvas to bypass mount/lifecycle ownership.

### 8.2 Public host IDs

```text
chroma-canvas
player-background
canvas-background-effects
player-header
player-collapse
header-collapse-icon
player-source
player-source-title
player-source-subtitle
player-header-actions
player-cast
header-cast-icon
player-native-queue
header-queue-icon
player-content
player-cover-region
cover-art
player-cover
player-lyrics
lyrics-heading
lyrics-dot
lyrics-mode
lyrics-provider
lyrics-scroll
lyrics-state
lyrics-retry
lyrics-addon-before
lyrics-lines
lyrics-addon-after
lyrics-credit
lyrics-focus-stage
lyrics-focus-previous
lyrics-focus-text
lyrics-focus-next
lyrics-custom
lyrics-addon-overlay
lyrics-addon-compact
lyrics-addon-fullscreen
lyrics-addon-controls
lyrics-follow
lyrics-options
lyrics-contract
player-metadata
player-title
album-title
player-artists
artist-name
share
details
player-extra-info
player-progress
seek
position
duration
player-controls
dislike
previous
play
play-glyph
next
like
player-actions
repeat
repeat-one
lyrics
report
shuffle
canvas-widgets
status
canvas-tools
canvas-close
addons
addons-disable
lyrics-mini-player
lyrics-mini-progress
lyrics-mini-track
lyrics-mini-cover
lyrics-mini-title
lyrics-mini-artist
lyrics-mini-like
lyrics-mini-play
lyrics-mini-play-glyph
lyrics-addon-mini
canvas-foreground-effects
canvas-modals
modal-title
modal-close
modal-body
```

Dynamic source-row IDs are lyrics-line-0, lyrics-line-1, and so on. Reacquire them after elementsChanged. These HTML IDs differ from LyricLine.id. SVG definitions outside #chroma-canvas are listed separately below.

### 8.3 Public classes and state selectors

```text
.svg-defs
.player-header-row
.header-button
.metadata-link
.cover-placeholder
.track-heading
.track-names
.icon
.metadata-icon
.times
.skip
.reaction
.effect-layer
.addon-frame
.list-item
.sr-only
.lyrics-heading
.lyrics-dot
.lyrics-line
.lyrics-line.active
.lyrics-line.near
.lyrics-line.unsynced
.lyrics-addon-layer
.lyrics-mini-names
```

Host SVG symbol IDs, available to full-access code in the shared document:

```text
i-play
i-pause
i-next
i-previous
i-heart
i-dislike
i-repeat
i-shuffle
i-lyrics
i-report
i-share
i-more
i-close
i-addons
i-clean
i-contract
```

An isolated iframe must ship its own SVG definitions; its document does not inherit host symbols.

Useful host state is also exposed through attributes such as `aria-pressed`, `hidden`, disabled state, and lyrics panel `data-phase`. Prefer SDK state for behavior and selectors for styling.

Additional state selectors:

- `#player-cover-region[data-playing="true"]` / `[data-playing="false"]`: enlarged playing cover versus paused cover; it does not enlarge the lyrics panel.
- `#lyrics[data-availability="checking"]`, `[aria-busy="true"]`, `:disabled`: automatic lyrics probe UI.
- `#shuffle[aria-pressed="true"]`, `[hidden]`, `:disabled`: active/unsupported/pending command UI.
- `#player-cast[aria-pressed="true"]`: currently casting.
- `[hidden]` always stays hidden in host CSS. Do not override it to reveal unavailable actions.

Do not couple layouts to a fixed number of bottom actions: shuffle and report are conditional. Removed IDs `addon-count` and `queue` are not available; the native header action is `player-native-queue`. Host control buttons are icon-only but retain accessible labels. Preserve those labels and keyboard/focus behavior.

### 8.4 CSS variables and cascade layers

Variables:

```text
--canvas-bg
--canvas-text
--canvas-muted
--canvas-accent
--canvas-radius
--canvas-button-radius
--canvas-action-bg
--canvas-track
```

Cascade order:

```css
@layer chromaBase, chromaTheme, chromaAddons, chromaUser;
```

The host wraps `theme.primary` CSS in `chromaTheme` and `style.patch` CSS in `chromaAddons`. Do not add a new top-level layer that attempts to reorder host layers. Keep `chromaUser` available for future direct user overrides.

The source of truth for selectors is the current host `index.html` and `host.css`. If source access is unavailable, use the lists above. Do not invent object names from visible labels.

The native header also uses `--canvas-safe-top`, supplied by the host. Preserve its inset rather than hard-coding a second top gap. Existing cover scaling lives on `#cover-art`; avoid piling an extra transform onto it unless the theme deliberately replaces both playing and paused styles.

### 8.5 Lyrics styling and presentation objects

The default compact view uses a large current line with fading previous/next copies inside the cover square. Fullscreen lyrics retain the header, scrolling source rows, collapse control, and bottom mini-player/progress. Automatic expansion normally occurs after 10 seconds of active lyrics playback. Pausing must not shrink the lyrics square like cover art. Preserve host transforms, timestamps, line identity, and native handlers when restyling.

Use `#chroma-canvas[data-lyrics-full="true"]` for fullscreen; `#player-lyrics[data-focused="true"]` for compact focus; `#lyrics-scroll[data-manual="true"]` for manual browsing; `#player-lyrics[data-renderer="addon"]` for an active custom renderer. The source .lyrics-line nodes expose active/near/unsynced and data-distance. `#lyrics-focus-previous`, `#lyrics-focus-text`, `#lyrics-focus-next` are reusable compact copies, not separate source records. `#lyrics-mini-progress` is the fullscreen progressbar; `#lyrics-mini-player` owns the runtime progress ratio `--lyrics-mini-progress` (0..1). Do not write that ratio to fake playback.

CSS custom properties (set on `#chroma-canvas` or the appropriate subtree so both the panel and sibling mini-player can inherit):

| Variables | Purpose / built-in fallback |
| --- | --- |
| --lyrics-font-family | Compact focus and fullscreen font; YSMusic Headline, YaMa Text, sans-serif. |
| --lyrics-font-weight, --lyrics-color | 700 and --canvas-text. |
| --lyrics-focus-font-size | Explicit compact font override. Omit to retain measured fit and configure ceilings. |
| --lyrics-focus-inset | Compact stage inset, 44px 8px 14px. |
| --lyrics-focus-line-height, --lyrics-focus-letter-spacing | 1.25 and -.035em. |
| --lyrics-focus-mask | Compact gradient fade: transparent, black 16%, black 84%, transparent. |
| --lyrics-full-top, --lyrics-full-padding | Fullscreen scroll top 88px and horizontal padding 40px. |
| --lyrics-full-anchor | Row-list top padding, fallback 60px. This is a CSS length; configure.fullAnchor is a viewport ratio used for following, not this variable. |
| --lyrics-full-font-size, --lyrics-full-line-height, --lyrics-full-letter-spacing | clamp(28px,10.5vw,40px), 1.05, -.025em. |
| --lyrics-full-gap, --lyrics-line-transition | 32px row gap and .38s row opacity/scale transition. |
| --lyrics-active-opacity, --lyrics-near-opacity, --lyrics-next-opacity, --lyrics-far-opacity | Fullscreen current / near / distance-2 / far opacity: 1 / .32 / .16 / .06. |
| --lyrics-active-scale, --lyrics-inactive-scale | Fullscreen source-row scale: 1 and .8. |
| --lyrics-full-mask | Normal fullscreen fade: transparent, black 9%, black 64%, transparent. |
| --lyrics-manual-mask, --lyrics-manual-opacity | Manual browsing fade: transparent, black 4%, black 96%, transparent; opacity 1. Preserve legibility while browsing. |
| --lyrics-mini-inset, --lyrics-mini-bottom | Left/right inset 16px; bottom max(36px, safe-area bottom + 12px). |
| --lyrics-mini-height, --lyrics-mini-radius, --lyrics-mini-background | 56px, 12px, color-mix of canvas background and text 16%. |
| --lyrics-progress-color | Fullscreen mini-player progress fill, #ffffff18. |

The focus opacity/scale/timing configure options affect compact copies; similarly named fullscreen CSS variables affect source rows. They are separate presentations. Unsynced rows have their own normal readable style rather than active-line animation. Honor prefers-reduced-motion; do not override the host's no-motion rule.

```css
#chroma-canvas {
  --lyrics-font-family: "YSMusic Headline", sans-serif;
  --lyrics-color: #fff7e8;
  --lyrics-focus-inset: 44px 10px 14px;
  --lyrics-full-padding: 24px;
  --lyrics-full-gap: 28px;
  --lyrics-near-opacity: .45;
  --lyrics-manual-opacity: 1;
  --lyrics-mini-radius: 18px;
  --lyrics-progress-color: #ffffff26;
}
```

Use the fonts already available in the host. The public asset() contract returns images, not arbitrary fonts; do not assume a remote font loader exists. When an existing font becomes ready or CSS geometry changes, call lyrics.refreshLayout() once. Do not rename/remove source rows, replace the scroll root, overwrite active classes, or recalculate native progress in a theme.

### 8.6 Built-in actions and native boundaries

| DOM element | Behavior | Availability / important boundary |
| --- | --- | --- |
| `player-collapse` | Collapse the full player | `header.collapse`; unlike `canvas-close`, it also hides the native player |
| `player-cast` | Native device picker | `header.cast`; native sheet/dialog, not addon DOM |
| `player-native-queue` | Native history or upcoming queue | `header.queue`, `header.queueKind`; retains Canvas while visiting the native screen and returns to it on dismissal |
| `album-title` | Native album navigation from the song title | `capabilities.album`; Canvas closes when navigating |
| `artist-name` | Native artist navigation | `capabilities.artist`; native multi-artist picker if needed; Canvas closes for navigation |
| `share` | Share current track | `capabilities.share`; native operation |
| `details` | Native track action menu above Canvas | `capabilities.menu`; no longer a custom Canvas details popup |
| `play`, `previous`, `next`, `like`, `dislike`, `seek`, `repeat` | Transport, rating, seek, repeat | Corresponding command in `state.commands` |
| `shuffle` | Toggle native queue shuffle | `capabilities.shuffle` + `TOGGLE_SHUFFLE`; hidden when unsupported |
| `lyrics` | Swap cover for lyrics | Host probe, `disabled`, `data-availability`; no force-enable |
| `report` | Native ChromaSync report UI | `capabilities.report`; hidden when unavailable |
| `canvas-close` | Close Canvas, return to native player | Host bootstrap must have completed |
| `addons` | Open native addon manager above Canvas | Changes may reload the Canvas document |
| `addons-disable` | Disable addons and reload a clean Canvas | All addon-owned resources must be disposable |

These host buttons already own the correct handlers and native guards. Style the original elements; do not clone them, replace their handlers, or copy their image URLs. If a trusted full-access theme needs a second user-triggered entrypoint, it may forward the gesture to the existing button's `click()` after checking it exists, is not hidden, and is not disabled. That reuses the host action; it is not a new SDK method. Never invoke it from a render/state callback or an automatic loop, especially for close, disable, or navigation.

```javascript
// Full-access theme only; the target remains the original host button.
const menuButton = document.createElement('button');
menuButton.type = 'button';
menuButton.textContent = 'Track actions';
const forward = () => {
  const target = document.getElementById('details');
  if (target && !target.hidden && !target.disabled) target.click();
};
menuButton.addEventListener('click', forward);
root.append(menuButton);
chroma.canvas.onDispose(() => {
  menuButton.removeEventListener('click', forward);
  menuButton.remove();
});
```

The public `chroma.player` API does **not** expose `openArtist`, `openAlbum`, `openMenu`, `openCast`, `openHistory`, or `closeCanvas`. Private `host.*` RPC and caller `_host` are not addon APIs. Do not forge RPC messages, call bridge internals, or impersonate the host. Isolated addons cannot use the forwarding pattern and cannot style the host DOM.

The device picker, track menu, history/queue, artist picker, report UI, and addon/settings manager remain Android-native UI. Their contents cannot be queried, themed, or extended with DOM/CSS. Native sheets may cover Canvas without disposing addons; do not duplicate subscriptions on return or assume every overlay causes `onDispose`/`visibilitychange`. Reload/close, in contrast, must release all addon resources.

## 9. Settings

Optional `settings.schema.json` supports a deliberately small object schema:

```json
{
  "type": "object",
  "properties": {
    "enabled": {
      "type": "boolean",
      "default": true
    },
    "density": {
      "type": "integer",
      "default": 24,
      "minimum": 0,
      "maximum": 48
    },
    "tone": {
      "type": "string",
      "default": "accent",
      "enum": ["accent", "neutral"]
    },
    "opacity": {
      "type": "number",
      "default": 0.7,
      "minimum": 0,
      "maximum": 1
    }
  }
}
```

Rules:

- top-level `type` must be `object`;
- `properties` is required;
- maximum 16 properties;
- key regex: `[a-zA-Z][a-zA-Z0-9_]{0,39}`;
- supported types: `boolean`, `string`, `number`, `integer`;
- every property requires `default`;
- string length is at most 256 and optional `enum` is enforced;
- numbers must be finite; default range is -1e9 to 1e9 and explicit min/max are enforced;
- nested objects and arrays are unsupported;
- unknown saved keys are rejected.

Read values from `chroma.canvas.settings`. Treat this object as a startup snapshot. Saving settings causes a Canvas document reload; do not expect live mutation callbacks.

## 10. Storage

Requires `storage`.

```javascript
const model = (await chroma.storage.get('model')) ?? { version: 1, count: 0 };
model.count += 1;
await chroma.storage.set('model', model);
```

Rules:

- key length 1-100;
- values must be JSON-compatible;
- total serialized storage is 64 KiB per addon;
- storage is isolated by addon ID;
- uninstall removes storage and settings;
- writes are atomic, but do not implement high-frequency counters by writing every frame.

Version stored structures and migrate them defensively.

## 11. Package assets

`chroma.canvas.asset(path)` reads an image from the current package and returns a data URL. Supported extensions are PNG, JPG/JPEG, and WebP. Maximum returned asset size is 256 KiB.

```javascript
const icon = document.createElement('img');
icon.alt = '';
icon.src = await chroma.canvas.asset('images/icon.webp');
root.append(icon);
```

Relative CSS URLs are not automatically resolved against ZIP contents. Use `asset()` and assign the result to DOM properties, or embed a very small data URL at build time. Do not request arbitrary package files through `asset()`; unsupported extensions are rejected.

## 12. External network

Requires `network` and exact `networkDomains` entries.

```json
{
  "permissions": ["canvas.widget", "network"],
  "networkDomains": ["api.example.com"]
}
```

```javascript
async function fetchStatus() {
  const response = await chroma.network.request({
    method: 'GET',
    url: 'https://api.example.com/v1/status?lang=ru'
  });
  if (!response.contentType.startsWith('application/json')) {
    throw new Error('UNEXPECTED_CONTENT_TYPE');
  }
  return JSON.parse(response.body);
}
```

Hard restrictions:

- HTTPS only;
- port 443 only;
- GET only;
- exact hostname membership; `example.com` does not authorize `api.example.com` and vice versa;
- no wildcard domains;
- no URL username/password;
- no request headers or body;
- no automatic redirects;
- no cookie jar;
- no system proxy;
- DNS results must all be public addresses; private, loopback, link-local, multicast, carrier-grade NAT and documentation ranges are denied;
- 25-second native deadline;
- 512 KiB response limit;
- only `application/json` and `text/*` are returned to JS.

The device VPN can still route the native connection. Never claim that a request bypasses a VPN.

Do not place API secrets in addon source: a ZIP is user-readable. Use only public endpoints or a backend designed for untrusted clients.

### 12.1 Pinned remote JavaScript

Ordinary `network.request` never returns executable MIME types and cannot load a `<script src>`. To execute a remote dependency, declare it in `remoteScripts` and request both `network` and `code.remote`:

```json
{
  "permissions": ["player.read", "yandex.read", "canvas.style", "canvas.fullAccess", "network", "code.remote"],
  "networkDomains": ["cdn.jsdelivr.net"],
  "remoteScripts": [{
    "url": "https://cdn.jsdelivr.net/npm/node-vibrant@3.1.6/dist/vibrant.min.js",
    "mimeType": "application/javascript",
    "sizeBytes": 59298,
    "sha256": "6b0711da39e8e242395f4ccbf1ec74be2db3f8a324176e21516f1678f480f473"
  }]
}
```

Validation and runtime rules:

- descriptor keys are strict: `url`, `mimeType`, `sizeBytes`, `sha256`, with no extras;
- URL is HTTPS/443, has no credentials or fragment, and its exact host must be listed in `networkDomains`;
- allowed base MIME values are exactly `application/javascript` and `text/javascript`; charset parameters in the response are ignored after the base MIME is compared;
- `sizeBytes` is exact, not a maximum; range 1-262144 bytes per file;
- `sha256` is exactly 64 lowercase hexadecimal characters and is compared against the downloaded bytes;
- at most four scripts and 512 KiB declared total;
- redirects remain disabled and public-address DNS checks still apply;
- every check must pass before any byte is executed;
- a verified dependency is cached by SHA-256 and revalidated by size/hash when read from cache;
- scripts execute in array order before the addon entrypoint, in the addon's normal execution realm;
- isolated addons keep the dependency inside their sandbox iframe; full-access style/theme addons execute it in the shared host realm;
- changing a URL, MIME, size, hash, or dependency version requires a new addon package version and renewed user review.

Do not implement fallback `eval(await chroma.network.request(...))`. It bypasses the immutable dependency contract and is intentionally rejected for JavaScript MIME responses.

### 12.2 Next-track artwork with node-vibrant

A full-access style addon can use the pinned dependency and the queue artwork API without exposing remote image URLs:

```javascript
let generation = 0;

async function refreshBackground() {
  const current = ++generation;
  const [next] = await chroma.player.getQueue({ offset: 1, limit: 1 });
  if (!next?.artworkAvailable) return;
  const artwork = await chroma.player.getQueueArtwork(next.queueId);
  if (current !== generation) return;

  const image = new Image();
  image.src = artwork.url;
  await image.decode();
  if (current !== generation) return;

  const palette = await Vibrant.from(image).getPalette();
  const accent = palette.Vibrant?.getHex() || palette.Muted?.getHex();
  document.getElementById('player-background')
    ?.style.setProperty('background-image', `url('${artwork.url}')`);
  if (accent) document.documentElement.style.setProperty('--canvas-accent', accent);
}

const offTrack = chroma.player.on('trackChanged', () => refreshBackground().catch(chroma.log));
const offQueue = chroma.player.on('queueChanged', () => refreshBackground().catch(chroma.log));
chroma.canvas.onDispose(() => { generation++; offTrack(); offQueue(); });
refreshBackground().catch(chroma.log);
```

This example needs `player.read`, `canvas.style`, `canvas.fullAccess`, `network`, and `code.remote`. Add `yandex.read` for the catalog fallback when MediaSession publishes identity but no artwork payload. Restore any inline styles or CSS variables your addon owns during disposal if disabling it without a full document reload becomes supported later.

## 13. Yandex catalog broker

Requires `yandex.read`. Credentials remain in native memory and are never returned to JavaScript. Response JSON is scrubbed of keys matching token, authorization, cookie, password, secret, passport, or download and of string values containing the current token.

Allowed routes:

```text
GET /artists/{1-20 digit id}/brief-info
GET /albums/{1-20 digit id}/with-tracks
GET /tracks/{1-20 digit id}/credits
GET /tracks/{1-20 digit id}/similar
```

Only query `lang=ru|en` is allowed. Custom body and headers are denied. Example:

```javascript
const response = await chroma.yandex.request({
  method: 'GET',
  path: '/artists/12345/brief-info',
  query: { lang: 'ru' }
});
```

Do not use `state.track.id` as a catalog ID. Current-track catalog identity is resolved internally for host lyrics/share but is not exposed as a public addon field in API v1. An addon may call catalog routes only with an explicit numeric ID it already knows from its own data or from an allowed catalog response.

`yandex.write` is reserved and currently returns `YANDEX_WRITE_UNAVAILABLE_IN_PREVIEW`. Do not request it for a read-only addon.

## 14. Downloads

Requires `downloads`, `network`, and an allowed exact domain. The operation downloads native bytes directly to MediaStore; bytes and local filesystem paths never pass through JavaScript.

```javascript
const result = await chroma.downloads.enqueue({
  url: 'https://cdn.example.com/audio/file.mp3',
  filename: 'Artist - Track.mp3'
});
```

Rules:

- Android 10 or newer;
- explicit HTTPS URL only;
- current Yandex Music track downloads are unavailable;
- filename regex effectively permits ASCII letters/digits, spaces, `_`, `.`, `-`, length up to 80, ending in `.mp3`, `.m4a`, `.ogg`, or `.flac`;
- response must be `audio/*`;
- maximum 32 MiB;
- no redirects;
- the user receives a native confirmation dialog every time;
- destination is `Music/ChromaCanvas`;
- result resolves after completion, not when a background job is queued;
- failed writes remove the pending MediaStore row.

Do not make a download button look automatic. The user confirmation is part of the security contract.

## 15. RPC, quotas, and concurrency

Package limits:

- at most 24 installed addons;
- at most 128 ZIP entries;
- at most 8 MiB total unpacked data;
- at most 2 MiB per file;
- entrypoint at most 256 KiB;
- total CSS at most 256 KiB;
- at most four remote scripts, 256 KiB each and 512 KiB declared total;
- enabled addon bundle JSON at most 2 MiB;
- image returned by `asset()` at most 256 KiB;
- storage at most 64 KiB per addon;
- settings at most 16 properties.

Runtime limits:

- 24 RPC requests per second per addon identity;
- shared lyrics local requests have a separate 48-per-second budget per addon; unsubscribe is exempt. Reuse state/events rather than polling. Native seek submission still goes through the native command permission/rate checks;
- 24 pending native RPC requests across the screen;
- 16 pending requests inside one isolated iframe;
- JS-side timeout approximately 36 seconds;
- native network deadline 25 seconds;
- up to two background Canvas IO tasks;
- generic text/JSON response 512 KiB;
- queue artwork source download 2 MiB, decoded source dimensions at most 32768 per side, rendered result at most 768 px per side, and eight ephemeral results retained per screen;
- audio download 32 MiB;
- inbound RPC message over 96 KiB is ignored.

Design implications:

- debounce rapid user actions;
- do not issue network calls on every playback state callback;
- coalesce track-change work and discard stale responses;
- use a loading/disabled state for buttons;
- cache only small derived data in storage;
- do not serialize images or audio through RPC.

## 16. Errors

Catch every SDK Promise. Branch on stable code values, not localized host text.

Common codes:

- `PERMISSION_DENIED`: missing manifest permission.
- `ADDON_DISABLED`: request came after disable/reload.
- `METHOD_UNKNOWN`: unsupported SDK/native method.
- `PLAYER_UNAVAILABLE`: no active MediaSession.
- `QUEUE_UNAVAILABLE`, `QUEUE_POSITION_UNKNOWN`, `QUEUE_RANGE`: queue unavailable or bad options.
- `COMMAND_UNSUPPORTED`, `SEEK_RANGE`, `REPEAT_UNAVAILABLE`: invalid command in current state.
- `TRACK_CHANGED`: asynchronous request targets an old track.
- `LYRICS_TRACK_UNRESOLVED`, `LYRICS_NOT_AVAILABLE`, `LYRICS_LOCAL_TRACK`: lyrics unavailable for a known reason.
- `LYRICS_STALE_LINE`, `LYRICS_LINE_MISSING`, `LYRICS_UNTIMED_LINE`: obsolete document reference, invalid/missing row, or seeking an untimed row.
- `LYRICS_OPTIONS`, `LYRICS_PAYLOAD`, `LYRICS_EVENT_INVALID`, `LYRICS_METHOD_UNSUPPORTED`: invalid shared-lyrics input/event/method.
- `LYRICS_RENDERER_BUSY`, `LYRICS_RENDERER_INVALID`, `LYRICS_RENDERER_ASYNC`: exclusive renderer already owned, malformed renderer, or Promise-returning render.
- `CANVAS_FULL_ACCESS_REQUIRED`, `CANVAS_OBJECT_MISSING`, `CANVAS_STYLE_INVALID`: DOM access or reversible style failure.
- `MOUNT_NOT_DECLARED`, `MOUNT_UNSUPPORTED`: additional mount absent from the manifest or unsupported by this host/type.
- `COMMAND_UNAVAILABLE`: current native seek command unavailable for seekToLine; check current commands and still catch races.
- `YANDEX_SESSION_NOT_OBSERVED`, `YANDEX_SESSION_CHANGED`: catalog session unavailable or changed.
- `YANDEX_ROUTE_DENIED`, `YANDEX_QUERY_DENIED`, `YANDEX_OPTIONS_DENIED`: catalog request outside allowlist.
- `NETWORK_DOMAIN_DENIED`, `NETWORK_ADDRESS_DENIED`, `NETWORK_METHOD_UNSUPPORTED`, `NETWORK_OPTIONS_UNSUPPORTED`: network policy failure.
- `NETWORK_HTTP_<status>`, `NETWORK_EMPTY`, `NETWORK_RESPONSE_TOO_LARGE`, `NETWORK_USE_DOWNLOAD`: remote response failure.
- `DOWNLOAD_REQUIRES_ANDROID_10`, `CURRENT_TRACK_DOWNLOAD_UNAVAILABLE`, `DOWNLOAD_FILENAME`, `DOWNLOAD_NOT_AUDIO`: export contract failure.
- `RPC_BUSY`, `RPC_TIMEOUT`, `CANCELLED`, `ADDON_UNLOADED`: overload, timeout, close, or reload.

Recommended pattern:

```javascript
async function guarded(button, task) {
  if (button.disabled) return;
  button.disabled = true;
  try {
    await task();
  } catch (error) {
    chroma.log(error.message || 'ADDON_REQUEST_FAILED');
    chroma.canvas.showMessage('Действие не выполнено');
  } finally {
    button.disabled = false;
  }
}
```

Do not parse user-visible host text to detect errors. Do not write tests that assert exact localized bot or Canvas text. Test state, callback IDs, data structures, permissions, error codes, and behavior.

In the current SDK, chroma.log also uses the on-screen message channel; it is not a private developer log. Logging raw error.message in examples is for development diagnostics. Production lyrics UI must not expose provider overload/auth diagnostics; use neutral pending/empty states and reserve a generic action failure message for an explicit failed user action.

## 17. Complete isolated widget template

`manifest.json`:

```json
{
  "id": "dev.example.control-card",
  "name": "Control Card",
  "version": "1.0.0",
  "apiVersion": 1,
  "type": "widget",
  "entrypoint": "index.js",
  "styles": ["style.css"],
  "permissions": ["player.read", "player.control", "canvas.widget", "storage"],
  "mounts": ["@player.afterMetadata"],
  "priority": 100,
  "minimumHostVersion": "1.0.0"
}
```

`index.js`:

```javascript
(() => {
  const card = document.createElement('section');
  card.className = 'control-card';
  const title = document.createElement('strong');
  const artist = document.createElement('span');
  const next = document.createElement('button');
  next.type = 'button';
  next.textContent = 'Дальше';
  card.append(title, artist, next);
  root.append(card);

  function render(state) {
    title.textContent = state.track?.title || 'Нет трека';
    artist.textContent = state.track?.artists || '';
    next.disabled = !state.commands.includes('NEXT');
  }

  async function skip() {
    if (next.disabled) return;
    next.disabled = true;
    try {
      await chroma.player.command('NEXT');
      const count = (await chroma.storage.get('skips')) ?? 0;
      await chroma.storage.set('skips', count + 1);
    } catch (error) {
      chroma.log('NEXT:', error.message);
    } finally {
      next.disabled = false;
    }
  }

  next.addEventListener('click', skip);
  chroma.player.getState().then(render).catch(error => chroma.log(error.message));
  const unsubscribe = chroma.player.on('stateChanged', render);

  chroma.canvas.onDispose(() => {
    unsubscribe();
    next.removeEventListener('click', skip);
    card.remove();
  });
})();
```

`style.css`:

```css
.control-card {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 4px 12px;
  align-items: center;
  box-sizing: border-box;
  width: 100%;
  padding: 14px;
  border-radius: 14px;
  background: #ffffff12;
  color: #fafafa;
}
.control-card strong,
.control-card span {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.control-card span { opacity: .65; }
.control-card button { grid-column: 2; grid-row: 1 / span 2; }
```

## 18. Complete full-access style patch template

Use only when the requested change targets host selectors.

`manifest.json`:

```json
{
  "id": "dev.example.rounded-player",
  "name": "Rounded Player",
  "version": "1.0.0",
  "apiVersion": 1,
  "type": "style.patch",
  "entrypoint": "index.js",
  "styles": ["style.css"],
  "permissions": ["canvas.style", "canvas.fullAccess"],
  "priority": 200,
  "minimumHostVersion": "1.0.0"
}
```

`index.js`:

```javascript
// CSS is applied by the host. Keep the required entrypoint intentionally inert.
chroma.canvas.onDispose(() => {});
```

`style.css`:

```css
:root {
  --canvas-radius: 24px;
  --canvas-button-radius: 18px;
}
#player-cover-region { box-shadow: 0 24px 70px #0006; }
#player-actions button { border: 1px solid #ffffff12; }
```

## 19. Effect template and performance rules

An effect receives a full-size iframe root inside a pointer-events-none host layer. Keep its own UI noninteractive unless the design explicitly requires otherwise.

```javascript
const layer = document.createElement('canvas');
const context = layer.getContext('2d');
root.append(layer);

let frame = 0;
let disposed = false;
const reduced = matchMedia('(prefers-reduced-motion: reduce)');

function resize() {
  const ratio = Math.min(devicePixelRatio || 1, 2);
  layer.width = Math.max(1, Math.floor(root.clientWidth * ratio));
  layer.height = Math.max(1, Math.floor(root.clientHeight * ratio));
  layer.style.width = '100%';
  layer.style.height = '100%';
}

function draw() {
  if (disposed || reduced.matches || document.hidden) return;
  context.clearRect(0, 0, layer.width, layer.height);
  // Keep work bounded. Reuse objects; do not allocate large arrays each frame.
  frame = requestAnimationFrame(draw);
}

function visibility() {
  cancelAnimationFrame(frame);
  if (!document.hidden && !reduced.matches) frame = requestAnimationFrame(draw);
}

const resizeObserver = new ResizeObserver(resize);
resizeObserver.observe(root);
document.addEventListener('visibilitychange', visibility);
reduced.addEventListener('change', visibility);
resize();
visibility();

chroma.canvas.onDispose(() => {
  disposed = true;
  cancelAnimationFrame(frame);
  resizeObserver.disconnect();
  document.removeEventListener('visibilitychange', visibility);
  reduced.removeEventListener('change', visibility);
  layer.remove();
});
```

Cap device pixel ratio, particle count, blur radius, and DOM node count. Prefer transforms and opacity. Do not run layout reads after layout writes in a loop.

## 20. Development workflow

When the Canvas SDK source tree is available:

```sh
cd canvas-sdk
npm test
npm run dev
# open http://127.0.0.1:8788
node pack.mjs examples/your-addon your-addon.zip
```

The browser mock has no real audio, account, catalog, or native download behavior. It is suitable for layout, event wiring, cleanup, ordering, and pure contract helpers. Native features must be tested on Android.

Useful preview query flags:

- `?addons=0`: host without addons;
- `?addons=0&paid=0`: no report capability;
- `?lyrics=missing`: missing lyrics result;
- `?lyrics=error`: lyrics request error.
- `?shuffle=1`: playlist-like queue with shuffle enabled as a capability (initially off).

The browser mock deliberately rejects native menu/navigation/header actions. A painted native button in preview is not evidence that its Android action works. There is no separate "Add examples" button: the app installs Snow Particles and Square Controls disabled on first use; the Telegram example downloads are separate study material. Removed Track Downloader is not a supported built-in example.

Packing requirements are enforced again on Android. A ZIP produced by another tool is acceptable if it follows the same root layout, paths, and quotas.

## 21. Test matrix

Do not stop after the happy path. Verify:

### Installation and lifecycle

- fresh install remains disabled until confirmation;
- enable, disable, remove;
- update with same ID and higher version;
- rollback to previous version;
- settings edit and reload;
- Canvas close and reopen;
- safe mode after forced process termination;
- no duplicate event handlers after reload.

### Playback

- no active track;
- normal online track;
- replacement track;
- local inserted track;
- play, pause, resume, seek, next, previous;
- end of track;
- rapid skips while async work is pending;
- unsupported reaction/repeat/queue;
- Wave -> playlist/album -> Wave: shuffle appears/disappears, supports toggling, and reflects native state;
- very long title and artist;
- missing cover and unknown duration.

### UI and performance

- narrow phone, tall phone, landscape, display cutout;
- Android font scale and accessibility focus;
- light/dark assumptions: Canvas currently declares dark color scheme, but addon contrast must remain sufficient;
- reduced motion;
- document hidden/visible;
- CPU/frame profile with Canvas open and closed;
- no work continues after dispose.
- native device picker, track menu, history/queue return, collapse, close, and artist/album navigation preserve the intended host behavior;
- optional header/capabilities missing in an older build; disabled/hidden actions stay unavailable;
- bottom action layout with and without shuffle/report; icon-only buttons retain accessible labels;

### Network and data

- offline, timeout, DNS denial, 4xx/5xx, wrong content type, oversized response;
- track changes during response;
- storage quota and malformed old data;
- catalog session missing/changed;
- download cancel and failed MediaStore write.

### Shared lyrics and composition

- shared-cache reads cause no duplicate provider/network request;
- Yandex/LRCLIB, source override, stale cached data, checking/deferred/unavailable, empty/untimed/duplicate-timestamp rows;
- subscribe then initial read without an older initial snapshot overwriting an event;
- offset changes update position and preserve references; track/text/source changes reject old row references;
- seekToLine preserves pause and applies offset once; scrollToLine never seeks or opens closed lyrics;
- compact/fullscreen auto expansion, manual browse legibility, follow, mini-player progress and controls;
- getLineElements after closed/open, line/source replacement, compact-copy reuse, and elementsChanged;
- isolated read-only widget can read shared data but cannot use objects, configure, custom renderer, or createMount;
- missing player.read/player.control/canvas.style, undeclared mounts, unknown options, and renderer conflicts are handled;
- overlapping configure/style leases released out of order restore preceding values;
- custom renderer toggled off, throws, or addon unloads: default presentation and other addons recover;
- no listeners, network polling, timers, or frame loops remain after disposal.

Tests should assert behavior, structures, callback IDs, permissions, and error codes. Never assert exact localized text.

## 22. Review checklist for the AI agent

Before reporting completion, verify all of the following:

- [ ] `manifest.json` passes ID, version, type, path, permission, domain, and mount rules.
- [ ] The archive root is correct.
- [ ] Permissions are minimal.
- [ ] `canvas.fullAccess` is absent unless host DOM modification is essential.
- [ ] Every event listener, timer, observer, animation, and temporary DOM node has cleanup.
- [ ] All SDK Promises have intentional error handling.
- [ ] Async work discards stale track results.
- [ ] New lyrics/object/multi-mount features are checked independently from apiVersion.
- [ ] Line actions retain trackKey/revision/index and handle null timestamps.
- [ ] getState is not polled per frame; positionChanged is subscribed only when needed.
- [ ] Renderer/config/style leases are released; shared host controls and progress remain intact.
- [ ] Player commands are gated by `state.commands`.
- [ ] `state.track.id` is never treated as a guaranteed catalog ID.
- [ ] External data is inserted via `textContent`, not `innerHTML`.
- [ ] Network domains are exact and no secret is embedded in source.
- [ ] UI handles missing values and small screens.
- [ ] Effects honor reduced motion and visibility.
- [ ] Browser tests pass where applicable.
- [ ] The final ZIP is below quotas and installs on Android.
- [ ] No exact user-facing text assertions were added to tests.

## 23. Known API v1 limitations

Do not promise or emulate unsupported behavior:

- no dependency graph or package registry;
- no marketplace or automatic addon updates;
- no package signature verification;
- no ES module loader;
- no background addon execution;
- no arbitrary browser `fetch` from addon code;
- no custom request headers/body or redirects in `network.request`;
- no arbitrary authenticated Yandex endpoints;
- no `yandex.write`;
- no public current-track catalog ID;
- no current Yandex audio or protected replacement download;
- no download progress/job handle;
- no public SDK method for native navigation, menus, header actions, or Canvas management;
- no public launcher for tool/page roots in the current UI;
- no guarantee that Yandex Music exposes queue, ratings, cover, repeat, or every transport command;
- iframe isolation does not provide a separate JavaScript thread and cannot guarantee protection from all WebView CPU abuse;
- fullAccess is trusted shared-DOM execution, not permission isolation.

When a requested feature needs one of these, stop and state the missing host API. Do not work around it with undocumented internals.

## 24. Complete Lyrics Workbench example

Download Lyrics Workbench 1.0.0.zip with the existing Telegram examples button below the human guide. The ZIP is a separate study addon, not a default-installed app package. Its source is reproduced below. It requires the new lyrics/object/multi-mount host extensions; do not claim compatibility with an older API-v1-only host.

manifest.json:

```json
{
  "id": "dev.chromasync.lyrics-workbench",
  "name": "Lyrics Workbench",
  "version": "1.0.0",
  "apiVersion": 1,
  "type": "widget",
  "entrypoint": "index.js",
  "styles": ["style.css"],
  "permissions": ["player.read", "player.control", "canvas.widget", "canvas.style", "canvas.fullAccess"],
  "mounts": ["@lyrics.controls", "@lyrics.miniPlayer"]
}
```

index.js:

```javascript
if (!chroma.lyrics) throw Error('LYRICS_SDK_REQUIRED');
const button = document.createElement('button');
button.textContent = 'Custom lyrics';
button.className = 'lyrics-workbench-toggle';
root.append(button);
const mini = chroma.canvas.createMount('@lyrics.miniPlayer');
const counter = document.createElement('small');
mini.append(counter);
let release = null;
function updateCounter(value) {
  counter.textContent = value.activeIndex < 0 ? '' : String(value.activeIndex + 1);
}
const off = chroma.lyrics.on('activeLineChanged', updateCounter);
chroma.lyrics.getPosition().then(updateCounter).catch(() => {});
button.onclick = () => {
  if (release) { release(); release = null; return; }
  release = chroma.lyrics.registerRenderer({
    render(state, container) {
      container.className = 'lyrics-workbench';
      const title = document.createElement('small');
      title.textContent = `${state.source || ''} ${state.activeIndex + 1}/${state.lines.length}`;
      const list = document.createElement('div');
      const index = Math.max(0, state.activeIndex);
      for (const row of state.lines.slice(Math.max(0, index - 1), index + 3)) {
        const item = document.createElement('button');
        item.textContent = row.text;
        item.className = row.index === state.activeIndex ? 'current' : '';
        item.disabled = row.timeMs === null;
        item.onclick = () => chroma.lyrics.seekToLine(row).catch(() => {});
        list.append(item);
      }
      container.replaceChildren(title, list);
    }
  });
};
chroma.canvas.onDispose(() => { off(); release?.(); });
```

style.css:

```css
.lyrics-workbench-toggle { display:block;margin:4px auto;padding:6px 12px;border-radius:16px;background:var(--canvas-action-bg); }
.lyrics-workbench { position:absolute;inset:48px 12px 56px;display:grid;align-content:center;text-align:center;overflow:auto; }
.lyrics-workbench > small { color:var(--canvas-muted); }
.lyrics-workbench button { display:block;background:transparent;width:100%;font-size:20px;padding:12px;opacity:.5; }
.lyrics-workbench button.current { color:var(--canvas-accent);opacity:1; }
#lyrics-addon-mini small { margin:0 8px;font-variant-numeric:tabular-nums; }
```

Install the ZIP through Canvas addons, inspect/confirm full-access permissions, enable it, and open lyrics on a cached timed track. Custom lyrics toggles the renderer; clicking a timed line submits seeking while preserving pause. Wait for the normal fullscreen expansion during playback to inspect the source/line counter and the additional mini-player counter. Toggle back, browse the native list, return to its current line, and disable/remove the addon. The original renderer, controls, progress, and other addons must retain their behavior. No external network permission or additional provider request is needed.

For a production addon, adapt the example with feature detection, accessible toggle state, deliberate error handling, and guarded initial-state delivery as shown in section 6.5. Keep provider/network diagnostics out of user-facing lyrics. Legacy Lyrics Focus 1.0.1 remains in the example downloads for older builds; its old DOM approach is not the recommended template for this new API.
