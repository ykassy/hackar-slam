# hackar-slam — Web SLAM Engine Architecture

## Overview

Lightweight visual SLAM engine for mobile WebAR. Runs entirely in the browser using WebAssembly + Web Workers. No WebXR API dependency (iOS Safari non-support).

**Target**: 20-30fps on mobile (iPhone 12+, Android mid-range+)
**License**: MIT
**Input**: Camera feed via getUserMedia (320x240 grayscale)
**Output**: 6DoF camera pose (4x4 matrix) + plane data

## Pipeline

```
Camera (getUserMedia 640x480)
    ↓ downscale to 320x240 grayscale
[Web Worker]
    ↓
┌──────────────────────────────────┐
│ 1. Feature Detection (FAST-9)    │  ~3ms
│ 2. Descriptor (BRIEF 256-bit)    │  ~2ms
│ 3. Matching (Hamming + Ratio)    │  ~3ms
│ 4. Pose Estimation (Essential/PnP)│  ~4ms
│ 5. Triangulation (DLT)           │  ~2ms
│ 6. Plane Detection (RANSAC)      │  ~3ms
└──────────────────────────────────┘
    ↓ postMessage (pose matrix + plane)
[Main Thread]
    ↓
Three.js rendering (AR overlay)
```

## Module Breakdown

### Core (C++ → WASM via Emscripten)

| Module | File | Description | Complexity |
|--------|------|-------------|------------|
| Feature Detection | `src/core/fast.cpp` | FAST-9 corner detection + Harris scoring | Low |
| Descriptor | `src/core/fast.cpp` | BRIEF 256-bit binary descriptor | Low |
| Matching | `src/core/fast.cpp` | Brute-force Hamming + Lowe's ratio test | Low |
| Pose Estimation | `src/core/pose.cpp` | Essential matrix (8-point) + decomposition | High |
| PnP Solver | `src/core/pose.cpp` | EPnP for 3D-2D tracking | High |
| Triangulation | `src/core/pose.cpp` | Linear DLT triangulation | Medium |
| Plane Detection | `src/core/pose.cpp` | RANSAC plane fitting from 3D points | Medium |
| Map Management | `src/core/map.cpp` | Keyframes, map points, covisibility | Medium |
| Bundle Adjustment | `src/core/ba.cpp` | Sparse Levenberg-Marquardt (local BA) | High |

### JavaScript/TypeScript Layer

| Module | File | Description |
|--------|------|-------------|
| Worker Entry | `src/js/worker.ts` | WASM init, frame processing loop |
| Main API | `src/js/slam.ts` | Public API, Worker communication |
| Three.js Adapter | `src/js/three-adapter.ts` | Pose → Three.js camera, plane → mesh |
| Camera Utils | `src/js/camera.ts` | getUserMedia, grayscale conversion |

### Demo

| File | Description |
|------|-------------|
| `demo/index.html` | Camera + cube fixed in 3D space |
| `demo/plane.html` | Plane detection + object placement |

## Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Core computation | C++ → WASM (Emscripten) | SIMD support, performance |
| Threading | Web Worker (dedicated) | Non-blocking main thread |
| Feature detection | FAST-9 (custom C++) | Lightest viable detector |
| Tracking | BRIEF descriptor + Hamming matching | Binary = fast, SIMD-friendly |
| Pose estimation | 8-point + EPnP | Well-understood, implementable |
| Plane detection | RANSAC | Standard, simple |
| API wrapper | TypeScript | Type safety, Three.js compat |
| Build | CMake + Emscripten | Standard WASM toolchain |
| Package | npm | Easy integration |

## Browser Compatibility

| Feature | iOS Safari | Android Chrome | Notes |
|---------|-----------|---------------|-------|
| getUserMedia | 16.4+ | ✅ | Camera access |
| WASM | ✅ | ✅ | Core requirement |
| WASM SIMD | 16.4+ | ✅ | 2-4x speedup for feature ops |
| Web Worker | ✅ | ✅ | Off-main-thread processing |
| SharedArrayBuffer | ✅* | ✅* | *Requires COOP/COEP headers |
| DeviceMotion (IMU) | ⚠️ permission | ✅ | Optional, improves quality |
| WebGPU | iOS 26+ | Limited | Future: GPU feature extraction |

### Required HTTP Headers (for SharedArrayBuffer)
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

## Phase Plan

### Phase 1: Visual Odometry MVP (2-3 weeks)
- [x] Project setup (repo, build system, demo page)
- [ ] FAST-9 feature detection (C++/WASM)
- [ ] BRIEF descriptor computation
- [ ] Brute-force matching with ratio test
- [ ] Essential matrix estimation (8-point algorithm)
- [ ] Camera pose decomposition (R, t from E)
- [ ] Web Worker integration
- [ ] Demo: camera feeds, feature points visualized, basic pose output

**Milestone**: Camera moves → 3D cube stays roughly fixed

### Phase 2: Map Building (2 weeks)
- [ ] Keyframe selection (parallax threshold)
- [ ] DLT triangulation for new map points
- [ ] Map point management (add/remove/merge)
- [ ] PnP tracking against map (EPnP)
- [ ] Local bundle adjustment (last N keyframes)
- [ ] Scale initialization from first two keyframes

**Milestone**: Objects stay fixed with minimal drift for 30+ seconds

### Phase 3: Plane Detection (1-2 weeks)
- [ ] RANSAC plane fitting from map points
- [ ] Floor plane identification (largest horizontal plane)
- [ ] Hit test API (screen tap → 3D plane intersection)
- [ ] Multiple plane tracking
- [ ] Plane visualization in demo

**Milestone**: Tap screen → object placed on detected floor

### Phase 4: Stabilization & Polish (2 weeks)
- [ ] Tracking lost detection + recovery
- [ ] Optional IMU fusion (DeviceMotion API)
- [ ] Performance profiling + SIMD optimization
- [ ] npm package build
- [ ] Three.js integration example
- [ ] Hackar:S integration prototype

**Milestone**: Production-usable in Hackar:S

## Performance Budget

| Operation | Target | Resolution |
|-----------|--------|------------|
| Grayscale conversion | <1ms | 320x240 |
| FAST detection | <3ms | ~500 corners |
| BRIEF descriptors | <2ms | 256-bit × 300 |
| Matching | <3ms | 300 × 300 Hamming |
| Pose estimation | <4ms | RANSAC 100 iterations |
| Triangulation | <2ms | ~50 new points |
| Plane RANSAC | <3ms | 200 iterations |
| **Total per frame** | **<18ms** | **~55fps headroom** |
| Worker overhead | ~2ms | postMessage serialization |
| **Effective budget** | **<20ms** | **50fps → 30fps safe** |

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| iOS Safari no IMU | Visual-only = more drift | High | Aggressive keyframing, lower resolution |
| WASM perf insufficient | <30fps on older phones | Medium | Reduce to 160x120, skip frames |
| Plane detection poor | Can't place objects | Medium | Fallback: manual height setting |
| Scale ambiguity | Objects wrong size | High | Reference object (marker) for scale init |
| Tracking loss | Objects jump/disappear | High | Re-localization from keyframe DB |
| SVD in WASM | Numerical instability | Low | Use well-tested decomposition code |

## Key Algorithms Reference

### FAST-9 Corner Detection
- Test 16 pixels on Bresenham circle (radius 3) around candidate
- If ≥9 contiguous pixels brighter/darker than center ± threshold → corner
- Quick reject: test 4 cardinal points first
- Score with Harris corner response for ranking

### BRIEF Descriptor
- 256 binary comparisons of pixel pairs in patch around keypoint
- Result: 256-bit (32 bytes) binary string
- Matching: Hamming distance (popcount of XOR) — extremely fast

### Essential Matrix → Pose
- E = K2^T × F × K1 (F from 8-point, K = intrinsic matrix)
- SVD of E → 4 possible [R|t] solutions
- Disambiguate by triangulating a point → check positive depth (cheirality)

### EPnP (3D-2D pose)
- Given N 3D map points and their 2D projections
- Express 3D points as weighted sum of 4 control points
- Solve for control point projections → recover pose
- More stable than DLT for pose estimation

### RANSAC Plane Detection
- Sample 3 random 3D points → fit plane (cross product of edges)
- Count inliers within threshold distance
- Repeat N times, keep best plane
- Refit plane using all inliers

## API Design (Draft)

```typescript
import { HackarSlam } from 'hackar-slam'

const slam = new HackarSlam({
  resolution: [320, 240],
  maxKeypoints: 300,
  enableIMU: true,  // optional, if available
})

// Start tracking
await slam.start(videoElement)

// Per-frame update (called from requestAnimationFrame)
slam.onPose((pose: Float32Array) => {
  // 4x4 column-major matrix, directly usable with Three.js
  camera.matrix.fromArray(pose)
  camera.matrixWorldNeedsUpdate = true
})

slam.onPlane((plane: { normal: [number,number,number], distance: number }) => {
  // Place objects on detected plane
})

// Hit test
const hit = slam.hitTest(screenX, screenY)
if (hit) {
  object.position.set(hit.x, hit.y, hit.z)
}

// Cleanup
slam.stop()
```

## Directory Structure

```
hackar-slam/
├── src/
│   ├── core/                  # C++ WASM modules
│   │   ├── fast.cpp           # FAST detection + BRIEF + matching
│   │   ├── pose.cpp           # Essential/PnP + triangulation + plane
│   │   ├── map.cpp            # Keyframe + map point management
│   │   └── ba.cpp             # Local bundle adjustment
│   ├── js/                    # TypeScript modules
│   │   ├── slam.ts            # Public API
│   │   ├── worker.ts          # Web Worker entry
│   │   ├── camera.ts          # getUserMedia + grayscale
│   │   └── three-adapter.ts   # Three.js helpers
│   └── wasm/                  # Build output
│       └── slam.wasm
├── demo/
│   ├── index.html             # Basic tracking demo
│   └── plane.html             # Plane detection demo
├── CMakeLists.txt             # Emscripten build config
├── package.json
├── tsconfig.json
├── ARCHITECTURE.md            # This file
├── LICENSE                    # MIT
└── README.md
```

## References

- stella_vslam (BSD-2) — architecture reference
- Nister, "An Efficient Solution to the Five-Point Relative Pose Problem" (2004)
- Rublee et al., "ORB: An Efficient Alternative to SIFT or SURF" (2011)
- Lepetit et al., "EPnP: An Accurate O(n) Solution to the PnP Problem" (2009)
- Rosten & Drummond, "Machine Learning for High-Speed Corner Detection" (2006)
- JSFeat — JS computer vision reference implementation
