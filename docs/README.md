# Documentation

This folder contains project documentation for **xlive-ableton-import** — a tool that converts Behringer/Midas mixer PolyWAV multitrack recordings into fully-labeled, color-coded Ableton Live Sets.

## Contents

| File | Description |
|---|---|
| [`workflow-diagram.html`](./workflow-diagram.html) | Interactive end-to-end workflow diagram |

---

## Workflow Diagram

[**→ Open Workflow Diagram**](./workflow-diagram.html)

The workflow diagram is a self-contained interactive HTML document that maps every step of the xlive-ableton-import pipeline — from raw PolyWAV files on an SD card through to finished, labeled Ableton Live Sets.

It covers all six stages of the architecture:

| Stage | Color | What it shows |
|---|---|---|
| **0 — Inputs** | 🟢 Green | SD Card, Mixer Scene Files (USB), Engineer Configuration |
| **1 — Front Half (Node.js)** | 🔵 Blue | Scan → Detect → Split PolyWAV → Normalize → Organize stems |
| **2 — Intermediate** | 🟡 Amber | Stems Directory Tree hand-off between Front and Back Half |
| **3 — Back Half (M4L + JS)** | 🟣 Purple | Load device → Discover sessions → Import Mode → Create Tracks → Load Clips → Apply Metadata → Save |
| **4 — Phase 3 (Future)** | 🩷 Pink | Mixer Scene metadata pipeline (dashed, planned) |
| **5 — Outputs** | 💚 Emerald | Session_NNN.als Live Sets, future metadata.json |

### Interactive Features

#### 🖱️ Navigation
- **Scroll wheel** — zoom in and out on the diagram canvas
- **Click + drag** — pan around the canvas freely
- **Reset View button** — snaps the canvas back to the default centered view

#### 🔍 Node Inspection
- **Click any node** — opens a detail panel on the right side of the screen showing:
  - Node name and stage badge
  - Full description of what the step does
  - Inputs and outputs for that step
  - Implementation notes: which file and function handles it (e.g. `front-half/index.js`, `back-half/js/import.js`)
- **Press `Esc`** or click the × button — closes the detail panel

#### 🔗 Path Highlighting
- **Click any node** — highlights all incoming and outgoing arrows connected to that node; all other arrows are dimmed so you can trace the data flow at a glance

#### 👁️ Phase 3 Toggle
- **Toggle Phase 3 button** (top toolbar) — shows or hides the future-state Phase 3 nodes (Mixer Scene metadata pipeline), letting you focus on the current implemented architecture or preview the full planned system

#### 💎 Decision Points
- Diamond-shaped nodes mark every branching decision in the pipeline (e.g. *PolyWAV Files Found?*, *Import Mode?*, *Import All Sessions?*)
- Both YES and NO paths are labeled and connected to their respective downstream nodes

---

## Project Architecture Overview

The tool is split into two independently runnable halves:

### Front Half — Node.js
Runs on any machine with access to the SD card. Scans for PolyWAV container files, demultiplexes them into per-channel mono WAV stems, normalizes filenames, and organizes the output into a consistent directory structure.

```
/Sessions/
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

### Back Half — Max for Live Device
Runs inside Ableton Live. The `device.amxd` file is dragged onto any MIDI track. The JS engine (`import.js`) scans the stems path, creates one audio track per stem, loads each WAV into the corresponding clip slot, and applies track names, colors, and scene names. In **Import All** mode it also saves a `.als` file per session.

### Phase 3 — Mixer Metadata (Planned)
Will parse USB-sourced Behringer/Midas scene files to extract channel names, colors, and fader positions, producing a `metadata.json` that feeds into the Back Half for richer, mixer-accurate labeling.

---

## See Also

- [Project README](../README.md) — setup, usage, and contributing guide
- [examples/](../examples/) — sample session folders and expected output
