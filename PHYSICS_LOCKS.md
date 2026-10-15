# Gorilla Guard physics-lock inventory

This inventory was checked against the installed `Assembly-CSharp.dll` with SHA-256 `91BFCEDB05B466714374A2C0B925E66A260EC6E7205CFDDFD9FE391EAA65F826` (file timestamp 2026-10-08 UTC). It records what Gorilla Guard 1.11.1 locks and why. A new Gorilla Tag build changes the signed host baseline and requires this inventory to be reviewed before a new signed guard release is made.

## Fixed values

| Runtime member | Locked value | Reason |
| --- | ---: | --- |
| `GorillaTagger.maxTagDistance` | `1.2` | Verified gameplay tag reach requested for the current game build; the serialized runtime value is authoritative over the decompiler's constructor fallback. |
| `GorillaTagger.maxStiltTagDistance` | `3.2` | Constructor-only stilt tag limit. |
| `GTPlayer.MaxArmLengthAllowed` | `1.0` | Base hand reach; writing the property also updates both internal hand-state limits. |
| Forest lower wall `slidePercentageOverride` | `0.55` | Known map-specific value. |
| Forest upper wall `slidePercentageOverride` | `1.0` | Known map-specific value. |

The 1.11.0 collision, movement-asset, swimming, rope, zipline, surface-boost, glider, time-scale, and gravity baseline locks were removed because they could fight legitimate initialization and runtime changes. They remain observable in telemetry but are not continuously rewritten.

## Game-authorized dynamic values

`GTPlayer.jumpMultiplier` and `GTPlayer.maxJumpSpeed` are read from `GorillaGameManager.LocalPlayerSpeed()` and restored only after an unapproved mod/component is blocked. The base result is `1.1` and `6.5`; built-in infection, hunt, paintbrawl, freeze-tag, and custom-map modes legitimately return other values.

## Frame-rate policy

Allowed configured rates are `72`, `80`, `90`, and `120`. A headset report within `±3` Hz is accepted. Measured FPS drops are display-only evidence and never cause a block. An unsupported configured XR refresh is mapped to the nearest allowed value; Gorilla Guard uses Gorilla Tag's own refresh setter and caps/verifies `Application.targetFrameRate` in the overlay and logs. `Time.fixedDeltaTime` remains game-controlled.

On PC VR, Unity may expose the headset refresh as read-only. Gorilla Guard can request a supported rate and cap the game target but cannot reprogram every headset or SteamVR runtime mode. The overlay uses `LIMITED display -> game FPS` wording when that distinction applies.

## Intentionally not frozen

These values are essential live state, not configuration defaults: player/hand transforms, rigidbody velocity, surface-contact state, current scale, active size volumes, active gravity-zone direction/strength, climbing/rope/zipline attachment state, water-contact state, game timers, and Photon/network state. Freezing them would damage the rig or prevent normal play.

Names such as `armBoost`, `xPred`, `tagThreshold`, `wallwalkForce`, and `momentumRamp` are cheat-menu variables, not Gorilla Tag fields with official defaults. Gorilla Guard handles their assembly/component as unapproved code instead of inventing a value to write into the game.

No finite in-process list can cover a cheat that directly writes position or velocity for one frame, replaces native code, or changes an as-yet unknown field. Those cases require server/referee-side movement validation in addition to this client guard.
