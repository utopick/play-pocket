# PLAY.Pocket

A pocket guitar rig in one HTML file: plug a guitar into an audio interface or a USB guitar cable, and play it through pedals, amps and cabs laid out on a board, over your own backing tracks and loops, in the browser. The sister app of [TRACK.Pocket](https://github.com/utopick/track-pocket), [TUNE.Pocket](https://github.com/utopick/tune-pocket) and [SING.Pocket](https://github.com/utopick/sing-pocket): no install, no dependencies, no network requests.

**Status: prototype.** Not released yet. It has 16 pedals, 4 amps and 4 cabs (where each comes from: [docs/UNITS.md](docs/UNITS.md)), a board to order them, a synthesized test riff, a looper (TUNE's, with edit tools, effects and kept loops), a synth, a voice changer, backing tracks from your device, a mixer, a latency test and the usual Files tab (board export and import, themes, options, help in English and Spanish). Next: the rest of TUNE's looper (effects, quantize, editing, keeping loops), and install as an app.

## How to run it

Serve the folder from `localhost` or HTTPS (the browser only gives the microphone to secure pages), for example with the workspace's `dev-server.js`, and open it in Chrome or Firefox on a computer, or Firefox on Android. Use headphones or the interface's output: with a phone's own mic and speaker the sound feeds back.

## The basics

- **PEDALS, AMPS, CABS** (the bottom bar) show one unit at a time; **◁ ▷** or a swipe changes it. Drag a knob up or down; the switch turns the unit on.
- **BOARD** (the squares icon at the top): the signal enters at the top right and snakes square by square to the speaker at the bottom right. Tap an empty square to add a unit, tap a unit twice to edit it, drag it to move it. **RIFF ▶** plays a test riff through the board.
- **MIXER** (the faders icon, or the speaker square): guitar, test riff, tracks, loops, click and master volumes, mutes and meters.
- **LOOPER** (the loop icon, TUNE.Pocket's): tempo, click, loop length (1, 2, 4 or 8 bars), recording delay and auto quantize; tap an empty pad to count in and record your board, tap a loop to play or stop it on the bar. EDIT: volume, overdub, reverse, normalize, nudge, quantize, crop, clear. FX: repeat, reverse, octave. UNDO. The loops are kept on the device.
- **SYNTH** (the keyboard icon): eight instruments made by the app (subtractive, FM and organ synthesis: no samples), played from a MIDI controller (notes, velocity, sustain pedal, pitch bend, mod wheel) or the keys on the screen, with filter and envelope knobs, optionally through the pedal board.
- **VOICE** (the face icon): the microphone through a voice changer (pitch shifter plus robot, megaphone, alien, choir and echo). Use headphones.
- **MIXER** has a knob and a mute for each source (guitar, test riff, synth, voice, tracks, loops, click, master) and a lower button that swaps it for **FX PADS** on the whole mix (lo-fi, tape, radio, stutter, reverb... to latch or hold).
- **TRACKS** (the note icon): add audio files from the device (kept in the browser, never uploaded) and play along.
- **RECORD** starts listening. The thin bar under the top icons is the input level.
- **The disk** (top right) opens Files: **FILES** (export, import, reset the board), **THEME**, **OPTIONS** and **HELP**.
- **OPTIONS** has the stats the browser reports, the **LATENCY TEST** (a ball bounces; pick a string on each click; or plug a cable from the output into the input and let it measure itself), the input, channel, gain and output, and the engine switches (sample rate, buffer, oversampling, raw input).

## Browsers and systems: what works where

The Pocket apps that listen (TUNE's tuner and looper, SING, PLAY) depend a lot on the browser and the system, more than on the phone or computer. Hearing yourself through the app while you play is the hard case: it needs the sound to go out and come back in quickly. These are round trips we measured in November 2026, with a cable from the audio interface's output back into its input, using [LAB.Pocket](https://utopick.github.io/test/latency-lab/) and PLAY.Pocket's latency test:

| System | Browser | Round trip | Play through it? |
|---|---|---|---|
| Mac + Focusrite Scarlett Solo | Chrome | 18–20 ms | yes |
| Mac + Focusrite Scarlett Solo | Firefox | 18 ms | yes (mind the output, below) |
| Mac + Focusrite Scarlett Solo | Safari 26 | about 110 ms | no |
| Android 16 phone, USB guitar cable or its mic | Firefox | 30–33 ms | yes |
| Android 16 phone | Chrome, and every browser built on it | 110–340 ms | no |
| iPad (2019), iPadOS 18 | Safari, and every other iOS browser | 77–127 ms, built-in mic only | no |

Under about 30 ms it feels like an amp; up to 60 ms it is playable with a little lag; beyond that it is an echo. Tuning, recording and playback work everywhere: only live monitoring needs the fast ones.

### The nitpicks

- **Use one interface for input and output.** Two devices run on two clocks: the Scarlett in and the Mac's headphone jack out gave 52–55 ms instead of 19.
- **Match the sample rate.** If the app runs at 44.1 kHz and the interface at 48 kHz, the browser converts between them, which costs about 6 ms. Chrome tells the app the input's rate, so it matches it by itself; Firefox does not, so set the rate by hand there (48000 Hz for most interfaces).
- **Firefox and the output choice.** Chrome can send the sound straight to any output. Firefox can only do it through its media player, which buffers for smooth music: 72 ms instead of 18. In Firefox, leave the app's output on system default and choose the output in the system's sound settings.
- **Raw input.** The browser's echo cancelling, noise suppression and automatic gain are made for calls: they add delay and change your tone. The apps turn them off.
- **The buffer setting barely matters.** The browser's "interactive" mode is already its smallest buffer; asking for 5 ms gained 1–2 ms on a computer and nothing on Android.
- **Distortion oversampling** (smoother drive) costs 3 ms at 2× and 4 ms at 4×.
- **Android:** Chrome's microphone path is slow, and it is the same in Edge, Brave, Samsung Internet and in-app browsers. Firefox uses Android's fast audio path: use Firefox to play through the app.
- **iPhone and iPad:** every browser there is Safari underneath. Once a page opens the microphone it records the built-in mic even when a USB interface is chosen ([WebKit bug 211192](https://bugs.webkit.org/show_bug.cgi?id=211192)), moves the sound to the iPad's own output, and lags more and more after the iPad sleeps. Pages that only play sound (TRACK) do play through an interface. Native apps like AUM use the interface fine: it is the browser, not the iPad.
- **Your hands count too.** Played by hand, the same test reads about 20 ms more than with a cable: most people play slightly after a click they hear.
- **The apps stop listening in the background**, so they never come back with a stuck microphone after the screen locks.

## Files

- `index.html`: the complete app (markup, styles, audio engine, help)
- `docs/UNITS.md`: where every pedal, amp and cab comes from and how it is built
