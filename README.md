# AHANOX GitHub Update System Architecture & Specification

## 1. Executive Summary

AHANOX will transition its update delivery channel from legacy hardcoded endpoints to a native **GitHub Releases & Raw CDN** architecture. The system supports live version checking, changelog rendering, and seamless 1-click binary updating for both the C++20 Direct2D standalone (`AHANOX.exe`) and the PyQt6 loader (`ahanox_loader.py`).

---

## 2. Architecture & Data Flow

```mermaid
flowchart TD
    A[User clicks 'Check for updates' or App Startup] --> B[GitHub Version Checker Thread]
    B --> C{Query Channel}
    C -->|Channel 1: Official API| D["api.github.com/repos/{owner}/{repo}/releases/latest"]
    C -->|Channel 2: Fallback CDN (No Rate Limits)| E["raw.githubusercontent.com/{owner}/{repo}/main/version.json"]
    
    D --> F[Parse JSON: version, tag, title, changelog, asset_url]
    E --> F
    
    F --> G{Compare with Current Version v2.0.1}
    G -->|Current == Latest| H[Status: Up to Date]
    G -->|Latest > Current| I[Status: Update Available]
    
    H --> J[Render 'Up to date' Badge + Latest Commit Info]
    I --> K[Render 'Update Available' Banner + Release Notes + Download Button]
    
    K --> L[User clicks '1-Click Update']
    L --> M[Download latest AHANOX.exe -> AHANOX_update.exe with Progress Bar]
    M --> N[Execute update.bat: Wait for exit -> Swap binary -> Relaunch new version]
```

---

## 3. GitHub Manifest & Release Format

### 3.1 Raw Version Manifest (`version.json`)
Placed in repository root `main` branch to bypass GitHub API unauthenticated rate limits (60 req/hr):

```json
{
  "version": "2.0.2",
  "tag": "v2.0.2",
  "title": "AHANOX v2.0.2 - Performance & Discord Sync",
  "date": "2026-11-01",
  "required": false,
  "changelog": [
    "Seamless Discord loopback authorization on port 8923",
    "Hardware-clipped circular Discord avatar rendering",
    "Direct GitHub Releases update pipeline",
    "Multi-threaded MSVC build pipeline hardening"
  ],
  "download_url": "https://github.com/OWNER/AHANOX/releases/download/v2.0.2/AHANOX.exe",
  "sha256": ""
}
```

### 3.2 GitHub Releases API Payload
Queried from `https://api.github.com/repos/{owner}/{repo}/releases/latest`:
- `tag_name`: Semantic version tag (e.g., `v2.0.2`)
- `name`: Release header title
- `body`: Markdown changelog description
- `html_url`: Direct browser link to GitHub release page
- `assets`: Array containing `AHANOX.exe` with `browser_download_url` and size

---

## 4. UI Design & States (Updates Tab)

The Updates screen in `ahanox_main.cpp` and `ahanox_loader.py` will feature two primary high-tech HUD cards:

### Card 1: `GITHUB RELEASE CHANNEL`
- **Header**: GitHub octocat / cloud sync icon with channel identifier.
- **Repository Link**: Clickable link to `github.com/{owner}/{repo}` with configurable override in `C:\ProgramData\ahanox.ini` (`github_repo=owner/repo`).
- **Current Version**: `AHANOX v2.0.1` (`Current Production Build`).
- **Dynamic Status Badge**:
  - `[ 🟢 Up to Date — v2.0.1 ]`
  - `[ 🚀 Update Available: v2.0.2 ]` (pulsing accent glow)
  - `[ ⏳ Checking GitHub Releases... ]`
- **Controls**:
  - `[ Check for updates ]`: Asynchronous query with animated refresh indicator.
  - `[ View on GitHub ]`: Opens the release in Google Chrome/default browser.
  - `[ 1-Click Update ]`: Initiates background binary download and hot-swap.

### Card 2: `LATEST RELEASE CHANGELOG`
- Displays release title, author (`!Aha_nov`), and publication timestamp.
- Styled terminal/glass changelog box listing the exact changes from GitHub.

---

## 5. Hot-Swap 1-Click Updater Engine (`update.bat`)

When dj or a user clicks **Download & Update**:
1. App downloads the binary to `AHANOX_update.exe` via WinHTTP/urllib with progress bar feedback.
2. App writes temporary hot-swap script `_apply_update.bat`:
   ```bat
   @echo off
   timeout /t 1 /nobreak >nul
   :wait_loop
   tasklist | find /i "AHANOX.exe" >nul
   if not errorlevel 1 (
       timeout /t 1 /nobreak >nul
       goto wait_loop
   )
   move /y "AHANOX_update.exe" "AHANOX.exe" >nul
   start "" "AHANOX.exe"
   del "%~f0"
   ```
3. App launches `_apply_update.bat` detached and gracefully exits.
4. The batch script swaps the binary and relaunches the new `AHANOX.exe` automatically in ~1.5 seconds.

---

## 6. Implementation Scope

| Target File | Modifications |
|---|---|
| `ahanox_core.hpp` | Add `github_repo` config field, settings load/save, and version comparator. |
| `ahanox_main.cpp` | Implement `render_tab_updates` GitHub card, async WinHTTP release check, changelog display, and update runner. |
| `ahanox_loader.py` | Replace legacy `wispbyte` URL with GitHub Releases API & `version.json` fallback, update Qt page. |
| `build_ahanox.bat` | Verify deliverable synchronization and compilation. |
| `version.json` | Create sample GitHub release manifest for the repository. |

