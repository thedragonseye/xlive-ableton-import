# Contributing to xlive-ableton-import

Thank you for your interest in contributing! This project is built for live sound engineers and is maintained as a focused, practical tool. Contributions that improve reliability, extend the pipeline, or improve documentation are warmly welcomed.

---

## Table of Contents

1. [Code of Conduct](#1-code-of-conduct)
2. [How to Contribute](#2-how-to-contribute)
3. [Development Setup](#3-development-setup)
4. [Repository Structure](#4-repository-structure)
5. [Coding Standards](#5-coding-standards)
6. [Branch and Commit Workflow](#6-branch-and-commit-workflow)
7. [Pull Request Process](#7-pull-request-process)
8. [Testing](#8-testing)
9. [Roadmap and Good First Issues](#9-roadmap-and-good-first-issues)
10. [Getting Help](#10-getting-help)

---

## 1. Code of Conduct

This project follows a simple standard: be respectful, b
## 3. Development Setup

### Front Half (Node.js)

**Prerequisites**

- Node.js **18 or later** (check with `node --version`)
- npm 9 or later (bundled with Node 18+)
- A Behringer/Midas mixer SD card with X-Live recordings, or the example sessions in `examples/`

**Steps**

```bash
# 1. Fork and clone the repository
git clone https://github.com/YOUR_USERNAME/xlive-ableton-import.git
cd xlive-ableton-import

# 2. Install Front Half dependencies
cd front-half
npm install

# 3. Run against example data
node index.js --input ../examples/sample-session --output ../examples/output --dry-run --verbose

# 4. Run tests
npm test
```

**Environment variables (optional)**

| Variable | Description |
|---|---|
| `XLIVE_LOG_LEVEL` | Set to `debug` for verbose internal logging |
| `XLIVE_OUTPUT_ROOT` | Override the default output path without passing `--output` |

### Back Half (Max for Live)

**Prerequisites**

- Ableton Live **11 or later** (Suite or with Max for Live add-on)
- Max 8 or later (bundled with Ableton's Max for Live)
- The Front Half must have been run at least once to produce a sessions directory

**Steps**

1. Open Ableton Live and create or open a project.
2. Drag `back-half/device.amxd` onto any **MIDI track**.
3. In the device UI, set the **Session Folder Path** to your sessions directory.
4. Open the Max patcher (`Options -> Open Max Window`) to see JS console output.
5. Edit `back-half/js/import.js` in any text editor; save and click **Reload** in the Max patcher to pick up changes without restarting Ableton.

**Tip:** Enable Max's JS debugger (`Options -> JavaScript Debugger`) to set breakpoints and step through `import.js` interactively.

---

## 4. Repository Structure

```
xlive-ableton-import/
├── front-half/            # Node.js PolyWAV -> stems tool
│   ├── index.js           # CLI entry point
│   ├── package.json
│   └── lib/               # Core splitting and naming utilities
│       ├── scanner.js     # scanForSessions, detectPolyWAV
│       ├── splitter.js    # splitPolyWAV
│       └── namer.js       # normalizeStemName
│
├── back-half/             # Max for Live device
│   ├── device.amxd        # Compiled M4L device (binary)
│   ├── js/
│   │   └── import.js      # JS engine: session discovery, track creation, LOM calls
│   └── ui/                # Max UI patches and assets
│
├── docs/                  # Project documentation
│   ├── README.md          # Docs index and overview
│   ├── API.md             # Full API reference
│   ├── CONTRIBUTING.md    # This file
│   └── workflow-diagram.html  # Interactive pipeline diagram
│
├── examples/              # Sample sessions and reference output
├── src/                   # Shared utilities (future use)
├── LICENSE                # MIT License
└── README.md              # Project overview and quick-start
```

**Key rule:** `device.amxd` is a binary Max patcher file. Do not edit it with a text editor — make all logic changes in `back-half/js/import.js` and UI changes through the Max application.

---

## 5. Coding Standards

### JavaScript / Node.js

- **Style:** Follow the existing code style. Keep things readable and consistent with surrounding code.
- **ES version:** Use ES2022+ features freely (Node 18 supports them all). Prefer `async/await` over raw Promises or callbacks.
- **Imports:** Use `import`/`export` (ESM) in all new modules. Do not mix with `require()`.
- **Error handling:** Never silently swallow errors. Always either re-throw, log with context, or handle explicitly. Use the error codes from [API.md](./API.md#5-error-reference).
- **Side effects:** Keep utility functions (`lib/`) pure where possible. Side effects belong in `index.js` or explicitly named `*Runner`/`*Writer` modules.
- **Comments:** Comment the *why*, not the *what*.

### Max for Live JS

- Max's JS environment is **not** a standard browser or Node.js context. It does not support ESM `import`/`export`.
- Avoid blocking the Max event loop. Keep LOM calls lightweight and batch track creation in a single pass where possible.
- Log with context using `post()`: `post("Creating track for stem: " + stemName + "\n");`
- Test every LOM path change in a live Ableton session.

### Naming Conventions

| Entity | Convention | Example |
|---|---|---|
| JS functions | `camelCase` | `splitPolyWAV`, `normalizeStemName` |
| JS variables | `camelCase` | `sessionPath`, `trackIndex` |
| JS constants | `UPPER_SNAKE_CASE` | `DEFAULT_OUTPUT_ROOT` |
| Files (JS) | `kebab-case` | `session-scanner.js` |
| Session directories | `Session_{NNN}` (3-digit zero-padded) | `Session_001` |
| Stem files | `{NN}_{Name}.wav` (2-digit zero-padded) | `03_Bass_DI.wav` |
| Branch names | `type/short-description` | `feat/phase3-metadata` |

---

## 6. Branch and Commit Workflow

### Branches

Create feature branches off `main`:

```bash
git checkout main
git pull origin main
git checkout -b feat/your-feature-name
```

Branch naming prefixes:

| Prefix | Use for |
|---|---|
| `feat/` | New features |
| `fix/` | Bug fixes |
| `docs/` | Documentation only |
| `refactor/` | Code restructuring without behavior change |
| `test/` | Test additions or improvements |
| `chore/` | Dependency updates, build config |

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
**Scopes:** `front-half`, `back-half`, `docs`, `examples`, `deps`

**Examples:**

```
feat(front-half): add --force flag to overwrite existing stems

## 7. Pull Request Process

1. **Open an issue first** for any non-trivial change so there's an agreed direction before you invest time coding.
2. **Push your branch** and open a PR against `main`.
3. **Fill out the PR description** — include what changed, why, and how to test it. Link the related issue.
4. **Self-review your diff** before requesting review. Check for debug logs, commented-out code, and unintended file changes.
5. **One approval required** from a maintainer before merging.
6. **Squash merge** is preferred for feature branches to keep the `main` history clean.

### PR Checklist

Before marking your PR ready for review, confirm:

- [ ] Code follows the standards in Section 5
- [ ] New functions are documented with JSDoc comments
- [ ] `docs/API.md` is updated if any interface changed
- [ ] Tests pass (`npm test` in `front-half/`)
- [ ] No debug `console.log` or `post()` calls left in
- [ ] `workflow-diagram.html` updated if the pipeline architecture changed

---

## 8. Testing

### Front Half

Tests live in `front-half/test/` and use Node's built-in test runner:

```bash
cd front-half
npm test
```

**What to test:**

- `scanner.js` — provide a fixture directory tree and assert correct session detection
- `splitter.js` — provide a known PolyWAV fixture and assert correct channel count, filenames, and audio content
- `namer.js` — unit-test the normalization logic for edge cases: empty names, special characters, indices over 9 and 99

**Adding test fixtures:** Place small anonymized WAV fixtures in `front-half/test/fixtures/`. Keep fixtures under 1 MB.

### Back Half

The Max for Live JS engine has no automated test harness yet. Test Back Half changes manually:

1. Run the Front Half against `examples/sample-session` to produce a stems directory.
2. Load the M4L device in Ableton and point it at that directory.
3. Test both **Import Selected Session** and **Import All Sessions** modes.
4. Verify track names, track count, clip placement, and (in Import All mode) that `.als` files are saved correctly.

---

## 9. Roadmap and Good First Issues

| Phase | Status | Good contribution areas |
|---|---|---|
| **Phase 1 — Front Half** | Complete | Bug fixes, edge cases, `--dry-run` improvements, test coverage |
| **Phase 2 — Back Half** | In Progress | Import reliability, LOM error handling, UI polish |
| **Phase 3 — Mixer Metadata** | Planned | Scene file parser, `metadata.json` writer, Back Half consumer |
| **Phase 4 — Packaging** | Planned | Installers, CI/CD, release automation |

### Suggested First Issues

- **Add `--version` flag to the Front Half CLI** — print the version from `package.json` and exit
- **Improve empty-session handling** — log a clear warning and skip gracefully when a session folder has no stems
- **Add JSDoc to all `lib/` functions** — the functions exist but lack inline documentation
- **Write unit tests for `namer.js`** — edge cases: names with slashes, emoji, very long strings, all-numeric names
- **Document the PolyWAV binary format** — add a `docs/polywav-format.md` describing the header structure for future parser contributors

---

## 10. Getting Help

- **GitHub Issues** — the best place for bug reports and feature discussions
- **GitHub Discussions** — for open-ended questions, architecture ideas, or "is this a good idea?" conversations
- **PR comments** — ask questions directly on your draft PR if you're unsure about an implementation choice

When asking for help, please include your OS and version, Node.js version, Ableton Live and Max for Live versions (for Back Half issues), and mixer model and firmware (for PolyWAV parsing issues).

---

*This document follows the project conventions in Section 5. If you find something unclear or missing, a PR to improve it is always welcome.*

fix(back-half): prevent crash when stems directory is empty

docs(docs): add Phase 3 metadata schema to API reference
```

- Keep the subject line under 72 characters.
- Use the imperative mood ("add", "fix", "update").
- Reference issues in the footer: `Closes #12`, `Related to #8`.
e constructive, and assume good faith. Harassment or dismissive behavior of any kind will not be tolerated.

---

## 2. How to Contribute

There are several ways to contribute:

- **Bug reports** — Open a GitHub Issue with a clear description, steps to reproduce, your OS, Node version, Ableton version, and mixer model.
- **Feature requests** — Open a GitHub Issue tagged `enhancement`. Check the roadmap first to see if it's already planned.
- **Code contributions** — Fork the repo, create a branch, make your changes, and open a Pull Request.
- **Documentation** — Improvements to any file in `docs/` are always welcome, including corrections, clarifications, and new examples.
- **Example sessions** — Anonymized example PolyWAV structures or reference output in `examples/` help new contributors understand the format.
