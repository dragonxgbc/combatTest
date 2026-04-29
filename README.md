# Sorcery Fight

Built with [Rojo](https://github.com/rojo-rbx/rojo) 7.5.1.

---

# Combat System

> A server-authoritative combat system with M1 combos, blocking/posture, Black Flash, Divergent Fist, dashing, knockback, wall daze, spell profiles (Death Curse, Disarm, Chain, Combo Spell), and NPC support.

## Table of Contents

- [Architecture](#architecture)
- [Shared](#shared)
- [Paired Systems](#paired-systems)
  - [M1 Punch](#m1-punch)
  - [Black Flash](#black-flash)
  - [Divergent Fist](#divergent-fist)
  - [Dash](#dash)
  - [Death Curse (Wizard Beam)](#death-curse-wizard-beam)
  - [Disarm Spell](#disarm-spell)
  - [Chain](#chain)
  - [Combo Spell](#combo-spell)
  - [Posture / Block](#posture--block)
  - [Knockback](#knockback)
  - [Charge Orb VFX](#charge-orb-vfx)
  - [Cooldowns](#cooldowns)
  - [Character Profiles](#character-profiles)
- [Server — Core Infrastructure](#server--core-infrastructure)
- [Server — NPCs](#server--npcs)
- [Server — Utility Scripts](#server--utility-scripts)
- [Client — Shared Utilities](#client--shared-utilities)
- [Controls](#controls)
- [Remote Events Reference](#remote-events-reference)
- [Character Attributes Reference](#character-attributes-reference)

---

## Architecture

| Layer | Path | Purpose |
| :--- | :--- | :--- |
| Shared | `src/shared/` | Config, character profile definitions |
| Server | `src/server/` | Authority for all game state, damage, physics |
| Client | `src/client/` | Input, animation, VFX, HUD |

Remotes flow **client → server** for requests and **server → client / all** for broadcasts and feedback. The server validates all hit reports before applying damage.

---

## Shared

### `shared/CombatConfig.luau`

All tunable values. Both server and client read from it via `rawget` to avoid triggering metamethods.

<details>
<summary>Config sections</summary>

1. Core Combo Timing & Damage
2. Hitbox & Visualization
3. Blocking & Guard / Suppression
4. Stun, Endlag & Reactions
5. Knockback & Movement Effects
6. Jump / Mobility (Dash, Jump Lock)
7. Animation & Remote LOD · Wall Daze · Invulnerability Punch Penalty
8. AI Helpers
9. Debug & Instrumentation
10. Endlag Marker Display
11. Asset References (Animations)
12. Misc / Pooling
13. Sound Effects
14. Punch Aura
15. Camera
16. Black Flash
17. Shift Lock Visual (stub)
18. Divergent Fist
19. Spellslinger (Death Curse, Disarm Spell, Chain, Combo Spell)

</details>

### `shared/CharacterProfiles.luau`

Defines the selectable character profiles. Each profile has an `id`, `name`, up to four `moves` slots (move IDs or `nil`), and an optional `passive`.

Universal systems — M1, Block, Dash — are not listed in profiles; they are always active.

**Profiles:**

| id | Name | Slot 1 | Slot 2 | Slot 3 |
| :--- | :--- | :--- | :--- | :--- |
| `black_sparks` | Black Sparks | Black Flash | Divergent Fist | — |
| `wizard` | Spellslinger | Death Curse | Disarm | Chain |

**API:** `list` (array of profiles), `hasMove(profile, moveId)` (slot lookup).

---

## Paired Systems

Each section covers the server module and its client counterpart together.

---

### M1 Punch

**Server:** `server/combat/m1System.luau` · **Client:** `client/combat/m1Client.client.luau`

#### Server — `m1System.luau`

**`request(player)`** — Validates blocking, guard broken, stunned, hitslow, endlag, per-hit cooldown, anti-duplicate threshold, blocked punch penalty, and `nextStringAvailableTime`. Advances combo index, applies endlag, broadcasts to all clients. For NPC targets, runs a server-side hitbox scan directly.

**`validateClientHit(player, payload)`** — Processes client-reported hit lists: verifies punch index against `activePunchIndex`, checks time window, validates range, applies block detection with back-hit bypass, post-dash stun/damage modifiers, `State.registerHit` debounce, HP damage, posture updates, hitstun, knockback, and hit feedback.

**Divergent Fist interaction:** When DF is active and an unblocked M1 lands, a delayed follow-up is scheduled with a 1.5–4s random delay: `DIVERGENT_FIST_DAMAGE`, `HITSTUN_TIME` hitslow, `lightKnockback`, fires `Combat_HitFeedback` type `"DivergentFistDelayed"`.

#### Client — `m1Client.client.luau`

- **Hold-to-M1:** `holdLoop` fires `Combat_Punch` every ~0.02s while mouse is held and timing gates pass.
- **`playOptimisticPunch`** — Plays animation + swing sound immediately on input.
- **`Combat_PunchBroadcast`** — If local attacker: plays confirmed animation and starts `runLocalScan`. If remote attacker: plays limb aura and an optimistic victim reaction if the local player is in range and facing the attacker.
- **`runLocalScan`** — Client-side hitbox sweep; reports hits via `Combat_HitReport`.
- **`Combat_HitFeedback`** — Shows hit aura, plays reaction animation and sounds; deduplicates sounds pre-played by optimistic code.

---

### Black Flash

**Server:** `server/combat/blackFlashSystem.luau` · **Client:** `client/combat/blackFlashClient.client.luau`

#### Server — `blackFlashSystem.luau`

Special move. Validates the same gates as M1 plus its own `lastUsed` timestamp against `CooldownManager.get("blackFlash")`. Applies full-duration endlag immediately, broadcasts to all clients, then after `windupTime` performs the hitbox scan.

| Target state | Damage | Knockback | Posture | Hitstun |
| :--- | :---: | :---: | :---: | :---: |
| Unblocked | `BLACK_FLASH_DAMAGE` | ✓ | — | ✓ |
| Unblocked + DF active | `BLACK_FLASH_DAMAGE + 5` | ✓ | — | ✓ |
| Blocked (front) | None | ✗ | +30 | ✓ |
| Blocked (back-hit bypass) | `BLACK_FLASH_DAMAGE` | ✓ | — | ✓ |

**DF+BF interaction:** +5 bonus damage; `DivFist = true` flag included in `Combat_HitFeedback` for client cursed energy burst.

| Function | Description |
| :--- | :--- |
| `request(player)` | Full validation + windup + hitbox |
| `resetCooldown(player)` | Clear `lastUsed` |
| `cleanup(player)` | Remove state on leave |

#### Client — `blackFlashClient.client.luau`

- **Input:** Client-side gates + `ClientCooldowns.isReady("blackFlash")` guard. Fires `Combat_BlackFlash`.
- **`Combat_BlackFlashBroadcast`** — Advances arm cycle, plays animation at reduced speed, starts black/red arm aura via `VisualAura.startBlackFlash`.
- **`Combat_HitFeedback` (BlackFlash):**
  - Cancels arm aura
  - Plays hit or block sound
  - Shows black hit aura on target
  - **Three-phase screen flash** (local attacker only): red → white → black, 0.06s each, no tweening
  - **LightningBolt cone**: 8 bolts (+ 8 overlapping) from target torso toward attacker; red+black pairs, lingers 0.5–0.9s then fades
  - If `DivFist = true`: spawns `cursedEnergy` on punching arm + plays DF+BF combo sound
  - **Multi-hit dedup:** `lastBFVfxTime` table; if the same attacker hits a second target within 0.1s, heavy VFX are skipped for the duplicate
- **`Combat_CooldownReset`** — Resets local cooldown state via `ClientCooldowns.resetAll()`.

> **Lighting required in Studio:** `whiteFlash` (ColorCorrectionEffect), `blackFlash` (Brightness −1), `redFlash` (high saturation + red tint).

---

### Divergent Fist

**Server:** `server/combat/divergentFistSystem.luau` · **Client:** `client/combat/divergentFistClient.client.luau`

#### Server — `divergentFistSystem.luau`

5-second buff. Cannot stack — activating while active does nothing. Validates same gates as M1.

When active:
- Fires `Combat_DivergentFistBroadcast` to all clients
- Any unblocked M1 hit schedules a delayed follow-up at 1.5–4s: `DIVERGENT_FIST_DAMAGE`, hitslow, `lightKnockback`, `Combat_HitFeedback` type `"DivergentFistDelayed"`
- Black Flash landing while active: +5 BF damage, `cursedEnergy` burst client-side

| Function | Description |
| :--- | :--- |
| `request(attacker)` | Activate buff if gates pass |
| `isActive(attacker)` | Query buff state (auto-expires stale entries) |
| `resetCooldown(attacker)` | Reset buff cooldown |
| `cleanup(attacker)` | Remove state on leave |

#### Client — `divergentFistClient.client.luau`

- **Input:** Same gates as server plus local `buffActive` guard. Fires `Combat_DivergentFist`.
- **`Combat_DivergentFistBroadcast`** — Calls `VisualAura.startDivergentFist`, spawns persistent `potential` attachment (idle knuckle glow) on both fists for the buff duration.
- **`Combat_PunchBroadcast`** — If DF active: emits `cursedEnergy` on the punching arm (suppresses knuckle glow during burst, restores after).
- **`Combat_HitFeedback` (DivergentFistDelayed)** — Teal `startHit` aura + impact sound.

| Attachment | Role |
| :--- | :--- |
| `potential` | Idle knuckle glow while buff active |
| `cursedEnergy` | Per-punch burst on lower arm |

---

### Dash

**Server:** `server/combat/modules/Dashing.luau` · **Client:** `client/combat/DashingClient.client.luau` + `client/combat/StaminaWheel.client.luau`

#### Server — `Dashing.luau`

Validates state gates and `dashCDUntil` from `CooldownManager.get("dash")`. Applies a `BodyVelocity` via `BVPool`, records `lastDashServerTime` for post-dash penalty windows, applies visual endlag via `Lockouts.applyEndlag("dash")`.

#### Client — `DashingClient.client.luau`

Q-key dash. Client CD guard via `ClientCooldowns.getDash()`. Sends `hum.MoveDirection` to server. Plays dash sound and spawns dash smoke particle attachment.

#### Client — `StaminaWheel.client.luau` (Dash Bar)

Vertical yellow fill bar in a `BillboardGui` above the local player's head. Appears on dash use, fills bottom-to-top as the cooldown expires. Flickers green/yellow in the final 1 second before ready. Hides when full.

---

### Death Curse (Wizard Beam)

**Server:** `server/combat/wizardSystem.luau` · **Client:** `client/combat/spellslinger/wizardClient.client.luau`

#### Server — `wizardSystem.luau`

Tracking instant-kill lightning beam. Server finds the closest living target (players + NPCs) within `DEATH_CURSE_RANGE` studs horizontally. Fires `Combat_CooldownSync` before the target check — cooldown starts regardless of hit or miss. If no target, spell dissipates silently. Cooldown duration read from `CooldownManager.get("deathCurse")`.

On hit: `hum:TakeDamage(10000)` (instant kill). If target is blocking, damage is skipped. Fires `Combat_HitFeedback` and `Combat_DeathCurseBroadcast` to all clients.

| Function | Description |
| :--- | :--- |
| `fire(player, _, lockedOrigin)` | Full validation + target search + damage |
| `resetCooldown(player)` | Clear cooldown entry |
| `cleanup(player)` | Remove state on leave |

#### Client — `wizardClient.client.luau`

- **Input:** Gates + `ClientCooldowns.isReady("deathCurse")`. Starts `SpellCharge.begin` (green orb, `ClientCooldowns.get("deathCurseCharge")`). Fires `Combat_DeathCurseFire` after charge.
- **`Combat_DeathCurseBroadcast`** — Renders a triple-layered green `LightningBolt` between cast origin and target.
- **`Combat_HitFeedback` (DeathCurse)** — Draws a green lightning X across the target's body; plays hit sound; green `VisualAura` hit flash.

---

### Disarm Spell

**Server:** `server/combat/disarmSystem.luau` · **Client:** `client/combat/spellslinger/disarmClient.client.luau`

#### Server — `disarmSystem.luau`

Tracking beam that sets `Disarmed = true` on the target for `DISARM_SPELL_DURATION` seconds, blocking all special moves. Same charge-then-fire flow as Death Curse. Fires `Combat_CooldownSync` before target check. Cooldown duration read from `CooldownManager.get("disarmSpell")`.

| Function | Description |
| :--- | :--- |
| `fire(player, _, lockedOrigin)` | Validation + target search + disarm |
| `isReady(player)` | Returns true if cooldown has expired |
| `setCooldown(player, untilTime)` | Set cooldown end timestamp directly (used by Combo Spell) |
| `resetCooldown(player)` | Zero the cooldown entry |
| `cleanup(player)` | Remove state on leave |

#### Client — `disarmClient.client.luau`

- **Input:** Gates + `ClientCooldowns.isReady("disarmSpell")`. Starts `SpellCharge.begin` (red orb, `ClientCooldowns.get("disarmSpellCharge")`). Fires `Combat_DisarmSpellFire` after charge.
- **`Combat_DisarmSpellBroadcast`** — Renders a triple-layered red `LightningBolt`.
- **`Combat_HitFeedback` (DisarmSpell)** — Spawns a red orbiting lightning ring around the target's HRP for the disarm duration. Plays hit sound.

---

### Chain

**Server:** `server/combat/chainSystem.luau` · **Client:** `client/combat/spellslinger/chainClient.client.luau`

#### Server — `chainSystem.luau`

Beam that locks the target's movement and bounces to nearby targets. On hit: sets `Chained = true`, sets `WalkSpeed = 0` for `CHAIN_DURATION` seconds. `propagateChain` then runs every `CHAIN_BOUNCE_INTERVAL` seconds, picking a random unchained living target within `CHAIN_BOUNCE_RADIUS` of the current victim and applying the chain to them. Caps at `MAX_CHAIN_BOUNCES = 10`. Fires `Combat_CooldownSync` before target check. Cooldown duration read from `CooldownManager.get("chain")`.

| Function | Description |
| :--- | :--- |
| `fire(player, _, lockedOrigin)` | Validation + target search + chain + bounce |
| `isReady(player)` | Returns true if cooldown has expired |
| `setCooldown(player, untilTime)` | Set cooldown end timestamp directly (used by Combo Spell) |
| `resetCooldown(player)` | Zero the cooldown entry |
| `cleanup(player)` | Remove state on leave |

#### Client — `chainClient.client.luau`

- **Input:** Gates + `ClientCooldowns.isReady("chain")`. Starts `SpellCharge.begin` (blue orb, `ClientCooldowns.get("chainCharge")`). Fires `Combat_ChainFire` after charge.
- **`Combat_ChainBroadcast`** — Renders a triple-layered blue `LightningBolt` from cast origin to first target.
- **`Combat_HitFeedback` (Chain)** — Spawns a blue orbiting lightning ring around the chained target for the chain duration. Plays hit sound.
- **`Combat_HitFeedback` (ChainBounce)** — Renders a bounce beam from the previous victim to the new target + spawns swirl on new target.

---

### Combo Spell

**Server:** `server/combat/comboSpellSystem.luau` · **Client:** `client/combat/spellslinger/comboSpellClient.client.luau`

The Combo Spell fires when Disarm and Chain are pressed within `COMBO_SPELL_WINDOW` seconds of each other. It cancels both individual charges and fires a single larger purple orb. On hit it roots and disarms every target, then bounces infinitely (capped at 30 real-time seconds) between nearby victims dealing `COMBO_SPELL_DAMAGE` per hop.

#### Server — `comboSpellSystem.luau`

Gates: standard state checks + `comboCooldownEnds` + `DisarmSystem.isReady` + `ChainSystem.isReady`. Fires `Combat_CooldownSync` with `{ disarmSpell, chain, comboSpell }` before the target check so all three cooldowns advance together. Cooldown duration from `CooldownManager.get("comboSpell")`.

On hit: `applyComboHit` sets `Chained = true`, `Disarmed = true`, `WalkSpeed = 0` on the target for `COMBO_SPELL_DURATION` seconds. `propagateCombo` then bounces every `COMBO_SPELL_BOUNCE_INTERVAL` seconds to a random unchained living target within `COMBO_SPELL_BOUNCE_RADIUS`.

| Function | Description |
| :--- | :--- |
| `fire(player, _, lockedOrigin)` | Full validation + target search + hit + propagate |
| `resetCooldown(player)` | Zero combo, disarm, and chain cooldowns |
| `cleanup(player)` | Remove state on leave |

#### Client — `comboSpellClient.client.luau`

- **Combo detection:** `InputBegan` records timestamps for disarm and chain keys. If both are pressed within `COMBO_SPELL_WINDOW` and both individual cooldowns have expired (`ClientCooldowns.getRemaining > 0` check on expiry only, not pending), a combo charge begins.
- **Charge:** Sets `ComboPending` attribute on character (signals disarm/chain `SpellCharge` instances to cancel silently). Spawns a larger purple orb (2→8 studs over `ClientCooldowns.get("comboSpellCharge")`). Generation counter prevents stale fires.
- **Cancel:** Stunned / guard-broken attribute changes abort the charge.
- **Fire:** `clearPending("comboSpell")` → `Combat_ComboSpellFire:FireServer` → `startCooldown("comboSpell", comboCooldown)`.
- **`Combat_ComboSpellBroadcast`** — Renders the initial triple-layered purple `LightningBolt` from cast origin to first target.
- **`Combat_HitFeedback` (ComboSpell)** — Spawns a dual-ring CW+CCW purple swirl around the target for `COMBO_SPELL_DURATION` seconds. Plays hit sound.
- **`Combat_HitFeedback` (ComboSpellBounce)** — Renders a bounce beam from source to target + spawns swirl on new target.

---

### Posture / Block

**Server:** `server/combat/modules/Posture.luau` · **Client:** `client/combat/postureClient.client.luau` + `client/combat/PostureBar.client.luau`

#### Server — `Posture.luau`

Guard pressure bar (0 → `POSTURE_MAX`). Background loop decays linearly every 0.1s, clearing over `POSTURE_DECAY_TIME` (22s).

| Function | Description |
| :--- | :--- |
| `applyBlockedHit(char)` | +`POSTURE_BLOCK_GAIN` (10); triggers guard break at max |
| `applyCustomGain(char, amount)` | Fixed gain (Black Flash uses +30, blocked hits only) |
| `applyHitReduction(char)` | −`POSTURE_HIT_REDUCTION` (5) on unblocked hits |
| `trySetBlocking(attacker, bool)` | Set block with rate limiting and suppression checks |
| `applyGuardBreak(char, context)` | Full guard break: clears block, stun + stagger + movement lock, schedules recovery, fires feedback |

#### Client — `postureClient.client.luau`

F-key blocking. Deduplicates state changes, checks `BlockSuppressUntil`, fires `Combat_Block`, plays optimistic animation. Safety revert: if server `Blocking` attribute doesn't confirm within 0.35s, drops the animation. Syncs from `Blocking` and `GuardBroken` attribute changes for server-forced unblocks.

#### Client — `PostureBar.client.luau`

`BillboardGui` above the local player's head showing posture as a horizontal white fill bar on dark grey. Only visible while blocking. Updates on `Posture` and `Blocking` attribute changes. Has a `Heartbeat` fallback to re-create the UI if destroyed on respawn.

---

### Knockback

**Server:** `server/combat/modules/MovementMechanics.luau` · **Client:** `client/combat/knockbackClient.client.luau`

#### Server — `MovementMechanics.luau`

| Function | Description |
| :--- | :--- |
| `finisherKnockback` | Used for NPC targets; player targets receive knockback via `Combat_Knockback` remote |
| `lightKnockback` | Applied on unblocked M1 hits and Divergent Fist delayed hits |

Internal `applyKnockback` handles direction calculation, minimum separation enforcement, and method dispatch (BodyVelocity vs AssemblyLinearVelocity).

#### Client — `knockbackClient.client.luau`

Listens on `Combat_Knockback`. Creates a `BodyVelocity` with the server-sent velocity vector, parents it to `HumanoidRootPart`, destroys it after the given duration. Calls `AnimationController.cancelReaction()` to clear lingering reactions during flight.

---

### Charge Orb VFX

**Server relay:** `MainCombat` (`Combat_ChargeOrbFire` handler) · **Client (local):** `client/combat/modules/SpellCharge.luau` · **Client (remote):** `client/combat/chargeOrbClient.client.luau`

#### Client — `SpellCharge.luau`

Shared charge-then-fire lifecycle used by all spell clients (Death Curse, Disarm, Chain).

- Spawns a local neon orb that grows from 0.5→2.5 studs over `chargeTime`
- Fires `Combat_ChargeOrbFire` to server so other clients see the orb
- `generations` table (keyed by `cdKey`) prevents stale fires when a new charge starts mid-cooldown
- Listens on `Stunned` / `GuardBroken` attribute changes; calls `ChargeNotifier.cancelled` if interrupted. If cancelled by a pending combo (`ComboPending` attribute), skips `ChargeNotifier` so no penalty cooldown is applied.
- After `chargeTime`: calls `ClientCooldowns.clearPending(cdKey)` then fires the spell remote. Clearing pending before the fire ensures a server reject doesn't leave the slot in grey limbo.

#### Client — `chargeOrbClient.client.luau`

Listens on `Combat_ChargeOrbBroadcast`. For all clients except the caster (who has a local orb already), renders the charge orb with identical grow animation and charge sound. Cancels early if the caster is stunned or guard-broken. Deduplicates — destroys any existing orb for the same caster before creating a new one.

#### Server relay — `MainCombat`

Receives `Combat_ChargeOrbFire`, validates origin proximity (≤ 20 studs from root), and relays as `Combat_ChargeOrbBroadcast` to all clients.

---

### Cooldowns

**Server:** `server/combat/modules/CooldownManager.luau` · **Client:** `client/combat/modules/ClientCooldowns.luau` + `client/combat/hud/CooldownHUD.client.luau`

#### Server — `CooldownManager.luau`

Runtime cooldown and charge-time duration store. Values can be changed at runtime without touching Config (used by `FastCooldowns`). Syncs to clients via `Combat_CooldownSync` on `PlayerAdded`. Snapshots are tagged `_snapshot = true` so the HUD handler can distinguish a duration update from a spell-fired notification.

| Function | Description |
| :--- | :--- |
| `get(name)` | Returns current duration (override or default) |
| `set(name, value)` | Set a runtime override |
| `reset(name)` | Clear override, revert to default |
| `resetAll()` | Clear all overrides |
| `syncAll()` | Fire `Combat_CooldownSync` snapshot to all clients |
| `snapshot()` | Returns all current durations (tagged `_snapshot = true`) |

**Cooldown keys:** `"blackFlash"`, `"dash"`, `"dashEndlag"`, `"divergentFistDuration"`, `"divergentFistCooldown"`, `"deathCurse"`, `"disarmSpell"`, `"chain"`, `"comboSpell"`

**Charge-time keys** (synced to clients so `FastCooldowns` can reduce them): `"deathCurseCharge"`, `"disarmSpellCharge"`, `"chainCharge"`, `"comboSpellCharge"`

#### Client — `ClientCooldowns.luau`

Single source of truth for all client-side cooldown state. Spell scripts gate firing through this module; `CooldownHUD` reads it for visual display.

Two internal tables:
- `values` — duration defaults; updated when `Combat_CooldownSync` arrives (processes all keys, including `_snapshot`-tagged data)
- `expiry` — absolute `tick()` when each cooldown expires
- `pending` — boolean per key; true while a charge is in flight

| Function | Description |
| :--- | :--- |
| `get(name)` | Returns stored duration value |
| `getDash()` | Returns `dash + dashEndlag` (total reuse time) |
| `isReady(key)` | True if expiry has passed AND not pending |
| `getRemaining(key)` | Seconds until expiry (expiry only, ignores pending) |
| `startCooldown(key, duration)` | Set `expiry = tick() + duration`; clear pending |
| `setPending(key)` | Mark a charge as in-flight (blocks `isReady`) |
| `clearPending(key)` | Unmark in-flight state |
| `isPending(key)` | Query pending flag (used by HUD for grey display) |
| `resetAll()` | Zero all expiries and clear all pending (CooldownReset / profile switch) |

`Combat_CooldownSync` listener updates `values` for all received keys.

#### Client — `CooldownHUD.client.luau`

Profile-driven move slot bar anchored bottom-center of the screen. Rebuilds when the active profile changes. Each profile move gets its own slot.

**Slot states:**

| State | Appearance |
| :--- | :--- |
| Ready | Bright background, white text, green border pulse on transition |
| On cooldown | Dark background, dim text, countdown timer |
| Pending (charging) | Dark background, no timer |
| Rejected | Red border flash when pressed while on CD |
| DF buff active | Teal background + text + border, buff countdown |

**Event sources:**
- Spell broadcast remotes — call `ClientCooldowns.clearPending` + `startCooldown` on confirmed server hit
- `Combat_CooldownSync` (non-snapshot) — calls `clearPending` + `startCooldown` for the dissipate (no-target) case
- `ChargeNotifier.onCancelled` — calls `clearPending` + `startCooldown` when a charge is interrupted
- `Combat_DivergentFistBroadcast` / `Combat_DivergentFistExpired` — DF buff tracking
- `Combat_CooldownReset` — calls `ClientCooldowns.resetAll()`

On input: presses while `getRemaining > 0` trigger a red reject flash. Spell clients own `setPending` exclusively — the HUD does not call it.

On profile switch: `clearSlots()` destroys old frames and calls `ClientCooldowns.resetAll()`.

---

### Character Profiles

**Shared:** `shared/CharacterProfiles.luau` · **Client:** `client/combat/modules/ActiveProfile.luau` + `client/combat/ProfileSelectorUI.client.luau`

#### Client — `ActiveProfile.luau`

Holds the currently selected profile for the local player. All spell client scripts check `ActiveProfile.hasMove(moveId)` before handling input.

| Function | Description |
| :--- | :--- |
| `getCurrent()` | Returns active profile table |
| `setCurrent(profileId)` | Switch profile; fires `Changed` signal |
| `hasMove(moveId)` | Returns true if active profile has the move in any slot |

`ActiveProfile.Changed` — `BindableEvent.Event` fired on every profile switch.

#### Client — `ProfileSelectorUI.client.luau`

Bottom-right dropdown UI for picking the active profile. Clicking the header expands the list upward. Selected profile highlighted teal. When a different profile is selected, fires `Combat_ProfileSwitch` to the server, which resets all server-side cooldowns and kills the character for a clean respawn.

---

## Server — Core Infrastructure

### `server/MainCombat.server.luau`

Entry point. Tags every character and NPC humanoid with a stable `CombatGUID` attribute on spawn. Resets all state flags and fires `Combat_CooldownReset` to the client on `CharacterAdded`. Wires all remotes:

| Remote | Handler |
| :--- | :--- |
| `Combat_Punch` | `m1System.request` |
| `Combat_Block` | `Posture.trySetBlocking` |
| `Combat_HitReport` | `m1System.validateClientHit` |
| `Combat_Dash` | `Dashing.tryDash` |
| `Combat_BlackFlash` | `BlackFlashSystem.request` |
| `Combat_DivergentFist` | `DivergentFist.request` |
| `Combat_DeathCurseFire` | `WizardSystem.fire` |
| `Combat_DisarmSpellFire` | `DisarmSystem.fire` |
| `Combat_ChainFire` | `ChainSystem.fire` |
| `Combat_ComboSpellFire` | `ComboSpellSystem.fire` |
| `Combat_ChargeOrbFire` | Relay → `Combat_ChargeOrbBroadcast` |
| `Combat_ProfileSwitch` | Reset all CDs + fire `CooldownReset` to client + kill character |

On `PlayerRemoving`: calls `cleanup` on all system state tables. On `CharacterAdded`: calls `cleanup` (not `resetCooldown`) on spell systems so the new life starts with a clean slate, and clears all lingering combat attributes.

### `server/combat/Remotes.luau`

Creates and ensures all required `RemoteEvent` instances exist in `ReplicatedStorage`. Auto-creates any missing ones with a warning. Single source of truth for which remotes the combat system uses.

### `server/combat/modules/State.luau`

Per-combatant state store keyed by `Player` or `Model`. Holds combo index, punch timing, blocking/stunned/guardBroken flags, cooldown timers, and a per-action hit debounce map.

| Function | Description |
| :--- | :--- |
| `ensure(attacker)` | Lazy-init state record |
| `ensureForChar(char)` | Same but takes a Model (NPC support) |
| `getForChar(char)` | Find state by character model (linear scan) |
| `registerHit(attacker, actionId, target, window)` | Debounce a hit; returns false if already registered |
| `resetForRespawn(attacker)` | Clear all transient timers/flags on new character |
| `cleanup(attacker)` | Remove state entry on player leave |

### `server/combat/modules/Lockouts.luau`

Manages all movement and action restrictions. A background coroutine per affected character (20 Hz) applies WalkSpeed/JumpPower overrides and restores them when all locks expire.

| Function | Description |
| :--- | :--- |
| `applyHitslow(char, duration, isFinisher?)` | Movement-capping stun with stacking cap |
| `applyStagger(char, duration)` | Full-freeze within hitslow (only extends, never shrinks) |
| `applyEndlag(char, duration, source)` | Named attacker lock (allows selective removal) |
| `removeEndlagSource(char, source)` | Remove one specific endlag source |
| `applyMovementLock(char, duration)` | Generic movement block |
| `isHitslow(char)` | Query active hitslow |
| `isEndlag(char)` | Query any active endlag |
| `clearAll(char)` | Force-clear all endlag, hitslow, and movement locks |

**Hitslow stacking rules:** Fresh application sets full duration. Extensions add half duration (finisher adds full). Total capped at `HITSTUN_STACK_CAP`.

### `server/combat/modules/WallDaze.luau`

After a finisher knockback, `monitor(target, attacker)` attaches `Touched` listeners to all body parts for `WALL_DAZE_MONITOR_WINDOW` (0.8s). Mark a wall by parenting any Instance named `"WallDaze"` to the Part.

On wall contact: applies bonus damage, fires `Combat_HitFeedback` type `"WallHit"`, releases knockback velocity, anchors root briefly, applies hitslow + invulnerability. Ignores horizontal surfaces via a size ratio check.

### `server/combat/modules/BVPool.luau`

`BodyVelocity` object pool (max 64).

| Function | Description |
| :--- | :--- |
| `acquire(maxForce, p)` | Get from pool or create new |
| `acquireReplacing(parent, maxForce, p)` | Reuse the BV currently on a specific part |
| `release(bv)` | Return to pool (or destroy if pool full) |
| `releaseFor(parent)` | Release by parent part reference |

### `server/combat/modules/JumpLock.luau`

Jump suppression during combos. Always restores to `ROBLOX_DEFAULT_JUMPPOWER = 50`.

| Function | Description |
| :--- | :--- |
| `apply(char)` | Set JumpPower = 0 |
| `clearIfNeutral(char)` | Restore if not stunned/blocking/in-combo/in-endlag |
| `forceClear(char)` | Unconditional restore |

---

## Server — NPCs

### `server/combat/npcController.luau`

NPC lifecycle manager.

- **`register(model, options)`** — Captures a template clone for respawning, hooks `Humanoid.Died`, sets up animations, optionally starts auto-punch and auto-block loops.
- **`unregister(model)`** — Stops all tracks and disconnects signals.
- **Respawn** — Clones the original template after `respawnDelay` seconds.
- Two `Heartbeat` loops: idle/walk animation switching + auto-block; auto-punch (`m1System.request` on the NPC model).

<details>
<summary>register() options</summary>

| Option | Type | Description |
| :--- | :--- | :--- |
| `respawn` | bool | Auto-respawn on death (default true) |
| `respawnDelay` | number | Seconds before respawn (default 3) |
| `template` | Model | Custom clone source |
| `spawnCFrame` | CFrame | Spawn location override |
| `animations` | table | `{Idle, Walk}` asset IDs |
| `autoPunch` | bool | Enable simple punch loop |
| `punchInterval` | number | Seconds between punch attempts |
| `stringPause` | number | Pause after finishing a full combo |
| `autoBlock` | bool | Enable auto-block toggling |
| `autoBlockInterval` | number | Seconds between block attempts |
| `deathCleanupDelay` | number | Seconds after death before model destroyed |

</details>

### `server/combat/NPCRegistry.server.luau`

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

### `server/combat/HitboxDebug.luau`

Server-side visual debug. Pools up to 64 `Part` instances with `SelectionBox` outlines. `spawn(cframe, size, color)` shows a hitbox part for `HITBOX_DURATION` seconds then returns it to the pool.

---

## Server — Utility Scripts

> These scripts use **CollectionService tags** or are parented directly to a Part. Tag the relevant Parts in Studio.

### `server/utility/CooldownRefresh.server.luau`

**Tag:** `CooldownRefreshZone`

Touch resets all active cooldowns for the touching player: BF `lastUsed`, DF cooldown, all spell `cooldownEnds` (wizard, disarm, chain, combo), `dashCDUntil`. Also calls `Lockouts.clearAll` on the character to remove any active endlag or hitslow. Fires `Combat_CooldownReset` to that client so the HUD clears instantly. 1-second debounce per player.

**Print:** `[CooldownRefresh] All cooldowns reset for <player>`

### `server/utility/FastCooldowns.server.luau`

Touch toggles all combat cooldowns and charge times between fast values and their defaults. **ON:** sets all cooldowns to 0.5s and all charge times (`deathCurseCharge`, `disarmSpellCharge`, `chainCharge`, `comboSpellCharge`) to 0.3s. **OFF:** restores all defaults via `CooldownManager.resetAll()`. Both directions sync to all clients via `CooldownSync` snapshot, reset all active per-player cooldowns, and call `Lockouts.clearAll` on each player's character. 1-second debounce per player.

**Print:** `[FastCooldowns] ON — all CDs set to 0.5 s` / `[FastCooldowns] OFF — CDs restored`

### `server/utility/Immortality.server.luau`

**Tag:** `ImmortalityZone`

Touch **toggles** invulnerability. On activation: heals to full HP, then hooks `HealthChanged` to restore HP to max whenever it drops. On deactivation: removes `Invulnerable` attribute and disconnects the heal hook.

**Prints:** `[Immortality] ON/OFF — <player>`, `[FullHeal] <player> healed <N> HP`

### `server/utility/InvulnZone.server.luau`

**Tag:** `InvulnZone`

Touch sets or clears the `Invulnerable` attribute based on the Part's `InvulnGive` attribute.

| `InvulnGive` | Behavior |
| :---: | :--- |
| `true` | Grants `Invulnerable` on touch |
| absent / `false` | Removes `Invulnerable` on touch |

### `server/utility/FullHeal.server.luau`

Script parented directly to a `BasePart`. Touch restores the player to full health. 1-second debounce per player.

### `server/WallDazeMarker.server.luau`

Empty script. Rename it to `"WallDaze"` and place it inside a Part — `WallDaze.luau` detects it via `part:FindFirstChild("WallDaze")`.

---

## Client — Shared Utilities

### `client/combat/modules/AnimationController.luau`

Manages all combat animation tracks for the local player.

- Caches punch tracks (1–4), block track, and reaction tracks (1–4) against the current Humanoid/Animator.
- Weight-tweened cross-fading for punch transitions.
- `remoteCaches` table for other players' punch animations with manual step-based fade.
- `reset()` called by `postureClient` on `CharacterAdded`.

| Function | Description |
| :--- | :--- |
| `preloadAll()` | Load and warm-start all tracks |
| `playPunch(index)` | Cross-fade to punch animation |
| `setBlocking(bool)` | Play/stop block animation |
| `playReaction(slot)` | Play reaction animation slot |
| `playRemotePunch(model, index)` | Play punch on another player/NPC |
| `forceNeutral()` | Fade out active punch/block |
| `reset()` | Clear all tracks on respawn |

### `client/combat/modules/SoundEffects.luau`

Round-robin pool of 10 `Sound` instances parented to the local HumanoidRootPart.

- `warm(id)` pre-fetches assets by briefly playing at volume 0 to eliminate first-play latency.
- `preloadBulk(ids)` called at startup with all known sound IDs.
- `play(id)` assigns ID to the next pool slot and replays from start.
- Re-parents pool sounds to the new character root on `CharacterAdded`.

### `client/combat/visual/VisualAura.luau`

Manages `Highlight` instance lifecycle for hit flashes and arm auras.

| Function | Description |
| :--- | :--- |
| `startHit(model, params?)` | Full-body flash (fade in → hold → fade out) |
| `startPunchLimb(attacker, punchIndex, params?)` | Per-arm highlight; alternates R/L by combo index |
| `startBlackFlash(attacker, side?)` | Black fill + red outline arm aura |
| `cancelBlackFlash(attacker)` | Remove arm aura on hit |
| `startDivergentFist(attacker, duration)` | Register teal aura state for DF buff duration |
| `isDivergentFistActive(attacker)` | Query DF buff aura |
| `cancelHit(model)` | Force-disable hit aura |
| `cancelPunchLimb(attacker)` | Disable all limb highlights |
| `clearAllFor(model)` | Destroy all aura state for a model |

**Hit aura colors:**

| Event | Color |
| :--- | :--- |
| M1 hit | White |
| Blocked | Blue |
| Guard Break | Red |
| Invulnerable | Yellow |
| Black Flash hit | Black |
| Wall hit | Red (guard break color) |
| Divergent Fist delayed hit | Teal |

### `client/combat/modules/M1Buffer.luau`

Tracks a 0.3s grace window after the `Endlag` attribute clears. `blocked()` returns true while endlag is active OR within the buffer window, preventing spells from firing in the gap between client attribute update and server-side endlag.

### `client/combat/modules/ChargeNotifier.luau`

Lightweight module-level signal. `SpellCharge` calls `cancelled(cdKey, duration)` when a charge is interrupted mid-way (stunned/guard-broken). `CooldownHUD` listens via `onCancelled` to immediately call `clearPending` + `startCooldown` without waiting for a server broadcast. Not called when a charge is silently cancelled by a combo (`comboCancelled = true`).

### `client/combat/CameraClamp.client.luau`

Enforces `CAMERA_MIN_ZOOM` (5) and `CAMERA_MAX_ZOOM` (18) via `player.CameraMinZoomDistance/MaxZoomDistance`. Re-applies every `RenderStepped` as a guard against overrides.

---

## Controls

| Key | Action | Profile |
| :--- | :--- | :--- |
| LMB (hold) | M1 combo (up to 4 hits) | All |
| F (hold) | Block | All |
| Q | Dash | All |
| 1 | Black Flash | Black Sparks |
| 2 | Divergent Fist | Black Sparks |
| 1 | Death Curse (Wizard Beam) | Spellslinger |
| 2 | Disarm Spell | Spellslinger |
| 3 | Chain | Spellslinger |
| 2 + 3 (within window) | Combo Spell | Spellslinger |

---

## Remote Events Reference

All created and ensured by `server/combat/Remotes.luau`.

| Remote | Direction | Description |
| :--- | :---: | :--- |
| `Combat_Punch` | C → S | Request M1 punch |
| `Combat_Block` | C → S | Set blocking state (bool) |
| `Combat_HitReport` | C → S | Report client hitbox scan results |
| `Combat_Dash` | C → S | Request dash (Vector3 direction) |
| `Combat_BlackFlash` | C → S | Request Black Flash |
| `Combat_DivergentFist` | C → S | Request Divergent Fist buff |
| `Combat_DeathCurseFire` | C → S | Fire Death Curse after charge |
| `Combat_DisarmSpellFire` | C → S | Fire Disarm Spell after charge |
| `Combat_ChainFire` | C → S | Fire Chain after charge |
| `Combat_ComboSpellFire` | C → S | Fire Combo Spell after charge |
| `Combat_ChargeOrbFire` | C → S | Notify server of local charge orb (for relay) |
| `Combat_ProfileSwitch` | C → S | Switch active character profile |
| `Combat_PunchBroadcast` | S → All | Notify all clients of a punch (attacker, index) |
| `Combat_HitFeedback` | S → All | Hit result for VFX/audio (type, target, attacker, blocked, divFist) |
| `Combat_Reaction` | S → All | Force reaction animation on a target |
| `Combat_Knockback` | S → Target | Send knockback velocity vector to target's client |
| `Combat_BlackFlashBroadcast` | S → All | Black Flash windup + hit feedback |
| `Combat_DivergentFistBroadcast` | S → All | DF buff active (attacker, duration) |
| `Combat_DivergentFistExpired` | S → All | DF buff expired; start cooldown |
| `Combat_DeathCurseBroadcast` | S → All | Render Death Curse beam (startPos, endPos) |
| `Combat_DisarmSpellBroadcast` | S → All | Render Disarm beam + trigger swirl |
| `Combat_ChainBroadcast` | S → All | Render Chain beam; bounces via `HitFeedback` type `"ChainBounce"` |
| `Combat_ComboSpellBroadcast` | S → All | Render initial Combo Spell beam (startPos, endPos) |
| `Combat_ChargeOrbBroadcast` | S → All | Show remote charge orb on all other clients |
| `Combat_CooldownSync` | S → One/All | Cooldown + charge-time durations (tagged `_snapshot`) or spell-fired notification |
| `Combat_CooldownReset` | S → One | Reset all active client-side cooldown timers and pending state |

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
| `Disarmed` | bool | Cannot use any special moves |
| `Chained` | bool | Movement locked by Chain or Combo Spell; WalkSpeed = 0 |
| `ComboPending` | bool | Set briefly by Combo Spell client to signal disarm/chain charges to cancel silently |
| `BlockSuppressUntil` | number | Server timestamp; block raise blocked until this time |
| `PunchLockUntil` | number | Server timestamp; punches blocked until this time |
| `NPC_Controlled` | bool | Marks models managed by npcController |
