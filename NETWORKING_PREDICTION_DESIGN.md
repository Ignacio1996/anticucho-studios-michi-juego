# Design: Client-side prediction for online co-op

> **Status: IMPLEMENTED** (M1–M3) in `index.html`. The old host-authoritative *video-streaming*
> model has been fully replaced by state replication + client-side prediction. This doc is kept as
> the design rationale; see "Implementation notes / deviations" at the bottom for what actually
> shipped vs. this plan.
>
> Key symbols in `index.html`: `netSerializeSnapshot()` / `netBroadcast()` (host),
> `netOnSnapshot()` / `netApplyGuest()` / `netPredictOwned()` (guest), and the `initNet` IIFE.

## Why the current model has a latency floor

Today the game is **host-authoritative video streaming**, not state replication:

- The host runs the whole simulation, captures its `<canvas>` with `captureStream()` and sends
  it to each guest as a WebRTC **video track**.
- The guest runs **no game logic**. It hides its own canvas (`wrap` → `display:none`) and shows
  only the host's video (`#remotevideo`). Its keys are sent to the host over a PeerJS data
  connection and fed into Michi (slot 2+) via `players[slot].keysrc`.

So every guest keypress travels: `guest key → data channel → host applies → host simulates a tick
→ host renders → encodes → WebRTC → guest decodes → displays`. The **render + encode + network +
decode** portion is unavoidable while the guest is only watching a video. The stream-tuning work
(event-driven input, zero jitter buffer, per-frame capture, high start bitrate) removes the
*avoidable* delays but cannot touch this floor.

The only way to make the guest's own character respond **instantly** is to simulate it locally and
reconcile against the host — i.e. client-side prediction.

**Bonus:** in this model the guest renders real sprites locally, so the "blurry on join" problem
disappears entirely (native assets, no video ramp-up). This is where the asset preloader
(`PRELOAD_IMGS`, `index.html` ~line 3495) — which is currently dead weight on the guest — finally
earns its keep.

## Target architecture

Standard "server-authoritative + local prediction + entity interpolation" model, as used by most
real-time multiplayer games. Two distinct treatments:

1. **The guest's own player** → *predicted* locally from input, reconciled against host snapshots.
2. **Everything else** (enemies, other players, items, particles) → *interpolated* between the
   two most recent host snapshots, rendered ~100 ms in the past for smoothness.

We do **not** attempt full deterministic lockstep. The sim leans heavily on `Math.random()`
(spawns, particles, drops), so guests can't reproduce host RNG. Predicting only the local player's
*movement* sidesteps that — the guest never needs to predict enemy behaviour or damage.

### Data flow (per guest)

```
Guest input ──(unreliable data ch)──▶ Host: apply to players[slot].keysrc, simulate (authoritative)
Guest input ──▶ apply locally to own player immediately (PREDICTION)          │
                                                                              ▼
Guest ◀──(snapshots @ ~20Hz)────────────────────────────── Host: serialize world each N ticks
   │
   ├─ own player: snap to authoritative pos at snapshot.tick, then REPLAY buffered inputs since
   │              that tick; if diverged, smoothly correct (lerp) rather than hard-snap
   └─ other entities: buffer 2+ snapshots, render interpolated at (now − 100ms)
```

## Snapshot format (host → guest)

Sent over the existing data connection at ~15–20 Hz (replace the video track entirely). Full
snapshots (not deltas) so packet loss on the unreliable channel self-heals within one frame.

```js
{
  seq,  tick,                     // monotonic; guest drops out-of-order older seqs
  state, round, roundTimer,       // STATE machine + round/break timers for HUD
  players: [ { id, kind, x, y, facing, hp, punchT, dead,
               tierra, piedra, shields, special, specialCharges } ],  // enough to render + HUD
  enemies: [ { id, t, x, y, facing, hp, st } ],   // t=type index, st=state enum
  bosses:  [ { id, x, y, hp, maxhp, phase } ],
  items:   [ { id, kind, x, y } ],                // apuchi drops during BREAK
  hearts:  [ { x, y } ],
  fx:      [ ... ],               // only gameplay-relevant fx (beams/lightning); skip cosmetic leaves
}
```

Rough bandwidth: 4 players + ~30 enemies + items ≈ a few hundred bytes/snapshot × 20 Hz ≈
**10–20 KB/s per guest** — two orders of magnitude below the 6 Mbps video. Optional later:
quantize coords to int16, delta-encode, `MessagePack`.

IDs are required so the guest can match entities across snapshots for interpolation. Host must
assign a stable `id` to each enemy/item at spawn (small refactor to the spawn helpers).

## Guest changes

1. **Stop hiding the canvas / drop the video.** Show `wrap`, hide `#remotevideo`. Don't mute
   `masterGain`; the guest plays its own audio (see Audio below).
2. **Populate the shared globals from snapshots.** `players`, `enemies`, `apuchis`, `hearts`,
   `fx_effects` become mirrors of the latest authoritative state (with an interpolation buffer).
3. **Run a guest-only frame path:**
   - Advance interpolation clock; lerp non-owned entities between snapshot N-1 and N.
   - Predict own player: apply local input via the *existing* movement code (`effSpeed`, etc.)
     every frame so movement is zero-latency.
   - Reuse the existing `draw*()` functions unchanged — they already read from the globals.
4. **Reconciliation** (own player only): keep an input history ring keyed by tick. On snapshot,
   set own player to the authoritative pose, then re-apply inputs with tick > snapshot.tick. If
   the corrected position differs from the currently displayed one by more than a small epsilon,
   ease toward it over a few frames instead of snapping.
5. **Combat stays host-authoritative.** Predict only the punch *animation* (visual), never damage
   or knockback — those come from snapshots. Prevents ugly mispredicted hits.

## Host changes

- Keep the authoritative sim exactly as-is.
- Add `serializeSnapshot()` and broadcast it to every guest conn every ~3rd tick (assign entity
  IDs at spawn first).
- Remove the `captureStream` / `peer.call` video path (or keep behind a flag as fallback).

## Audio

Guest currently plays the host's streamed audio and mutes everything local. Under prediction the
guest plays audio locally, driven off snapshot state transitions (round start, hits, deaths). This
means SFX fire on the guest's own machine — lower latency, but the round/boss cue logic in
`playRoundIntro` / event hooks must be triggered from snapshot deltas rather than direct calls.

## Rollout milestones (each independently shippable)

- **M1 — Replicated rendering, no prediction.** Guest renders from snapshots + interpolation;
  input still round-trips to the host. *Already fixes the blur natively* (real assets) and cuts
  bandwidth ~100×. Input lag unchanged. Lowest-risk first step; validates serialization.
- **M2 — Local movement prediction.** Predict + reconcile the guest's own player. This is the step
  that actually removes the *movement* input lag.
- **M3 — Predicted punch animation + reconciliation smoothing + audio localization + polish.**

## Risks / open questions

- **Refactor surface:** entities need stable IDs; the sim step and render step must be cleanly
  separable per-guest (they're mostly separated already via `update*`/`draw*`).
- **Determinism not required** (we interpolate non-owned entities), which is what makes this
  tractable — but mispredicting the *local* player near walls/enemies needs the smooth-correct
  path to avoid rubber-banding.
- **Cheating:** guest is authoritative only over its own *input*; host validates all movement, so
  a tampered guest can't do more than move its own character within the host's rules.
- **>4 players / weak clients:** could keep the video path as a runtime fallback and choose per
  connection.

## Effort

Large — this is a networking-model change, not a tweak. M1 is the bulk of the serialization work;
M2 adds the prediction/reconciliation loop; M3 is polish.

## Implementation notes / deviations (as shipped)

All three milestones landed together and the video path was removed outright (no `?predict=1`
A/B flag — the replicated path is now the only path). Concrete choices that differ from the plan:

- **Full-object snapshots, not minimal deltas.** The host sends `~18` snapshots/s over the existing
  unreliable data channel. Each enemy is serialized with a *primitive-field filter* (`netSerEnt`:
  copies only number/string/boolean fields, which auto-drops object refs like a boss's target and
  any functions → always BinaryPack-safe) rather than the hand-picked `{id,t,x,y,facing,hp,st}`.
  Players use an explicit whitelist (`netSerPlayer`, facing flattened to `fx`/`fy`). This is heavier
  than the doc's "few hundred bytes" target (~single-digit KB/snapshot) but still ~100× below the
  old 6 Mbps video, and it guarantees the detailed `draw*()` funcs get every field they read
  (skin, variant indices `gvar`/`lvar`/`gpvar`, boss anim state, etc.). Delta-encoding / int16
  quantization / MessagePack remain the same "optional later" wins.
- **Entity IDs** are stamped at spawn via a module-level `_entId` counter on enemies (all 4 spawn
  sites incl. bosses) and apuchis — used for cross-snapshot matching / interpolation.
- **animTime advanced locally** on the guest (walk cycles stay smooth regardless of snapshot rate);
  only x/y are interpolated between the two snapshots bracketing `now − 100 ms`.
- **Prediction** (`netPredictOwned`) applies local input through the real `effSpeed` + `resolveCollision`
  each frame, then eases toward the latest authoritative pose (hard-snap past 110px for
  round-start / revive / cliff-death teleports). Input is only applied during PLAY/BREAK.
- **Predicted punch** = animation only, on the input rising edge, gated by the character's cooldown;
  damage/knockback stay 100% host-authoritative.
- **Audio localized** minimally: round-start cue on round delta + background music mirrored from
  state; all wrapped in try/catch so an audio hiccup can never break rendering.
- **Not replicated (known gaps):** cosmetic/secondary visuals that aren't in the snapshot — enemy
  projectiles (`leaves`/arrows), `rays`, `iceBeams`, `nuggets`, machu `wallets`, and host-side
  spark particles. The guest sees players, enemies, bosses, apuchis, hearts, the amuleto/key, and
  `fx_effects` (beams/zaps/shock/camflash). Adding the projectile arrays to the snapshot is the
  natural next increment if their absence proves distracting.
- **Per-guest camera**: each guest now frames the group independently (its own `updateCamera`),
  rather than all guests sharing the host's single viewport as under the video model.
