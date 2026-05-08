# **README — X Live Ableton Import**

## **Overview**
**X Live Ableton Import** is a two‑stage workflow designed for live engineers who record multitrack sessions on a digital mixer and need to rapidly assemble fully‑labeled, color‑coded Ableton Live Sets for rehearsal review, production, or mixdown.

The system consists of:

1. **Front Half (Node.js)**  
   Converts PolyWAV files from the mixer’s SD card into individual stems, organized by session.

2. **Back Half (Max for Live + JavaScript)**  
   Runs inside Ableton Live to automatically build a new Live Set per session, populate tracks with stems, apply mixer‑derived names/colors, and save the finished `.als` files.

This workflow eliminates manual track creation, clip placement, labeling, and coloring — turning hours of repetitive work into a single button press.

---

## **Architecture**

### **1. Front Half — Node.js PolyWAV → Stems**
The front‑half tool:

- Scans the SD card for session folders  
- Detects PolyWAV multitrack files  
- Splits them into individual WAV stems  
- Normalizes naming  
- Organizes output into a predictable directory structure:

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

This stage runs entirely outside Ableton and prepares the raw audio for import.

---

### **2. Back Half — Max for Live Device (JS Engine)**
The back‑half M4L device:

- Discovers all session folders created by the front half  
- Allows the engineer to:
  - Import a **single selected session** into the current Live Set  
  - Import **all sessions**, generating a **new Live Set per session**  
- Creates tracks automatically  
- Loads stems into clips  
- Applies:
  - Track names  
  - Track colors  
  - Scene names  
- Saves each Live Set as:

```
Session_001.als
Session_002.als
...
```

This stage runs inside Ableton and handles all Live Set construction.

---

## **Future Feature: Mixer Scene Metadata Integration**
A planned enhancement will allow the system to:

- Read mixer scene files from USB  
- Parse channel names, colors, and layout  
- Output a metadata JSON file  
- Apply that metadata during Live Set creation

This will ensure Ableton tracks match the mixer’s labeling and color scheme exactly.

---

## **Repository Structure**

```
x-live-ableton-import/
│
├── front-half/              # Node.js polywav → stems tool
│   ├── index.js
│   ├── package.json
│   └── lib/
│
├── back-half/               # M4L device + JS scripts
│   ├── device.amxd
│   ├── js/
│   │   └── import.js
│   └── ui/
│
├── docs/
│   └── architecture.md
│
├── examples/
│
└── README.md
```

---

## **Installation**

### **Front Half (Node.js)**
Requires Node.js 18+.

```
cd front-half
npm install
node index.js
```

### **Back Half (M4L)**
- Open Ableton Live  
- Drag `device.amxd` into a MIDI track  
- Configure the session folder path  
- Use the UI to import sessions

---

## **Usage**

### **Import All Sessions**
Creates a new Live Set per session:

1. Load the M4L device  
2. Press **Import All Sessions**  
3. Ableton generates:
   ```
   Session_001.als
   Session_002.als
   ...
   ```

### **Import Selected Session**
1. Choose a session from the dropdown  
2. Press **Import Selected Session**  
3. Stems populate the current Live Set

---

## **Roadmap**

### **Phase 1 — Front Half (Complete)**
- PolyWAV detection  
- Stem splitting  
- Session folder organization  

### **Phase 2 — Back Half (In Progress)**
- M4L device UI  
- JS importer  
- New Live Set per session  
- Import selected session  

### **Phase 3 — Mixer Metadata Integration**
- Parse mixer scene files  
- Output JSON metadata  
- Apply names/colors in Ableton  

### **Phase 4 — Packaging**
- Installers  
- Documentation  
- Example sessions  

---

## **License**
MIT License.
