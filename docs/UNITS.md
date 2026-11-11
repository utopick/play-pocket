# Where the pedals, amps and cabs come from

Every unit in PLAY.Pocket is written from scratch for this app, from **published descriptions of how the circuits or algorithms work**: circuit analyses, papers, textbooks and one public-domain program (Freeverb). No code was copied from other effect projects. GPL projects (Guitarix, for example) were not used, so the app stays MIT.

Unit names are generic. "TS-style" or "RAT-style" below names the family of circuit a design follows, the way guitarists describe pedals. The names are not products, and the app does not claim to copy any one.

## How a unit is built

All units live in `index.html` (`PEDALS`, `AMPS`, `CABS`). Each one is a **box**: `in → dry → out`, and `in → (effect) → wet → out`.

- **insert** units (drives, amps, cabs, EQ, dynamics, tremolo, vibrato): when on, the dry path fades out and the wet path fades in. When off, the sound passes untouched.
- **send** units (chorus, flanger, phaser, delay, reverbs): the dry path stays, and the effect is added at the MIX level.

Switching a unit on or off is a 20 ms crossfade (`setTargetAtTime`), so nothing is rebuilt while you play. The **board** sets the order. Units not on the board are disconnected.

Most units are made of the browser's own audio nodes:

- `GainNode`
- `BiquadFilterNode` (the RBJ "Audio EQ Cookbook" filters, built into Web Audio)
- `WaveShaperNode`
- `DelayNode`
- `ConvolverNode`
- `OscillatorNode` (LFOs)

These run as compiled code inside the browser and add no delay. Three units need sample-by-sample code, so they run in an **AudioWorklet** (`WORKLET_JS`): the noise gate, the compressor and the room reverb. A worklet adds no delay either: it works inside the same 128-sample blocks.

No unit uses look-ahead, and no cab uses a recorded impulse, so **no unit adds delay**. The only exception is distortion oversampling, which is an option and off by default.

## Pedals

| Unit | Kind | Based on | How it is made |
|---|---|---|---|
| NOISE GATE | dynamics | the standard downward gate with hysteresis, as in Zölzer, *DAFX* (2nd ed., Wiley 2011), ch. 4 "Nonlinear processing" | Worklet. A peak envelope; the gate opens above THRESHOLD in 1 ms and closes over RELEASE once the level is 6 dB under it (the gap stops it chattering). |
| COMPRESSOR | dynamics | D. Giannoulis, M. Massberg, J. D. Reiss, "Digital Dynamic Range Compressor Design — A Tutorial and Analysis", *J. Audio Eng. Soc.* 60(6), 2012 | Worklet. Feed-forward, level detected in dB, soft knee (6 dB), ratio 4:1, smoothed attack (ATTACK knob) and 150 ms release, no look-ahead. SUSTAIN lowers the threshold and adds matching make-up gain. |
| BOOST | drive | a clean booster: a gain stage with a treble lift | Gain and a high shelf at 2 kHz. |
| OVERDRIVE | drive | TS-style. ElectroSmash, "Tube Screamer Analysis": the clipping stage only clips above about 720 Hz, which gives the mid hump; soft diode clipping; a tone control after it | A low shelf (−12 dB at 720 Hz) before a `tanh` soft clipper (DRIVE sets how hard), then a lowpass for TONE. |
| DISTORTION | drive | RAT-style. ElectroSmash, "ProCo RAT Analysis": a very high-gain op-amp stage, hard clipping, and a FILTER that is a lowpass (turned up = darker) | A high-pass at 60 Hz, up to ×200 gain, a hard clipping curve, a lowpass from 12 kHz down to about 500 Hz. |
| FUZZ | drive | Fuzz Face-style. ElectroSmash, "Fuzz Face Analysis": two transistors, very high gain, the two halves of the wave clip differently | Up to ×120 gain into an **asymmetric** curve (`tanh` on top, a harder and lower ceiling below), then a lowpass. |
| AUTO-WAH | filter | an envelope-controlled band-pass ("envelope filter"), as in Zölzer, *DAFX*, ch. 2 and 4 | Built from nodes: the signal is rectified (a `\|x\|` WaveShaper), smoothed by an 8 Hz lowpass, scaled by SENS and added to a band-pass filter's frequency. PEAK is its Q. |
| EQ-5 | filter | a graphic EQ | Five peaking filters at 100, 250, 800, 2000 and 5000 Hz, ±12 dB. |
| CHORUS | modulation | J. Dattorro, "Effect Design, Part 2: Delay-Line Modulation and Chorus", *J. Audio Eng. Soc.* 45(10), 1997 | A 15 ms delay swept by a sine LFO (up to ±6 ms), added to the dry sound. |
| FLANGER | modulation | Dattorro, Part 2 (as above) | A 3.5 ms delay swept by a slow LFO (±2.5 ms) and fed back into itself (REGEN), added to the dry sound. |
| PHASER | modulation | J. O. Smith, *Physical Audio Signal Processing* (CCRMA, online book), "Phasing with first-order allpass filters"; Zölzer, *DAFX* | Four allpass filters whose frequency an LFO sweeps; added to the dry sound, they cut moving notches. |
| TREMOLO | modulation | the classic amplitude tremolo | A gain swung by a sine LFO between 1 and 1 − DEPTH. |
| VIBRATO | modulation | Zölzer, *DAFX*, ch. 2 (vibrato as a modulated delay with no dry signal) | A 6 ms delay swept by an LFO; only the delayed sound is heard, so the pitch wobbles. |
| DELAY | space | the basic feedback delay line | A delay (up to 0.9 s) with a feedback loop (REPEATS), added at MIX. |
| PLATE REVERB | space | convolution with a synthetic impulse: exponentially decaying noise, the textbook way to approximate a dense plate or hall tail | A `ConvolverNode` with a stereo noise burst shaped by (1 − t)³ over DECAY seconds, then a lowpass (TONE). The burst is rebuilt when DECAY is let go. |
| ROOM REVERB | space | **Freeverb** by Jezar at Dreampoint, public domain (2000): 8 parallel low-pass feedback comb filters and 4 series allpasses (the Schroeder–Moorer design) | Worklet. The same comb lengths (1116–1617 samples), allpass lengths (556, 441, 341, 225), allpass feedback 0.5, input gain 0.015, room size 0.7–0.98 and damping 0–0.4 as Freeverb, scaled to the sample rate. Mono. |

## Amps

All four share one recipe (`ampDef`):

1. input gain
2. a low cut, so the lows do not turn to mud once clipped
3. a WaveShaper clipping curve
4. for HIGH GAIN only, a second low cut, gain and clipping stage, as in cascaded preamps
5. the tone stack, then MASTER

The tone stack is a low shelf (BASS), a peaking band (MID) and a high shelf (TREBLE). Each amp also has a fixed "voicing" band, such as the mid scoop of the clean and high-gain amps. The voicings follow the measured shapes of classic tone stacks, as described in D. T. Yeh and J. O. Smith, "Discretization of the '59 Fender Bassman Tone Stack", *Proc. DAFx-06*, 2006, and in the analog references it cites. They are approximations by ear and by those curves, not the discretized circuit.

The clipping curves are static waveshapers, the method described in Zölzer, *DAFX*, ch. 4. A `tanh` curve stands in for a tube stage. An offset or uneven curve makes the two halves of the wave clip differently, which adds even harmonics ("warmth").

| Unit | Voicing |
|---|---|
| CLEAN 65 | American clean: low gain, a nearly linear curve, a mid scoop at 450 Hz, a bright top |
| CRUNCH 50 | British crunch: an uneven `tanh` (the negative side clips later), mids at 700 Hz |
| CHIME 30 | class-A chime: an offset `tanh` (even harmonics), mids up at 1.2 kHz, breaks up early |
| HIGH GAIN 100 | modern high gain: two clipping stages with a low cut between them, a mid scoop at 500 Hz |

## Cabs

A guitar speaker in a box behaves like a fixed filter. A recorded impulse response (IR) is the usual way to model one, but long IRs cost CPU on phones, and good free ones carry licences that would have to travel with the app. So the cabs are **filter models**:

- the speaker's low cut (a high-pass)
- the box's low resonance (LOW, a peaking filter)
- the cone's presence peak (2–3 kHz)
- for closed boxes, a notch in the low mids
- two lowpasses where the speaker stops (BRIGHT moves them)

The shapes follow the typical frequency response of 10" and 12" guitar speakers published by speaker makers, and the open-back / closed-back differences described in guitar-amp literature.

| Unit | Low cut | Resonance | Presence | Notch | Top |
|---|---|---|---|---|---|
| 1×12 OPEN BACK | 75 Hz | 110 Hz | 2.5 kHz +3 dB | — | 5 kHz |
| 1×10 TWEED | 110 Hz | 140 Hz | 2 kHz +4 dB | — | 4.5 kHz |
| 2×12 CLOSED | 70 Hz | 100 Hz | 2.2 kHz +2 dB | 1.2 kHz | 4.8 kHz |
| 4×12 BRITISH | 65 Hz | 90 Hz | 2.8 kHz +5 dB | 1.5 kHz | 4.2 kHz |

Loading your own IR files (a convolver per cab) is a natural next step. It would add no delay beyond the IR's own pre-delay, but it costs CPU.

## The test riff

The RIFF ▶ button plays a riff that the app **synthesizes** when you press it, so there is no audio file and no licence to track. It uses Karplus–Strong plucked-string synthesis: K. Karplus and A. Strong, "Digital Synthesis of Plucked-String and Drum Timbres", *Computer Music Journal* 7(2), 1983. The method is public domain.

How it works: a burst of noise circulates in a delay line one period long, and each pass averages neighbouring samples, so the string loses treble the way a real one does. The riff is in E minor at 112 BPM: power-chord chugs, a G5–A5 move and a pentatonic lick, 32 eighth notes long and looped. It sounds like a guitar plugged straight into an interface, which is what the drives, amps and cabs expect.

## The output stage

After the master fader comes a soft limiter: gain ×0.5, then `tanh(2x)`. It is a fixed curve, not a compressor, so it has no look-ahead and adds no delay, and a hot board never hard-clips.

## Adding a unit

1. Add an object to `PEDALS` (or a call to `ampDef` / `cabDef`):
   - `id`: new and never reused, since saved boards refer to it
   - `kind`, `mode`, `name`, `col`
   - `knobs`: `[id, LABEL, min, max, start, format]`
   - `build(h)`: returns a box from `h.box()`, or `h.worklet(name)`
   - `set(n, s, knob, h)`
2. If it needs sample-by-sample code, add a processor class to `WORKLET_JS`, and set `wl: true` on the unit as a reminder.
3. Add a row to this file with its source.
4. Add it to the help (EN and ES).
