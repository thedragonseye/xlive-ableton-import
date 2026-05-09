# xlive-ableton-import — End-to-End Workflow

This document describes the full workflow from XLive session media ingestion (SD card / USB drive)
through conversion and creation of finished Ableton Live Sets. The steps are designed to be
**traceable**: each action and system step has a stable ID (E# for engineer actions, S# for software/system steps).

> Diagram is authored in Mermaid so it renders directly on GitHub.

---

## 1) Workflow Overview (Traceable)

```mermaid
flowchart TD
  %% =========================================================
  %% xlive-ableton-import End-to-End Workflow (Traceable)
  %% =========================================================

  start([Start]) --> E1

  %% -------- Engineer lane --------
  subgraph AV["A/V Engineer Actions (E#)"]
    direction TB
    E1["E1 Insert media (SD card or USB drive) into computer"]
    E2["E2 Confirm media is mounted / accessible"]
    E3["E3 Locate session folder(s) on media"]
    E4["E4 Choose target session(s) to import"]
    E5["E5 Choose import options\n• destination folder\n• track naming strategy\n• tempo/timebase assumptions\n• render/convert settings"]
    E6["E6 Run xlive-ableton-import (CLI/GUI)"]
    E7["E7 Review import report / warnings"]
    E8["E8 Open generated Live Set(s) in Ableton Live"]
    E9["E9 Verify audio alignment, track layout, routing"]
    E10["E10 Save final Live Set(s) / package for delivery"]
  end

  %% -------- System lane --------
  subgraph SYS["xlive-ableton-import System Steps (S#)"]
    direction TB
    S1["S1 Detect media type and root path"]
    S2["S2 Scan for supported XLive session structures"]
    S3["S3 Validate session integrity\n• expected files present\n• readable audio assets\n• metadata parseable"]
    S4["S4 Extract session metadata\n• track list\n• take/clip regions\n• timestamps/timebase info\n• sample rate/bit depth"]
    S5["S5 Build conversion plan\n• track mapping\n• file transforms\n• set structure layout"]
    S6["S6 Convert or normalize audio (as configured)\n• resample (if needed)\n• channel format normalization\n• loudness/trim (optional)"]
    S7["S7 Generate Ableton project artifacts\n• tracks\n• clips\n• arrangement timeline\n• markers/locators (if available)\n• metadata notes"]
    S8["S8 Write outputs to destination\n• Ableton project folder\n• referenced audio files\n• logs/report"]
    S9["S9 Produce final report\n• outputs created\n• warnings/errors\n• next actions"]
  end

  %% -------- Decisions & loops --------
  E1 --> E2 --> S1

  S1 --> D1{Media readable?}
  D1 -- "No" --> X1["S-ERR1 Stop: Media not readable\nRecommend: check adapter/cable, reinsert, try another port"] --> endstop([End])
  D1 -- "Yes" --> S2 --> D2{Sessions found?}

  D2 -- "No" --> X2["S-ERR2 Stop: No supported sessions detected\nRecommend: verify folder path and supported session format"] --> endstop
  D2 -- "Yes" --> E3 --> E4 --> E5 --> E6 --> S3 --> D3{Validation OK?}

  D3 -- "No" --> X3["S-ERR3 Report validation failures\n• missing/corrupt files\n• unsupported encoding\n• metadata parse failure"] --> E7 --> D4{Try a different session / fix media?}
  D4 -- "Fix & retry" --> E2
  D4 -- "Abort" --> endstop

  D3 -- "Yes" --> S4 --> S5 --> S6 --> D5{Conversion warnings?}
  D5 -- "Yes" --> S8 --> S9 --> E7 --> D6{Proceed anyway?}
  D6 -- "Adjust options" --> E5
  D6 -- "Proceed" --> E8

  D5 -- "No" --> S7 --> S8 --> S9 --> E8 --> E9 --> D7{Meets expectations?}
  D7 -- "No" --> E5
  D7 -- "Yes" --> E10 --> end([End: Delivered Live Sets])
