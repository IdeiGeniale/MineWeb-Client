# ⛏ MineWeb

A browser-based voxel sandbox game inspired by Minecraft, built as a single HTML file using [Three.js](https://threejs.org/). No server required — just open and play.

---

## How to use

If you want to use the version without the three.module.js file, you can download the com.ideigeniale.mineweb-client.latest file.

---

## Status

This is a remake of MineWeb 1. It is still in progress, and MineWeb 1 is basically a finished project to me. I will no longer be working on MineWeb 1. I will make make MineWeb 1 open-source and contributions are happily accepted.

---

## Features

- **Procedural world generation** using seeded 3D Perlin noise with 3 distinct biomes
- **18 block types** including ores, transparent blocks, and decorative blocks
- **Hardness-based block breaking** — hold to break, harder blocks take longer
- **Fly mode** for free camera movement
- **Export & import worlds** as `.mcweb` files (MineWeb 2.0 World Format)
- **Full mobile support** with virtual joystick, touch-to-look, and on-screen buttons
- **Seeded worlds** — enter any text seed to generate the same world every time
- **Instanced rendering** with face culling for performance

---

## Getting Started

MineWeb uses ES Modules, so it must be served over HTTP — it will not work when opened directly as a `file://` URL.

### Option 1 — Python (no install needed)
```bash
python -m http.server 8000
```
Then open `http://localhost:8000/mineweb.html` in your browser.

### Option 2 — Node.js
```bash
npx serve .
```

### Option 3 — VS Code
Install the **Live Server** extension, right-click `mineweb.html`, and select *Open with Live Server*.

---

## Three.js Dependency

MineWeb loads Three.js from a local file. Make sure `three.module.js` is in the **same folder** as `mineweb.html`:

```
your-folder/
  mineweb.html
  three.module.js   ← ES module build (not three.min.js)
```

Download the correct file from:
```
https://unpkg.com/three@0.160.0/build/three.module.js
```

> ⚠️ Make sure you download `three.module.js` specifically — the UMD builds (`three.js`, `three.min.js`) will **not** work with the importmap.

---

## Controls

### Desktop

| Key / Action | Function |
|---|---|
| `W A S D` | Move |
| `Space` | Jump |
| `Shift` | Fly down (fly mode only) |
| `F` | Toggle fly mode |
| `Left Click` (hold) | Break block |
| `Right Click` | Place block |
| `1` – `8` | Select hotbar slot |
| `Scroll Wheel` | Cycle hotbar slots |
| `E` | Export world |
| `I` | Import world |
| `Esc` | Pause / resume |

### Mobile

| Control | Function |
|---|---|
| Left joystick | Move |
| Swipe (centre of screen) | Look / rotate camera |
| ✈ Fly | Toggle fly mode |
| ↑ Jump (hold) | Jump / fly up |
| ↓ Down (hold) | Fly down (fly mode only) |
| ⛏ Break | Break targeted block |
| 🧱 Place | Place selected block |
| ◀ ▶ | Cycle hotbar slots |
| 💾 | Export world |

---

## Block Types

| Block | Notes |
|---|---|
| Grass | Surface block in plains biome |
| Dirt | Sub-surface layer under grass |
| Stone | Most common underground block |
| Wood Log | Tree trunk |
| Leaves | Tree canopy (transparent) |
| Sand | Surface block in desert biome |
| Glass | Transparent, placeable |
| Water | Fills areas below sea level (y=14) |
| Bedrock | Indestructible bottom layer |
| Snow | Mountain biome surface |
| Cactus | Spawns in desert biome |
| Planks | Craftable wood block |
| Gravel | Underground fill block |
| Coal Ore | Found below y=18 |
| Iron Ore | Found below y=18 |
| Gold Ore | Found below y=6 |
| Brick | Decorative block |
| Cobblestone | Decorative block |

---

## Biomes

| Biome | Description |
|---|---|
| **Plains** | Flat-ish green terrain with dense tree coverage |
| **Mountains** | Tall rocky peaks with snow caps above a certain height |
| **Desert** | Low sandy terrain dotted with cacti |

Biomes are determined by a secondary Perlin noise layer and blend naturally at their borders.

---

## World Files

Worlds are exported as `.mcweb` files. When opened in a text editor the first line reads:

```
MineWeb 2.0 World Format
```

Followed by a JSON object containing the world seed and all block data. These files can be re-imported from the main menu or the pause menu.

---

## World Size

| Dimension | Size |
|---|---|
| Width (X) | 76 blocks |
| Length (Z) | 76 blocks |
| Height (Y) | 54 blocks |
| Total volume | ~312,000 blocks |

The world radius is defined by `WRAD = 38` in the source. Increasing this value will generate a larger world at the cost of longer load times.

---

## Technical Details

- **Engine:** Three.js r160
- **Rendering:** Instanced meshes with per-face textures (canvas-generated, pixel-art style)
- **Terrain:** Seeded 3D Perlin noise with fractional Brownian motion (fBm)
- **Raycasting:** DDA (Digital Differential Analysis) for precise block targeting
- **Physics:** AABB collision detection with gravity and jump
- **Textures:** Generated at runtime using `CanvasTexture` with pixel noise for a voxel look
- **Max visible instances:** 25,000 blocks per block type
- **Sea level:** y = 14

---

## License

This project can be used freely without licensing or contacting the author for personal and educational purposes.
