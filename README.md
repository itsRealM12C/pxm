# PIXIMOD1 Extractor — Technical README

A single-file HTML application that decodes **PIXIMOD1**, an undocumented proprietary binary format used by PixiTracker (NightRadio/SunDog engine), and renders its patterns, instruments, and audio playback natively in the browser.

Unlike the companion SunVox tool, there is no public specification for this format and no reference implementation to embed. Everything below was determined by direct binary analysis — hex inspection, cross-validation against byte counts, and iterative correction against real project files. Where something is inferred rather than confirmed, it's marked as such.

---

## 1. Container structure

PIXIMOD1 uses a flat, chunk-based container, structurally similar to RIFF/IFF but with 4-byte little-endian sizes and no nesting depth beyond the pattern/instrument sub-blocks described below.

```
Offset 0        "PIXIMOD1"           (8-byte magic, no version field found)
Offset 8..EOF   sequence of chunks:
                  TAG   (4 bytes, ASCII, space-padded if <4 chars)
                  SIZE  (4 bytes, uint32 LE)
                  DATA  (SIZE bytes)
```

Parsing terminates cleanly at the last chunk whose bounds fit inside the file — any trailing bytes beyond that point are treated as footer/padding and intentionally excluded from extraction.

---

## 2. Top-level header chunks

| Tag | Size | Type | Meaning |
|---|---|---|---|
| `BPM ` | 4 | int32 | Song tempo |
| `LPB ` | 4 | int32 | Lines per beat (grid subdivision; also drives the beat-highlight column in the UI) |
| `TPL ` | 4 | int32 | Ticks per line |
| `SHFL` | 4 | int32 | Shuffle/swing amount |
| `VOL ` | 4 | int32 | Master volume |
| `PATT` | variable | struct | Song order list (see §3) |

Any unrecognized 4-byte chunk falls back to a generic int32 read; anything else falls back to a raw hex dump in the header table, so no data is silently dropped even when a tag's meaning is unknown.

---

## 3. Song order (`PATT`)

This chunk was **decoded incorrectly in an earlier version of this tool** — the first implementation assumed a simple `[count, int32 entries...]` layout, which happened to produce plausible-looking (but wrong) output on early test files. The actual structure, confirmed by exact byte-size matching across four independent project files:

```
int32   unknown          (constant 2 in every sample observed; purpose unconfirmed)
int32   numOrders
int16   restartPosition  (loop target when the song wraps)
int16   padding
int16[numOrders]  order  (0-based pattern indices — directly matches PATN, no offset)
```

Total chunk size always equals `12 + numOrders*2` exactly, which is what confirmed this layout over the earlier guess.

**Order entries can repeat.** A real project's order list may look like `[0,1,2,3,0,1,2,3,4,5,6,7,...]` — the same pattern reused many times in a musical arrangement, not a simple "play each pattern once" sequence. The sequencer walks this list directly rather than iterating unique patterns.

---

## 4. Pattern data (`PATD`)

Each pattern is a `PATN` (index) → `PATD` (event data) → metadata chunks → `PEND` block.

### 4.1 `PATD` header

```
int32   cellSize     (bytes per note-slot × slots-per-cell; observed: 4)
int32   channels
int32   rows
```

**This field order was also wrong in an earlier version** (originally assumed `[channels, cellSize, rows]`). The correct order was confirmed the same way as `PATT` — by finding the interpretation that made `channels × rows × cellSize` equal the actual remaining chunk bytes *and* produced musically coherent note placement (evenly-spaced, alternating-channel patterns) instead of clustered noise.

### 4.2 Cell layout

**Row-major**, not channel-major: `for each row: for each channel: read cellSize bytes`. (The earlier, incorrect version assumed channel-major, which silently reshuffled every note's position.)

Each cell splits into `cellSize / 4` note-event slots of 4 bytes each:

```
byte 0     note
byte 1     instrument   (index into the instrument list)
byte 2-3   volume       (uint16 LE)
```

A slot is considered populated if `note || instrument || volume` is nonzero.

---

## 5. Instruments

`SNDN` (int32 index) opens an instrument block; the following fields populate it until `SND1` closes it:

| Tag | Meaning |
|---|---|
| `CHAN` | Channel count (mono/stereo) |
| `RATE` | Sample rate, Hz |
| `FINE` | Finetune, **1/128-semitone units** (confirmed by matching known instrument tuning behavior) |
| `RELN` | Relative note, **whole semitones**, added to the note before pitch calculation |
| `SVOL` | Instrument-level default volume (0–100+; not always 100) |
| `SOFF` | Sample trim: start offset, **in samples** (not bytes) |
| `SOF2` | Sample trim: end offset, in samples (0 = play to end) |
| `LPST` / `LPLN` / `LPTP` | Loop start / length / type |
| `SND1` | Raw sample payload (see §6) |

### 5.1 Pitch formula

```
semitones = (note - 60) + RELN + (FINE / 128) + globalPitchOffset
playbackRate = 2 ^ (semitones / 12)
```

Note 60 is the unshifted reference pitch. An earlier version had the `RELN` sign inverted (subtracting instead of adding), which shifted every instrument with nonzero `RELN` in the wrong direction — audible as instruments that were supposed to transpose up instead transposing down by the same amount.

---

## 6. Raw sample payload (`SND1`)

```
int32   bytesPerSample     (observed: 2, i.e. 16-bit PCM)
int32   declaredSampleCount
int16 × 3   (flags/reserved — meaning not determined)
──────────────────────────────
int16[]  PCM samples, little-endian, mono, from byte offset 14 onward
```

Verified against real data: declared sample count consistently matches `(chunk_size - 14) / 2` within rounding, and the decoded waveform is a smooth, correctly-ranged (`-32768..32767`) audio signal rather than noise — confirming both the header length and the sample format.

Playback trims this buffer using `SOFF`/`SOF2` before use, and scales amplitude by `SVOL / 100` at decode time (baked into both the live playback buffer and any exported WAV).

---

## 7. Note icons

16 instrument color slots map to a fixed palette (`#ffffff` through `#ff5cc5`, with a 17th slot added as pure black for an extra/overflow instrument). Icon *bitmaps* are not part of the PIXIMOD1 format — they come from a separately-provided 7×7 sprite sheet (`sound_icons.png`, 16 icons × 7px, extracted as literal 1-bit bitmaps), with a second sheet (`sound_icons2.png`) supplying alternate animation frames. Instrument icons in the UI alternate between the two frame sets every 200ms.

Each note cell renders as: instrument-color background fill, with the corresponding icon bitmap drawn on top in solid black at true 1:1 pixel scale (no smoothing/scaling artifacts).

---

## 8. Playback engine

Fully custom Web Audio implementation — there is no reference engine to embed (unlike the SunVox tool), so timing, pitch, and channel behavior are all reimplemented from scratch based on the decoded data.

- **Row timing**: `rowSeconds = TPL × 2.5 / BPM` (classic tracker tempo formula).
- **Monophonic channels**: triggering a new note on a channel fades out (10ms) and stops whatever was already sounding on that same channel, matching standard tracker "one note per channel" behavior — notes don't stack/overlap indefinitely.
- **Resume-from-pause**: rounds the resume position **up** to the next unplayed row (`Math.ceil`), not down — an earlier version rounded down, which re-triggered the last-heard note on every resume.
- **AudioContext gesture handling**: `resume()` is awaited before scheduling, and triggered directly inside the click handler — scheduling before the context has actually resumed produced silent playback with no thrown error in earlier testing.
- **Sequencer**: walks the real `PATT` order list (§3), including repeats; Play/Pause is a genuine toggle that preserves position rather than always restarting from the beginning.

---

## 9. Known unknowns

Documented honestly rather than papered over:

- The leading `int32` in `PATT` (always `2` in every sample seen) — purpose unconfirmed.
- The 3× `int16` reserved fields in the `SND1` header — never seen nonzero, so their function is untested.
- Loop fields (`LPST`/`LPLN`/`LPTP`) are parsed and exposed but not yet wired into playback (samples always play their full trimmed range rather than looping).
- No instrument type ever produced a nonzero `SOF2`/loop combination in testing, so that code path is unverified against real data.
