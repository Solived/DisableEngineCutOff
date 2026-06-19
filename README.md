# Disable Engine Cut Off — System Documentation
###### Standalone | Open-Source | No dependencies
## Overview

#### Vanilla Engine - Explaination:
GTA V's native flight model includes a built-in mechanic where holding the brake/reverse input (the **S** key on keyboard, or the equivalent analog brake/reverse trigger on controller) for roughly one second while flying or taxiing will automatically shut the aircraft's engine off. This is hardcoded into the base game's vehicle task system — it isn't something exposed through a single toggle native, and it isn't configurable through any standard FiveM convar.

For ground vehicles this rarely matters in practice. For aircraft, it's a constant hazard: a pilot taps the brake a moment too long during a steep dive, a precision landing, or a tight maneuver, and the engine cuts mid-air. On tactical, aviation-focused, or simulation-style servers, this turns routine flying into a recurring annoyance and, in the wrong moment, a scripted-feeling "death" that has nothing to do with player skill or combat damage.

`Disable Engine Cut Off` is a small, standalone, client-side resource that neutralizes this specific behavior — and only this behavior — scoped specifically to aircraft, without touching ground vehicles, boats, or any other engine-related system.

## How It Works:

There is no native that directly disables this mechanic. The engine shutoff is enforced internally by the game's input/vehicle-task layer every tick, based on sustained brake input while flying. Because it can't be intercepted at the source, this resource takes the standard, reliable approach used across the FiveM scripting community: **detect and correct**.

Every frame, while the local player is in the driver's seat of an eligible aircraft, the script checks the vehicle's current engine state. If it finds the engine has just turned off, it checks *why* before reacting:

- If the engine's health is above zero (i.e. it wasn't destroyed by damage or an explosion), the script assumes this is the native auto-stall and immediately forces the engine back on using `SetVehicleEngineOn`.
- If the engine's health is at or below zero (genuinely destroyed), the script does nothing — a dead engine stays dead, exactly as it should.

Because this check runs on every tick (`Wait(0)`) rather than on a delay, the correction happens within a single frame. There's no audible stutter, no visible "engine dies then restarts" moment — to the player, the engine simply never turns off from holding the brake.

A second, narrower check listens for the moment a player enters the driver's seat, to catch the edge case where the native stall completes in the same instant a player takes control (e.g. boarding an aircraft that was already mid-stall).

## Scope: Aircraft Only

This is the most important behavioral change from earlier internal revisions of this script: **it does not affect ground vehicles, boats, or any vehicle class outside the configured target classes.**

Targeting is done via `GetVehicleClass`, checked against a configurable table:

- **Class 16 — Planes** (enabled by default)
- **Class 15 — Helicopters** (disabled by default, one line to enable)

Every other vehicle class in the game is completely untouched by this resource. A player driving a car, riding a bike, or piloting a boat will experience zero difference in behavior — the script's per-tick loop checks class eligibility first and exits immediately for anything outside the configured set.

## Features

- **Class-scoped by design.** Only vehicles matching `Config.TargetClasses` are ever acted on. Default is planes only; helicopters are supported and can be enabled with a single config line.
- **Blacklist / whitelist model filtering.** `Config.ModelListMode` supports three modes:
  - `"blacklist"` (default) — every aircraft in the targeted classes is affected *except* models you list (useful for excluding a custom add-on aircraft that already has its own stall-handling logic, or any aircraft you intentionally want stall-prone).
  - `"whitelist"` — inverts this: *only* the models you list are affected, everything else in the target classes is left alone.
  - `"off"` — ignores the model list entirely and applies to every vehicle in the target classes.
  
  Model matching is done by hashing the spawn name once and storing it in a lookup table, so the per-tick cost is a single table lookup rather than a string comparison loop.
- **Driver-only scope.** The script explicitly confirms the local player is in the driver's seat (`GetPedInVehicleSeat(vehicle, -1)`) before doing anything. Passengers, NPC-piloted aircraft, and other players' vehicles are never touched by your client.
- **Damage-respecting logic.** The auto-recovery check explicitly excludes any vehicle with engine health at or below zero. A genuinely destroyed engine — from gunfire, collision, or explosion — stays dead. This script only reverses the *specific* native auto-stall, never legitimate damage states.
- **No network events, no server component.** This is a purely client-side input-correction fix. There is nothing to validate, nothing exploitable, and no server-side resource file at all — just a client script and a shared config.
- **Configurable check interval.** `Config.CheckIntervalMs` defaults to `0` (every tick), which is what produces a seamless, imperceptible correction. This is documented as the recommended setting — raising it introduces a brief, visible engine-cut before the script catches up.
- **Seat-enter recheck.** `Config.RecheckOnSeatEnter` hooks `CEventNetworkPlayerEnteredVehicle` to catch the specific edge case of boarding an aircraft mid-stall-cycle, on top of the main per-tick loop.
- **Built-in debug mode.** `Config.Debug = true` prints the model name, vehicle class, and eligibility result to console every time the player's vehicle changes — making it straightforward to verify class targeting and blacklist/whitelist behavior without reading code.
- **Zero dependencies.** Pure standalone Lua. No ESX, QBCore, vRP, or other framework required — works identically on any server setup.

## What This Script Does NOT Do

To set expectations clearly for anyone evaluating it:

- It does not affect any vehicle class outside the configured target classes (ground vehicles and boats are untouched by default).
- It does not prevent engine damage from collisions, gunfire, or explosions.
- It does not add a custom manual engine on/off keybind — players use whatever your server already provides.
- It does not touch fuel, vehicle health, or any other vehicle system beyond engine on/off state caused by the native auto-stall.
- It does not run server-side or store any persistent state — it's a pure local-client behavior correction.

## Resource Structure

```
Solived-DisableEngineCutOff/
├── fxmanifest.lua
├── config.lua
└── client/
    └── main.lua
```

A single shared config file and a single client script. No server scripts, no shared exports beyond config, no UI.

## Installation

1. Drop the `Solived-DisableEngineCutOff` folder into your `resources` directory.
2. Add `ensure Solived-DisableEngineCutOff` to your `server.cfg`.
3. Script is scoped to aircraft (configurable vehicle classes) only, with optional blacklist / whitelist filtering by model.
   Open `config.lua` and add any custom aircraft to `Config.ModelList`.
4. Start your server. No keybinds registered ~ depending on player's brake/reverse binds.

No SQL, no exports to configure, no other resources required.

## Configuration Reference

| Option | Default | Description |
|---|---|---|
| `Config.TargetClasses` | `{ [16] = true }` | Table of GTA V vehicle class IDs this resource applies to. 16 = Planes, 15 = Helicopters. |
| `Config.ModelListMode` | `"blacklist"` | `"blacklist"`, `"whitelist"`, or `"off"` — controls how `Config.ModelList` is applied. |
| `Config.ModelList` | `{}` (empty) | List of model spawn names (case-insensitive) used as either a blacklist or whitelist. |
| `Config.CheckIntervalMs` | `0` | Delay between checks while driving an eligible aircraft. `0` recommended for seamless correction. |
| `Config.RecheckOnSeatEnter` | `true` | Adds a recovery check triggered on vehicle-entry, covering a narrow edge case alongside the main loop. |
| `Config.Debug` | `false` | Prints model/class/eligibility info to console on vehicle change. Recommended only during setup. |

## Compatibility Notes

If your server runs a fuel resource that sets the engine off directly via script when fuel reaches zero, that interaction should be reviewed before deployment — the eligibility/damage check in this resource is structured so a fuel-based shutdown condition can be incorporated cleanly, but out of the box it only distinguishes "native auto-stall" from "destroyed by damage," not "out of fuel."

## Performance Profile

This resource runs one lightweight `CreateThread` loop plus a single event handler. It does no networking, no database access, no NUI, and no heavy per-frame math. For players not in an eligible aircraft, the loop overhead is a couple of cheap boolean/native checks per tick. For players actively flying an eligible aircraft, it adds two additional lightweight native calls per tick (`GetVehicleEngineHealth`, `GetIsVehicleEngineRunning`). This is well within normal tolerances for any live server and won't show up as a hotspot in profiling, even at high player counts.
