# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An HTML5 canvas arcade game — "Freddy & Michi: Espíritu del Amazonas" — a top-down round-based wave survival brawler with 1- or 2-player local co-op and online co-op over WebRTC. All markup, CSS, and JavaScript logic live in `index.html` (~0.2MB, ~3500 lines of readable code). Every image and audio asset is an external binary file under `assets/` (`img_*.png`, `img_*.jpg`, `snd_*.mp3`), referenced by relative path; the JS loads them into `Image`/`Audio` objects. There is no build system, package manager, framework, or test suite.

> Assets were previously inlined as base64 `data:` URIs (making `index.html` ~23MB); they have since been externalized into `assets/`. The file names are sequential (not semantic) — to find which asset a sprite/sound uses, grep `index.html` for the variable assignment, not the filename.

## Running / developing

- **Run it**: open `index.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server`) and open the page. Online multiplayer needs the page served over a real origin (PeerJS signaling) and internet access to load `peerjs` from unpkg.
- **No build, lint, or test commands exist.** Edits to `index.html` take effect on browser reload.
- **Editing**: `index.html` is now ~0.2MB of readable code and can be `Read` directly. Assets are separate files in `assets/`, so the long base64 lines are gone. Use `grep -n` to locate functions/sections by name.

## Architecture (all within `index.html`)

The code is one IIFE running a `requestAnimationFrame` loop (`frame()`, ~line 1484) driven by a string state machine:

`STATE = { MENU, INTRO, PLAY, BREAK, PAUSE, OVER, VICTORY }` with a global `let state`. `frame()` dispatches to `update*(dt)` then draws via `draw*()` per state. `dt` is clamped to 0.05s to survive tab stalls. Victory is reached at round 20.

Key global mutable arrays/objects updated each tick: `players`, `enemies`, `apuchis`, `particles`, `leaves`, `hearts`, `fx_effects`. Game flow: `startGame()` → `startIntro()` → rounds alternate `startRound()` (PLAY, spawn waves via `enemiesForRound(r)`) and `startBreak()` (10s BREAK shop where players grab dropped power-up items).

**Players** (`makePlayer`, line ~357): two kinds, `freddy` (tanky, slow punch, high damage) and `michi` (fragile, fast). Each player reads input from a `controls` map of `e.code` keys. P1 = WASD + Space; P2 = arrows + Comma. A player can have its input redirected via `keysrc` (used for the networked guest).

**Apuchi upgrade system**: power-ups picked up during BREAK. Per-round temporary stacks `tierra` (speed, `TIERRA_MULT`) and `piedra` (damage, `PIEDRA_MULT`), plus persistent `shields` (Cuarzo) and a `special` slot (`morganita` = beam, `safiro` = lightning) with `specialCharges`. Effective stats are computed through `effSpeed`/`effDmg`/etc., not read raw — change multipliers there.

**Enemies**: `ENEMY_TYPES` (`normal`/`fast`/`large`) plus bosses (`spawnBoss`, every 5th round). Difficulty scales by spawn count per round, not per-enemy buffs.

**Rendering**: arena background and tree border are pre-rendered once to offscreen canvases (`arenaBG`, `arenaTrees`) and blitted each frame. Sprites are loaded from base64 `data:` URIs into `Image` objects; `tinted()` caches recolored sprite variants. Audio is base64 `data:` URIs in `AUDIO_SRC`, played via cloned `Audio` elements; `initAudio()` must run on a user gesture (wired to the first keydown).

## Online multiplayer (`initNet` IIFE + net-sync block above it)

**State replication + client-side prediction** — host-authoritative *simulation*, but each guest renders its own world (no video). See `NETWORKING_PREDICTION_DESIGN.md` for the full rationale.
- **Host** runs the only authoritative simulation. Every ~1/18s `netBroadcast()` sends a full world snapshot (`netSerializeSnapshot()`: players + enemies + bosses + apuchis + hearts + amuleto + fx) over the PeerJS **data** channel to each guest, tagged with that guest's player slot (`you`). It still feeds each guest's `{u,dn,l,r,p}` input packet into that guest's player via `applyRemote` → the connection's `keys` object (`players[slot].keysrc`).
- **Guest** is detected by the `?r=<peerId>` query param. It buffers snapshots (`netOnSnapshot`), and each frame `netApplyGuest()` replicates the world into the shared globals (`players`/`enemies`/`apuchis`/…), **interpolating** non-owned entities ~100ms in the past and **predicting** its own player from local input via the real `effSpeed`/`resolveCollision` (`netPredictOwned`), reconciled toward the latest authoritative pose. It renders with the normal `draw*()` code and plays its own audio.
- `_frameBody` branches on `NET.role`: a guest skips all `update*()` sim and instead runs `netApplyGuest`; the host additionally calls `netBroadcast`. Entities carry a stable `id` (`_entId`, stamped at spawn) for cross-snapshot matching.

Snapshots use a static/dynamic split for enemies: constant fields (skin, variant indices, radius, type/kind…) are sent only for an enemy's first ~1s and re-broadcast every ~2s (keyframe), while the small mutable set streams every frame. Projectiles/pickups (`leaves`/arrows, `rays`, `iceBeams`, `nuggets`, `wallets`) are replicated and dead-reckoned locally between snapshots. Not replicated (cosmetic only): host-side spark particles and floating text. UI strings throughout are in Spanish.

## Workflow (owner preference)

After completing and pushing any piece of work, ALWAYS make sure an open pull request exists for it without being asked: if the branch has no open PR, create one; if the branch's previous PR was already merged/closed, rebase the branch onto the latest `main` and open a fresh PR for the new commits. Never leave pushed work sitting without an open PR.
