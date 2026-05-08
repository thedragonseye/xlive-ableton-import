# API Reference

This document describes every configurable input, output, and interface point in **xlive-ableton-import** — covering the Front Half CLI, the Back Half Max for Live JS engine, shared data structures, and the planned Phase 3 metadata contract.

---

## Table of Contents

1. [Front Half — Node.js CLI](#1-front-half--nodejs-cli)
   - [Entry Point](#entry-point)
   - [Configuration](#configuration)
   - [CLI Options](#cli-options)
   - [Functions](#functions)
   - [Output Contract](#output-contract)
   - [Error Conditions](#error-conditions)
2. [Back Half — Max for Live JS Engine](#2-back-half--max-for-live-js-engine)
   - [Device Entry Point](#device-entry-point)
   - [Configuration](#configuration-1)
   - [JS Engine API — import.js](#js-engine-api--importjs)
   - [Import Modes](#import-modes)
   - [Live Object Model Interactions](#live-object-model-interactions)
3. [Shared Data Structures](#3-shared-data-structures)
   - [Session Directory Layout](#session-directory-layout)
   - [Stem Naming Convention](#stem-naming-convention)
4. [Phase 3 — Metadata JSON Contract (Planned)](#4-phase-3--metadata-json-contract-planned)
   - [Schema](#schema)
   - [Field Reference](#field-reference)
   - [Integration Point](#integration-point)
5. [Error Reference](#5-error-reference)

---

## 1. Front Half — Node.js CLI

The Front Half is a standalone Node.js script that reads PolyWAV multitrack files from a Behringer/Midas mixer SD card and writes individual WAV stems to a structured output directory.

**Requirements:** Node.js 18 or later.

### Entry Point

```
front-half/index.js
```

Run directly:

```bash
cd front-half
npm install
node index.js [options]
```

### Configuration

All configuration is passed via CLI flags. There is no separate config file in Phase 1/2.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `--input` | `string` | (required) | Absolute or relative path to the SD card mount point or session source root |
| `--output` | `string` | `./Sessions` | Destination root for all split stem directories |
| `--dry-run` | `boolean` | `false` | Scan and log actions without writing any files |
| `--verbose` | `boolean` | `false` | Emit detailed per-file logging |
| `--force` | `boolean` | `false` | Overwrite existing stems if session output already exists |

### CLI Options

```bash
node index.js --input /Volumes/XLIVE_SD --output /path/to/Sessions
node index.js --input /Volumes/XLIVE_SD --dry-run --verbose
node index.js --input /Volumes/XLIVE_SD --force
```

### Functions

The following functions are exposed by `front-half/lib/`:

#### `scanForSessions(inputPath: string): string[]`

Recursively walks `inputPath` and returns an array of absolute paths to every directory that contains at least one PolyWAV file.

- **Input:** Absolute path to SD card root
- **Output:** `string[]` — array of session folder paths, sorted alphanumerically
- **Throws:** `Error` if `inputPath` is not readable

---

#### `detectPolyWAV(sessionPath: string): string[]`

Scans a single session folder for PolyWAV container files (`.wav` files that encode multiple channels in one file using the Behringer/Midas X-Live format).

- **Input:** Path to a single session folder
- **Output:** `string[]` — paths to all PolyWAV files found in that session
- **Throws:** `Error` if the path does not exist

---

#### `splitPolyWAV(polyWAVPath: string, outputDir: string): StemResult[]`

Demultiplexes a single PolyWAV container into individual mono WAV files. Creates `outputDir` if it does not exist.

- **Input:**
  - `polyWAVPath` — absolute path to the source PolyWAV file
  - `outputDir` — absolute path to the destination stems directory
- **Output:** `StemResult[]` — one entry per extracted channel
- **Throws:** `Error` on read failure or unsupported channel count

---

#### `normalizeStemName(channelIndex: number, rawName: string): string`

Produces a zero-padded, filesystem-safe stem filename from a channel index and raw channel name.

- **Input:**
  - `channelIndex` — 0-based channel number
  - `rawName` — raw channel label from the mixer (may be empty)
- **Output:** e.g. `"01_Kick.wav"`, `"02_Snare.wav"`, `"16_FX_Return.wav"`
- **Fallback:** If `rawName` is empty, uses `"CH{N}"` (e.g. `"07_CH7.wav"`)

---

#### `StemResult`

```js
{
  channelIndex: number,   // 0-based
  channelName: string,    // normalized label used in the filename
  filename: string,       // e.g. "03_Bass.wav"
  outputPath: string,     // absolute path to the written file
  durationSamples: number,
  sampleRate: number,     // typically 48000
  bitDepth: number        // typically 24
}
```

### Output Contract

For each detected session the Front Half produces:

```
{outputRoot}/
  Session_001/
    stems/
      01_Kick.wav
      02_Snare.wav
      03_Bass.wav
      ...
  Session_002/
    stems/
      ...
```

- Session directories are named `Session_{NNN}` with zero-padded three-digit indices.
- Stem files are named `{NN}_{ChannelName}.wav` with zero-padded two-digit channel indices.
- All stems are **mono, 24-bit, 48 kHz** WAV (matching t
## 2. Back Half — Max for Live JS Engine

The Back Half runs inside Ableton Live as a Max for Live device. It reads the stems directory produced by the Front Half and constructs complete Ableton Live Sets automatically.

**Requirements:** Ableton Live 11 or later with Max for Live.

### Device Entry Point

```
back-half/device.amxd
```

Drag `device.amxd` onto any MIDI track in Ableton Live. The device exposes a UI panel and connects to the JS engine.

### Configuration

Configuration is set through the device UI and stored in the Max patcher:

| Setting | Type | Description |
|---|---|---|
| **Session Folder Path** | `string` | Absolute path to the root sessions directory produced by the Front Half |
| **Import Mode** | `enum` | `"selected"` or `"all"` — see Import Modes below |
| **Selected Session** | `string` | Session folder name chosen from the UI dropdown; only used when Import Mode is `"selected"` |

### JS Engine API — import.js

```
back-half/js/import.js
```

This script is loaded by the Max patcher and communicates with Ableton's Live Object Model (LOM) via the Max JS API.

#### `discoverSessions(basePath: string): string[]`

Scans `basePath` for subdirectories matching the `Session_{NNN}` naming pattern.

- **Input:** Absolute path to the sessions root
- **Output:** `string[]` — sorted array of session folder names (e.g. `["Session_001", "Session_002"]`)
- **Side effect:** Populates the session dropdown in the device UI

---

#### `importSession(sessionPath: string, mode: "new" | "current"): void`

Builds a Live Set from a single session's stems directory.

- **Input:**
  - `sessionPath` — absolute path to a single `Session_NNN/` folder
  - `mode` — `"new"` creates a blank Live Set before import; `"current"` populates the currently open Live Set
- **Side effects:** Creates audio tracks, loads clips, applies names and colors, optionally saves the `.als` file

---

#### `createTrack(stemPath: string, trackIndex: number): LiveTrack`

Creates a single audio track in the current Live Set for a given stem file.

- **Input:**
  - `stemPath` — absolute path to a `.wav` stem file
  - `trackIndex` — 0-based position in the track list
- **Output:** Reference to the newly created `LiveTrack` object

---

#### `loadClip(track: LiveTrack, stemPath: string): void`

Places the WAV file into the first clip slot of the given track.

---

#### `applyMetadata(track: LiveTrack, name: string, color: number): void`

Sets the track name and Ableton color index (0–69). In Phase 1/2 these are derived from the stem filename; in Phase 3 they come from `metadata.json`.

---

#### `saveLiveSet(outputPath: string): void`

Saves the current Live Set to disk as an `.als` file.

- **Input:** `outputPath` — absolute path including filename (e.g. `/path/to/Sessions/Session_001.als`)
- **Behavior:** Calls Live's save API; blocks until save completes

### Import Modes

**Import Selected Session:** Engineer selects a session from the UI dropdown. Stems are loaded into the currently open Live Set — no new file is created and no save is triggered.

**Import All Sessions:** For each session: a new blank Live Set is created, stems are loaded and metadata applied, then `saveLiveSet` writes `Session_NNN.als` alongside the session folder:

```
/Sessions/
  Session_001/stems/...
  Session_001.als
  Session_002/stems/...
  Session_002.als
```

### Live Object Model Interactions

| LOM Path | Operation | Purpose |
|---|---|---|
| `live_set tracks` | read | Enumerate existing tracks |
## 3. Shared Data Structures

### Session Directory Layout

Both halves agree on this directory contract:

```
{sessionsRoot}/
  Session_{NNN}/          <- produced by Front Half
    stems/
      {NN}_{Name}.wav
      ...
  Session_{NNN}.als       <- produced by Back Half (Import All mode only)
```

### Stem Naming Convention

Stem filenames follow this pattern: `{channelIndex}_{channelName}.wav`

| Component | Format | Example |
|---|---|---|
| `channelIndex` | Two-digit zero-padded integer (1-based) | `01`, `12`, `32` |
| `channelName` | Alphanumeric + underscores, no spaces | `Kick`, `Snare`, `FX_Return` |
| Extension | Always `.wav` | `.wav` |

**Examples:** `01_Kick.wav`, `02_Snare.wav`, `03_HiHat.wav`, `04_Bass_DI.wav`, `16_FX_Return.wav`

---

## 4. Phase 3 — Metadata JSON Contract (Planned)

> **Status:** Planned. This interface does not exist yet. It will be finalized during Phase 3 development.

The Phase 3 metadata pipeline reads Behringer/Midas USB scene files and produces a `metadata.json` file that the Back Half consumes to apply mixer-accurate track names, colors, and scene labels.

### Schema

```json
{
  "version": "1.0",
  "sessionName": "string",
  "exportedAt": "ISO 8601 datetime",
  "mixer": {
    "model": "string",
    "firmware": "string"
  },
  "channels": [
    {
      "index": 1,
      "name": "Kick",
      "color": "#FF4500",
      "abletonColorIndex": 5,
      "faderLevel": 0.0,
      "muted": false,
      "soloed": false
    }
  ],
  "scenes": [
    { "index": 1, "name": "Verse" }
  ]
}
```

### Field Reference

#### Top Level

| Field | Type | Description |
|---|---|---|
| `version` | `string` | Schema version; currently `"1.0"` |
| `sessionName` | `string` | Human-readable session name from the mixer |
| `exportedAt` | `string` | ISO 8601 UTC timestamp of export |
| `mixer.model` | `string` | Mixer model string (e.g. `"Behringer X32"`, `"Midas M32"`) |
| `mixer.firmware` | `string` | Firmware version string |

#### Channel Object

| Field | Type | Description |
|---|---|---|
| `index` | `number` | 1-based channel number, matches stem filename prefix |
| `name` | `string` | Channel label as set on the mixer |
| `color` | `string` | Hex color string (e.g. `"#FF4500"`) from the mixer's color palette |
| `abletonColorIndex` | `number` | Nearest Ableton color index (0-69), pre-computed during export |
| `faderLevel` | `number` | Fader position in dB (0.0 = unity) |
| `muted` | `boolean` | Channel mute state at time of export |
| `soloed` | `boolean` | Channel solo state at time of export |

#### Scene Object

| Field | Type | Description |
|---|---|---|
| `index` | `number` | 1-based scene number |
| `name` | `string` | Scene label as set on the mixer |

### Integration Point

`metadata.json` is placed in the session root alongside `stems/`. The Back Half's `import.js` detects its presence and prefers it over stem-filename-derived labels for all `applyMetadata` calls.

```
Session_001/
  stems/
    01_Kick.wav
    ...
  metadata.json       <- Phase 3 addition
```

---

## 5. Error Reference

| Code | Source | Meaning | Resolution |
|---|---|---|---|
| `ERR_NO_POLYWAV` | Front Half | No PolyWAV files found at the input path | Verify the SD card is mounted and the path is correct |
| `ERR_OUTPUT_NOT_WRITABLE` | Front Half | Cannot create or write to the output directory | Check filesystem permissions |
| `ERR_CORRUPT_POLYWAV` | Front Half | PolyWAV file failed to parse | File may be incomplete; re-record or skip |
| `ERR_SESSION_EXISTS` | Front Half | Session output directory already exists | Use `--force` to overwrite |
| `ERR_NO_SESSIONS` | Back Half | No `Session_{NNN}` directories found at configured path | Confirm the Front Half has run and the path is correct |
| `ERR_LOM_TRACK_CREATE` | Back Half | Ableton refused track creation | Ensure a Live Set is open and Max for Live is active |
| `ERR_SAVE_FAILED` | Back Half | `save_live_set` call failed | Check disk space and Ableton's file permissions |

| `live_set create_audio_track` | call | Create a new audio track |
| `live_set tracks N name` | set | Set track display name |
| `live_set tracks N color` | set | Set track color (Ableton color index) |
| `live_set tracks N clip_slots 0 clip` | set | Place WAV clip in slot 0 |
| `live_set scenes N name` | set | Set scene name |
| `live_set save_live_set` | call | Save current set to disk |

---
he X-Live recorder output).
- The `stems/` subdirectory is always present, even if only one stem was extracted.

### Error Conditions

| Condition | Behavior |
|---|---|
| No PolyWAV files found at `--input` | Logs error and exits with code `1` |
| Output directory not writable | Throws and exits with code `1` |
| PolyWAV read failure (corrupt file) | Logs warning, skips file, continues |
| Session output already exists (no `--force`) | Logs warning, skips session, continues |

---
