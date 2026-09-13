# AeroTwin — Drone Video to 3D Reconstruction Pipeline

<div align="center">

![AeroTwin Banner](https://img.shields.io/badge/AeroTwin-v1.0.0-blue?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Windows_x64-lightgrey?style=for-the-badge)
![Team](https://img.shields.io/badge/Team-Synapse_X-purple?style=for-the-badge)
![Language](https://img.shields.io/badge/C%2B%2B-17-orange?style=for-the-badge)

**Built for Hack With Vizag 2026 | Team Synapse X**

*Transform drone footage into photorealistic 3D digital twins — automatically.*

</div>

---

## Overview

**AeroTwin** is a Windows desktop application that converts raw drone video footage into a complete 3D reconstruction dataset ready for **Gaussian Splatting** and **Neural Radiance Fields (NeRF)** workflows.

Point AeroTwin at any drone video file, set a frame rate, and it handles everything:
- Frame extraction at the specified FPS
- GPS coordinate embedding from DJI SRT telemetry files
- COLMAP-based Structure-from-Motion (SfM) reconstruction
- Sparse 3D point cloud and camera pose generation
- Image undistortion for downstream AI/3D pipelines

---

## Features

| Feature | Details |
|---------|---------|
| **Input** | MP4, AVI, MOV, and any FFmpeg-supported format |
| **SRT GPS Embed** | Auto-detects DJI SRT telemetry and embeds GPS into EXIF |
| **Frame Extraction** | Configurable FPS (1–30 fps) via FFmpeg |
| **3D Reconstruction** | COLMAP SfM (CPU & GPU), Agisoft Metashape, RealityScan 2.0 |
| **Output** | Sparse point cloud, camera poses, undistorted images |
| **GUI** | Native Win32 GUI — no dependencies, no install required |

---

## Architecture

`
┌─────────────────────────────────────────────────────────────────┐
│                       AeroTwin Application                      │
│                      (DroneRecon.exe)                           │
├──────────────────────┬──────────────────────────────────────────┤
│      GUI Layer       │          Pipeline Layer                   │
│    (gui.cpp/.h)      │        (pipeline.cpp/.h)                  │
│                      │                                          │
│  ┌────────────────┐  │  ┌────────────┐  ┌──────────────────┐   │
│  │   Win32 GUI    │  │  │   FFmpeg   │  │  GPS/SRT Parser  │   │
│  │  - File picker │──┼─▶│  Extractor │  │  (gps_embed.h)   │   │
│  │  - FPS input   │  │  └─────┬──────┘  └────────┬─────────┘   │
│  │  - Method sel. │  │        │                   │             │
│  │  - Progress log│  │        ▼                   ▼             │
│  └────────────────┘  │  ┌──────────────────────────────────┐   │
│                      │  │          Frame Directory          │   │
│                      │  │   frames/<video>/<name>_NNNN.jpg  │   │
│                      │  └──────────────┬───────────────────┘   │
│                      │                 │                        │
│                      │                 ▼                        │
│                      │  ┌──────────────────────────────────┐   │
│                      │  │      3D Reconstruction Engine     │   │
│                      │  │  ┌─────────┐  ┌───────────────┐  │   │
│                      │  │  │ COLMAP  │  │   Metashape   │  │   │
│                      │  │  │  (SfM)  │  │  (optional)   │  │   │
│                      │  │  └────┬────┘  └───────────────┘  │   │
│                      │  └───────┼───────────────────────────┘   │
└──────────────────────┴──────────┼──────────────────────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │        Output Directory      │
                    │  database/database.db        │
                    │  sparse/0/cameras.bin        │
                    │  sparse/0/images.bin         │
                    │  sparse/0/points3D.bin       │
                    │  images/ (undistorted)       │
                    └─────────────────────────────┘
`

---

## Pipeline Flowchart

`
 ┌──────────────────────────────────────────────┐
 │             START: User Input                 │
 │  - Video File Path                            │
 │  - Output Directory                           │
 │  - Frame Rate (fps)                           │
 │  - Reconstruction Method                     │
 └─────────────────────┬────────────────────────┘
                       │
                       ▼
 ┌─────────────────────────────────────────────┐
 │          STEP 1: Frame Extraction            │
 │                                             │
 │  FFmpeg extracts frames at N fps            │
 │  Output: frames/<video>/<name>_NNNN.jpg     │
 └─────────────────────┬───────────────────────┘
                       │
                       ▼
 ┌─────────────────────────────────────────────┐
 │         STEP 1b: GPS Embedding (Optional)    │
 │                                             │
 │  Detect <video>.SRT file alongside video    │
 │  Parse DJI GPS telemetry per frame          │
 │  Embed GPS coords into JPEG EXIF via        │
 │  ExifTool                                   │
 └─────────────────────┬───────────────────────┘
                       │
                       ▼
 ┌─────────────────────────────────────────────┐
 │          STEP 2: 3D Reconstruction           │
 │                                             │
 │  Kill any stale COLMAP processes            │
 │  Clean previous database/sparse dirs        │
 │                                             │
 │  ┌────────────────────────────────────┐     │
 │  │  2a: Feature Extraction            │     │
 │  │  colmap feature_extractor          │     │
 │  │  --FeatureExtraction.use_gpu 0     │     │
 │  │  ~9,000-15,000 SIFT features/img  │     │
 │  └──────────────┬─────────────────────┘     │
 │                 │                           │
 │  ┌──────────────▼─────────────────────┐     │
 │  │  2b: Feature Matching              │     │
 │  │  colmap exhaustive_matcher         │     │
 │  │  --FeatureMatching.use_gpu 0       │     │
 │  │  All pairs matched (CPU BF)        │     │
 │  └──────────────┬─────────────────────┘     │
 │                 │                           │
 │  ┌──────────────▼─────────────────────┐     │
 │  │  2c: Sparse Reconstruction         │     │
 │  │  colmap mapper                     │     │
 │  │  Incremental SfM, bundle adj.      │     │
 │  │  Outputs: cameras + point cloud    │     │
 │  └──────────────┬─────────────────────┘     │
 │                 │                           │
 │  ┌──────────────▼─────────────────────┐     │
 │  │  2d: Image Undistortion            │     │
 │  │  colmap image_undistorter          │     │
 │  │  Ready for NeRF / Gaussian Splat   │     │
 │  └────────────────────────────────────┘     │
 └─────────────────────┬───────────────────────┘
                       │
                       ▼
 ┌─────────────────────────────────────────────┐
 │                   OUTPUT                    │
 │                                             │
 │  sparse/0/cameras.bin   ← Camera intrinsics│
 │  sparse/0/images.bin    ← Camera poses     │
 │  sparse/0/points3D.bin  ← 3D point cloud   │
 │  images/                ← Undistorted imgs │
 │                                             │
 │  → Import into Gaussian Splatting           │
 │  → Import into Luma AI / NeRF Studio       │
 └─────────────────────────────────────────────┘
`

---

## Project Structure

`
AeroTwin/
├── src/
│   ├── main.cpp         — WinMain entry point
│   ├── gui.cpp          — Win32 GUI: window, controls, progress log
│   ├── gui.h            — GUI header
│   ├── pipeline.cpp     — Core pipeline: FFmpeg, COLMAP, ExifTool orchestration
│   ├── pipeline.h       — Pipeline function declarations & config
│   └── gps_embed.h      — DJI SRT parser + EXIF GPS embedding logic
├── vendor/              — (populated at build time, not committed)
│   ├── ffmpeg/          — FFmpeg static binary
│   ├── colmap/          — COLMAP binary (CPU, no CUDA)
│   └── exiftool/        — ExifTool binary
├── build/               — CMake build output
├── CMakeLists.txt       — Build configuration (MinGW / MSVC)
├── BUILD.md             — Build instructions
├── LICENSE              — MIT License
└── README.md            — This file
`

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Windows | 10 / 11 x64 | — |
| GCC (MinGW) | 13+ | scoop install gcc |
| CMake | 3.20+ | scoop install cmake |
| Make | Any | scoop install make |
| FFmpeg | Auto-downloaded | Handled by build |
| COLMAP (nocuda) | 4.2+ | Handled by build |
| ExifTool | Latest | scoop install exiftool |

> **Install Scoop** (no admin required):
> `powershell
> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
> irm get.scoop.sh | iex
> scoop install gcc cmake make
> `

---

## Build Instructions

`powershell
# 1. Clone the repo
git clone https://github.com/rohzyy/AeroTwin.git
cd AeroTwin

# 2. Download vendor binaries
scoop install ffmpeg exiftool

# Copy ffmpeg into vendor
New-Item -ItemType Directory -Force build\vendor\ffmpeg\bin
Copy-Item (Get-Command ffmpeg).Source build\vendor\ffmpeg\bin\ffmpeg.exe

# Download COLMAP (CPU-only, no CUDA required)
New-Item -ItemType Directory -Force build\vendor\colmap\bin
# Download from: https://github.com/colmap/colmap/releases
# Extract colmap.bat and its DLLs into build\vendor\colmap\bin\

# 3. Configure and build
cmake -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release .
cd build
make

# 4. Run
.\DroneRecon.exe
`

See [BUILD.md](BUILD.md) for detailed step-by-step instructions.

---

## Usage

1. **Launch** uild/DroneRecon.exe
2. **Select Video File** — click "File" and pick your drone .mp4
3. **Select Output Directory** — where results will be saved
4. **Set Frame Rate** — recommended: 10.0 fps for a 5–10s clip
5. **Choose Method** — select "COLMAP (bundled)"
6. **Click "Start Processing"** and wait (~10–15 min on CPU)

### Output Files

After successful completion, your output directory will contain:

`
<output>/
├── frames/
│   └── <videoname>/
│       ├── <videoname>_frame_0001.jpg
│       └── ...
├── database/
│   └── database.db          ← COLMAP feature database
├── sparse/
│   └── 0/
│       ├── cameras.bin      ← Camera intrinsics
│       ├── images.bin       ← Camera extrinsics (poses)
│       └── points3D.bin     ← Sparse 3D point cloud
└── images/
    └── *.jpg                ← Undistorted images for NeRF/Splat
`

### Using Output with Gaussian Splatting

`ash
# Using gaussian-splatting (https://github.com/graphdeco-inria/gaussian-splatting)
python train.py -s <output_directory>
`

---

## Technical Details

### SIFT Feature Extraction
Each frame yields ~9,000–15,000 SIFT keypoints at 1920×1080. COLMAP uses a SIMPLE_RADIAL camera model with focal length estimated from sensor metadata.

### Exhaustive Feature Matching
All image pairs are matched (O(n²)) — appropriate for short video clips. For longer sequences, sequential_matcher or ocab_tree_matcher should be used instead.

### Incremental Structure-from-Motion
COLMAP's incremental mapper finds the best initial image pair, then progressively registers all remaining cameras using bundle adjustment at each step.

### CPU-Only Mode
AeroTwin uses --FeatureExtraction.use_gpu 0 and --FeatureMatching.use_gpu 0 to ensure compatibility on all Windows machines without a CUDA GPU.

---

## Team

**Team Synapse X** — Hack With Vizag 2026

| Name | Role |
|------|------|
| **Rohan Malyadri** | Lead Developer & Architecture |

---

## License

`
MIT License
Copyright (c) 2026 Rohan Malyadri — Team Synapse X
`

See [LICENSE](LICENSE) for the full text.

---

<div align="center">
Made with ❤️ by Team Synapse X for Hack With Vizag 2026
</div>
