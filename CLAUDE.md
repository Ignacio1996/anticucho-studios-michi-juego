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

## Online multiplayer (`initNet` IIFE, line ~1533)

Host-authoritative streaming model, **not** state replication:
- **Host** runs the entire simulation, captures its `<canvas>` via `captureStream(30)` and sends it to the guest as a WebRTC video track. It receives the guest's normalized key state over a PeerJS data connection and feeds it into Michi (player 2) through the shared `remoteKeys` object (`players[1].keysrc = remoteKeys`).
- **Guest** is detected by the `?r=<peerId>` query param. It renders only the incoming video (`#remotevideo`), runs no game logic, captures its own keys in the capture phase to keep them out of the local hidden game instance, and sends a compact `{u,dn,l,r,p}` input packet every 33ms.

Consequence: the guest's experience is entirely the host's screen; only the host's machine simulates anything. UI strings throughout are in Spanish.
