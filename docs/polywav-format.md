# PolyWAV Binary Format Reference

This document describes the binary structure of the X-Live PolyWAV multitrack files produced by the Behringer X-Live and Midas M-Live SD card recording expansion boards, as used in the X32, M32, and WING digital mixing consoles. It is intended for contributors writing or auditing the Front Half parser (`front-half/lib/splitter.js`).

> **Source note:** The X-Live format is not officially published by Behringer/Music Tribe. This document is derived from community-reverse-engineered sources, publicly available tooling (pmaillot's Live2Wav/Wav2Live utilities, the `wav-extract` project), and the standard RIFF/WAV specification. Fields marked ⚠️ are inferred from tool behavior rather than confirmed from primary documentation.

---

## Table of Contents

1. [Overview](#1-overview)
2. [SD Card Directory Structure](#2-sd-card-directory-structure)
3. [RIFF/WAV Header](#3-riffwav-header)
   - [RIFF Chunk](#riff-chunk)
   - [fmt Chunk — WAVE_FORMAT_EXTENSIBLE](#fmt-chunk--wave_format_extensible)
   - [data Chunk](#data-chunk)
4. [Audio Data Layout](#4-audio-data-layout)
   - [Channel Interleaving](#channel-interleaving)
   - [Sample Encoding](#sample-encoding)
   - [File Splitting at 4 GB](#file-splitting-at-4-gb)
5. [SE_LOG.bin — Session Log File](#5-se_logbin--session-log-file)
6. [Channel Count by Console Model](#6-channel-count-by-console-model)
7. [Parsing Guide for splitter.js](#7-parsing-guide-for-splitterjs)
   - [Step-by-Step Algorithm](#step-by-step-algorithm)
   - [Key Constants](#key-constants)
   - [Edge Cases](#edge-cases)
8. [Known Quirks and Gotchas](#8-known-quirks-and-gotchas)
9. [Reference Tools](#9-reference-tools)

---

## 1. Overview

The X-Live expansion board records all active console channels simultaneously into a single multichannel WAV file on an SD card. This design avoids the overhead of writing many small files in parallel and maximizes SD card write throughput.

Key properties of the format:

- **Container:** Standard RIFF/WAV with `WAVE_FORMAT_EXTENSIBLE` format tag
- **Channels:** Up to 32, all interleaved into one file
- **Bit depth:** 32-bit signed little-endian PCM (container); audio content is 24-bit padded to 32-bit ⚠️
- **Sample rates:** 44,100 Hz or 48,000 Hz (set on the console; 48 kHz is most common in live sound)
- **File size limit:** ~4 GB per file (FAT32 filesystem constraint on SD cards)
- **Continuation files:** When the 4 GB limit is reached, the recorder seamlessly opens a new file; audio is gapless across the boundary
- **Session log:** A companion binary file `SE_LOG.bin` is written in the same directory

---

## 2. SD Card Directory Structure

The SD card root contains a `RECORD/` directory. Inside it, each recording session occupies its own subdirectory named as the **Unix timestamp of the recording start**, encoded as an 8-character uppercase hexadecimal string:

```
SD_CARD_ROOT/
  RECORD/
    4B728846/               <- 8-char hex Unix timestamp
      00000001.wav           <- First segment (up to ~4 GB)
      00000002.wav           <- Second segment, seamlessly continues
      00000003.wav           <- Third segment (if needed)
      SE_LOG.bin             <- Binary session log (markers, session name)
    5C839957/               <- Another session
      00000001.wav
      SE_LOG.bin
```

**Segment file naming:** Files are numbered sequentially starting at `00000001.wav`, zero-padded to 8 digits.

**Timestamp directory:** The name is the 32-bit Unix epoch timestamp of when REC was pressed, as 8 uppercase hex characters. Example: `4B728846` = Unix time 1265331270 = 2010-02-04T22:14:30Z.

---

## 3. RIFF/WAV Header

Each segment file is a self-contained, valid RIFF/WAV file. All multi-byte integer fields are **little-endian**.

### RIFF Chunk

| Offset | Size | Type | Value | Description |
|---|---|---|---|---|
| 0 | 4 | ASCII | `"RIFF"` | RIFF magic number |
| 4 | 4 | uint32 | file size − 8 | Total file size minus 8 bytes |
| 8 | 4 | ASCII | `"WAVE"` | WAVE form type |

### fmt Chunk — WAVE_FORMAT_EXTENSIBLE

X-Live files use the `WAVE_FORMAT_EXTENSIBLE` format tag (0xFFFE), required for files with more than 2 channels.

| Offset | Size | Type | Value | Description |
|---|---|---|---|---|
| 12 | 4 | ASCII | `"fmt "` | fmt chunk ID (note trailing space) |
| 16 | 4 | uint32 | 40 | fmt chunk size (always 40 for EXTENSIBLE) |
| 20 | 2 | uint16 | 0xFFFE | Format tag: WAVE_FORMAT_EXTENSIBLE |
| 22 | 2 | uint16 | N | Number of channels (typically 32) |
| 24 | 4 | uint32 | 44100 or 48000 | Sample rate in Hz |
| 28 | 4 | uint32 | `SR x N x 4` | Average bytes per second |
| 32 | 2 | uint16 | `N x 4` | Block align: bytes per sample frame |
| 34 | 2 | uint16 | 32 | Bits per sample (container width) |
| 36 | 2 | uint16 | 22 | cbSize: size of extension |
| 38 | 2 | uint16 | 24 | Valid bits per sample (actual audio bit depth) ⚠️ |
| 40 | 4 | uint32 | channel mask | Bitmask of speaker positions (0 = no position) |
| 44 | 16 | GUID | PCM SubFormat | `{00000001-0000-0010-8000-00aa00389b71}` |

**Reading `wNumChannels` from bytes 22-23** is the primary way to determine how many stems to extract. Always read this dynamically.

**Valid bits per sample (offset 38):** Set to 24 in X-Li
## 4. Audio Data Layout

### Channel Interleaving

Audio samples are interleaved in the standard WAV manner — one complete sample frame contains one 32-bit sample for every channel, in channel order:

```
Frame 0:  [CH1_s0] [CH2_s0] [CH3_s0] ... [CH32_s0]
Frame 1:  [CH1_s1] [CH2_s1] [CH3_s1] ... [CH32_s1]
Frame 2:  [CH1_s2] [CH2_s2] [CH3_s2] ... [CH32_s2]
```

Each sample is a **4-byte (32-bit) signed little-endian integer**. Frame size = `numChannels x 4` bytes (128 bytes for 32 channels).

To extract channel K (0-based) from frame F:

```
dataOffset + (F x numChannels x 4) + (K x 4)
```

### Sample Encoding

| Property | Value |
|---|---|
| Container width | 32-bit signed integer, little-endian |
| Audio bit depth | 24 bits (valid bits per sample = 24) |
| Bit alignment | Left-aligned: audio in bits 31-8; bits 7-0 are always 0 |
| Value range | -2,147,483,648 to +2,147,483,520 (multiples of 256) |

To write 24-bit output stems, right-shift each 32-bit value by 8:

```js
const sample24 = sample32 >> 8;  // arithmetic right shift
```

### File Splitting at 4 GB

FAT32 imposes a 4 GB maximum file size. The X-Live handles this transparently:

1. When a segment approaches 4 GB, the recorder **closes it** and **opens the next** (e.g., `00000001.wav` -> `00000002.wav`).
2. The audio stream is **gapless** across the boundary.
3. Each segment is a **valid, self-contained WAV file** with its own complete header.
4. The `data` chunk size in earlier segments may be `0xFFFFFFFF`. ⚠️

To reconstruct a full session, concatenate audio data from all segments in numeric order.

**Approximate duration per segment** at 32ch / 48 kHz / 32-bit: 4 GB / (32 x 4 x 48,000) = ~687 s = ~11.5 minutes per file.

---

## 5. SE_LOG.bin — Session Log File

`SE_LOG.bin` is a binary file in the session directory alongside the WAV segments. It is **not a WAV file**.

> ⚠️ The SE_LOG.bin format is not officially documented. The following is based on community reverse engineering.

| Field | Notes |
|---|---|
| Session name | Up to 20 ASCII characters as set on the console at record time |
| Marker list | Up to 100 time markers, each stored as a frame offset or timestamp |
| Recording metadata | Start timestamp, channel count, sample rate ⚠️ |

Markers correspond to points the engineer pressed the marker button during recording, stored relative to frame 0 of `00000001.wav`.

The Front Half does not currently read `SE_LOG.bin`. Phase 3 may extract the session name and markers to populate Ableton scene names. See [API.md](./API.md#4-phase-3--metadata-json-contract-planned).

---

## 6. Channel Count by Console Model

| Console | Expansion Board | Max Channels | Common Recording Modes |
|---|---|---|---|
| Behringer X32 | X-Live | 32 | 8, 16, 32 |
| Midas M32 | M-Live | 32 | 8, 16, 32 |
| Behringer WING | W-Live | 32+ ⚠️ | varies |

Always read channel count from `wNumChannels` at header offset 22. Do not assume 32. **Inactive channels** are recorded as silent all-zero frames and extracted as valid (silent) WAV files unless filtered.

---

## 7. Parsing Guide for splitter.js

### Step-by-Step Algorithm

**1. Discover segments**

Scan the session directory for files matching `\d{8}\.wav`, sort numerically ascending.

**2. Read the header from segment 1**

Parse the RIFF/WAV header from `00000001.wav`:
- Verify RIFF magic and WAVE type
- Confirm format tag is `0xFFFE` (WAVE_FORMAT_EXTENSIBLE)
- Extract `wNumChannels` (offset 22, uint16 LE)
- Extract `nSamplesPerSec` (offset 24, uint32 LE)
- Extract `wBitsPerSample` (offset 34, uint16 LE) — container bits (32)
- Extract `wValidBitsPerSample` (offset 38, uint16 LE) — actual bits (24)
- Locate the `data` chunk to find `dataOffset`

**3. Calculate frame geometry**

```js
const bytesPerSample = wBitsPerSample / 8;          // 4
const bytesPerFrame  = wNumChannels * bytesPerSample; // 128 for 32ch
```

**4. Open output files**

For each channel K (0-based): construct the stem filename via `normalizeStemName(K+1, channelNames[K])` and open an output WAV file with a standard mono header (24-bit, `nSamplesPerSec`, 1 channel).

**5. Stream and demux**

```
for each segment file (in order):
  seek to dataOffset
  while bytes remain in data chunk:
    read one frame (bytesPerFrame bytes)
    for each channel K:
      extract 4 bytes at frame position K*4
      right-shift by 8 to get 24-bit value
      write 3 bytes to output[K]
```

**6. Finalize output files**

Seek back to start of each output, patch the RIFF and `data` chunk sizes, close all files.

### Key Constants

```js
const RIFF_MAGIC       = 0x46464952; // 'RIFF' LE
const WAVE_TYPE        = 0x45564157; // 'WAVE' LE
const FMT_CHUNK_ID     = 0x20746d66; // 'fmt ' LE
const DATA_CHUNK_ID    = 0x61746164; // 'data' LE
const EXTENSIBLE_TAG   = 0xFFFE;
const MAX_DATA_SIZE    = 0xFFFFFFFF; // sentinel for unknown size
const BYTES_PER_SAMPLE = 4;         // 32-bit container
const OUTPUT_BIT_DEPTH = 24;
const OUTPUT_SHIFT     = 8;         // right-shift to extract 24-bit content
```

### Edge Cases

| Scenario | Handling |
|---|---|
| Single-segment session | Normal path; no stitching required |
| `data` chunk size = 0xFFFFFFFF | Read until EOF; do not trust reported size |
| Fewer than 32 channels | Always read `wNumChannels` from header |
| Silent (all-zero) channels | Extract normally; caller filters post-extraction |
| Mismatched headers across segments ⚠️ | Warn; use segment 1 header as authoritative |
| Truncated final segment | Discard incomplete trailing frame |

---

## 8. Known Quirks and Gotchas

- **32-bit container, 24-bit content:** `wBitsPerSample` = 32 but `wValidBitsPerSample` = 24. Audio lives in bits 31-8. Some DAWs misidentify X-Live files as "32-bit float" — they are **integer PCM**.

- **No fact chunk in earlier firmware:** Some X-Live firmware versions omit the `fact` chunk. Locate the `data` chunk by scanning for the `"data"` fourCC, not by assuming a fixed offset.

- **Segment 1 reported size may be wrong:** The RIFF chunk size is written at the start of recording and may not be updated if recording is interrupted. Count frames for actual sample count.

- **Session directory name is a timestamp, not a counter:** The 8-char hex name is a Unix timestamp. The Front Half's `Session_NNN` numbering is imposed during import, not from the SD card.

- **Always read sample rate from header:** Engineers sometimes use 44.1 kHz for USB playback compatibility. Never assume 48 kHz.

---

## 9. Reference Tools

| Tool | Language | Author | Notes |
|---|---|---|---|
| `wav-extract` | Go | calebmcelroy | Extracts channels from interleaved WAV; handles multi-segment sessions |
| Live2Wav / Wav2Live | Windows/macOS binary | pmaillot | Reference implementation for X-Live format |
| `x32Live-CleanUp` | PowerShell + ffmpeg | Topslakr | Uses ffmpeg to split channels; useful for verifying channel order |
| `ffprobe` | — | FFmpeg project | Reports channel count, codec, sample rate from the WAV header |

Verify a PolyWAV file's header without custom code:

```bash
ffprobe -show_streams 00000001.wav 2>&1 | grep -E 'channels|sample_rate|codec_name|bits_per_sample'
```

Expected output for a typical X-Live file:

```
codec_name=pcm_s32le
sample_rate=48000
channels=32
bits_per_raw_sample=32
```
ve files — audio is 24-bit despite the 32-bit container. Lower 8 bits of each sample are always zero.

### data Chunk

| Offset | Size | Type | Description |
|---|---|---|---|
| Variable | 4 | ASCII | `"data"` |
| Variable+4 | 4 | uint32 | Size of audio data in bytes |
| Variable+8 | — | raw bytes | Interleaved PCM audio frames |

The data chunk size may be `0xFFFFFFFF` (−1 as uint32) in continuation segments. Parsers must handle this by reading until EOF. ⚠️

---
