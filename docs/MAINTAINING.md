# Maintaining PLAY.Pocket

A guide for whoever looks after this code next. It explains how the app is put together, in plain words, so you can
find your way before you change anything. It describes the installable web app (PWA).

## What it is
PLAY.Pocket is a pocket guitar rig: an instrument plugged into the phone or an audio interface plays through pedals,
amps and cabs laid out on a board, with a looper (TUNE.Pocket's), backing tracks, a mixer and a latency test. It runs
in any modern browser and installs as a web app (PWA). Where each pedal, amp and cab comes from is in `docs/UNITS.md`.

## The one-file rule
The whole app is **one file: `index.html`** (`sw.js`, `manifest.webmanifest` and `icons/` only make it installable).
There is no build step, no framework and no dependencies, so anyone can open it, read it and change it.
- Inside, in this order: a comment at the top with the layout of the file and every storage key (read it first),
  `<style>`, the markup (one `<section>` per view), one big `<script>`.
- To find something, search for its name. Buttons have ids; the code wires them with `$("#id").onclick = ...`.
- `$` is `document.querySelector` and `$$` is `querySelectorAll`. `toast("...")` shows a short message. `esc()` cleans
  any text that goes into `innerHTML`.
- The help texts live in the `HELP` array, in English (`en`) and Spanish (`es`). When you change what a button does,
  change its help in both. The interface itself is in English.

## The views
The header buttons: `live` (LIVE PLAY: one pedal, amp or cab full screen, section `rig`), `board` (the pedal board),
`loop` (looper), `synth` (sixteen instruments, MY SOUND and four samples, MIDI or the keys on the screen), `voice` (voice changer), `tracks`, `mix` (mixer, with the FX pads on the whole mix), `files` (FILES, THEME, OPTIONS, HELP). `showView` switches between them and
asks the view to draw itself (`renderUnit`, `boardFace`, `lpFace`, `trFace`, `mixFace`). The bottom ribbon of LIVE PLAY
(RECORD, CABS, AMPS, PEDALS) has room for a fifth button that is not decided yet.

## The state: where things are kept
- `PEDALS`, `AMPS`, `CABS`: the units, each with its knobs (`[id, label, min, max, default, how to show the value]`) and the function that
  builds its audio nodes. `FXI` finds a unit by id; `KINDS` names the three kinds.
- `S`: everything the user chose: the unit shown per kind (`S.sel`), every unit's switch and knobs (`S.fx`), the 27
  squares of the board in signal order (`S.slots`), the mixer (`S.mix`), the looper settings (`S.loop`), the synth (`S.syn`) and the voice (`S.voice`). Saved as
  `play.state`.
- `A`: the audio engine while it runs (`A.ctx`, `A.nodes`, `A.mic`). `LP`: the looper's loops and recording. `TR`: the
  backing tracks. `OPT`, `AIN`, `AOUT`, `LAT`: the engine options, the input, the output and the last latency test.

Rule of thumb: change the state, call `save()`, then the view's `...Face()` or `render...()` function.

## Sound
All sound is the Web Audio API. The `AudioContext` is made on the first tap, at the input's own sample rate (no
resampling), with the raw input (no echo cancelling, noise suppression or auto gain).
- `engineStart(mic)` builds everything once: input gain, the mixer's gains, every unit, the master and a limiter.
  Switching a unit off crossfades to its dry path. Moving units only reconnects them (`rewire`).
- The worklets (`WORKLET_JS`) hold the code that needs to run sample by sample, the pick detector of the latency test
  and the looper's recorder, which stamps every block with the audio clock so loops are cut exactly.
- The looper is TUNE.Pocket's (`lp...` functions, the `CLK` clock). The recording is moved by the measured latency
  (`lpLat`, REC DELAY).
- The app stops listening when it goes to the background.
- **MIDI** is the sister apps' block: `<script id="midi-device">` (the panel, identical in all apps), `micard-css`, and the MIDI core in the main
  script (`MI`, `midiMsg`, ASSIGN, MONITOR, MAP, MOD). PLAY adds pitch bend (`MI.bend`). What the controls do follows the screen:
  `MI.defaults()` lists the knobs (`miKnobs()`: the unit's knobs, the mixer with the master first, the synth, voice and tracks knobs through
  `mkPanel`'s `root._mk`) and the pads (`miPadsOf()`: the board squares, looper pads or loop FX, the buttons of TRACKS, the mixer FX pads, the voice pads;
  on the rig pad 1 is on/off and pad 2 TEST). `miPadPress` clicks or sends pointerdown/up to the element, so the pads do exactly what a tap does.
- **SYNTH waves and samples:** `SYN` has `MY SOUND` (`type: "sub"`, its `osc` is `S.syn.mine`, up to six `[wave, detune, octave]`) and four `type: "usr"`
  instruments (`SMPS[slot]`, mono Float32, up to 12 s, IndexedDB store `samples`). `synChainFace` draws the six squares and the buttons under them.
  The recorder hooks the looper's capture worklet (`SREC`). The FM piano samples were removed (430 KB). Idea: a slicer for the samples (see the roadmap).

## Storage
- localStorage: `play.state`, `play.opts`, `play.audioIn`, `play.audioOut`, `play.lat`, `play.theme`, `play.lang`.
- IndexedDB `play-pocket`: store `tracks` (the backing tracks as blobs) and store `loops` (the looper's loops).
- Everything that comes back is checked: `vetFx`, `vetSlots`, `vetMix`, `vetSyn`, `vetVoice` for the board and the other settings (saved or imported), the checks
  in `lpRestore` for the loops and in `trLoad` for the tracks. Keep it that way for anything new.
- Saved data must keep loading: never rename a unit id or a knob id, add new ones.

## The web app (PWA)
- `manifest.webmanifest`, `sw.js` and `icons/` make the page installable. The service worker keeps the app files for
  offline use and tries the network first, so an online visit gets the newest version. If the page ever needs a new
  file at start, add it to the list in `sw.js`. More in `docs/PWA.md`.
- It is published by GitHub Pages from `main`, root folder, at `https://utopick.github.io/play-pocket/`. Pushing
  `main` publishes it. There is no build and no workflow.
- The version is `APP_VERSION` in `index.html`, shown in the help title. The last help section, "What's new", lists
  the changes of each version in both languages.

## Trying a change
1. The microphone needs `localhost` or https: run the small server in the parent folder (`Pocket dev server.command`,
   or `node dev-server.js`) and open `/play-pocket/`.
2. Look at it on a phone-sized window (about 390 × 844): the apps are designed for phones first.
3. Open the browser console. A change is not done until it shows no red errors.
4. Run the automated checks (a real Chrome driven by Selenium, with Chrome's pretend microphone):
   `python3 -m venv .venv && .venv/bin/pip install -r tests/requirements.txt` once, then `.venv/bin/python tests/e2e.py`.
   `tests/e2e.py` is the same file in the four apps. Add a test there when you fix a bug.
5. The pretend microphone only proves that the code runs. Tone, feedback and every latency number need a real
   instrument, interface and phone.

## Rules of thumb
- One file, no dependencies, no network. Keep it that way.
- Phones first: big squares, nothing that needs a hover or a right click.
- Every new button gets a `title`, a line of help in English and Spanish, and a toast that says what it did.
- Fonts and code must be free and open.
- Small commits with clear messages. Try each change on a phone before a release.
