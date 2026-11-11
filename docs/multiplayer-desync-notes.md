# Multiplayer desync notes (release build)

Working notes from cooperative debugging of the mod against the Steam
release (app 5160800, custom Godot 4.7.2 build). Mostly discovered the
hard way on 2026-11-10.

## Debugging setup

- Launch both instances on one PC: host through the launcher, second
  instance with `SteamAppId=5160800 SteamGameId=5160800 FindTheNeedle.exe`,
  join via direct IP 127.0.0.1:7777.
- Set `MP_DESYNC=1` for: per-process trace files (`user://mp_trace_<pid>.log`),
  prop checksum ticks, payload notes on placement ops, machine discovery dumps.
- Both sides log `[host/<pid>]` / `[guest/<pid>]` tagged lines; compare the two
  files at matching timestamps to catch divergence.
- `MP_DEBUG_PROPS=1` enables the mod's older verbose prop logging.

## Hard-won engine/game facts

- **`builds.to_array()` has side effects.** The game's build serializer
  normalizes build-system state while serializing. Calling it on a timer
  (we tried 1 Hz) keeps resetting the power grid and machines - even solo.
  It is safe occasionally (join-time full sync), never poll it.
- **The item-added signal fires before `add_child`.** A freshly spawned prop
  is outside the scene tree; `global_position` reads as (0,0,0). Defer any
  position-dependent handling until `is_inside_tree()`.
- **`CLIENT_FROZEN` does not stop client machines.** `set_process(false)`
  stops `_process`, but machines also react to property setters. Syncing a
  `stored` counter into a client machine can trigger its consume-and-eject
  logic, spawning ghost output locally.
- Client machines also produce ghosts when the host stops (full belt) and
  the client's stale state keeps "exporting" - those ghosts became real
  props on the host (664 in one session) until suppressed.
- Build dicts (`builds.to_array()`) carry content state: `cycle_recipe`,
  `queued`, `queued_needles`, `out_left`, `jam`, `member_priority`
  (power_tags), `cannon_queue` (scanner), shape keys (span/sections/rows).
  This state rides the join-time full sync but has no incremental channel.
- **Generic build-dict content sync is unsafe.** Diffing whole dicts and
  demolish-rebuilding on change destabilizes the sim (volatile fields, guest
  state laundering). If content sync returns, it must be a targeted
  whitelist (recipe/jam/contents) with the volatile-key and rate filters.
- Release machine groups seen so far: `conveyors`, `corners`, `platforms`,
  `paint_boards`, `power_poles`, `water_mains`, `power_tags` (exported);
  cabinets are the storage the UI calls "depot" - their contents are NOT
  under any common field name (`stored`, `contents`, `items`... all null).

## Fixes in place (this branch)

- Guest pickup of a puppet prop re-adopts the same id (claim-on-replace)
  instead of minting a duplicate.
- Client-side producer double-spawn suppressed: a local spawn matching a
  just-received foreign puppet (same item, <1.5 m, <2 s) is removed.
- Guest machine-origin spawns suppressed (<3.5 m of a machine, no player
  within 6 m, not held) - kills ghost exports onto full belts.
- Prop move updates carry linear_velocity; smoothing dead-reckons 0.1 s
  ahead so eject arcs track the host.
- Machine fields heartbeat: host resends a full snapshot every 2.5 s so
  client-mutated counters self-correct.
- New prop handling deferred until inside the scene tree.

## Open items

- Depot/cabinet contents sync (field name still unknown - probe returned
  nothing for common names; needs deeper introspection).
- Fabricator recipe sync: field is `cycle_recipe`; wire a targeted,
  event-driven sync for it (and `jam`) - not the generic channel.
- Belt-pickup identity: belt records are anonymous; picking a rider off a
  belt on a guest mints a new id (duplication). Needs mp-ids stamped into
  belt record state.
- Drones: frozen on clients, no pathing sync.
- Delivery counter (350-rod progress), delivery bay position offset,
  compressor double eject animation (cosmetic), carry crash
  (`p_surface` material errors - 300+ per session, likely release model).
