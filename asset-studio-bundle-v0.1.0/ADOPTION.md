# Adopting `@laststand/asset-studio` in another game

This package is engine-agnostic. The studio itself is a Node HTTP server +
browser UI. For Three.js games, also install `@laststand/asset-studio-three`
to consume the studio's `tune.json` output (bone offsets + prop poses).

## 1. Install

```bash
npm install -D ./path/to/laststand-asset-studio-0.1.0.tgz
npm install   ./path/to/laststand-asset-studio-three-0.1.0.tgz   # Three.js games only
```

Or, with a workspace setup, reference the local paths:

```json
// package.json (your game)
{
  "devDependencies": {
    "@laststand/asset-studio": "file:../packages/asset-studio"
  },
  "dependencies": {
    "@laststand/asset-studio-three": "file:../packages/asset-studio-three"
  }
}
```

## 2. Create `asset-studio.config.json`

Drop this at the root of your game project (same dir as `package.json`):

```json
{
  "$schema": "./node_modules/@laststand/asset-studio/config.schema.json",
  "title": "My Game",
  "assetsDir": "./public/assets",
  "lintTargets": ["src/game/main.js"],
  "slots": [
    { "id": "char_hero", "label": "Hero", "kind": "glb",
      "fills": ["hero.glb"], "criticality": "blocker" }
  ]
}
```

A copy of this template ships at `node_modules/@laststand/asset-studio/templates/asset-studio.config.json` for reference.

| Field | Purpose |
|---|---|
| `assetsDir` | Where the studio reads/writes asset files. Vite users typically point at `./public/assets`. |
| `lintTargets` | Source files the boot-time drift-lint scans for asset name string literals — flags slots whose `fills[]` aren't referenced anywhere. |
| `slots[]` | Game requirements. Each `id` is stable; the studio derives status from disk (🟢 polished if fill exists, 🟡 placeholder if `usedInGame` is `proc`/`vfx`, 🔴 empty otherwise). |

## 3. Add the npm script

```json
// package.json
{ "scripts": { "asset-studio": "asset-studio" } }
```

Then `npm run asset-studio` boots the server at `http://localhost:3010`. The
`asset-studio` binary is provided by the package — equivalent to running
`node --watch node_modules/@laststand/asset-studio/asset-server.mjs`.

**Flags:**

```bash
asset-studio --project=./   # project root (default: cwd)
asset-studio --config=./other.json   # alt config path
asset-studio --assets=./elsewhere    # override assetsDir
asset-studio --port=3010
asset-studio --no-watch              # disable auto-restart on file change
```

## 4. Wire the Three.js runtime adapter (optional)

In your scene code:

```js
import * as THREE from 'three';
import { createTuneLoader } from '@laststand/asset-studio-three';

// After your character GLB loads…
const tune = createTuneLoader({
  characterAsset: 'hero.glb',
  isDev: import.meta.env?.DEV,
  onReload: () => {
    // re-apply prop poses if relevant
    tune.applyPropPose(weapon, 'weapon.glb', armatureRoot, rightHandBone);
  },
});
await tune.load();
tune.subscribe();      // hot-reload via studio SSE in dev

// Each frame, AFTER the AnimationMixer updates:
tune.applyBoneOffsets(armatureRoot, activeClipName);
```

`tune.json` is read from `assetsDir/<character>.tune.json` (served by Vite from `public/assets/` in your game) at boot. In dev, the loader also opens `EventSource('http://localhost:3010/api/events')`; when the Rig Tuner saves, the game refetches and re-applies without a reload.

## 5. Per-project state

The studio writes runtime state (`queue.json`, `history.json`) to `<project>/.asset-studio/`. Add this to `.gitignore` if you don't want to commit it (some teams do commit history for shared lineage):

```
.asset-studio/cache/
```

The `assets/.history/` archive (versioned snapshots of each generation) lives inside `assetsDir`.

## 6. Optional: Layer.ai integration

The studio's `Generate` buttons assemble Layer.ai prompts based on each slot's `note` field. To actually generate, you need to enqueue tasks via the studio UI and process them with a Claude Code session (see `CLAUDE_PROCESSING_GUIDE.md` in the package).

## What you DON'T need to set up

- Three.js CDN — the studio vendors three locally at `/vendor/three/*`. Works offline, no ad-blocker conflicts.
- Wildlife Asset Hub — orthogonal. If you want it, drop a `.mcp.json` in your project (see `game/CLAUDE.md` if migrating from the Last Stand template).
- CI pipeline — none required. The studio is a local dev tool.

## Verifying the setup

```bash
npm run asset-studio
# Studio boots at http://localhost:3010
# Boot banner shows: title, project, assets path, config path, slot counts.
```

If you see `[config] no asset-studio.config.json — using defaults`, the studio couldn't find your config — check the file is at the project root and that you're running the command from there.

If lint warnings like `[lint] slot "X" resolves to Y.glb but no lintTarget references any of its fills[]` show up, either (a) your game code does reference it but via a non-standard pattern (open an issue), or (b) the slot isn't actually used yet.
