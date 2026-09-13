# AeroTwin — Drone Video to 3D Reconstruction Pipeline

<div align="center">

![AeroTwin](https://img.shields.io/badge/AeroTwin-v1.0.0-blue?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Windows_x64-lightgrey?style=for-the-badge)
![Team](https://img.shields.io/badge/Team-Synapse_X-purple?style=for-the-badge)
![C++17](https://img.shields.io/badge/C%2B%2B-17-orange?style=for-the-badge)

**Built for Hack With Vizag 2026 | Team Synapse X**

*Transform drone footage into photorealistic 3D digital twins — automatically.*

</div>

---

## Overview

**AeroTwin** is a Windows desktop application that converts raw drone video footage into a complete 3D reconstruction dataset ready for **Gaussian Splatting** and **Neural Radiance Fields (NeRF)** workflows.

Point AeroTwin at any drone video file, set a frame rate, and it handles everything:
- Frame extraction at the specified FPS via FFmpeg
- GPS coordinate embedding from DJI SRT telemetry files
- COLMAP-based Structure-from-Motion (SfM) 3D reconstruction
- Sparse 3D point cloud and camera pose generation
- Image undistortion ready for downstream AI/3D pipelines

---

## Features

| Feature | Details |
|---------|---------|
| **Input** | MP4, AVI, MOV, and any FFmpeg-supported format |
| **SRT GPS Embed** | Auto-detects DJI SRT telemetry and embeds GPS into EXIF |
| **Frame Extraction** | Configurable FPS (1–30 fps) via FFmpeg |
| **3D Reconstruction** | COLMAP SfM (CPU), Agisoft Metashape, RealityScan 2.0 |
| **Output** | Sparse point cloud, camera poses, undistorted images |
| **GUI** | Native Win32 GUI — no dependencies, no install required |

---

## Architecture

```mermaid
graph TD
    A["🚁 AeroTwin — DroneRecon.exe"] --> B["🖥️ GUI Layer\ngui.cpp / gui.h"]
    A --> C["⚙️ Pipeline Layer\npipeline.cpp / pipeline.h"]

    B --> D["Win32 GUI\n• File / Folder picker\n• Frame rate input\n• Method selector\n• Real-time progress log"]

    C --> E["📹 FFmpeg\nFrame Extractor"]
    C --> F["🛰️ GPS / SRT Parser\ngps_embed.h"]

    E --> G["🗂️ Frame Directory\nframes/<video>/<name>_NNNN.jpg"]
    F --> G

    G --> H["🔷 3D Reconstruction Engine"]

    H --> I["COLMAP SfM\n(bundled, CPU)"]
    H --> J["Agisoft Metashape\n(optional, external)"]
    H --> K["RealityScan 2.0\n(optional, external)"]

    I --> L["📦 Output Directory"]

    L --> M["database/database.db"]
    L --> N["sparse/0/cameras.bin"]
    L --> O["sparse/0/images.bin"]
    L --> P["sparse/0/points3D.bin"]
    L --> Q["images/ (undistorted)"]
```

---

## Pipeline Flowchart

```mermaid
flowchart TD
    START(["▶ START\nUser provides Video, Output Dir,\nFPS, Reconstruction Method"])

    START --> STEP1

    STEP1["📹 STEP 1 — Frame Extraction\nFFmpeg extracts frames at N fps\nOutput → frames/<video>/<name>_NNNN.jpg"]

    STEP1 --> SRT_CHECK{SRT file\nfound?}

    SRT_CHECK -- Yes --> GPS["🛰️ STEP 1b — GPS Embedding\nParse DJI SRT telemetry\nEmbed GPS into JPEG EXIF via ExifTool"]
    SRT_CHECK -- No --> SKIP["⏭ Skip GPS\n(no telemetry file)"]

    GPS --> STEP2
    SKIP --> STEP2

    STEP2["🔷 STEP 2 — 3D Reconstruction\nKill stale COLMAP processes\nClean previous database & sparse dirs"]

    STEP2 --> FE["2a — Feature Extraction\ncolmap feature_extractor\n--FeatureExtraction.use_gpu 0\n~9,000–15,000 SIFT features per image"]

    FE --> FM["2b — Feature Matching\ncolmap exhaustive_matcher\n--FeatureMatching.use_gpu 0\nAll image pairs matched (CPU brute-force)"]

    FM --> SR["2c — Sparse Reconstruction\ncolmap mapper\nIncremental SfM + Global Bundle Adjustment\nOutputs: camera poses + 3D point cloud"]

    SR --> UD["2d — Image Undistortion\ncolmap image_undistorter\nReady for NeRF / Gaussian Splatting"]

    UD --> OUT

    OUT(["✅ OUTPUT\nsparse/0/cameras.bin — Camera intrinsics\nsparse/0/images.bin  — Camera poses\nsparse/0/points3D.bin — 3D point cloud\nimages/               — Undistorted frames"])

    OUT --> GSPLAT["→ Gaussian Splatting\n(graphdeco-inria/gaussian-splatting)"]
    OUT --> NERF["→ NeRF / Luma AI / NeRF Studio"]
```

---

## Project Structure

```
AeroTwin/
├── src/
│   ├── main.cpp          — WinMain entry point
│   ├── gui.cpp           — Win32 GUI: window, controls, progress log
│   ├── gui.h             — GUI header
│   ├── pipeline.cpp      — Core pipeline: FFmpeg, COLMAP, ExifTool orchestration
│   ├── pipeline.h        — Pipeline function declarations & config
│   └── gps_embed.h       — DJI SRT parser + EXIF GPS embedding logic
├── vendor/               — (populated at build time, not committed)
│   ├── ffmpeg/           — FFmpeg static binary
│   ├── colmap/           — COLMAP binary (CPU, no CUDA required)
│   └── exiftool/         — ExifTool binary
├── build/                — CMake build output
├── CMakeLists.txt        — Build config (MinGW / MSVC)
├── BUILD.md              — Detailed build instructions
├── CHANGELOG.md          — Version history
├── LICENSE               — MIT License
└── README.md             — This file
```

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Windows | 10 / 11 x64 | — |
| GCC (MinGW) | 13+ | `scoop install gcc` |
| CMake | 3.20+ | `scoop install cmake` |
| Make | Any | `scoop install make` |
| FFmpeg | Auto | `scoop install ffmpeg` |
| ExifTool | Latest | `scoop install exiftool` |
| COLMAP (no-CUDA) | 4.2+ | [GitHub Releases](https://github.com/colmap/colmap/releases) |

> **Install Scoop** (no admin required):
> ```powershell
> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
> irm get.scoop.sh | iex
> scoop install gcc cmake make ffmpeg exiftool
> ```

---

## Build Instructions

```powershell
# 1. Clone the repo
git clone https://github.com/rohzyy/AeroTwin.git
cd AeroTwin

# 2. Place vendor binaries into build/vendor/
#    FFmpeg  → build/vendor/ffmpeg/bin/ffmpeg.exe
#    COLMAP  → build/vendor/colmap/bin/colmap.bat + DLLs
#    ExifTool→ build/vendor/exiftool/exiftool.exe

# 3. Configure and build
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release .
cd build
make

# 4. Run
.\DroneRecon.exe
```

See [BUILD.md](BUILD.md) for detailed step-by-step instructions.

---

## Usage

1. **Launch** `build/DroneRecon.exe`
2. **Select Video File** — click "File" and pick your drone `.mp4`
3. **Select Output Directory** — where results will be saved
4. **Set Frame Rate** — recommended `10.0` fps for a 5–10 s clip
5. **Choose Method** — select "COLMAP (bundled)"
6. **Click "Start Processing"** — wait ~10–15 min on CPU

### Output Files

```
<output>/
├── frames/<videoname>/
│   ├── <videoname>_frame_0001.jpg
│   └── ...
├── database/
│   └── database.db          ← COLMAP feature database
├── sparse/
│   └── 0/
│       ├── cameras.bin      ← Camera intrinsics
│       ├── images.bin       ← Camera extrinsics (poses)
│       └── points3D.bin     ← Sparse 3D point cloud
└── images/
    └── *.jpg                ← Undistorted images
```

### Using Output with Gaussian Splatting

```bash
python train.py -s <output_directory>
```

---

## Technical Details

### SIFT Feature Extraction
Each frame yields ~9,000–15,000 SIFT keypoints at 1920×1080. COLMAP uses a `SIMPLE_RADIAL` camera model with focal length estimated from sensor metadata.

### Exhaustive Feature Matching
All image pairs are matched — appropriate for short video clips. For longer sequences, `sequential_matcher` or `vocab_tree_matcher` should be used instead.

### Incremental Structure-from-Motion
COLMAP finds the best initial image pair, then progressively registers all cameras using bundle adjustment at each step.

### CPU-Only Mode
AeroTwin passes `--FeatureExtraction.use_gpu 0` and `--FeatureMatching.use_gpu 0` to ensure compatibility on any Windows machine without a CUDA GPU.

---

## Team

**Team Synapse X** — Hack With Vizag 2026

| Name | Role |
|------|------|
| **Rohan Malyadri** | Lead Developer & Architecture |

---

## License

```
MIT License
Copyright (c) 2026 Rohan Malyadri — Team Synapse X
```

See [LICENSE](LICENSE) for the full text.

---

<div align="center">
Made with ❤️ by Team Synapse X for Hack With Vizag 2026
</div>
