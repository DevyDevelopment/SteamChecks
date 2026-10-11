# Gorilla Guard physics-lock inventory

This inventory was checked against the installed `Assembly-CSharp.dll` with SHA-256 `91BFCEDB05B466714374A2C0B925E66A260EC6E7205CFDDFD9FE391EAA65F826` (file timestamp 2026-10-08 UTC). It records what Gorilla Guard 1.11.0 locks and why. A new Gorilla Tag build changes the signed host baseline and requires this inventory to be reviewed before a new signed guard release is made.

## Fixed values

| Runtime member | Locked value | Reason |
| --- | ---: | --- |
| `GorillaTagger.maxTagDistance` | `1.2` | Verified gameplay tag reach requested for the current game build; the serialized runtime value is authoritative over the decompiler's constructor fallback. |
| `GorillaTagger.maxStiltTagDistance` | `3.2` | Constructor-only stilt tag limit. |
| `GTPlayer.MaxArmLengthAllowed` | `1.0` | Base hand reach; writing the property also updates both internal hand-state limits. |
| `GTPlayer.unStickDistance` | `1.0` | Constructor-only locomotion setting. |
| `GTPlayer.minimumRaycastDistance` | `0.05` | Constructor-only collision setting. |
| `GTPlayer.defaultSlideFactor` | `0.03` | Constructor-only slide setting. |
| `GTPlayer.slidingMinimum` | `0.9` | Constructor-only slide threshold. |
| `GTPlayer.defaultPrecision` | `0.995` | Constructor-only collision precision. |
| `GTPlayer.teleportThresholdNoVel` | `1.0` | Constructor-only teleport/collision threshold. |
| `GTPlayer.frictionConstant` | `1.0` | Constructor-only friction setting. |
| `GTPlayer.stickDepth` | `0.01` | Constructor-only surface-contact setting. |
| `GTPlayer.siJumpMultiplier` | `1.0` | Initialized once; no trusted game call site writes it. |
| `GTPlayer.iceThreshold` | `0.9` | Constructor-only surface setting. |
| `GTPlayer.bodyLerp` | `0.17` | Constructor-only body collision setting. |
| `GTPlayer.exitMovingSurfaceThreshold` | `6.0` | Constructor-only moving-surface setting. |
| `Time.timeScale` | `1.0` | No trusted game call site writes it in this build. |
| Forest lower wall `slidePercentageOverride` | `0.55` | Known map-specific value. |
| Forest upper wall `slidePercentageOverride` | `1.0` | Known map-specific value. |

`Physics.gravity` is locked to the trusted value observed at signed-runtime startup rather than a guessed constant.

## Game-authorized dynamic values

`GTPlayer.jumpMultiplier` and `GTPlayer.maxJumpSpeed` are checked every frame. Their expected values are refreshed from `GorillaGameManager.LocalPlayerSpeed()` every 0.5 seconds. The base result is `1.1` and `6.5`; built-in infection, hunt, paintbrawl, freeze-tag, and custom-map modes legitimately return other values. This closes direct field edits without breaking those game modes.

## Per-object trusted baselines

The guard discovers loaded official objects after startup, captures supported scalar/vector configuration members, and enforces the captured values every frame. New official objects are discovered again during the 20-second audit.

- `GTPlayer.velocityLimit` and `GTPlayer.slideVelocityLimit`
- every `GorillaSurfaceOverride.extraVelMultiplier` and `extraVelMaxMultiplier`
- public float, integer, Boolean, `Vector2`, and `Vector3` fields/properties on `PlayerSwimmingParameters`
- the same supported settings on `GorillaZiplineSettings` and `GorillaRopeSwingSettings`
- selected glider lift, drag, gravity-compensation, handling, slip, and infected-speed configuration members

## Intentionally not frozen

These values are essential live state, not configuration defaults: player/hand transforms, rigidbody velocity, surface-contact state, current scale, active size volumes, active gravity-zone direction/strength, climbing/rope/zipline attachment state, water-contact state, game timers, and Photon/network state. Freezing them would damage the rig or prevent normal play.

Names such as `armBoost`, `xPred`, `tagThreshold`, `wallwalkForce`, and `momentumRamp` are cheat-menu variables, not Gorilla Tag fields with official defaults. Gorilla Guard handles their assembly/component as unapproved code instead of inventing a value to write into the game.

No finite in-process list can cover a cheat that directly writes position or velocity for one frame, replaces native code, or changes an as-yet unknown field. Those cases require server/referee-side movement validation in addition to this client guard.
