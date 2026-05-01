---
title: Meshalyzer shader fix for Intel Haswell / older Mesa drivers
status: validated   # patch applied locally on cdc-ThinkPad-X240, vertex picking confirmed working
use_when: meshalyzer fails to compile shaders, prints `ERROR::SHADER::VERTEX::COMPILATION_FAILED`, or vertex picking does not select nodes — Intel Haswell or WSL
last_updated: 2026-04-28
---

# Meshalyzer shader fix for Intel Haswell / older Mesa drivers

Documents the patch applied on `cdc-ThinkPad-X240` (Ubuntu 24.04, Intel HD Graphics 4400) so it can be replicated on the Windows WSL setup that has the same symptoms.

## Symptoms

When running `/usr/local/meshalyzer/meshalyzer <mesh> <data.igb>`:

1. Terminal prints repeated shader errors:
   ```
   ERROR::SHADER::VERTEX::COMPILATION_FAILED
   0:2(12): warning: extension `GL_EXT_clip_cull_distance' unsupported in vertex shader
   0:5(7): error: redeclaration cannot change qualification of `gl_ClipDistance'
   ERROR::SHADER::PROGRAM::LINKING_FAILED
   error: linking with uncompiled/unspecialized shader
   ```
   (Other variants: `0:1(10): error: GLSL 4.10 is not supported`, `0:39(2): error: unsized array index must be constant`.)

2. Mesh and vm.igb render fine, animation plays — but **clicking on the mesh never selects a vertex**. Hovering only highlights surface fill.

## Root cause

Two independent bugs:

| # | Bug | Where |
|---|---|---|
| A | Shaders declare `#version 410 core` but the driver only supports up to GLSL 4.00 (Intel Haswell maxes at OpenGL 4.0). All version-410 shaders fail to compile. | All visual + picking shaders |
| B | The **picking shader** (separate render pass that encodes vertex IDs as colors so the click handler can decode "which vertex did the user click") declares `float gl_ClipDistance[6];` *without* the `out` qualifier. The built-in `gl_ClipDistance` is implicitly `out`, so Mesa correctly rejects this as `redeclaration cannot change qualification`. This is a meshalyzer source bug, not a driver issue. | `src/select_util.h::get_select_shader()` in meshalyzer 6.1 |

Bug A makes everything fail spectacularly. Bug B is the reason picking *specifically* still doesn't work even after Bug A is fixed: rendering uses a different shader pipeline than picking, so visual rendering can succeed while picking silently fails.

## Fix overview

The shaders are GLSL string literals baked into the meshalyzer binary. Patch the binary in-place, preserving byte length so other offsets stay valid.

| # | Original string (bytes) | Replacement | Count | Reason |
|---|---|---|---|---|
| 1 | `#version 410 core` (17) | `#version 400 core` | 11 | Driver only supports GLSL ≤4.00 |
| 2 | `#extension GL_EXT_clip_cull_distance : enable\n` (46) | `<spaces>\n` | 1 | Extension unsupported on Mesa Intel |
| 3 | `float gl_ClipDistance[6];\n` (26) | `<spaces>\n` | 1 | Removes the no-`out` redeclaration that triggers the qualifier error |
| 4 | `  for( int i=0; i<6; i++ )\n    gl_ClipDistance[i] = dot(vec4(aPos,1.0), clipEqn[i]);\n` (84) | `<spaces>` | 1 | The for-loop body would error with "unsized array index must be constant" once #3 is removed |

Trade-off from patches 3 + 4: the picking shader no longer respects user-defined clip planes. For normal use you'll never notice; only edge case is "click a vertex that's currently clipped away" — picking might select a vertex you can't see.

## Step-by-step (works on any Linux x86_64 with the v6.1 AppImage)

### 1. Install meshalyzer

```bash
sudo mkdir -p /usr/local/meshalyzer
sudo curl -L "https://git.opencarp.org/api/v4/projects/39/packages/generic/meshalyzer-appimage/6.1/Meshalyzer-6.1-x86_64.AppImage" \
     -o /usr/local/meshalyzer/meshalyzer.appimage
sudo chmod +x /usr/local/meshalyzer/meshalyzer.appimage
```

### 2. Extract the AppImage

The AppImage's binary is read-only inside the squashfs, so we extract it.

```bash
cd /tmp
/usr/local/meshalyzer/meshalyzer.appimage --appimage-extract
sudo cp -r /tmp/squashfs-root /usr/local/meshalyzer/extracted
sudo ln -sf /usr/local/meshalyzer/extracted/AppRun /usr/local/meshalyzer/meshalyzer
```

After this, running `/usr/local/meshalyzer/meshalyzer` invokes the extracted binary.

### 3. Apply the binary patches

```bash
python3 - <<'PY'
target = '/usr/local/meshalyzer/extracted/usr/bin/meshalyzer'
out    = '/tmp/meshalyzer-patched'
data   = bytearray(open(target, 'rb').read())

# Patch 1: downgrade GLSL 4.10 -> 4.00 in all shaders
old, new = b'#version 410 core', b'#version 400 core'
n = 0
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    data[p:p+len(old)] = new
    n += 1
    i = p + 1
print(f'#version 410 -> 400: {n} replacements')

# Patch 2: remove unsupported clip_cull_distance extension
old = b'#extension GL_EXT_clip_cull_distance : enable\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Removed clip_cull_distance extension at offset {p}')

# Patch 3: blank out the no-qualifier gl_ClipDistance redeclaration
# (only the one without preceding 'out ')
old = b'float gl_ClipDistance[6];\n'
new = b' ' * len(old)
i = 0
while True:
    p = data.find(old, i)
    if p == -1: break
    if data[max(0, p-4):p] != b'out ':
        data[p:p+len(old)] = new
        print(f'Wiped no-out gl_ClipDistance redeclaration at {p}')
    i = p + 1

# Patch 4: blank out the for-loop using gl_ClipDistance[i]
old = b'  for( int i=0; i<6; i++ )\n    gl_ClipDistance[i] = dot(vec4(aPos,1.0), clipEqn[i]);\n'
new = b' ' * len(old)
p = data.find(old)
if p != -1:
    data[p:p+len(old)] = new
    print(f'Wiped picking for-loop at {p}')

open(out, 'wb').write(data)
print('Wrote', out)
PY
sudo cp /tmp/meshalyzer-patched /usr/local/meshalyzer/extracted/usr/bin/meshalyzer
sudo chmod +x /usr/local/meshalyzer/extracted/usr/bin/meshalyzer
```

Expected output:
```
#version 410 -> 400: 11 replacements
Removed clip_cull_distance extension at offset 2128674
Wiped no-out gl_ClipDistance redeclaration at 2128796
Wiped picking for-loop at 2129102
Wrote /tmp/meshalyzer-patched
```

(Offsets will differ slightly by build, but counts should match: 11 + 1 + 1 + 1.)

### 4. Verify

```bash
cd ~/tutorials/02_EP_tissue/00_simple
/usr/local/meshalyzer/meshalyzer meshes/*/block.pts 2*/vm.igb simple.mshz
```

- No `ERROR::SHADER::*` lines should appear in the terminal.
- Clicking on the mesh should select individual vertices.

## WSL-specific notes

- WSL2 with WSLg uses an OpenGL stack that delegates to the Windows GPU via D3D12 (Mesa+Zink) or to Mesa software (llvmpipe). The `#version 410 core` failure has been reported on both paths, so the same patches apply.
- AppImages need FUSE in WSL. If the AppImage refuses to mount, use `--appimage-extract` (step 2) and run `/usr/local/meshalyzer/extracted/AppRun` directly. The extracted approach is what we use here anyway, so this isn't extra work.
- If on WSL you still see GLSL 4.10 errors after patching, double-check the count says `11 replacements`. If it says fewer, the build version differs and there are remaining `#version 410 core` strings — re-run the Python with that string repeated until count is 0.

## Reverting

```bash
sudo rm -rf /usr/local/meshalyzer/extracted /usr/local/meshalyzer/meshalyzer
sudo mv /usr/local/meshalyzer/meshalyzer.appimage /usr/local/meshalyzer/meshalyzer
sudo chmod +x /usr/local/meshalyzer/meshalyzer
```

## Upstream report (recommended)

The picking-shader bug (#B above) is a real meshalyzer issue and should be reported at <https://git.opencarp.org/openCARP/meshalyzer/-/issues>. Suggested patch in `src/select_util.h`:

```diff
-    "float gl_ClipDistance[6];\n"
+    "out float gl_ClipDistance[6];\n"
```

That alone would let the picking shader compile on any GLSL 4.00+ driver. The `#version 410` issue is a separate compatibility decision (some drivers cap at 4.00) and would warrant a CMake feature check or a `#version 400` baseline.
