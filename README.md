# Sound-Reactive VJ- Gobelins Workshop

A live **VJ host** that plays each student's visual back-to-back in an
**infinite loop**, blinking between them, all reacting to the same music (mp3 or live mic).

Every student brings **their own self-contained page** (three.js), loaded in an iframe. The host runs one audio analyser and
sends each on-screen scene four simple signals. Any tech / stacks works, and one scene
crashing never touches the others.

Each scene is **independent**: open it on its own and it runs standalone (it automatically plays the identical MP3 tracks from the playlist); embedded in the host, the same code receives the master's
audio instead- auto-detected, nothing to change. The shared audio brain is the
module [`src/sounds/Analyzer.js`](src/sounds/Analyzer.js); the host engine is
the [`src/VJHost.js`](src/VJHost.js) singleton, started by [`src/main.js`](src/main.js).

```bash
pnpm install
pnpm dev
# drop more .mp3 in public/tracks/
```

## Add your scene

Add a **local folder** and open a **pull request**- it joins the loop with **no list to edit**:

- copy `public/vj-scenes/template/` → `public/vj-scenes/<your-name>/`
- edit `index.html`, save (the dev server auto-detects it), then open a PR if everythings is good.

### The contract

Import the `Analyzer` module, then implement any of the lifecycle hooks and read the audio:

```html
<script type="module">
  import Analyzer from '/sounds/Analyzer.js'
  const audio = new Analyzer()   // auto: standalone MP3 player/mic, or the host's broadcast

  audio.onLoad(  () => {} )   // create your renderer / load assets   (once; async ok)
  audio.onWarmup(() => {} )   // compile shaders / draw one frame      (once, after load)
  audio.onPlay(  () => {} )   // you're on screen → start your loop
  audio.onStop(  () => {} )   // you're off screen → pause your loop

  audio.onAudio( a => {
    a.volume             // instant loudness          0..1
    a.volumeSmooth       // loudness, smoothed         0..1
    a.kick               // spikes on each kick        0..1
    a.kickHard           // strong beats only          0..1
    a.volumeByFrequency  // loudness per frequency     Float32Array(256), 0..1
  } )
</script>
```

The same `audio` object holds the latest values to read inside your own render loop.
The lifecycle runs **load → warmup → play**, so the host can warm your shaders
while you're still off-screen- switching stays smooth. **Pause on `onStop`** so
off-screen scenes don't cook the GPU.

**See what the analyzer feels.** Press **`d`** to toggle the debug overlay (on by
default when standalone)- it draws the `volumeByFrequency` spectrum on top and
`volume` / `volumeSmooth` / `kick` / `kickHard` meters below (the kick rows show
their adaptive threshold). To embed it yourself: `import AnalyzerDebug from
'/sounds/AnalyzerDebug.js'` and `new AnalyzerDebug( audio )`.

**Change the track.** Drop `.mp3`s in `public/tracks/` (auto-discovered into
`tracks.json`), then switch with **`.`** / **`,`** or hit **`m`** for the mic (see
[Run it live](#run-it-live-keyboard)). Each track's kick/volume tuning is auto-picked
from its filename in [`src/sounds/TrackTuningConfig.js`](src/sounds/TrackTuningConfig.js).

### The helper modules (`AnalyzerDebug` · `SoundPlayer` · `PlayerControl`)

All three ship in `src/sounds/` and are wired up **for you** in standalone mode- you rarely
construct them by hand. Reach for the API only when you want manual control.

**`AnalyzerDebug`**- the draggable overlay (the `volumeByFrequency` spectrum on top,
`volume` / `volumeSmooth` / `kick` / `kickHard` meters below). It only *reads* the signals,
so it works with a **live or a receive** `Analyzer`. Standalone scenes show it by default and
`d` toggles it; to place your own:

```js
import AnalyzerDebug from '/sounds/AnalyzerDebug.js'
const debug = new AnalyzerDebug( audio, { width: 160, height: 95, bins: 8 } )  // opts optional
debug.toggle()   // show / hide   ·   debug.show()  ·  debug.hide()  ·  debug.dispose()
```

**`SoundPlayer`**- "choose & play the music" for a standalone scene: it owns the playlist
(`/tracks/tracks.json`), the `<audio>` element, the keyboard shortcuts (`m` mic · `.` / `,`
track), and the "now playing" control. The `Analyzer` constructs it **on the first user
gesture** (once the AudioContext unlocks) and keeps it at `audio.player`, so you normally just
drop mp3s in `public/tracks/`. To drive it from code once it exists:

```js
audio.player?.nextTrack()                       // also prevTrack()
audio.player?.useTrack( '/tracks/01-Tame.mp3' ) // load a specific file (optional 2nd arg: startTime)
audio.player?.useMic()                          // switch to the microphone
```

Embedded in the host there is **no** `SoundPlayer`- the host owns the audio and your
`Analyzer` runs in **receive** mode, so `audio.player` stays undefined (hence the `?.`).

**`PlayerControl`**- the bottom-right "now playing" widget: track name (click to skip),
hover/drag scrubber, `mm:ss` time. It is **pure UI**- it doesn't choose, load, or analyse
anything; you hand it a few accessors + callbacks and it draws. Both `SoundPlayer` and the
host's master player ([`src/main.js`](src/main.js)) create one, so a standalone scene already
has it. To build your own (or just re-theme it):

```js
import PlayerControl from '/sounds/PlayerControl.js'
const control = new PlayerControl( {
  getAudioEl:   () => audioEl,       // read duration / currentTime
  getSource:    () => 'mp3',         // 'mic' hides the scrubber
  getTrackName: () => 'My Track',    // display name (no extension)
  onSkip:       () => {},            // track name clicked
  onSeek:       ( seconds ) => {},   // scrub released- commit it
  // optional theming: fillBackground · fillShadow · nameHoverColor · idleOpacity
} )
control.dispose()                    // removes the widget + its <style>
```

Only one widget exists at a time (it bails if a `.vj-track-widget` is already on the page), so
the host and a standalone scene never draw two.

## Use it in your own project

The audio engine is **dependency-free**. Copy one folder, drop in mp3s, read four signals.

### Getting started

**Copy the engine.** Vite won't import a `/public` JS file from bundled code, so where it goes depends on your page:

- **Raw page in `public/`** (libs via importmap/CDN, like this repo's scenes):
  `src/sounds/` → `public/sounds/`. Import `'/sounds/Analyzer.js'`.
- **Bundled entry** (libs from npm, e.g. `import * as THREE from 'three'`):
  `src/sounds/` → your `src/sounds/`. Import `'./sounds/Analyzer.js'`.

**Add music.** Copy `public/tracks/` → `public/tracks/` (no mp3s → mic). Copy `vite-plugin-vj-tracks.js` to your root and register it- it keeps `tracks.json` in sync:

```js
// vite.config.js
import vjTracksPlugin from './vite-plugin-vj-tracks.js'
export default defineConfig( { plugins: [ vjTracksPlugin() ] } )
```

> **Cold start:** the plugin writes `tracks.json` just after Vite's first scan, so a fresh run can't serve it and falls back to the mic. Keep one `tracks.json` in the folder, or restart `pnpm dev` once.

**Read the signals** (both setups):

```js
const audio = new Analyzer()
audio.onAudio( a => { /* a.volume · a.kick · a.volumeByFrequency */ } )
```

`pnpm dev` and open your page. It plays the tracks (or mic); visuals react on their own. (`AnalyzerDebug.js` ships too- `new AnalyzerDebug( audio )` for the live overlay.)

### Integrate it here

Add your scene as a **local folder** and open a **pull request**- no list to edit:

```bash
cp -r public/vj-scenes/template public/vj-scenes/your-name
# edit index.html, save, then open a PR
```

Save and the dev server auto-detects the folder. Once merged it joins the loop: the
host loads it in an iframe and the same `Analyzer` auto-switches to **receive** mode-
it reads the show's live audio instead of its own. Nothing in your code changes.

**Examples to copy from** (`public/vj-scenes/`):
- `template`- three.js (WebGL) via CDN. Simplest starting point.
- `02-spectrum-landscape`- custom 3D **WebGL** landscape tunnel.
- `_warmup`- the host's instant idle scene (index 0).

## Run it live (keyboard)

```
→ / n   next scene          ← / p   previous scene
0-9 + ⏎ jump to scene #      space   pause / resume auto-advance
m       toggle microphone    . / ,  next / prev track
d       audio debug overlay
```

## Layout

```
src/main.js              bootstrap- start overlay, hands off to the VJHost singleton
src/VJHost.js            the host engine (iframe layers · blink · loop · broadcast)
src/sounds/Analyzer.js   the audio brain- analysis only (copy the whole folder to reuse: public/sounds/ for a raw page, src/sounds/ for a bundled entry)
src/sounds/SoundPlayer.js    standalone "choose & play the music" (playlist · <audio> · keys)- auto-loaded
src/sounds/PlayerControl.js  the "now playing" control- shared by the host and standalone scenes
src/sounds/ui/           the control's markup & styles (string modules, so they ride along with /sounds)
src/sounds/AnalyzerDebug.js  optional draggable debug overlay
public/
  vj-scenes/<name>/index.html   each student's self-contained page
  tracks/                drop mp3s here (auto-discovered; tracks.json generated inside)
vite-plugin-vj-tracks.js the "auto track update"- regenerates tracks.json (copy this to reuse)
vite.config.js           tiny plugins: discover vj-scenes/*, tracks/*, serve /sounds from src/sounds
```
