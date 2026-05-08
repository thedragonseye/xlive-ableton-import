# USB Metadata Snapshot Reference

This document describes the Behringer X32 / Midas M32 scene snapshot file (`.scn`) that engineers export to a USB drive, how it is parsed by the Phase 3 pipeline, and the `metadata.json` contract it produces for consumption by the Back Half.

> **Status:** The Phase 3 parsing pipeline does not yet exist. This document defines the format and the integration contract so contributors can implement it accurately. See [API.md](./API.md#4-phase-3--metadata-json-contract-planned) for the downstream `metadata.json` schema.

---

## Table of Contents

1. [What Is the Scene Snapshot?](#1-what-is-the-scene-snapshot)
2. [Exporting a Scene to USB](#2-exporting-a-scene-to-usb)
3. [File Format Overview](#3-file-format-overview)
4. [OSC-Path Line Syntax](#4-osc-path-line-syntax)
5. [Fields Relevant to xlive-ableton-import](#5-fields-relevant-to-xlive-ableton-import)
   - [Channel Configuration](#channel-configuration-chnn)
   - [Channel Mix (Fader, Mute, Pan)](#channel-mix-chnnmix)
   - [DCA Groups](#dca-groups-dcan)
   - [Mix Buses](#mix-buses-busnn)
   - [Main Bus](#main-bus)
6. [Color Palette Reference](#6-color-palette-reference)
7. [Ableton Color Index Mapping](#7-ableton-color-index-mapping)
8. [Full Annotated Scene Extract](#8-full-annotated-scene-extract)
9. [Phase 3 Parsing Contract](#9-phase-3-parsing-contract)
   - [Input](#input)
   - [Output: metadata.json](#output-metadatajson)
   - [Parser Behavior Rules](#parser-behavior-rules)
10. [Field Extraction Reference Table](#10-field-extraction-reference-table)
11. [Known Limitations and Gotchas](#11-known-limitations-and-gotchas)

---

## 1. What Is the Scene Snapshot?

A **scene** on the Behringer X32 and Midas M32 is a complete, point-in-time snapshot of the mixing console's state: all channel names, colors, fader levels, mute states, EQ settings, routing, dynamics, and more. Scenes are stored internally on the console and can be exported to a USB drive as a plain-text `.scn` file.

For the purposes of xlive-ableton-import, the scene snapshot is the primary source of **mixer-accurate metadata** — specifically the per-channel names and colors the engineer assigned during setup. This data is what Phase 3 uses to populate Ableton track names and colors to match the physical console layout.

The `.scn` file is **not JSON**. It is a line-oriented, plain-text file using an OSC-derived path syntax. Phase 3 parses it and produces the `metadata.json` contract described in this document and in [API.md](./API.md).

---

## 2. Exporting a Scene to USB

**On the console:**

1. Insert a FAT32-formatted USB 2.0 drive into the top USB port of the X32 or M32.
2. Press the **VIEW** button next to **Scenes** on the right-hand panel.
3. Select the **Scenes** tab using the arrow buttons.
4. Choose the scene slot you want to export.
5. Press the **UTILITY** button, then press the encoder labeled **Save** to write the scene to the console's memory.
6. Press **Save** again (or follow the on-screen prompt) to export to USB.

The file is written to the USB root as:

```
USB_ROOT/
  SCENE_001.scn      <- scene slot 001 (numbering matches the console's scene list)
  SCENE_002.scn
  ...
```

**Alternatively, via X32-Edit / M32-Edit software:**

1. Connect X32-Edit to the console.
2. In the Scenes panel, select the scene on the console side.
3. Click **Export** and save to disk.

The resulting `.scn` file is identical in format regardless of export method.

---

## 3. File Format Overview

A `.scn` file is a **plain UTF-8 text file** (no BOM). Each line is an independent record. Lines are not ordered in any semantically meaningful way relative to each other — every line is self-contained.

General properties:

- **Encoding:** UTF-8 (ASCII-safe for all field values in practice)
- **Line endings:** LF (`\n`) on console exports; CRLF (`\r\n`) may appear in X32-Edit exports on Windows
- **Line length:** Variable; typically 40-120 characters
- **Comments:** None — every non-empty line is a data record
- **Typical file size:** ~90 KB for a full 32-channel scene

The file begins with a version/header line:

```
#4.0#                    <- firmware-version marker; format is stable across 3.x and 4.x
```

Followed by thousands of OSC-path lines covering the complete console state.

---

## 4. OSC-Path Line Syntax

Each data line follows this grammar:

```
<path> <value1> [<value2> ...]
```

- **`<path>`** — a forward-slash-separated identifier string, e.g. `/ch/01/config`
- **`<value>`** — one or more space-separated values; strings are quoted with double quotes; numbers are unquoted; booleans are `ON` or `OFF`; special values include `-oo` (negative infinity, representing silence/-inf dB)

Examples:

```
/ch/01/config "Kick" 1 RD 1
/ch/01/mix ON +0.0 ON +0 OFF -oo
/ch/01/delay OFF 0.3
/dca/1/config "Drums" 43 CY
```

Values are always in the order documented per path (see section 5). There are no key names for individual values — position is the only identifier.

---

## 5. Fields Relevant to xlive-ableton-import

The full `.scn` file contains thousands of lines covering EQ curves, dynamics, routing, effects, outputs, and more. Phase 3 reads only the subset described below.

### Channel Configuration `/ch/NN/config`

```
/ch/NN/config "<name>" <icon> <color> <input>
```

| Position | Field | Type | Description |
|---|---|---|---|
| 1 | `name` | quoted string | Channel label, up to 12 characters. Shown on the console's scribble strip. |
| 2 | `icon` | integer | Icon index (1-74). Not used by xlive-ableton-import. |
| 3 | `color` | string | Two-letter color code; see section 6. |
| 4 | `input` | integer | Physical input number routed to this channel. |

**NN** is the two-digit zero-padded channel number, `01` through `32`.

**Example:**

```
/ch/01/config "Kick" 1 RD 1
/ch/02/config "Snare" 2 YE 2
/ch/03/config "HiHat" 3 GN 3
/ch/04/config "Bass DI" 4 BL 4
/ch/07/config "" 1 WH 7    <- empty name; fall back to "CH07"
```

### Channel Mix `/ch/NN/mix`

```
/ch/NN/mix <on> <fader_dB> <send_LR> <pan> <send_MC> <fader_MC_dB>
```

| Position | Field | Type | Description |
|---|---|---|---|
| 1 | `on` | `ON`/`OFF` | `OFF` means the channel is muted |
| 2 | `fader_dB` | float | Fader level in dB (+10.0 to -inf). `-oo` = silence |
| 3 | `send_LR` | `ON`/`OFF` | Send to L/R main bus |
| 4 | `pan` | integer | Pan position (-100 = full left, +100 = full right, 0 = center) |
| 5 | `send_MC` | `ON`/`OFF` | Send to mono/center bus |
| 6 | `fader_MC_dB` | float | Level sent to mono/center bus |

**For xlive-ableton-import Phase 3**, only fields 1 (`on` -> mute state) and 2 (`fader_dB` -> fader level) are consumed.

**Example:**

```
/ch/01/mix ON +0.2 ON +0 OFF -oo
/ch/05/mix OFF -5.3 ON +0 OFF -oo    <- channel 5 is muted
```

### DCA Groups `/dca/N/config`

```
/dca/N/config "<name>" <icon> <color>
```

| Position | Field | Type | Description |
|---|---|---|---|
| 1 | `name` | quoted string | DCA group label, up to 12 characters |
| 2 | `icon` | integer | Icon index |
| 3 | `color` | string | Two-letter color code |

**N** is 1-8 (no leading zero for DCA groups).

**Example:**

```
/dca/1/config "Drums" 43 CY
/dca/2/config "Vocals" 12 MG
/dca/3/config "" 1 WH
```

Phase 3 extracts DCA names and colors for potential use as Ableton Group Track labels (future feature, not in initial Phase 3 scope).

### Mix Buses `/bus/NN/config`

```
/bus/NN/config "<name>" <icon> <color>
```

**NN** is `01` through `16`.

**Example:**

```
/bus/01/config "Drums Mix" 53 WH
/bus/02/config "Vocals Mix" 12 GN
```

Phase 3 may use bus names to label Ableton return tracks (future feature).

### Main Bus

```
/main/st/config "" 73 GNi
/main/m/config "" 74 WH
```

The main stereo (`st`) and mono/center (`m`) bus configurations. Not consumed by Phase 3 in initial implementation.

---

## 6. Color Palette Reference

The X32/M32 supports 7 base colors plus an inverted variant of each. Colors are encoded as two-letter codes (or three letters with `i` suffix for inverted):

| Code | Color | Hex (approximate) | Ableton Index (see section 7) |
|---|---|---|---|
| `OFF` | No color (default grey) | `#808080` | 0 |
| `RD` | Red | `#E22222` | 14 |
| `RDi` | Red (inverted/dark) | `#7A1111` | 13 |
| `GN` | Green | `#22BB22` | 28 |

## 7. Ableton Color Index Mapping

Ableton Live uses a fixed palette of 70 colors (indices 0-69). The Phase 3 parser maps each X32/M32 color code to the nearest Ableton color index. The mapping is pre-computed (not derived at runtime from hex distance) for predictability:

| X32/M32 Code | Ableton Index | Ableton Color Name |
|---|---|---|
| `OFF` | 0 | Charcoal |
| `WH` | 1 | Cream White |
| `WHi` | 0 | Charcoal |
| `RD` | 14 | Coral Red |
| `RDi` | 13 | Dark Red |
| `GN` | 28 | Green |
| `GNi` | 27 | Dark Green |
| `YE` | 62 | Lime Yellow |
| `YEi` | 61 | Olive |
| `BL` | 41 | Cobalt Blue |
| `BLi` | 40 | Navy Blue |
| `MG` | 57 | Magenta |
| `MGi` | 56 | Dark Magenta |
| `CY` | 24 | Cyan |
| `CYi` | 23 | Dark Cyan |

> These indices are recommendations. If the project adopts a different mapping, update this table and the parser implementation together. The `abletonColorIndex` field in `metadata.json` is always the resolved integer, never the string code.

---

## 8. Full Annotated Scene Extract

The following is a representative extract from a real X32 scene file, with inline annotations. Only fields consumed by Phase 3 are annotated; others are shown for context.

```
#4.0#
/ch/01/config "Kick" 1 RD 1        <- ch1: name="Kick", color=RD (red), input=1
/ch/01/delay OFF 0.3               <- not consumed
/ch/01/preamp +0.0 OFF ON 24 136   <- not consumed
/ch/01/gate OFF EXP4 -38.5 27.0 20 100 576 0   <- not consumed
/ch/01/dyn ON COMP RMS LIN -23.0 2.0 1 8.00 74 0.03 576 POST 0 100 OFF  <- not consumed
/ch/01/eq ON                        <- not consumed
/ch/01/eq/1 VEQ 232.3 -6.00 4.3    <- not consumed
/ch/01/eq/2 PEQ 514.1 -5.00 3.4    <- not consumed
/ch/01/eq/3 PEQ 1k21 -4.25 4.3     <- not consumed
/ch/01/eq/4 HCut 11k91 +0.00 2.0   <- not consumed
/ch/01/mix ON +0.2 ON +0 OFF -oo   <- mute=OFF(not muted), fader=+0.2 dB
/ch/01/grp %00000001 %000000        <- not consumed
/ch/02/config "Snare" 2 YE 2       <- ch2: name="Snare", color=YE (yellow), input=2
/ch/02/mix ON -1.5 ON +0 OFF -oo   <- mute=OFF, fader=-1.5 dB
/ch/03/config "HiHat" 3 GN 3       <- ch3: name="HiHat", color=GN (green)
/ch/03/mix OFF -3.0 ON +0 OFF -oo  <- mute=ON (channel is muted)
/ch/04/config "Bass DI" 4 BL 4     <- ch4: name="Bass DI", color=BL (blue)
/ch/04/mix ON +0.0 ON +0 OFF -oo
/ch/07/config "" 1 WH 7            <- ch7: name="" (empty -> use fallback "CH07")
/ch/07/mix ON +0.0 ON +0 OFF -oo
...
/dca/1/config "Drums" 43 CY        <- DCA1: name="Drums", color=CY (cyan)
/dca/2/config "Vocals" 12 MG       <- DCA2: name="Vocals", color=MG (magenta)
/bus/01/config "Drum Bus" 53 WH    <- Mix bus 1
/main/st/config "" 73 GNi          <- Main stereo output
```

---

## 9. Phase 3 Parsing Contract

### Input

| Item | Description |
|---|---|
| **Source file** | `SCENE_NNN.scn` copied from the USB drive to the session working directory |
| **Placement** | Engineers place the `.scn` file in the session root: `Session_001/SCENE_001.scn` |
| **Multiplicity** | One `.scn` file per session; if multiple are present, use the most recently modified |
| **Encoding** | UTF-8; handle both LF and CRLF line endings |

### Output: metadata.json

The parser writes `metadata.json` to the session root alongside `stems/` and the `.scn` file:

```
Session_001/
  SCENE_001.scn     <- input
  stems/            <- from Front Half
    01_Kick.wav
    ...
  metadata.json     <- output of Phase 3 parser
```

**Full example output:**

```json
{
  "version": "1.0",
  "sessionName": "Sunday AM Service",
  "exportedAt": "2026-05-08T17:06:00Z",
  "sourceFile": "SCENE_001.scn",
  "mixer": {
    "model": "Behringer X32",
    "firmware": "4.0"
  },
  "channels": [
    {
      "index": 1,
      "name": "Kick",
      "color": "RD",
      "abletonColorIndex": 14,
      "faderLevel": 0.2,
      "muted": false,
      "soloed": false
    },
    {
      "index": 2,
      "name": "Snare",
      "color": "YE",
      "abletonColorIndex": 62,
      "faderLevel": -1.5,
      "muted": false,
      "soloed": false
    },
    {
      "index": 3,
      "name": "HiHat",
      "color": "GN",
      "abletonColorIndex": 28,
      "faderLevel": -3.0,
      "muted": true,
      "soloed": false
    },
    {
      "index": 7,
      "name": "CH07",
      "color": "WH",
      "abletonColorIndex": 1,
      "faderLevel": 0.0,
      "muted": false,
      "soloed": false
    }
  ],
  "dca": [
    {
      "index": 1,
      "name": "Drums",
      "color": "CY",
      "abletonColorIndex": 24
    },
    {
      "index": 2,
      "name": "Vocals",
      "color": "MG",
      "abletonColorIndex": 57
    }
  ],
  "scenes": [
    {
      "index": 1,
      "name": "Verse"
    },
    {
      "index": 2,
      "name": "Chorus"
    }
  ]
}
```

### Parser Behavior Rules

These rules must be implemented exactly - the Back Half's `import.js` depends on them:

#### Channel name fallback

If a channel's `name` field is an empty string (`""`), the parser **must** substitute the fallback name:

```
"CH" + String(index).padStart(2, '0')   // e.g. "CH07" for channel 7
```

This ensures every channel in `metadata.json` has a non-empty name.

#### Channel index alignment

Channel indices in `metadata.json` are **1-based** and must match the two-digit channel number in the `.scn` path (`/ch/01/` -> index 1). They must also match the stem filename prefix (`01_Kick.wav` -> index 1). The Back Half uses the index to correlate `metadata.json` entries with stem files.

#### Mute state derivation

The mute state is read from the first value of the `/ch/NN/mix` line:

```
/ch/03/mix OFF ...    <- ON=unmuted, OFF=muted
```

`"muted": true` when the mix line value is `OFF`; `"muted": false` when it is `ON`.

> Note: This is counter-intuitive - `OFF` on the mix line means the channel's contribution to the output is off (i.e., muted), not that the mute button is on.

#### Fader level

The fader level in dB is the second value on the `/ch/NN/mix` line. The special value `-oo` (negative infinity) is represented in `metadata.json` as the number `null`:

```json
"faderLevel": null
```

All other values are parsed as floating-point numbers with their sign. The `+` prefix on positive values (e.g., `+0.2`) must be accepted by the parser.

#### Solo state

The console `.scn` file does not persist solo states (solos are transient UI states). The `soloed` field is always written as `false` in `metadata.json`.

#### Missing channels

If the `.scn` file does not contain a `/ch/NN/config` line for a given channel number (e.g., because the console was configured with fewer than 32 channels), that channel is **omitted** from the `channels` array in `metadata.json`. The Back Half handles omitted channels by using the stem filename as the track label.

#### Firmware version

The firmware version is read from the first line of the `.scn` file:

```
#4.0#   ->  "firmware": "4.0"
#3.11#  ->  "firmware": "3.11"
```

The `#` characters are delimiters; strip them when writing to `metadata.json`.

#### Session name

The session name is **not** present in the `.scn` file itself. It must be obtained from one of:

1. `SE_LOG.bin` in the session directory (if Phase 3 also parses the session log), **or**
2. The stem directory name (`Session_001`), **or**
3. An empty string `""` as a fallback

In initial Phase 3 implementation, use the session directory name as the `sessionName` value.

---

## 10. Field Extraction Reference Table

A concise lookup for the parser implementation:

| metadata.json field | Source `.scn` path | Value position | Notes |
|---|---|---|---|
| `channels[].index` | `/ch/NN/config` | path component NN | 1-based integer |
| `channels[].name` | `/ch/NN/config` | position 1 (quoted string) | Fallback to `"CHNN"` if empty |
| `channels[].color` | `/ch/NN/config` | position 3 | Two-letter code; see section 6 |
| `channels[].abletonColorIndex` | derived | — | Looked up from section 7 table |
| `channels[].faderLevel` | `/ch/NN/mix` | position 2 | Float; `null` for `-oo` |
| `channels[].muted` | `/ch/NN/mix` | position 1 | `true` if `OFF`, `false` if `ON` |
| `channels[].soloed` | — | — | Always `false` |
| `dca[].index` | `/dca/N/config` | path component N | 1-based, no leading zero |
| `dca[].name` | `/dca/N/config` | position 1 | Fallback to

## 11. Known Limitations and Gotchas

#### The `.scn` file is console state, not session state

A scene snapshot captures the console's configuration at the moment the engineer saved it — which may not match the configuration at the moment of recording. Engineers sometimes change channel names between the recording and the scene save. Phase 3 assumes the scene exported to the USB drive alongside the SD card is the correct reference for that session.

#### Channel labels are limited to 12 characters

The console's scribble strip and internal storage cap channel names at 12 characters. Names longer than 12 characters cannot exist in a valid `.scn` file. However, the parser should not enforce this limit — it should accept whatever is present.

#### Color `OFF` means "no color assigned"

A channel with color `OFF` has no color set on the console. This should map to Ableton color index 0 (charcoal/default). Do not skip or null-out these channels.

#### kHz notation in EQ fields

Some numeric values in the `.scn` file use a compact notation for frequencies: `1k21` means 1210 Hz, `11k91` means 11910 Hz. The Phase 3 parser does not read EQ fields, but contributors extending the parser should be aware of this notation.

#### Negative infinity in fader fields

`-oo` (two lowercase letters 'oo', not zeros) represents -inf dB (channel fully faded out / silent). This is distinct from a muted channel. A channel can have `-oo` fader and be unmuted, or a 0 dB fader and be muted.

#### Scene files from X32-Edit on Windows may have CRLF

Line endings may be `\r\n` rather than `\n`. The parser must normalize line endings before splitting on newlines:

```js
const lines = fileContent.replace(/\r\n/g, '\n').split('\n');
```

#### Not all channel slots appear in every scene file

A scene file may omit channels that are unconfigured. Always check for the presence of `/ch/NN/config` before assuming a channel exists, and omit missing channels from `metadata.json` rather than emitting null entries.

#### DCA group indices have no leading zero

DCA paths use single-digit indices: `/dca/1/config`, `/dca/8/config`. Channel paths use two-digit zero-padded indices: `/ch/01/config`, `/ch/32/config`. The parser must handle both formats correctly.

#### The `#version#` line format may vary

Some older firmware exports the version differently (e.g., `#FW4.06#` or simply omit the header line). Parse defensively: if the first line does not match `#N.NN#`, set `firmware` to `"unknown"` and continue. `"DCA N"` if empty |
| `dca[].color` | `/dca/N/config` | position 3 | Two-letter code |
| `dca[].abletonColorIndex` | derived | — | Looked up from section 7 table |
| `scenes[].index` | `/show/scene/NN/name` | path component | If present in file |
| `scenes[].name` | `/show/scene/NN/name` | position 1 | Scene label |
| `mixer.firmware` | `#N.NN#` (first line) | between `#` chars | Strip delimiters |
| `mixer.model` | — | — | Infer from channel count or hardcode per project config |
| `sessionName` | Session directory name | — | e.g. `"Session_001"` |
| `exportedAt` | System time at parse time | — | ISO 8601 UTC |
| `version` | — | — | Always `"1.0"` |
| `GNi` | Green (inverted/dark) | `#115511` | 27 |
| `YE` | Yellow | `#CCCC00` | 62 |
| `YEi` | Yellow (inverted/dark) | `#888800` | 61 |
| `BL` | Blue | `#2255DD` | 41 |
| `BLi` | Blue (inverted/dark) | `#112266` | 40 |
| `MG` | Magenta | `#CC22CC` | 57 |
| `MGi` | Magenta (inverted/dark) | `#661166` | 56 |
| `CY` | Cyan | `#22CCCC` | 24 |
| `CYi` | Cyan (inverted/dark) | `#116666` | 23 |
| `WH` | White | `#DDDDDD` | 1 |
| `WHi` | White (inverted/dark) | `#888888` | 0 |

> Hex values are approximate — the exact appearance varies by console model, display calibration, and firmware version.
