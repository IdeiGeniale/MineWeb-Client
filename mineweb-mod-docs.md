# MineWeb Mod API Documentation

> **Version:** 2.6+  
> **Format:** `.mwebmod` (ZIP archive)  
> **Language:** JavaScript (ES2020)

---

## Table of Contents

1. [Overview](#1-overview)
2. [The .mwebmod Format](#2-the-mwebmod-format)
3. [mod.json Reference](#3-modjson-reference)
4. [mod.js & the ModAPI Object](#4-modjs--the-modapi-object)
5. [Registering Blocks](#5-registering-blocks)
6. [Modifying the Hotbar](#6-modifying-the-hotbar)
7. [Event Hooks](#7-event-hooks)
8. [Reading & Writing Blocks](#8-reading--writing-blocks)
9. [Accessing Game State](#9-accessing-game-state)
10. [Bundling Assets](#10-bundling-assets)
11. [Injecting CSS](#11-injecting-css)
12. [Block Definition Reference](#12-block-definition-reference)
13. [Full Example: Ruby Ore Mod](#13-full-example-ruby-ore-mod)
14. [Loading a Mod In-Game](#14-loading-a-mod-in-game)
15. [Tips & Common Mistakes](#15-tips--common-mistakes)

---

## 1. Overview

MineWeb supports mods through `.mwebmod` files — ZIP archives that contain JavaScript, optional JSON metadata, optional CSS, and any bundled assets (images, sounds). When a mod is loaded, its `mod.js` is executed and receives a `ModAPI` object, which is the complete interface to the game.

Mods can:
- Register new block types with custom colours and properties
- Add blocks to the player's hotbar
- React to in-game events (block break, block place, every frame)
- Read and write blocks in the world
- Access the Three.js scene, camera, and player state
- Bundle and use custom assets
- Inject custom CSS into the page

---

## 2. The .mwebmod Format

A `.mwebmod` file is a standard ZIP archive renamed with the `.mwebmod` extension.

```
MyMod.mwebmod  (ZIP)
├── mod.json       ← optional metadata
├── mod.js         ← required: your mod code
├── style.css      ← optional: auto-injected CSS
└── assets/
    └── texture.png  ← optional: any bundled files
```

### Rules

- `mod.js` is **required**. The mod will not load without it.
- `mod.json` is optional. If absent, the mod name defaults to the filename.
- All `.css` files are automatically injected into `<head>` on load.
- All other files are stored as object URLs accessible via `ModAPI.getAsset()`.
- You can include as many asset files as you want in any folder structure.

### Creating the archive

**On Windows:** Select all files → right-click → Send to → Compressed folder → rename `.zip` to `.mwebmod`.  
**On macOS:** Select all files → right-click → Compress → rename `.zip` to `.mwebmod`.  
**With a tool:**
```bash
zip -r MyMod.mwebmod mod.json mod.js style.css assets/
```

> ⚠️ **Important:** Do not zip the *folder* — zip the *contents*. `mod.js` must be at the root of the archive, not inside a subfolder.

---

## 3. mod.json Reference

```json
{
  "name": "My Mod",
  "version": "1.0.0",
  "description": "A short description of what this mod does."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | No | Display name shown in the mod list. Defaults to filename. |
| `version` | string | No | Version string. Defaults to `"1.0"`. |
| `description` | string | No | Short description shown in the mod list. |

---

## 4. mod.js & the ModAPI Object

Your `mod.js` is executed as:

```js
new Function('ModAPI', yourCode)(modAPIInstance);
```

This means:
- Your code runs in a **function scope** — no `import`/`export`.
- `ModAPI` is your only entry point to the game.
- `window` is accessible for global browser APIs (e.g. `window.MineWebTHREE`).
- `return` at the top level exits the mod early (useful for error handling).

### Minimal mod.js

```js
ModAPI.log('Hello from my mod!');
```

### Accessing Three.js

Three.js is exposed globally as `window.MineWebTHREE`:

```js
const THREE = window.MineWebTHREE;
const geometry = new THREE.BoxGeometry(1, 1, 1);
```

---

## 5. Registering Blocks

### `ModAPI.registerBlock(id, def, displayName)`

Registers a new block type and makes it available to place in the world.

| Parameter | Type | Description |
|---|---|---|
| `id` | string \| number | A string ID (e.g. `'ruby_ore'`) auto-assigns a numeric ID ≥ 100. Pass a number to use a specific ID. |
| `def` | object | Block definition — see [Block Definition Reference](#12-block-definition-reference). |
| `displayName` | string | Name shown in the hotbar tooltip. |

**Returns:** The numeric block ID assigned to this block.

```js
const RUBY = ModAPI.registerBlock('ruby_ore', {
  top:  0xcc2244,
  side: 0xaa1133,
  hard: 3,
}, 'Ruby Ore');

ModAPI.log('Ruby Ore registered with ID:', RUBY);
```

> ℹ️ Numeric IDs 0–99 are reserved for vanilla MineWeb blocks. Mod blocks start at 100 and are assigned sequentially unless you pass an explicit number.

---

## 6. Modifying the Hotbar

### `ModAPI.addToHotbar(blockId)`

Appends a block to the end of the hotbar if it isn't already there. Rebuilds the hotbar UI automatically.

```js
ModAPI.addToHotbar(RUBY);
```

### `ModAPI.buildHotbar()`

Forces a rebuild of the hotbar UI. Call this after manually modifying `ModAPI.HOTBAR`.

```js
// Manually add a block and rebuild
ModAPI.HOTBAR.push(RUBY);
ModAPI.buildHotbar();
```

### `ModAPI.HOTBAR`

Direct reference to the hotbar array. You can read, push, splice, or clear it. Always call `ModAPI.buildHotbar()` after any manual changes.

```js
// Empty the hotbar (survival-style start)
ModAPI.HOTBAR.length = 0;
ModAPI.buildHotbar();
```

---

## 7. Event Hooks

### `ModAPI.on(event, callback)`

Registers a callback for a game event. Multiple callbacks can be registered for the same event — all will fire.

| Event | Callback signature | Description |
|---|---|---|
| `'blockBreak'` | `(id, x, y, z)` | Fires after a block is broken. `id` is the block that was removed. |
| `'blockPlace'` | `(id, x, y, z)` | Fires after a block is placed. `id` is the block that was placed. |
| `'tick'` | `(dt)` | Fires every animation frame while the game is active and unpaused. `dt` is delta time in seconds. |
| `'worldGen'` | *(reserved)* | Reserved for future use. |

```js
// React to block breaks
ModAPI.on('blockBreak', (id, x, y, z) => {
  if (id === RUBY) {
    ModAPI.log(`Ruby mined at (${x}, ${y}, ${z})!`);
  }
});

// React to block placement
ModAPI.on('blockPlace', (id, x, y, z) => {
  ModAPI.log(`Block ${ModAPI.BNAMES[id]} placed at (${x}, ${y}, ${z})`);
});

// Per-frame logic
let elapsed = 0;
ModAPI.on('tick', (dt) => {
  elapsed += dt;
  if (elapsed > 5) {
    elapsed = 0;
    ModAPI.log('5 seconds have passed in-game.');
  }
});
```

> ⚠️ Errors thrown inside hook callbacks are caught and logged as warnings — they won't crash the game, but they will silently skip the rest of your callback. Wrap risky code in `try/catch` for better error visibility.

---

## 8. Reading & Writing Blocks

### `ModAPI.getBlock(x, y, z)`

Returns the numeric block ID at world coordinates `(x, y, z)`. Returns `0` (AIR) if the position is empty or outside the world.

```js
const id = ModAPI.getBlock(10, 48, 10);
ModAPI.log('Block at (10,48,10):', ModAPI.BNAMES[id] || id);
```

### `ModAPI.setBlock(x, y, z, id)`

Places a block at world coordinates. Use `0` (or `ModAPI.B.AIR`) to remove a block.

```js
// Place a Ruby Ore block
ModAPI.setBlock(0, 50, 0, RUBY);

// Remove a block
ModAPI.setBlock(0, 50, 0, ModAPI.B.AIR);
```

> ⚠️ `setBlock` updates both the world map and the rendered mesh. It is safe to call from hook callbacks or tick events, but avoid calling it excessively every frame as it triggers mesh rebuilds.

---

## 9. Accessing Game State

All game state properties on `ModAPI` are **live getters** — they always reflect the current value.

### Block Registry

| Property | Type | Description |
|---|---|---|
| `ModAPI.B` | object | Map of block name → numeric ID. e.g. `ModAPI.B.GRASS === 1`. |
| `ModAPI.BDEF` | object | Map of numeric ID → block definition object. |
| `ModAPI.BNAMES` | object | Map of numeric ID → display name string. |

```js
const B = ModAPI.B;
ModAPI.log(B.GRASS, B.STONE, B.WATER); // 1, 3, 8

// Check if a broken block was stone
ModAPI.on('blockBreak', (id) => {
  if (id === B.STONE) ModAPI.log('You broke stone!');
});
```

### World

| Property | Type | Description |
|---|---|---|
| `ModAPI.wmap` | `Map<string, number>` | The world block map. Keys are `"x,y,z"` strings. Read-only — use `setBlock` to modify. |

```js
// Count all torches in the world
let count = 0;
for (const [key, id] of ModAPI.wmap) {
  if (id === ModAPI.B.TORCH) count++;
}
ModAPI.log('Torches in world:', count);
```

### Player & Camera

| Property | Type | Description |
|---|---|---|
| `ModAPI.player` | object | Player state. Has `.pos` (Vector3), `.vel` (Vector3), `.flying` (bool), `.onGround` (bool). |
| `ModAPI.camera` | `THREE.Camera` | The active Three.js camera. |

```js
const pos = ModAPI.player.pos;
ModAPI.log(`Player is at (${pos.x.toFixed(1)}, ${pos.y.toFixed(1)}, ${pos.z.toFixed(1)})`);

// Teleport the player
ModAPI.player.pos.set(0, 60, 0);
ModAPI.player.vel.set(0, 0, 0);
```

### Input & Interaction

| Property | Type | Description |
|---|---|---|
| `ModAPI.inp` | object | Current keyboard input state. Keys: `w`, `s`, `a`, `d`, `sp` (space), `sh` (shift). Value is `1` (held) or `0`. |
| `ModAPI.slot` | number | Currently selected hotbar slot index. |
| `ModAPI.highlighted` | object \| null | The block currently targeted by the crosshair. Has `.block` (id), `.x`, `.y`, `.z`, `.face` (Vector3). `null` if nothing is targeted. |

```js
ModAPI.on('tick', (dt) => {
  if (ModAPI.inp.w) {
    ModAPI.log('Player is moving forward');
  }
});
```

### Scene

| Property | Type | Description |
|---|---|---|
| `ModAPI.scene` | `THREE.Scene` | The Three.js scene. Add or remove objects freely. |

```js
const THREE = window.MineWebTHREE;
const light = new THREE.PointLight(0xff0000, 2, 20);
light.position.set(0, 55, 0);
ModAPI.scene.add(light);
```

---

## 10. Bundling Assets

Any non-`.js`, non-`.css`, non-`mod.json` file in your archive is stored as a blob URL accessible via `ModAPI.getAsset()`.

### `ModAPI.getAsset(filename)`

Returns the blob object URL string for a bundled file, or `null` if not found.

```js
const texUrl = ModAPI.getAsset('assets/ruby_texture.png');
if (texUrl) {
  const loader = new (window.MineWebTHREE.TextureLoader)();
  loader.load(texUrl, texture => {
    // use texture
  });
}
```

File paths are matched against the full path as stored in the ZIP. If your asset is at `assets/ruby.png` in the archive, call `ModAPI.getAsset('assets/ruby.png')`.

---

## 11. Injecting CSS

Any `.css` file in the archive root (or any subfolder) is automatically injected as a `<style>` tag when the mod loads. This lets you style custom UI elements your mod adds to the DOM.

**style.css** (inside your `.mwebmod`):
```css
#my-mod-overlay {
  position: fixed;
  top: 10px;
  right: 10px;
  background: rgba(0,0,0,.7);
  color: #fff;
  padding: 8px 14px;
  border-radius: 6px;
  font-size: 13px;
  z-index: 5000;
}
```

**mod.js:**
```js
const el = document.createElement('div');
el.id = 'my-mod-overlay';
el.textContent = 'My Mod Active';
document.body.appendChild(el);
```

The style tag is given a `data-mod` attribute set to your mod's name, so you can target or remove it later if needed.

---

## 12. Block Definition Reference

A block definition object describes how a block looks and behaves.

| Field | Type | Default | Description |
|---|---|---|---|
| `top` | number (hex color) | required | Colour of the top face. |
| `side` | number (hex color) | `top` | Colour of the side faces. Defaults to `top` if omitted. |
| `bot` | number (hex color) | `top` | Colour of the bottom face. Defaults to `top` if omitted. |
| `hard` | number | `1` | Hardness — how long the block takes to break. `999` = unbreakable. |
| `tr` | boolean | `false` | Transparent — if `true`, adjacent faces are rendered (used for water, glass). |
| `emit` | number | `0` | Light emission level. `1` makes the block glow like a torch. |

```js
// Solid opaque block
{ top: 0xcc2244, side: 0xaa1133, bot: 0x881122, hard: 3 }

// Transparent block (like glass)
{ top: 0x88ccff, side: 0x88ccff, hard: 0.5, tr: true }

// Glowing block (like a torch)
{ top: 0xffaa00, side: 0xff8800, hard: 0.1, emit: 1 }

// Unbreakable block
{ top: 0x333333, side: 0x222222, hard: 999 }
```

---

## 13. Full Example: Ruby Ore Mod

This example registers a Ruby Ore block, adds it to the hotbar, and gives a toast notification when mined.

### File structure

```
RubyOre.mwebmod (ZIP)
├── mod.json
└── mod.js
```

### mod.json

```json
{
  "name": "Ruby Ore",
  "version": "1.0.0",
  "description": "Adds a rare ruby ore block to MineWeb."
}
```

### mod.js

```js
// ── Ruby Ore Mod for MineWeb ──────────────────
// Requires MineWeb v2.5.1+

// Register the block
const RUBY = ModAPI.registerBlock('ruby_ore', {
  top:  0xcc2244,
  side: 0xaa1133,
  bot:  0x881122,
  hard: 3,
}, 'Ruby Ore');

// Add to hotbar
ModAPI.addToHotbar(RUBY);

// Toast helper (simple DOM notification)
function toast(msg) {
  let el = document.getElementById('ruby-toast');
  if (!el) {
    el = document.createElement('div');
    el.id = 'ruby-toast';
    el.style.cssText = 'position:fixed;top:52px;left:50%;transform:translateX(-50%);' +
      'background:rgba(180,20,40,.9);color:#fff;padding:6px 18px;border-radius:6px;' +
      'font-size:13px;pointer-events:none;z-index:9999;transition:opacity .4s;';
    document.body.appendChild(el);
  }
  el.textContent = msg;
  el.style.opacity = '1';
  clearTimeout(el._t);
  el._t = setTimeout(() => el.style.opacity = '0', 2800);
}

// React to block break
ModAPI.on('blockBreak', (id, x, y, z) => {
  if (id === RUBY) {
    toast(`💎 Ruby Ore mined at (${x}, ${y}, ${z})!`);
  }
});

ModAPI.log('Ruby Ore mod loaded successfully.');
```

---

## 14. Loading a Mod In-Game

1. Open MineWeb in your browser.
2. On the **main menu**, click **🧩 Mods**.
3. Click **📦 Load .mwebmod file** and select your `.mwebmod` file.
4. The mod loads immediately — its name and version appear in the mod list.
5. Click **← Back** to return to the main menu, then start your game.

Mods persist for the duration of the browser session. Refreshing the page unloads all mods.

> ℹ️ Multiple mods can be loaded at once. Load them all before starting your game.

---

## 15. Tips & Common Mistakes

### ✅ Do

- Call `ModAPI.buildHotbar()` after any manual changes to `ModAPI.HOTBAR`.
- Wrap DOM manipulation in null checks — some elements only exist once the game starts.
- Use `ModAPI.log()` for debugging instead of `console.log()` directly — it prefixes your mod name.
- Use `return` at the top level to exit early if a required dependency is missing.
- Use block IDs ≥ 100 for custom blocks (auto-assigned when you pass a string to `registerBlock`).

### ❌ Don't

- Don't call `setBlock` inside a `tick` hook on every frame — it rebuilds meshes and will tank performance.
- Don't use `import` or `export` — your mod runs in a function scope, not an ES module.
- Don't zip the *containing folder* — only zip the *contents* (so `mod.js` is at the ZIP root).
- Don't rely on block IDs below 100 — those are reserved for vanilla blocks and may change.
- Don't add `<script>` tags from your mod CSS — only `mod.js` is executed.

### Checking if survival mode is active

```js
// Access the svOn variable via a tick hook side effect
// (svOn is a game-internal variable, not exposed on ModAPI)
// Instead, check indirectly — e.g. if hotbar has empty slots:
const isLikelySurvival = ModAPI.HOTBAR.includes(0);
```

### Accessing the Three.js library

```js
const THREE = window.MineWebTHREE;
// THREE is r160 — all standard Three.js classes are available
```

### Reacting only when in-game (not paused)

The `tick` hook only fires when the game is active and unpaused, so no additional check is needed for that. For DOM events (e.g. `keydown`), you may want to guard:

```js
document.addEventListener('keydown', e => {
  // ModAPI.player.pos is available once the game has started
  if (!ModAPI.player || !ModAPI.scene) return;
  if (e.code === 'KeyZ') {
    ModAPI.log('Z pressed while in game');
  }
});
```

---

*MineWeb is made by IdeiGeniale. Mod API available from v2.6.*
