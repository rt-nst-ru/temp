# Echo Music App — Viva Questions (30)

> **Codebase**: React Native + Expo music streaming app ("Echo Music")
> **Split**: 10 Conceptual · 10 Code (High-Level / Architecture) · 10 Code (Low-Level / Difficult Practices)

---

## SECTION A — Conceptual Questions (10)

---

### Q1. Why does this app NOT use Expo Router's file-based routing for tab navigation, even though Expo Router is installed and the project uses file-based routing for the root layout?

**Answer:**
The app has only a single route file ([`src/app/index.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/app/index.tsx)). Instead of creating separate route files for each tab (home, search, library, musicflow), it implements a **manual tab system** using React state (`activeTab`) and conditional rendering. This was done because the app needs **persistent overlays** (FloatingMiniPlayer, FloatingNavigationToolbar, EchoPlayerModal, QueueModal) that sit above all screens simultaneously. With Expo Router's native tab/stack navigators, managing these persistent floating UI elements that must remain visible across all tabs becomes significantly more complex. The single-route approach gives full control over the overlay layering.

---

### Q2. Explain the multi-tier audio stream resolution strategy used in `getAudioStreamUrl()`. Why are there 5 fallback levels and what problem does each solve?

**Answer:**
The 5-tier strategy in [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L424-L517):
1. **Pre-seeded `PRESET_STREAM_URLS`** — Hard-coded Saavn CDN URLs for featured songs → **0ms latency** for initial user experience, no network call.
2. **In-memory `STREAM_CACHE` (Map with 6hr TTL)** — Avoids re-resolving URLs for recently played songs → reduces API load and latency.
3. **JioSaavn API** — Search + detail fetch + DES decryption of `encrypted_media_url` → 320kbps high quality audio, no authentication required.
4. **Piped instances (3 mirrors)** — Open-source YouTube frontends that extract audio streams → fallback when Saavn has no match, iterates multiple mirrors for resilience.
5. **Ultimate fallback to a known preset URL** — Returns the Kesariya song URL → ensures the player *never* returns `null` for a critical path; prevents the app from appearing broken.

This handles: CDN failures, Saavn catalog gaps, Piped instance downtime, and new/obscure songs.

---

### Q3. The app declares `android.permission.FOREGROUND_SERVICE` and `android.permission.WAKE_LOCK` in `app.json`. What is the purpose of each in the context of a music player, and what happens if they are removed?

**Answer:**
- **`FOREGROUND_SERVICE`**: Required for audio playback to continue when the app is backgrounded on Android. Without it, Android's process killer will terminate playback after a few seconds/minutes when the app goes to the background.
- **`WAKE_LOCK`**: Prevents the CPU from sleeping during audio playback. Without it, the device may enter deep sleep and interrupt the audio stream, causing stuttering or stopping.
- **`INTERNET`**: Required for streaming audio from remote CDN URLs.

Removing `FOREGROUND_SERVICE` means music stops when the user switches apps. Removing `WAKE_LOCK` means playback may cut out when the screen turns off.

---

### Q4. What is Continuous Native Generation (CNG) and why does this project NOT have `ios/` and `android/` directories?

**Answer:**
CNG is Expo's approach where native platform directories (`ios/` and `android/`) are **generated at build time** rather than committed to source control. As stated in [`AGENTS.md`](file:///Users/ritesh/Documents/TEst%20Diff/sona/AGENTS.md#L39): *"If ios/ and android/ directories do not exist, they are generated."* All native configuration (permissions, icons, splash screen, bundle IDs) is declared in [`app.json`](file:///Users/ritesh/Documents/TEst%20Diff/sona/app.json) and config plugins. This means:
- No Xcode/Android Studio setup needed locally
- Builds happen via EAS Build in the cloud
- Native code is auto-generated from declarative config
- Upgrading React Native/Expo SDK doesn't require manual native file merging

---

### Q5. The `app.json` enables `"reactCompiler": true` under `experiments`. What does the React Compiler do and how does it affect this codebase?

**Answer:**
The React Compiler (formerly React Forget) is an **automatic memoization compiler** that analyzes React components at build time and inserts `useMemo`, `useCallback`, and `React.memo` equivalents where appropriate. In this codebase:
- Components like `EchoHomeScreen` with expensive `useMemo` computations (e.g., `sortedCommunityPlaylists`) may get additional automatic optimizations
- The many inline arrow functions passed as props (e.g., `onPress={() => setShowSettings(true)}`) get automatically memoized to prevent unnecessary child re-renders
- The PlayerContext's large value object with 20+ properties and methods would benefit from automatic memoization

With the compiler enabled, **manual memoization is less critical** but the developer-written `useMemo`/`useCallback` calls still work and are not removed.

---

### Q6. Why does the app use `expo-image` (`Image` from `expo-image`) everywhere instead of React Native's built-in `Image` component?

**Answer:**
`expo-image` provides significant advantages over RN's built-in `Image`:
1. **Automatic disk caching** — Images are cached to disk automatically; built-in `Image` only does in-memory caching
2. **`contentFit` prop** — More intuitive than `resizeMode` (uses CSS `object-fit` semantics: `cover`, `contain`, etc.)
3. **`transition` prop** — Built-in crossfade animations (used throughout: `transition={200}`, `transition={250}`, `transition={300}`)
4. **Better performance** — Uses native image loading libraries (SDWebImage on iOS, Glide on Android) instead of RN's slower bridge-based loading
5. **`blurRadius` prop** — Hardware-accelerated blur used in `EchoPlayerModal` and `MusicFlowScreen` for backdrop effects
6. **Progressive loading** and **blurhash** support

This is critical for a music app displaying many album art thumbnails in scrollable lists.

---

### Q7. Explain how the `BackHandler` integration works in `index.tsx`. What specific navigation behavior does it implement, and what does `return true` vs `return false` signify?

**Answer:**
In [`index.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/app/index.tsx#L19-L37):
- `return true` means "I handled this back press, don't propagate further" (prevents app exit)
- `return false` means "I didn't handle it, let the system handle it" (exits the app)

The hierarchy:
1. If Settings screen is open → close it, return `true` (stay in app)
2. If on a non-home tab (search/library/musicflow) → switch to home tab, return `true`
3. If already on home tab with settings closed → return `false` → Android's default back action (exit app)

The `useEffect` dependency array `[activeTab, showSettings]` ensures the listener is re-registered whenever these values change, so the closure always has fresh state. The cleanup `subscription.remove()` prevents memory leaks.

---

### Q8. Why does this app use `AsyncStorage` for persistence instead of SQLite or a database solution? What are the trade-offs?

**Answer:**
`AsyncStorage` (via `@react-native-async-storage/async-storage`) is used in [`storage.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/storage.ts) for: liked songs, playlists, recents, search history, settings, liked radio stations.

**Why it's suitable:**
- Simple key-value data with JSON serialization
- Small data volumes (50 recents cap, 15 search history cap)
- No relational queries needed
- Zero setup, no native module compilation beyond the library itself

**Trade-offs:**
- **No indexing** — `isSongLiked()` loads ALL liked songs into memory just to check one song
- **No transactions** — Concurrent writes to the same key could lose data (e.g., two rapid like toggles)
- **Full deserialization** — Every read parses the entire JSON blob; with 1000+ liked songs this becomes slow
- **No migration system** — Schema changes require manual migration logic
- SQLite would be better if the app scaled to thousands of songs/playlists

---

### Q9. What is the `scheme` property `"echomusic"` in `app.json` and how does it relate to deep linking in a React Native app?

**Answer:**
The `"scheme": "echomusic"` in [`app.json`](file:///Users/ritesh/Documents/TEst%20Diff/sona/app.json#L8) registers a **custom URL scheme** for the app. This means:
- URLs like `echomusic://` can open the app from external sources (browser, other apps, QR codes)
- Expo Router uses this for **deep linking** — mapping URLs to specific routes
- Combined with `expo-linking`, the app can handle incoming URLs and navigate to specific screens
- Universal links (https) can also be configured alongside this

However, in this codebase the deep linking capability is **declared but not actively utilized** — there are no route parameters or deep link handlers in the single `index.tsx` route. The Share functionality generates YouTube URLs (`https://youtu.be/`) rather than `echomusic://` URLs.

---

### Q10. The app requests `playsInSilentMode: true` via `setAudioModeAsync`. What does this control on iOS specifically, and why is `interruptionMode: 'doNotMix'` chosen over `'mixWithOthers'` or `'duckOthers'`?

**Answer:**
From [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L93-L97):
- **`playsInSilentMode: true`**: On iOS, the physical mute switch (Ring/Silent toggle) normally silences all audio. This setting overrides that — music plays even with the switch set to silent. Essential for a music app; without it, users would think the app is broken when their phone is on silent.
- **`interruptionMode: 'doNotMix'`**: When this app plays audio, it **pauses other apps' audio** (e.g., Spotify, podcast apps). Alternatives:
  - `'mixWithOthers'` — Both audio sources play simultaneously (useful for sound effects, not music)
  - `'duckOthers'` — Lowers other app's volume instead of pausing (used for navigation voice prompts, not music)

For a music player, `'doNotMix'` is correct — you don't want two songs playing at once.

---

## SECTION B — Code Questions: High-Level / Architecture (10)

---

### Q11. The `PlayerContext` exposes 20+ state values and actions in a single Context. What re-rendering problems does this create, and how could it be architecturally improved?

**Answer:**
Every time ANY value in the context changes (e.g., `position` updating every 250ms), ALL components consuming `usePlayer()` re-render — even if they only use `playSong`. This means:
- `FloatingMiniPlayer`, `SongItem`, `EchoPlayerModal`, `MusicFlowScreen`, all screens — ALL re-render **4 times per second** due to position updates.

**Architectural improvements:**
1. **Split into multiple contexts**: `PlayerStateContext` (position, duration, isPlaying) vs `PlayerActionsContext` (playSong, togglePlayPause) — actions rarely change.
2. **Use `useSyncExternalStore`** or a state manager like Zustand/Jotai with **selector-based subscriptions** so components only re-render when their specific slice changes.
3. **Memoize the value object** — Currently the Provider value is a new object literal every render. Wrapping it in `useMemo` would prevent unnecessary propagation.

---

### Q12. Why does `PlayerContext` use a `playSongInternalRef` pattern with `useRef` + `useEffect` to sync the ref? What bug does this prevent?

**Answer:**
In [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L150-L272):

```typescript
const playSongInternalRef = useRef<...>(async () => {});
// ...
useEffect(() => {
  playSongInternalRef.current = playSongInternal;
}, [playSongInternal]);
```

This is the **"latest ref" pattern** to break circular closure dependencies. The problem:
- `handleTrackFinished` calls `playSongInternal` to play the next song
- `playSongInternal` registers `onPlaybackStatusUpdate` as a listener
- `onPlaybackStatusUpdate` calls `handleTrackFinished`
- This creates a **circular dependency chain** in `useCallback` dependency arrays

If `handleTrackFinished` directly referenced `playSongInternal` via closure, every time `playSongInternal` changes (due to its own dependencies), `handleTrackFinished` would need to be recreated, which would cascade through the entire chain causing infinite re-registration of audio listeners.

The ref breaks this cycle — `handleTrackFinished` reads `playSongInternalRef.current` which always has the latest function without being a dependency.

---

### Q13. The app implements navigation without React Navigation or Expo Router tabs. Analyze the `index.tsx` architecture — what does `pointerEvents="box-none"` do on the `floatingOverlay` View, and why is it critical?

**Answer:**
In [`index.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/app/index.tsx#L58):
```jsx
<View style={styles.floatingOverlay} pointerEvents="box-none">
```

`pointerEvents="box-none"` means:
- The View **itself** does not receive touch events
- But its **children** (FloatingMiniPlayer, FloatingNavigationToolbar) DO receive touch events

This is critical because the `floatingOverlay` is positioned **absolutely** over the entire screen content (`position: 'absolute', bottom: 0, left: 0, right: 0`). Without `box-none`, this invisible View would **intercept all touch events** on the lower portion of the screen, making the underlying screen content (song lists, buttons) untappable. With `box-none`, touches pass through the transparent overlay to the screens below, but the mini player and nav bar still respond to taps.

---

### Q14. How does the `MusicFlowScreen` auto-play songs on scroll? Explain the `viewabilityConfigCallbackPairs` pattern and why `useRef` + `useState` lazy initializer is used instead of a simple `onViewableItemsChanged` prop.

**Answer:**
In [`MusicFlowScreen.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/screens/MusicFlowScreen.tsx#L49-L67):

The `viewabilityConfigCallbackPairs` pattern is used because React Native's `FlatList` requires the `onViewableItemsChanged` callback to be **referentially stable** (same function reference across renders). If passed inline, it causes the error: *"Changing onViewableItemsChanged on the fly is not supported."*

The `useState(() => [...])` **lazy initializer** ensures the callback pair is created once and never recreated. Inside the callback, `useRef` is used for `flowQueueRef`, `currentSongRef`, and `playSongRef` to access **latest values without being in the callback's closure dependencies**. This avoids:
1. FlatList's referential stability requirement being violated
2. Stale closures reading outdated `currentSong` or `playSong` values

The 60% `itemVisiblePercentThreshold` means a song auto-plays when 60% of its card is visible during the vertical paging scroll.

---

### Q15. The search screen implements debounced search suggestions. Trace the data flow from keystroke to suggestion display and explain why `useRef` is used for the timer instead of `useState`.

**Answer:**
Flow in [`EchoSearchScreen.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/screens/EchoSearchScreen.tsx#L72-L93):

1. User types → `handleQueryChange(text)` fires
2. Previous timer cleared via `clearTimeout(debounceTimer.current)`
3. If text > 1 char, new 250ms `setTimeout` starts
4. After 250ms of no typing, `getSearchSuggestions(text)` calls YouTube's suggest API
5. Response parsed (JSONP callback stripped) → `setSuggestions(suggs)` updates UI

**Why `useRef` not `useState` for the timer:**
- `useState` would cause a re-render every time the timer ID changes (on every keystroke) — but the timer ID is not needed for rendering
- `useRef` stores the timer ID **without triggering re-renders** — it's a mutable container that persists across renders
- Using `useState` would also create a stale closure problem: `clearTimeout` in the next keystroke would reference the old timer ID from the previous render's closure

---

### Q16. The `EchoHomeScreen` dynamically sorts community playlists based on recently played songs using `useMemo`. Explain the scoring algorithm and its computational complexity. Is `useMemo` justified here?

**Answer:**
From [`EchoHomeScreen.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/screens/EchoHomeScreen.tsx#L68-L92):

**Algorithm:** For each playlist, sum a score where each song in the playlist gets +1 for each recent artist that fuzzy-matches (bidirectional `includes()` check). Sort playlists descending by score.

**Complexity:** `O(P × S × R)` where P = playlists, S = avg songs per playlist, R = recent songs. With the 50-song recents cap and perhaps 10 playlists with 15 songs each: `10 × 15 × 50 = 7,500` string comparison operations per computation.

**Is `useMemo` justified?** Yes, because:
1. The dependency is `[recentSongs]` which only changes when navigating back to home or on mount
2. Without it, this `O(P×S×R)` sort runs on **every render** — and due to the PlayerContext re-rendering issue (position updates 4x/sec), this would run ~240 times per minute
3. The bidirectional `includes()` on lowercased strings is not trivial

---

### Q17. Why does the `EchoSettingsScreen` receive `visible` and `onClose` props and render as a `Modal`, instead of being a separate route/screen in the Expo Router stack?

**Answer:**
This is an architectural decision tied to the single-route pattern. The settings screen in [`EchoSettingsScreen.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/screens/EchoSettingsScreen.tsx#L18-L21) uses `Modal` with `presentationStyle="fullScreen"` and `animationType="slide"` because:

1. **State isolation** — Settings state (accent color, audio quality, etc.) is local `useState` that resets when the modal unmounts. No need to persist intermediate selections.
2. **Overlay consistency** — The FloatingMiniPlayer and navigation bar exist in `index.tsx`; a separate route would need its own overlay management.
3. **BackHandler integration** — The parent `index.tsx` handles `showSettings` in its back handler hierarchy before checking `activeTab`, giving a clean UX flow.
4. **No URL needed** — Settings doesn't need a shareable URL or deep link, so a route adds no value.

The trade-off is that this screen's state is **not preserved** when closed and reopened.

---

### Q18. The `SongItem` component conditionally renders a "Like Song" action in its action sheet. Why is the like button only shown when `isCurrent` is true? Is this a limitation or intentional design?

**Answer:**
From [`SongItem.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/components/SongItem.tsx#L188-L204):

```jsx
{isCurrent && (
  <Pressable onPress={toggleLikeCurrentSong}>...
```

This is a **limitation of the architecture**. The `toggleLikeCurrentSong` function from `PlayerContext` only operates on the **currently playing song**. The `SongItem` doesn't have an independent `toggleLikeSong(song)` action exposed from the context — only `toggleLikeCurrentSong()`. So if you show the like button for a non-playing song and the user taps it, it would like the wrong song (the currently playing one, not the one in the action sheet).

To fix this, the context would need to expose a `toggleLikeSong(song: Song)` action, or the `SongItem` could directly call `storage.toggleLikeSong()` without going through the context.

---

### Q19. How does the app handle the relationship between `playRadio()` and regular song playback? What type coercion is happening and what edge cases could arise?

**Answer:**
In [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L287-L298):

```typescript
const radioSong: Song = {
  ytid: station.streamUrl,  // URL stored in ytid field!
  title: station.name,
  artist: `${station.genre} Radio (Live)`,
  isLive: true,
  duration: 0,
};
```

A `RadioStation` is **coerced into a `Song`** by stuffing the stream URL into the `ytid` field. The `playSongInternal` function then checks `if (song.isLive && song.ytid.startsWith('http'))` to use the ytid directly as a stream URL.

**Edge cases:**
1. `isSongLiked(song.ytid)` would check if a URL is in liked songs — not meaningful
2. `addRecentlyPlayed(song)` stores a radio station in recents with a URL as its ID
3. The share URL would be `https://youtu.be/http://stream.url` — broken
4. If a radio stream URL doesn't start with `http`, it falls through to `getAudioStreamUrl()` which tries to resolve it as a YouTube video ID — will fail

---

### Q20. The `clearQueue` function keeps the current song. Explain why, and what state inconsistency would occur if it set `queue` to an empty array.

**Answer:**
From [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L412-L420):

```typescript
const clearQueue = () => {
  if (currentSong) {
    setQueue([currentSong]);
    setQueueIndex(0);
  } else {
    setQueue([]);
    setQueueIndex(0);
  }
};
```

If queue was set to `[]` while a song is playing:
1. `queueIndex` (0) would point to a non-existent array element
2. `nextSong` and `previousSong` check `queue.length === 0` and return early — user gets stuck
3. `handleTrackFinished` checks `queueIndex < queue.length - 1` which would be `0 < -1` → false, then checks `repeatMode === 'all' && queue.length > 0` → false → sets `isPlaying(false)` — music stops unexpectedly when current song ends
4. `removeFromQueue` has `if (index === queueIndex) return` — the currently playing song can never be removed, maintaining consistency with this design

---

## SECTION C — Code Questions: Low-Level / Difficult Practices (10)

---

### Q21. In the `decryptSaavnMediaUrl` function, explain what `CryptoJS.DES.decrypt` with `ECB` mode and `Pkcs7` padding does. Why is the decrypted URL modified with `.replace('_96', '_320')`?

**Answer:**
From [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L406-L419):

- **DES** (Data Encryption Standard) with **ECB** (Electronic Codebook) mode — each 64-bit block is encrypted independently with the same key `'38346591'`. ECB is the simplest mode (no IV needed), which is why JioSaavn likely uses it — simpler to implement.
- **PKCS7 padding** — Pads the plaintext to a multiple of 8 bytes (DES block size) before encryption.
- The key `'38346591'` is JioSaavn's publicly known encryption key for media URLs.
- After decryption, the URL contains `_96` indicating 96kbps quality. The `.replace('_96', '_320')` upgrades the CDN path to serve 320kbps audio instead. JioSaavn's CDN stores multiple quality versions at predictable URL patterns — only the bitrate segment differs.

This is **reverse-engineered API behavior** — the encryption is DRM-lite, not true security.

---

### Q22. The `parseLrc` function uses a regex `\[(\d{2}):(\d{2})\.(\d{2,3})\](.*)` to parse LRC lyrics. Explain what `match[3].padEnd(3, '0')` does and why it's necessary.

**Answer:**
From [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L572-L590):

LRC format timestamps can have either 2 or 3 digit milliseconds:
- `[01:23.45]` → match[3] = `"45"` (centiseconds)
- `[01:23.456]` → match[3] = `"456"` (milliseconds)

`padEnd(3, '0')` converts `"45"` → `"450"`, so when divided by 1000 it gives `0.450` seconds (correct).

Without `padEnd`: `"45" / 1000 = 0.045` seconds — **off by 10x**. The lyric would appear 405ms too early.

The regex captures `\d{2,3}` to handle both formats, and `padEnd` normalizes them to milliseconds before the division. The final result is sorted by time (`result.sort((a, b) => a.time - b.time)`) to handle LRC files with out-of-order timestamps.

---

### Q23. In `toggleShuffle`, explain the Fisher-Yates shuffle implementation. Why is the current song extracted first and what subtle bug exists in this implementation?

**Answer:**
From [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L363-L378):

```typescript
const current = queue[queueIndex];
const rest = queue.filter((_, i) => i !== queueIndex);
for (let i = rest.length - 1; i > 0; i--) {
  const j = Math.floor(Math.random() * (i + 1));
  [rest[i], rest[j]] = [rest[j], rest[i]];
}
setQueue([current, ...rest]);
setQueueIndex(0);
```

1. Current song is extracted so it stays at index 0 — user shouldn't hear a different song when enabling shuffle
2. Fisher-Yates iterates from end to start, swapping each element with a random earlier element — O(n) uniform shuffle

**Subtle bug:** When shuffle is **toggled OFF** (`nextShuffle = false`), the original queue order is **not restored**. The queue remains in its shuffled order. A proper implementation would save the original queue order before shuffling and restore it on un-shuffle. The `setIsShuffle` setter only updates the boolean — the queue stays scrambled.

---

### Q24. Explain why `getSearchSuggestions` strips a JSONP callback wrapper. What is the response format, and why can't it use regular JSON parsing directly?

**Answer:**
From [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L384-L404):

YouTube's suggest API (`suggestqueries-clients6.youtube.com`) returns **JSONP** (JSON with Padding), not plain JSON:

```
func && func(["query", [["suggestion 1", 0, [512]], ...], ...])
```

This format exists because the API was designed for **browser `<script>` tag injection** where CORS wasn't available. The `callback=func` parameter tells the server to wrap the response in a function call.

The code strips this:
```typescript
text.replace(/^func\s*&&\s*func\(/, '').replace(/\);?\s*$/, '')
```

This removes `func && func(` from the start and `);` from the end, leaving pure JSON for `JSON.parse()`. Without stripping, `JSON.parse()` throws `SyntaxError` because `func && func(...)` is not valid JSON — it's valid JavaScript.

---

### Q25. The `previousSong` function has a `position > 3` check. What UX behavior does this implement, and why is the value `3` (not 0 or 5)?

**Answer:**
From [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L348-L360):

```typescript
if (position > 3 && playerRef.current) {
  await seekTo(0);
  return;
}
```

This implements the **standard music player "previous" behavior** found in Spotify, Apple Music, etc.:
- If more than 3 seconds into the song → restart the current song from the beginning
- If within the first 3 seconds → go to the actual previous track

**Why 3 seconds:**
- 0 seconds: Would make it impossible to restart a song (always goes to previous track)
- 5 seconds: Too long — user has to wait 5 seconds before "previous" does the expected thing
- 3 seconds is the **industry standard** (Spotify uses ~3s, Apple Music uses ~3-5s). It gives enough time for the user to realize they pressed previous intentionally but not so much that the restart behavior feels sluggish.

---

### Q26. In the `searchEchoMusic` function, the YouTube Music Innertube API is called with `clientName: 'WEB_REMIX'`. What is WEB_REMIX and why is it used instead of `'WEB'` or `'ANDROID'`?

**Answer:**
From [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L94-L113):

`WEB_REMIX` is the internal client identifier for **YouTube Music's web interface** (`music.youtube.com`). Different client names return different response formats:
- `WEB` — Regular YouTube (`youtube.com`), returns video-centric results with `videoRenderer`
- `WEB_REMIX` — YouTube Music, returns music-centric results with `musicCardShelfRenderer`, `musicResponsiveListItemRenderer`, categorized by Songs/Artists/Albums
- `ANDROID` — Android YouTube app, includes DRM-protected streams

`WEB_REMIX` is used because:
1. Results are **music-categorized** (songs vs artists vs albums vs playlists) — the response structure has `SearchItemType` classification built in
2. Thumbnail URLs are **music-formatted** (square album art, not video thumbnails)
3. It's the same API YouTube Music's website uses, so it returns high-quality music metadata

The fallback `searchYouTubeFallback` uses `clientName: 'WEB'` as a degraded backup with simpler parsing.

---

### Q27. Explain the `isSettingUpRef` guard in `playSongInternal`. What race condition does it prevent, and why is `useRef` used instead of `useState`?

**Answer:**
From [`PlayerContext.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/context/PlayerContext.tsx#L191-L265):

```typescript
if (isSettingUpRef.current) return;
isSettingUpRef.current = true;
// ... async work ...
isSettingUpRef.current = false; // in finally block
```

**Race condition:** If the user rapidly taps two different songs, the first `playSongInternal` call starts an async fetch for a stream URL. Before it resolves, the second call starts. Without the guard:
1. Both calls create `AudioPlayer` instances
2. Both set `playerRef.current` — first player is orphaned (memory leak, still playing)
3. Both call `setCurrentSong` — UI flickers between songs
4. Status subscriptions from the first player may fire after the second player is set up

**Why `useRef` not `useState`:**
- `useState` updates are **asynchronous** — `setIsSettingUp(true)` doesn't take effect immediately, so a second call in the same event loop tick would still read `false`
- `useRef` mutations are **synchronous** — `isSettingUpRef.current = true` is immediately visible to any subsequent synchronous code
- No re-render needed — this is an internal guard, not a UI concern

---

### Q28. The `addRecentlyPlayed` function uses `.filter().slice()` chaining. What data integrity problem does this pattern solve, and why is the order `filter → unshift → slice` important?

**Answer:**
From [`storage.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/storage.ts#L136-L143):

```typescript
const filtered = recents.filter((s) => s.ytid !== song.ytid);
const updated = [song, ...filtered].slice(0, 50);
```

**Pattern:** Remove → Prepend → Cap

1. **`filter`**: Removes any existing instance of this song — prevents **duplicates**. Without this, playing the same song 50 times fills the entire recents list with one song.
2. **`[song, ...filtered]`**: Prepends the song to the front — most recently played is **always first** (MRU order).
3. **`.slice(0, 50)`**: Caps at 50 entries — prevents **unbounded growth** in AsyncStorage.

**Why this order matters:**
- If you slice first then filter: you might remove the existing entry from beyond position 50 and add a new entry, growing the list to 51
- If you unshift before filter: the same song appears twice momentarily
- Filter → Prepend → Slice guarantees: no duplicates, correct MRU order, bounded size, all in a single synchronous chain

---

### Q29. The `formatSongThumbnail` function replaces a regex in Google URLs. What does the replacement `=w${size}-h${size}-l90-rj` do, and why is this URL manipulation necessary?

**Answer:**
From [`api.ts`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/services/api.ts#L76-L82):

```typescript
url.replace(/=w\d+-h\d+[^"]*/, `=w${size}-h${size}-l90-rj`)
```

Google's image CDN (`googleusercontent.com`) serves dynamically resized images via URL parameters:
- `w544` — width 544px
- `h544` — height 544px
- `l90` — quality level 90% (JPEG compression)
- `rj` — format: JPEG

The regex matches existing size parameters like `=w120-h120-l90-rj` or `=w60-h60-s-c-k-c0x00ffffff-no-rj` and **replaces them** with the desired size.

**Why necessary:** YouTube Music API returns thumbnails at small default sizes (60x60 or 120x120). The app needs larger images for album art display (330px artwork in the player modal). Instead of making a separate API call for larger images, the URL is simply modified — Google's CDN dynamically resizes on the server side. The default `size = 544` provides crisp images on high-DPI mobile screens.

---

### Q30. In the `EchoPlayerModal`, the seek bar uses `e.nativeEvent.locationX` for touch-based seeking. Explain the calculation and what `Math.max(0, Math.min(1, ...))` (clamping) prevents.

**Answer:**
From [`EchoPlayerModal.tsx`](file:///Users/ritesh/Documents/TEst%20Diff/sona/src/components/EchoPlayerModal.tsx#L78-L84):

```typescript
const handleSeekTouch = (e: any) => {
  if (duration <= 0) return;
  const { locationX } = e.nativeEvent;
  const barWidth = SCREEN_WIDTH - 48;      // 24px padding × 2
  const fraction = Math.max(0, Math.min(1, locationX / barWidth));
  seekTo(fraction * duration);
};
```

1. `locationX` — the X coordinate of the touch **relative to the pressed element** (the seek bar), not the screen
2. `barWidth = SCREEN_WIDTH - 48` — total seek bar width accounting for 24px horizontal padding on each side of the `safeArea`
3. `fraction = locationX / barWidth` — converts pixel position to 0-1 progress fraction
4. `seekTo(fraction * duration)` — converts fraction to seconds

**What clamping prevents:**
- `Math.max(0, ...)` — if `locationX` is negative (touch started slightly before the bar's left edge due to `hitSlop` or touch imprecision), prevents seeking to a negative time
- `Math.min(1, ...)` — if `locationX > barWidth` (touch extends past the right edge), prevents seeking beyond the song's duration which could crash the audio player or cause undefined behavior
- Without clamping: `seekTo(-5)` or `seekTo(duration * 1.3)` could throw native audio errors
