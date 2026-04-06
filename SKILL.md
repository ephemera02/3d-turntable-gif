---
name: 3d-turntable-gif
description: Render spinning turntable GIF animations from 3D model files (.obj, .glb, .gltf, .fbx) using headless Three.js and Puppeteer. Use this skill whenever the user asks to create a spinning GIF, turntable animation, product spin, rotating 3D render, or any animated preview from a 3D model file. Also trigger when the user uploads a 3D model and asks for a GIF, animation, spin, or render of it. Covers OBJ+MTL, GLTF/GLB, and FBX formats with texture support, transparent backgrounds, and configurable frame count, speed, lighting, and materials. If the user mentions "spinning", "turntable", "rotate", "3D GIF", "product animation", or "model render" alongside a 3D file, use this skill.
---

# 3D Turntable GIF Renderer

Renders smooth spinning GIF animations from 3D model files using headless Chrome + Three.js + Puppeteer + ImageMagick.

## Pipeline Overview

1. **Extract and inspect** the model files (format, textures, dimensions)
2. **Set up a local HTTP server** serving the model, textures, and Three.js modules
3. **Create an HTML renderer** with Three.js (format-specific loader, lighting, materials, camera)
4. **Capture frames** via Puppeteer headless Chrome screenshots with transparent background
5. **Encode GIF** with ImageMagick `convert`

## Before You Start

Read `references/setup.md` for dependency installation and Three.js module setup.

## Format-Specific Handling

Read `references/formats.md` for loader setup per format (OBJ+MTL, GLTF/GLB, FBX).

## Step-by-Step Workflow

### 1. Inspect the Model

```bash
# Check zip contents
unzip -l model.zip

# For OBJ: check material file and dimensions
cat model.mtl
python3 -c "
mins=[float('inf')]*3; maxs=[float('-inf')]*3
with open('model.obj') as f:
    for l in f:
        if l.startswith('v '):
            p=l.split()
            for i in range(3): v=float(p[i+1]); mins[i]=min(mins[i],v); maxs[i]=max(maxs[i],v)
for a,n,x in zip('XYZ',mins,maxs): print(f'{a}: {n:.2f} to {x:.2f} (size: {x-n:.2f})')
"
```

### 2. Set Up Three.js Modules Locally

CDN imports in headless Chrome cause navigation timeouts. **Always download Three.js modules locally.**

```bash
mkdir -p js utils libs curves
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js" -o js/three.module.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/GLTFLoader.js" -o js/GLTFLoader.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/OBJLoader.js" -o js/OBJLoader.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/MTLLoader.js" -o js/MTLLoader.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/utils/BufferGeometryUtils.js" -o utils/BufferGeometryUtils.js
# FBX only:
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/FBXLoader.js" -o js/FBXLoader.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/libs/fflate.module.js" -o libs/fflate.module.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/curves/NURBSCurve.js" -o curves/NURBSCurve.js
curl -sL "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/curves/NURBSUtils.js" -o curves/NURBSUtils.js
```

Use a local import map in the HTML:
```html
<script type="importmap">
{ "imports": { "three": "./js/three.module.js" } }
</script>
```

### 3. Create render.html

See `references/formats.md` for format-specific loader code. Key principles:

- **Renderer**: `WebGLRenderer` with `antialias: true, alpha: true, preserveDrawingBuffer: true`
- **Tone mapping**: `ACESFilmicToneMapping`, exposure 1.3-1.4
- **Clear color**: `renderer.setClearColor(0x000000, 0)` for transparency
- **Lighting**: Studio setup with 4-5 lights (ambient, key, fill, rim, top)
- **Environment map**: Create a `WebGLCubeRenderTarget` with `CubeCamera` for reflections
- **Camera**: `PerspectiveCamera` FOV 28-30, distance `maxDim * 2.0-2.5`
- **Pivot group**: Wrap model in a `THREE.Group` for clean Y-axis rotation
- **Center model**: Compute bounding box, subtract center from model position
- **Window API**: Expose `window._ready` boolean and `window._renderFrame(angle)` function

### 4. Create capture.js

See `references/capture-template.md` for the full Node.js capture script template.

Critical settings:
- **Chrome path**: `/home/claude/.cache/puppeteer/chrome/linux-131.0.6778.204/chrome-linux64/chrome`
- **Chrome args**: `--no-sandbox --disable-setuid-sandbox --enable-unsafe-swiftshader --enable-webgl`
- **Navigation**: `waitUntil: 'load'` (NOT `networkidle0`, it times out)
- **Warm-up**: Render 10 throwaway frames before capture to stabilize WebGL context
- **Settle time**: 1000ms after warm-up before starting capture
- **Frame delay**: 60ms between frames during capture
- **Console logging**: Always attach `page.on('console')` and `page.on('pageerror')` for debugging
- **MIME types**: Include `.obj: 'text/plain'`, `.mtl: 'text/plain'`, `.gltf: 'application/json'`, `.bin: 'application/octet-stream'`

### 5. Encode GIF

```bash
convert -delay DELAY -loop 0 -dispose Background -alpha set frames/frame_*.png output.gif
```

Delay values (in 1/100ths of a second):
- Fast spin: `-delay 3` (30ms/frame)
- Medium spin: `-delay 5` (50ms/frame, good for 120 frames = 6 second rotation)
- Slow spin: `-delay 8` (80ms/frame)

**Do NOT use `-layers Optimize`** for quality renders; it degrades transparency.

## Common Gotchas

1. **FBX version 6100**: Three.js, Blender, and Assimp cannot read FBX files before version 7000. Use `fbx2gltf` (Facebook's converter) to convert to GLB first: `npm install -g fbx2gltf`, then use the Linux binary at `~/.npm-global/lib/node_modules/fbx2gltf/bin/Linux/FBX2glTF`
2. **MTL backslash paths**: Windows-exported MTL files use `\\` in texture paths. Fix with `sed -i 's|textures\\\\|textures/|g' model.mtl`
3. **Multi-material OBJ**: When an OBJ has multiple `usemtl` directives, OBJLoader creates a mesh with `Array.isArray(child.material) === true`. Handle both single and array cases.
4. **WebGL context loss**: Headless Chrome + SwiftShader drops the WebGL context during model loading. The warm-up phase after `_ready` fixes this. First few frames may be blank without it.
5. **CDN import maps**: Never use CDN URLs in import maps for headless Chrome; they cause navigation timeouts. Always download locally.
6. **Texture UV issues**: MTLLoader sometimes creates invalid UV channel references causing shader errors. When this happens, load textures manually with `TextureLoader` and apply as `MeshStandardMaterial` with `color` property instead.
7. **Port conflicts**: Use a different port each render attempt to avoid "address in use" errors.
8. **Puppeteer module location**: If running from a different directory, symlink `node_modules` from where `puppeteer-core` was installed.

## Quality Settings

| Quality | Frames | Delay | Size | Rotation Time |
|---------|--------|-------|------|---------------|
| Preview | 48 | 3 | ~1 MB | ~1.5s |
| Standard | 60 | 5 | ~1.5 MB | 3s |
| High | 120 | 5 | ~2.5 MB | 6s |
| Cinematic | 120 | 8 | ~2.5 MB | 10s |

## Material Presets

- **Metallic product** (iPod, hardware): `roughness: 0.15-0.2, metalness: 0.75-0.85, envMapIntensity: 0.5-0.7`
- **Matte/stylized** (Bitcoin coin): Keep GLTF materials, add `envMap` with `envMapIntensity: 0.8`
- **Glass/translucent** (PayPal logo): Use `MeshPhysicalMaterial` with `transmission: 0.6-0.7, thickness: 1.5-2.0, ior: 1.5, transparent: true, opacity: 0.75-0.85`
- **Clean white base**: `roughness: 0.2-0.3, metalness: 0.05, envMapIntensity: 0.3`
