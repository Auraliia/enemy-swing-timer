# CLAUDE.md — Boss Swing & Cast Timer addon (working title)

> Handoff context from a planning session. This is the project's starting brief.
> Update it as decisions change. Keep under ~200 lines.

## Goal

A WoW addon for **healers** that shows the **next melee swing** and **current cast**
of the relevant enemy, so heals can be timed to land right after a boss swing/cast.
The pain point with existing tools: they're locked to a single unit (e.g. target-of-target)
and aren't customizable enough.

## Platform

- WoW **Classic client**, **Anniversary realms** (progressing through TBC content).
- TOC `## Interface:` MUST match the live client build. Get it in-game:
  `/run print((select(4, GetBuildInfo())))` — do not hardcode from memory; it bumps each patch.
- Plan for multi-flavor TOC suffixes (e.g. `_TBC`, `_Cata`) as Anniversary advances.

## Form factor decision

**Standalone addon, not a WeakAura.** Reasons: needs a real options panel
(priority list + enable/disable toggles) and CurseForge/Wago distribution.

Stack:
- **Ace3**: AceAddon, AceConfig/AceGUI (options), AceDB (saved settings), AceEvent.
- **LibSharedMedia-3.0** for bar textures.
- A status-bar approach for rendering (LibCandyBar or hand-rolled StatusBar frames).

**Prototype first**: validate swing estimation + unit resolution in live combat
before building the full options UI. That's the only risky part.

## Core challenge 1 — Unit resolver (the main differentiator)

Replace the typical single-unit dropdown with a **priority resolver**, re-evaluated on
`PLAYER_TARGET_CHANGED`, `PLAYER_FOCUS_CHANGED`, `UNIT_TARGET`:

- target is **hostile** → watch `target`
- target is **friendly** (e.g. the tank) → watch `targettarget`
- fallback → `focus` (or focus's target)

Each source should be individually enable/disable-able, with configurable priority order.
Resolver outputs a single unit token + its GUID, consumed by both the swing and cast logic.

## Core challenge 2 — Cast bar (native, easy)

Modern Classic client has **native enemy cast bars**.
- Use `UnitCastingInfo(unit)` / `UnitChannelInfo(unit)` for spell name, icon, start/end time.
- `UNIT_SPELLCAST_*` events fire for `target`, `focus`, `nameplateN` — but NOT `targettarget`.
- For the `targettarget` case: **poll** `UnitCastingInfo`/`UnitChannelInfo` on the resolved
  unit via a lightweight OnUpdate (polling one unit per frame is cheap).
- Caveats: `notInterruptible` is always nil in Classic (uninterruptible styling unreliable);
  spell rank can't be distinguished.

## Core challenge 3 — Swing estimator (hybrid; beats the reference WA)

No API predicts the next enemy swing. Two strategies exist; we use **both**:
- **Seed** instantly from `UnitAttackSpeed(unit)` → no warmup, first swing already predicted.
- **Correct** continuously toward the *observed* interval between swings, measured from
  `COMBAT_LOG_EVENT_UNFILTERED` `SWING_DAMAGE`/`SWING_MISSED` filtered by the unit GUID.
  Reject outliers (boss pauses to cast, resets) above a threshold.

This lets us **delete** the reference WA's manual attack-speed-debuff table AND its hardcoded
parry-immune NPC list — both become unnecessary. Bar resets on `PLAYER_REGEN_DISABLED`.

**Verify in prototype:** how accurate `UnitAttackSpeed` is on a live boss (base vs. with
active haste/slow). The reference WA needed a manual slow table, implying it's not fully dynamic.
The hybrid is robust either way.

## Bars (visuals)

- Configurable: size, position, texture (LibSharedMedia), fill direction (L→R / R→L).
- Overlaid text: spell/source name, total duration, remaining.
- Separate styling for swing bar vs cast bar.

## Carry over from the reference WA (Wago JY04q6095)

- **Keep**: stash the player's own cast end-time (`playerCastTime`) from the player's
  `UNIT_SPELLCAST_START`/`_DELAYED`. This is the seed for the real end-goal: aligning
  when YOUR heal lands against the predicted swing.
- **Keep**: parry-haste model (reduce remaining by ~40% of weapon speed, floored near 20%).
- **Reimplement, don't copy**: the WA's parry block looks dead (it keys state by GUID
  `a[a8]` while the swing bar is stored under `a[""]` — keys never match).
- The WA ignores off-hand swings entirely (sidesteps dual-wield rather than solving it).
- Translation notes for porting its logic: `WA_GetUnitBuff` → `UnitAura`/`AuraUtil.ForEachAura`;
  `aura_env` → the addon's namespace table.

## v1 scope / explicit non-goals

- **No latency compensation** in v1 (planned for v2; can reuse logic from Quartz et al.).
- Dual-wield enemies treated as single-weapon — known imperfect.
- Uninterruptible-cast styling not reliable (Classic API limitation).

## Dev workflow

- Edit Lua/XML → `/reload`. TOC changes or new files may need a full relog.
- Install **BugGrabber + BugSack** to surface errors. Use `/eventtrace`, `ViragDevTool`.
- Reference addons (CHECK LICENSES before lifting code): BossSwingTimer, EnemySwing
  (Treeston), Quartz (swing module), ClassicCastbars (cast patterns, now native).

## Distribution

- GitHub repo + **BigWigsMods packager** GitHub Action + `.pkgmeta` → CurseForge / Wago Addons.
- CurseForge requires project approval.

## Suggested first steps in Claude Code

1. Scaffold: TOC + Ace3 embeds + AceDB defaults + a stub options panel.
2. Build the unit resolver as an isolated, testable module (print resolved unit on target change).
3. Build the hybrid swing estimator; print predicted vs. actual swing times to validate.
4. Wire one swing bar to the resolved unit. Then add the cast bar (native + targettarget poll).
5. Only then expand the options UI (textures, direction, per-source toggles, priority order).
