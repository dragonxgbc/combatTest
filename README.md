# combatTest

Created with [Rojo](https://github.com/rojo-rbx/rojo) 7.5.1.

# Combat System

> A server-authoritative combat system with M1 combos, blocking/posture, Black Flash, dashing, knockback, wall daze, and NPC support.

## Table of Contents

- [Architecture](#architecture)
- [Controls](#controls)
- [Configuration](#configuration)
- [Server](#server)
  - [MainCombat.server.luau](#maincombatserverluau)
  - [m1System.luau](#m1systemluau)
  - [blackFlashSystem.luau](#blackflashsystemluau)
  - [Remotes.luau](#remotesluau)
  - [HitboxDebug.luau](#hitboxdebugluau)
  - [npcController.luau](#npccontrollerluau)
  - [NPCRegistry.server.luau](#npcregistryserverluau)
  - [State.luau](#stateluau)
  - [Lockouts.luau](#lockoutsluau)
  - [Posture.luau](#postureluau)
  - [WallDaze.luau](#walldazeluau)
  - [Dashing.luau](#dashingluau)
  - [MovementMechanics.luau](#movementmechanicsluau)
  - [BVPool.luau](#bvpoolluau)
  - [JumpLock.luau](#jumplockluau)
  - [CooldownManager.luau](#cooldownmanagerluau)
- [Server — Utility Scripts](#server--utility-scripts)
- [Client](#client)
- [Remote Events Reference](#remote-events-reference)
- [Character Attributes Reference](#character-attributes-reference)

---

## Architecture

| Layer | Path | Purpose |
| :--- | :--- | :--- |
| Shared | `src/shared/` | Config only |
| Server | `src/server/` | Authority for all game state, damage, physics |
| Client | `src/client/` | Input, animation, VFX, HUD |

Remotes flow **server → client** for broadcasts and feedback, and **client → server** for requests. The server validates all hit reports before applying damage.

---

## Controls

| Key | Action |
| :--- | :--- |
| <kbd>LMB</kbd> (hold) | M1 combo (up to 4 hits) |
| <kbd>F</kbd> (hold) | Block |
| <kbd>Q</kbd> | Dash |
| <kbd>1</kbd> | Black Flash |

---

## Configuration

All tunable values live in `src/shared/CombatConfig.luau`. Both server and client read from it.

<details>
<summary>Config sections</summary>

1. Core Combo Timing & Damage
2. Hitbox & Visualization
3. Blocking & Guard / Suppression
4. Stun, Endlag & Reactions
5. Knockback & Movement Effects
6. Jump / Mobility (Dash, Jump Lock)
7. Animation & Remote LOD
   - 7b. Wall Daze
   - 7c. Invulnerability Punch Penalty
8. AI Helpers
9. Debug & Instrumentation
10. Endlag Marker Display
11. Asset References (Animations)
12. Misc / Pooling
13. Sound Effects
14. Punch Aura
15. Camera
16. Black Flash

</details>

> [!NOTE]
> `BLACK_FLASH_COOLDOWN` is commented out in config. The fallback `or 12` in `CooldownManager` and `ClientCooldowns` is what's active.

---

## Server

### `MainCombat.server.luau`

Entry point. Tags every character and NPC humanoid with a `CombatGUID` attribute (used by clients to report hit targets by ID). Resets all state flags on character spawn. Wires all remotes:

| Remote | Handler |
| :--- | :--- |
| `Combat_Punch` | `m1System.request` |
| `Combat_Block` | `Posture.trySetBlocking` |
| `Combat_HitReport` | `m1System.validateClientHit` |
| `Combat_Dash` | `Dashing.tryDash` |
| `Combat_BlackFlash` | `BlackFlashSystem.request` |

Cleans up `State` and `BlackFlashSystem` tables on `PlayerRemoving`.

### `combat/m1System.luau`

Core M1 system.

**`request(player)`** — Validates all gates (blocking, guard broken, stunned, hitslow, endlag, per-hit cooldown, anti-duplicate threshold, blocked punch penalty, `nextStringAvailableTime`), advances the combo index, applies endlag, and broadcasts to all clients for animation. For NPCs it also runs a server-side hitbox scan directly.

**`validateClientHit(player, payload)`** — Processes client-reported hit lists: verifies punch index matches active punch, checks time window, validates range, applies block detection with back-hit bypass, post-dash stun/damage modifiers, `State.registerHit` debounce, HP damage, posture updates, hitstun, knockback, and hit feedback remotes.

### `combat/blackFlashSystem.luau`

Special move (<kbd>1</kbd>). Validates the same gates as M1 plus its own per-player `lastUsed` timestamp against `CooldownManager.get("blackFlash")`. Applies full-duration endlag immediately, broadcasts to all clients (triggers animation + arm aura), then after `windupTime` delay performs the hitbox scan.

| Target state | Damage | Knockback | Posture | Hitstun/Stagger |
| :--- | :---: | :---: | :---: | :---: |
| Unblocked | Full | ✓ | +30 | ✓ |
| Blocked (front) | None | ✗ | +30 | ✓ |
| Blocked (back-hit bypass) | Full | ✓ | +30 | ✓ |

### `combat/Remotes.luau`

Creates and ensures all required `RemoteEvent` instances exist in `ReplicatedStorage`. Auto-creates any missing ones with a warning. Single source of truth for which remotes the combat system uses.

### `combat/HitboxDebug.luau`

Server-side visual debug. Pools up to 64 `Part` instances with `SelectionBox` outlines. `spawn(cframe, size, color)` shows a hitbox part for `HITBOX_DURATION` seconds then returns it to the pool.

### `combat/npcController.luau`

NPC lifecycle manager.

- **`register(model, options)`** — Captures a template clone for respawning, hooks `Humanoid.Died`, sets up idle/walk/block animations, optionally starts auto-punch and auto-block loops.
- **`unregister(model)`** — Stops all tracks and disconnects signals.
- **Respawn** — Clones the original template after `respawnDelay` seconds.
- Two `RunService.Heartbeat` loops: idle/walk animation switching + auto-block; auto-punch (`m1System.request` on the NPC model).

<details>
<summary>register() options</summary>

| Option | Type | Description |
| :--- | :--- | :--- |
| `respawn` | bool | Auto-respawn on death (default `true`) |
| `respawnDelay` | number | Seconds before respawn (default 3) |
| `template` | Model | Custom clone source |
| `spawnCFrame` | CFrame | Spawn location override |
| `animations` | table | `{Idle: string, Walk: string}` asset IDs |
| `autoPunch` | bool | Enable simple punch loop |
| `punchInterval` | number | Seconds between punch attempts |
| `stringPause` | number | Pause after finishing a full combo |
| `autoBlock` | bool | Enable auto-block toggling |
| `autoBlockInterval` | number | Seconds between block attempts |
| `deathCleanupDelay` | number | Seconds after death before model destroyed |

</details>

### `combat/NPCRegistry.server.luau`

Auto-discovers and registers NPC models. Scans workspace on start and watches `DescendantAdded`. A model qualifies if it has `NPC_AutoRegister = true` or lives inside `workspace.NPCs`.

<details>
<summary>Per-model attributes</summary>

| Attribute | Type | Description |
| :--- | :--- | :--- |
| `NPC_AutoRegister` | bool | Register even outside the NPCs folder |
| `NPC_AutoPunch` | bool | Enable auto-punch loop |
| `NPC_PunchInterval` | number | Punch interval override |
| `NPC_StringPause` | number | Post-combo pause |
| `NPC_AutoBlock` | bool | Enable auto-block |
| `NPC_AutoBlockInterval` | number | Block attempt interval |
| `NPC_AutoBlockPulse` | bool | Occasionally release/re-hold block |
| `NPC_Respawn` | bool | Respawn override |
| `NPC_RespawnDelay` | number | Respawn delay override |
| `NPC_IdleAnim` | string | Idle animation asset ID |
| `NPC_WalkAnim` | string | Walk animation asset ID |
| `NPC_SpawnX/Y/Z` | number | Spawn position override (all three required) |

</details>

### `combat/modules/State.luau`

Per-combatant state store keyed by `Player` or `Model`. Holds combo index, punch timing, blocking/stunned/guardBroken flags, cooldown timers, and a per-action hit debounce map (`recentHits`).

| Function | Description |
| :--- | :--- |
| `ensure(attacker)` | Lazy-init state record |
| `ensureForChar(char)` | Same but takes a Model (NPC support) |
| `getForChar(char)` | Find state by character model (linear scan) |
| `registerHit(attacker, actionId, target, window)` | Debounce a hit; returns `false` if already registered |
| `resetForRespawn(attacker)` | Clear all transient timers/flags on new character |
| `cleanup(attacker)` | Remove state entry on player leave |

### `combat/modules/Lockouts.luau`

Manages all movement and action restrictions. A background coroutine per affected character (20 Hz) applies WalkSpeed/JumpPower overrides and restores them when all locks expire.

| Function | Description |
| :--- | :--- |
| `applyHitslow(char, duration, isFinisher?)` | Movement-capping stun with stacking cap |
| `applyStagger(char, duration)` | Full-freeze phase within hitslow (only extends, never shrinks) |
| `applyEndlag(char, duration, source)` | Named attacker lock (allows selective removal) |
| `removeEndlagSource(char, source)` | Remove one specific endlag source |
| `applyMovementLock(char, duration)` | Generic movement block |
| `isHitslow(char)` | Query active hitslow |
| `isEndlag(char)` | Query any active endlag |
| `clearAll(char)` | Force-clear everything |

**Hitslow stacking rules:**
- Fresh application sets the full duration
- Extensions add half the duration (finisher adds full)
- Total is capped at `HITSTUN_STACK_CAP`

### `combat/modules/Posture.luau`

Guard pressure bar (0 → `POSTURE_MAX`). A background loop runs passive linear decay every `POSTURE_DECAY_TICK` (0.1s), decaying from current value to 0 over `POSTURE_DECAY_TIME` (22s).

| Function | Description |
| :--- | :--- |
| `applyBlockedHit(char)` | +`POSTURE_BLOCK_GAIN` (10); triggers guard break at max |
| `applyCustomGain(char, amount)` | Fixed gain (Black Flash uses +30) |
| `applyHitReduction(char)` | −`POSTURE_HIT_REDUCTION` (5) on unblocked hits |
| `trySetBlocking(attacker, bool)` | Set block with rate limiting and suppression checks |
| `applyGuardBreak(char, context)` | Full guard break: clears block, applies stun + stagger + movement lock, schedules recovery, fires feedback remotes |

### `combat/modules/WallDaze.luau`

After a finisher knockback, `monitor(target, attacker)` attaches `Touched` listeners to all body parts for `WALL_DAZE_MONITOR_WINDOW` (0.8s).

> [!TIP]
> To mark a wall: insert any Instance named `"WallDaze"` as a child of the Part (use `WallDazeMarker.server.luau`).

On wall contact:
1. Applies `WALL_DAZE_BONUS_DAMAGE`
2. Fires `Combat_HitFeedback` type `"WallHit"`
3. Releases existing knockback velocity
4. Anchors the root briefly (`WALL_DAZE_ANCHOR_TIME`)
5. Applies hitslow + invulnerability for `WALL_DAZE_INVULN_TIME`

Horizontal surfaces (floors/ceilings) are ignored via a size ratio check.

### `combat/modules/Dashing.luau`

Server-side dash execution. Validates state gates and `dashCDUntil` from `CooldownManager.get("dash")`. Applies a `BodyVelocity` via `BVPool`, records `lastDashServerTime` for post-dash penalty windows, and applies visual endlag via `Lockouts.applyEndlag("dash")`.

### `combat/modules/MovementMechanics.luau`

Knockback impulse application.

- `finisherKnockback` — Used for NPC targets only; player targets receive knockback via `Combat_Knockback` remote fired to their client.
- `lightKnockback` — Intentionally empty (removed).
- Internal `applyKnockback` handles direction calculation, minimum separation enforcement, and method dispatch (BodyVelocity vs AssemblyLinearVelocity).

### `combat/modules/BVPool.luau`

`BodyVelocity` object pool (max 64, set by `BODYVELOCITY_POOL_MAX`).

| Function | Description |
| :--- | :--- |
| `acquire(maxForce, p)` | Get from pool or create new |
| `acquireReplacing(parent, maxForce, p)` | Reuse the BV currently on a specific part |
| `release(bv)` | Return to pool (or destroy if pool full) |
| `releaseFor(parent)` | Release by parent part reference |

### `combat/modules/JumpLock.luau`

Jump suppression during combos.

| Function | Description |
| :--- | :--- |
| `apply(char)` | Set JumpPower = 0 |
| `clearIfNeutral(char)` | Restore JumpPower if not stunned/blocking/in-combo/in-endlag |
| `forceClear(char)` | Unconditional restore |

> [!NOTE]
> Always restores to `ROBLOX_DEFAULT_JUMPPOWER = 50`.

### `combat/modules/CooldownManager.luau`

Server-side runtime cooldown duration store. Values can be changed at runtime without touching Config (used by `FastCooldowns`). Changes are synced to all clients via `Combat_CooldownSync`.

Cooldown names: `"blackFlash"`, `"dash"`, `"dashEndlag"`.

| Function | Description |
| :--- | :--- |
| `get(name)` | Returns current duration (override or default) |
| `set(name, value)` | Set a runtime override |
| `reset(name)` | Clear override, revert to default |
| `resetAll()` | Clear all overrides |
| `syncAll()` | Fire `Combat_CooldownSync` to all clients |

---

## Server — Utility Scripts

> [!TIP]
> These scripts are placed **inside Parts in the workspace**. Each Part's `Touched` event drives the behavior.

### `FastCooldowns.server.luau`

Touch toggles all combat CDs between 0.5s and their Config defaults. Syncs to clients via `Combat_CooldownSync`. Resets all active per-player cooldowns on both toggle directions so no player is locked out when switching.

### `CooldownRefresh.server.luau`

Touch resets all active cooldowns for the touching player only (BF `lastUsed`, `dashCDUntil`). Fires `Combat_CooldownReset` to that client so the HUD and `lastLocalUseTime` also clear instantly.

### `FullHeal.server.luau`

Touch restores the player to full health. 1-second per-player debounce.

### `Immortality.server.luau`

Touch toggles invulnerability. While active, hooks `HealthChanged` to immediately restore HP to max. Cleans up the connection on `PlayerRemoving`.

### `InvulnZone.server.luau`

Touch sets or clears the `Invulnerable` attribute based on the Part's `InvulnGive` attribute.

| `InvulnGive` | Behavior |
| :---: | :--- |
| `true` | Grants `Invulnerable` on touch |
| absent / `false` | Removes `Invulnerable` on touch |

### `WallDazeMarker.server.luau`

Empty script. Place it inside a Part and rename it to `"WallDaze"` — its presence is what `WallDaze.luau` detects via `part:FindFirstChild("WallDaze")`.

---

## Client

### `m1Client.client.luau`

Handles M1 input, optimistic animation, local hitbox scan, and hit feedback.

- **Hold-to-M1:** `holdLoop` fires `Combat_Punch` every ~0.02s while mouse is held and timing gates pass.
- **`playOptimisticPunch`** — Plays animation + swing sound immediately on input before server confirms.
- **`Combat_PunchBroadcast`** — If local attacker: plays confirmed animation and starts `runLocalScan`. If remote attacker: plays limb aura and an optimistic victim reaction if local player is within range and in front.
- **`runLocalScan`** — Client-side hitbox sweep; reports hits via `Combat_HitReport`.
- **`Combat_HitFeedback`** — Shows hit aura, plays reaction animation and sounds; deduplicates sounds already pre-played by optimistic code.

### `blackFlashClient.client.luau`

Press-<kbd>1</kbd> special move client handler.

- **Input:** Same gates as server (blocking, stunned, guardbroken, endlag, hitslow) plus client-side cooldown via `ClientCooldowns`. Fires `Combat_BlackFlash`.
- **`Combat_BlackFlashBroadcast`** — Advances arm cycle (alternates Right/Left), plays animation at reduced speed, starts black/red arm aura via `VisualAura.startBlackFlash`.
- **`Combat_HitFeedback` (BlackFlash)** — Cancels arm aura, plays hit sound, shows black hit aura on target, plays B&W screen flash for local attacker, plays heavy reaction on target.
- **`Combat_CooldownReset`** — Resets `lastLocalUseTime`.

### `postureClient.client.luau`

<kbd>F</kbd>-key blocking.

- `requestBlock` deduplicates state changes, checks `BlockSuppressUntil`, fires `Combat_Block`, plays optimistic animation.
- Safety revert: if server `Blocking` attribute doesn't confirm within 0.35s, drops the animation.
- Syncs from `Blocking` and `GuardBroken` attribute changes for server-forced unblocks.

### `DashingClient.client.luau`

<kbd>Q</kbd>-key dash. Client CD guard via `ClientCooldowns.getDash()`. Sends `hum.MoveDirection` to server. Plays dash sound and spawns dash smoke particle attachment.

### `CooldownHUD.client.luau`

HUD slots for Dash [<kbd>Q</kbd>] and Black Flash [<kbd>1</kbd>], anchored bottom-center of screen.

- Dims + shows countdown while on cooldown.
- Lights up when ready; plays green pulse animation on cooldown expiry.
- Red reject flash when input is attempted while on cooldown.
- `Combat_CooldownReset` clears both slots and `lastDashTrigger`.

### `ClientCooldowns.luau`

Client-side cooldown duration store. Initialized from `CombatConfig` defaults and kept in sync via `Combat_CooldownSync` whenever the server changes values.

| Function | Description |
| :--- | :--- |
| `get(name)` | Returns cooldown duration |
| `getDash()` | Returns `dash + dashEndlag` (total reuse time) |

### `AnimationController.luau`

Manages all combat animation tracks for the local player.

- Caches punch tracks (1–4), block track, and reaction tracks (1–4) against the current Humanoid/Animator.
- Weight-tweened cross-fading for punch transitions using a `TweenService`-driven `NumberValue`.
- `remoteCaches` table for other players' punch animations with manual step-based fade.
- `reset()` stops all tracks and clears state — called by `postureClient` on `CharacterAdded`.

| Function | Description |
| :--- | :--- |
| `preloadAll()` | Load and warm-start all tracks |
| `playPunch(index)` | Cross-fade to punch animation |
| `setBlocking(bool)` | Play/stop block animation |
| `playReaction(slot)` | Play reaction animation slot |
| `playRemotePunch(model, index)` | Play punch on another player/NPC |
| `forceNeutral()` | Fade out active punch/block |
| `reset()` | Clear all tracks on respawn |

### `SoundEffects.luau`

Round-robin pool of 10 `Sound` instances parented to the local HumanoidRootPart.

- `warm(id)` pre-fetches assets by briefly playing at volume 0 to eliminate first-play latency.
- `preloadBulk(ids)` called at startup with all known sound IDs.
- `play(id)` assigns ID to the next pool slot, stops if already playing, replays from start.
- Re-parents pool sounds to the new character root on `CharacterAdded`.

### `visual/VisualAura.luau`

Purely visual — manages `Highlight` instance lifecycle for hit flashes and arm auras.

| Function | Description |
| :--- | :--- |
| `startHit(model, params?)` | Full-body flash (fade in → hold → fade out) |
| `startPunchLimb(attacker, punchIndex, params?)` | Per-arm highlight during active punch; alternates R/L by combo index |
| `startBlackFlash(attacker, side?)` | Black fill + red outline arm aura; auto-expires after `BLACK_FLASH_ARM_AURA_HOLD` |
| `cancelBlackFlash(attacker)` | Immediately remove arm aura on hit |
| `cancelHit(model)` | Force-disable the hit aura |
| `cancelPunchLimb(attacker)` | Disable all limb highlights |
| `clearAllFor(model)` | Destroy all aura state for a model |

**Hit aura colors:**

| Event | Color |
| :--- | :--- |
| Hit (M1) | White (`HIT_AURA_COLOR_HIT`) |
| Blocked | Blue (`HIT_AURA_COLOR_BLOCKED`) |
| Guard Break | Red (`HIT_AURA_COLOR_GUARDBREAK`) |
| Invulnerable | Yellow (`HIT_AURA_COLOR_INVULNERABLE`) |
| Black Flash hit | Black |
| Wall hit | Guard break color (red) |

### `knockbackClient.client.luau`

Listens on `Combat_Knockback`. Creates a `BodyVelocity` with the server-sent velocity vector, parents it to `HumanoidRootPart`, destroys it after the given duration. Also calls `AnimationController.cancelReaction()` to clear any lingering reaction during flight.

### `PostureBar.client.luau`

`BillboardGui` above the local player's head showing posture as a horizontal fill bar (white on dark grey). Only visible while blocking. Updates on `Posture` and `Blocking` attribute changes. Has a `Heartbeat` fallback to re-create the UI if it gets destroyed on respawn.

### `CameraClamp.client.luau`

Enforces `CAMERA_MIN_ZOOM` (5) and `CAMERA_MAX_ZOOM` (18) via `player.CameraMinZoomDistance/MaxZoomDistance`. Re-applies every `RenderStepped` as a guard against anything else overriding the property.

---

## Remote Events Reference

All created and ensured by `combat/Remotes.luau`.

| Remote | Direction | Description |
| :--- | :---: | :--- |
| `Combat_Punch` | C → S | Request M1 punch |
| `Combat_Block` | C → S | Set blocking state (bool) |
| `Combat_HitReport` | C → S | Report client hitbox scan results |
| `Combat_Dash` | C → S | Request dash (Vector3 direction) |
| `Combat_BlackFlash` | C → S | Request Black Flash |
| `Combat_PunchBroadcast` | S → All | Notify all clients of a punch (attacker, index, debug info) |
| `Combat_HitFeedback` | S → All | Hit result for visual/audio feedback (type, target, attacker) |
| `Combat_Reaction` | S → All | Force reaction animation on a target |
| `Combat_Knockback` | S → Target | Send knockback velocity vector to target's client |
| `Combat_BlackFlashBroadcast` | S → All | Notify all clients of a Black Flash windup |
| `Combat_CooldownSync` | S → All/One | Push current cooldown durations to clients |
| `Combat_CooldownReset` | S → One/All | Reset all active client-side cooldown timers |

---

## Character Attributes Reference

Set on the character `Model` by the server; read by both server and client.

| Attribute | Type | Description |
| :--- | :---: | :--- |
| `CombatGUID` | string | Stable ID for hit reporting |
| `Blocking` | bool | Currently blocking |
| `Stunned` | bool | In hitstun or guard broken |
| `GuardBroken` | bool | Guard broken, cannot block |
| `Endlag` | bool | Attacker is locked out (endlag active) |
| `Hitslow` | bool | Movement-capped stun active |
| `Posture` | number | Current posture value (0–max) |
| `Invulnerable` | bool | Combat system skips all hits |
| `BlockSuppressUntil` | number | Server timestamp; block raise blocked until this time |
| `PunchLockUntil` | number | Server timestamp; punches blocked until this time |
| `NPC_Controlled` | bool | Marks models managed by npcController |
