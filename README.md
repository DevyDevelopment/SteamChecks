# Gorilla Guard

Gorilla Guard is a defensive BepInEx 5 guard plus a user-mode launcher for a local Gorilla Tag installation. Version 1.11.1 runs in exclusive mode: `GorillaGuard.dll` is the only executable allowed under `BepInEx/plugins`, and `GorillaGuard.Preloader.dll` is the only allowed BepInEx patcher. Other executable files are moved recoverably into `BepInEx/quarantine` before ordinary plugin discovery.

The signed release manifest covers the launcher, both guard DLLs, Doorstop bootstrap/configuration, BepInEx core code, the game executable, UnityPlayer, and deterministic hashes of the managed and native game-code trees. It accepts the two observed official native layouts: with or without the optional Discord Game SDK binary; all files that are present must still match exactly, and any addition or modification produces another tree hash and is rejected. The launcher verifies that host baseline before and after acquiring write/delete-denying file locks, starts the game through Steam, watches guarded code directories, monitors native modules, and maintains an authenticated named-pipe heartbeat with the runtime DLL.

The preloader validates a fresh launcher session and its exact executable path before the managed game assembly loads, then proves that early execution to the launcher over a kernel-PID-bound pipe. The launcher refuses the later runtime connection unless that preload proof arrived first. The runtime then maintains the continuous heartbeat, inventories managed assemblies that were already present, watches later loads, inspects Harmony's active patch table, reports high-risk managed-code events back to the launcher, and closes a directly launched/unprotected game. The launcher also checks whether an external debugger attaches. The overlay turns red for launcher, manifest, signature, preloader, or unrecoverable runtime failures.

## Continuously enforced gameplay values

- tag-manager `maxTagDistance`: `1.2`
- tag-manager `maxStiltTagDistance`: `3.2`
- player maximum arm reach: `1.0`
- Forest lower slippery wall `slidePercentageOverride`: `0.55`
- Forest upper slippery wall `slidePercentageOverride`: `1.0`

Version 1.11.1 removes 1.11.0's broad per-object physics baselines. Those baselines could capture values before Gorilla Tag finished initializing and then fight legitimate game updates, making locomotion unusable. Jump multiplier, maximum jump speed, time scale, and global gravity are now restored only after an unapproved mod/component is actually blocked; otherwise Gorilla Tag retains control of them.

The in-game frame policy accepts only 72, 80, 90, or 120 Hz/FPS targets. A reported headset refresh within 3 Hz of one of those values is accepted. Real measured FPS is displayed but transient or sustained performance drops are not treated as tampering. If the XR display reports an unsupported configured rate, Gorilla Guard asks Gorilla Tag to use the nearest allowed rate and locks `Application.targetFrameRate` plus physics timing to it. The overlay shows `VERIFIED`, `LIMITED`, or `WAITING FOR XR` with configured and measured values.

A lightweight audit runs every 20 seconds and logs to both `BepInEx/LogOutput.log` and `BepInEx/plugins/GorillaGuard.log`. Native-module enumeration and dedicated-file writes run away from Unity's main thread to avoid an audit-frame hitch; the launcher debugger/module check runs every two seconds outside Unity's main thread.

The on-screen panel keeps the three most recent successful enforcement actions. Matching log records use `BLOCKED [PLUGIN]`, `BLOCKED [COMPONENT]`, `BLOCKED [FILE]`, or `BLOCKED [VALUE]` and include the affected name, path/member, previous value when relevant, and the action taken.

See `PHYSICS_LOCKS.md` for the audited member-by-member lock inventory and the intentionally dynamic state that is not frozen.

Version 1.11.1 separates recoverable cheat blocks from fatal trust failures. Disabled plugins/components, detected managed assemblies or Harmony patches, in-use code files, and suspicious late native modules are reported to the launcher while the game remains open. If an in-use file cannot be moved, the launcher attempts to lock it against replacement and continues component, patch, and value enforcement. Because Windows cannot safely unload arbitrary native code once it has already entered the process, this state is logged as containment rather than claimed as complete removal. Invalid signatures, modified protected binaries, missing preloader/runtime authentication, lost heartbeat, watcher boundary failure, and attached debuggers still fail closed.

## Build and install

Build from this folder:

```powershell
dotnet build .\GorillaGuard.Preloader\GorillaGuard.Preloader.csproj -c Release
dotnet build .\GorillaGuard\GorillaGuard.csproj -c Release
dotnet publish .\GorillaGuard.Launcher\GorillaGuard.Launcher.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
dotnet publish .\GorillaGuard.Bootstrap\GorillaGuard.Bootstrap.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
```

## Install and run

For a first installation, distribute only `GorillaGuard.Bootstrap.exe` from the official GitHub release. It always downloads the current descriptor and detached signature from the public release channel, verifies them with its pinned RSA public key, downloads the exact package named by that signed descriptor, verifies the complete ZIP SHA-256, safely extracts it, verifies the package's internal signed manifest, and only then starts the installer. It fails closed when GitHub is unavailable or any verification fails. `--verify-only` performs this complete online verification without installing or launching the game.

The complete signed ZIP remains available for offline/manual installation. Extract it, keep `GorillaGuard.Launcher.exe` beside its packaged `BepInEx` folder, and run the EXE. The package carries the signed BepInEx 5 core needed by the guard. When it is outside the game directory it:

1. verifies the extracted signed payload before copying anything;
2. discovers Gorilla Tag in the current user's registered Steam libraries;
3. makes a recoverable backup of an existing Gorilla Guard installation;
4. installs and verifies the launcher, runtime, preloader, BepInEx core, Doorstop bootstrap/configuration, manifest, and signature; and
5. starts the installed launcher, which starts Gorilla Tag through Steam.

Afterward, run the installed `GorillaGuard.Launcher.exe` rather than Steam's Play button. `--install-only` installs without starting the game, `--verify-package` verifies an extracted package, and `--self-test` checks Windows file-lock, debugger-query, job-object, and PID-bound pipe behavior without starting Gorilla Tag.

After verification and file locking, the launcher sends a sanitized `READY` event when a relay is configured, shows a ten-second countdown, and then starts Gorilla Tag through Steam. Runtime block events and a curated value snapshot are carried to the launcher over the authenticated PID-bound pipe. The snapshot reports tag distance, both guarded slippery surfaces, time scale, and available `GTPlayer` arm-length, velocity, jump, scale, slide, and ice-threshold fields. Cheat-menu labels such as `armBoost`, `xPred`, and `wallwalkForce` are not official Gorilla Tag fields and are therefore treated as indicators of an unapproved assembly rather than invented game defaults.

## Signed updates and event relay

`GorillaGuard.remote.json` configures two optional remote services. The update URLs point to a detached RSA-signed JSON descriptor and its signature. The descriptor contains the newer version, HTTPS package URL, and package SHA-256. The installed launcher verifies that signature with its pinned release public key, refuses rollbacks, verifies the complete downloaded ZIP hash, extracts with traversal and size limits, and hands off installation only after the running launcher exits. It downloads prebuilt signed releases; it never compiles or executes arbitrary GitHub source. A public GitHub release works for anonymous clients. A private repository must instead be exposed through a narrow HTTPS update relay that keeps GitHub credentials server-side; never place a GitHub token or SSH private deploy key in this file or the launcher.

The event endpoint must be an HTTPS Gorilla Guard relay and is deliberately rejected when it points directly at Discord. Store the Discord webhook only in the relay host's `GORILLAGUARD_DISCORD_WEBHOOK_URL` environment variable. The sample `GorillaGuard.Relay` project validates a client token, rate-limits source addresses, removes Discord mentions, and forwards sanitized events. Set its public `/events` URL and the matching limited client token in `GorillaGuard.remote.json`. This telemetry is visible evidence, not remote attestation; a player controlling the PC can still forge client reports.

Manual installation uses these paths:

```powershell
Copy-Item .\GorillaGuard.Launcher\bin\Release\net8.0-windows\win-x64\publish\GorillaGuard.Launcher.exe 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\GorillaGuard.Launcher.exe'
Copy-Item .\GorillaGuard.Preloader\bin\Release\netstandard2.1\GorillaGuard.Preloader.dll 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\BepInEx\patchers\GorillaGuard.Preloader.dll'
Copy-Item .\GorillaGuard\bin\Release\netstandard2.1\GorillaGuard.dll 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\BepInEx\plugins\GorillaGuard.dll'
Copy-Item .\release\GorillaGuard.manifest 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\BepInEx\GorillaGuard.manifest'
Copy-Item .\release\GorillaGuard.manifest.sig 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\BepInEx\GorillaGuard.manifest.sig'
Copy-Item .\BepInEx\core\* 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\BepInEx\core' -Recurse -Force
Copy-Item .\winhttp.dll 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\winhttp.dll'
Copy-Item .\doorstop_config.ini 'C:\Program Files (x86)\Steam\steamapps\common\Gorilla Tag\doorstop_config.ini'
```

The launcher discovers Gorilla Tag's app ID from its local Steam manifest and supports secondary Steam library folders. Logs are written to `BepInEx/GorillaGuard.Launcher.log`, `BepInEx/LogOutput.log`, and `BepInEx/plugins/GorillaGuard.log`.

If Gorilla Tag is elsewhere, build the DLLs with `/p:GorillaTagDir="D:\path\to\Gorilla Tag"`.

Security controls and the overlay have no configuration off switch. A stale `BepInEx/config/com.local.gorillaguard.cfg` from an older release is ignored. Do not distribute `signing-private/GorillaGuard.release-private.pk8`; it is the release-signing key and is deliberately excluded from packages.

## Security boundary

A plugin running inside the game cannot guarantee prevention of native DLL injection and cannot safely classify every module by name alone. The preloader verifies the expected game field before `Assembly-CSharp` executes but deliberately does not rewrite Gorilla Tag methods or the original game DLL on disk. Gorilla Guard therefore:

- verifies the expected tag-distance field during preload without rewriting Gorilla Tag methods;
- requires an ordered, kernel-PID-bound preloader proof before accepting the runtime heartbeat and closes the game if the proof never arrives;
- verifies a signed manifest covering the launcher, runtime, and preloader before normal BepInEx plugin discovery;
- installs and verifies the package-owned Doorstop and BepInEx core, moving unexpected core executables into a recoverable backup, then verifies the signed host baseline before launch and again under file locks;
- requires a live authenticated launcher/runtime heartbeat and fails closed if either side disappears;
- binds the named-pipe client to the actual Gorilla Tag PID returned by Windows;
- locks the per-launch session file against modification, replacement, and deletion while still allowing the preloader and runtime to read it;
- attaches Gorilla Tag to a Windows kill-on-close job object so the game cannot outlive its launcher;
- holds read-only handles that deny writes and deletion for the game executable, managed/native game code, BepInEx core, the manifest, and all three guard binaries while the protected session is active;
- watches those code locations and recoverably quarantines new executable files while leaving logs, configuration, caches, saves, networking, VR, and Photon data alone;
- recoverably quarantines unexpected root-level DLL/EXE sideload candidates before starting Gorilla Tag; the allowlist is limited to Gorilla Tag, Unity, Doorstop, and Gorilla Guard bootstrap files;
- rejects redirected/reparse-point code directories instead of following them outside the game folder;
- quarantines all other executable files under `BepInEx/plugins` and `BepInEx/patchers`, preserving their relative paths for manual restoration;
- rechecks signed file integrity every five seconds and restores the trusted runtime after an on-disk swap;
- locks only the verified tag/reach/slippery-wall values, limits unsupported configured frame timing to 72/80/90/120, and leaves other locomotion, surface contact, size-change, gravity-zone, and Photon state under Gorilla Tag's control;
- disables unapproved BepInEx plugin components;
- rejects Unity components created from injected/unapproved assemblies and removes any already present;
- quarantines newly created executable code files from game code-loading folders while leaving normal logs, configuration, caches, and other game data writable;
- logs managed assemblies loaded after startup;
- evaluates managed assemblies already present when the runtime starts, including fileless assemblies that reference Gorilla Tag or BepInEx;
- inspects Harmony's global patched-method table at startup, after Harmony loads, and during the 20-second audit; only exact patch methods from the signed BepInEx preloader runtime fixes are accepted, while another observed patch is reported and contained without closing the game; an uninspectable loaded Harmony runtime remains a fatal monitoring failure;
- logs native modules first observed after startup;
- terminates the protected session when Windows reports that an external debugger is attached;
- displays its current state on screen.

This makes casual BepInEx loading, managed Harmony patching, root-directory DLL sideloading, offline host-code replacement, late file drops, launcher swapping, debugger-assisted edits, and ordinary file replacement substantially harder, but it is not a kernel anti-cheat and cannot establish remote trust. A legitimate Gorilla Tag or BepInEx code update changes the signed baseline and therefore requires a newly signed Gorilla Guard package. An administrator/kernel driver or a native manual mapper invisible to normal module enumeration can still bypass a user-controlled machine. For match enforcement, combine it with server/referee-side movement validation. Do not treat the overlay alone as proof that the process is clean.

Gorilla Tag's current official code of conduct says no cheats or mods, and its fan-content policy limits mods to approved tools/pages. Use this project only for authorized private/offline defensive testing; it is not affiliated with or endorsed by Another Axiom. See https://www.gorillatagvr.com/ and https://www.gorillatagvr.com/fan-content-mod-policy.
