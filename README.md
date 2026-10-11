# SteamChecks release channel

This repository hosts signed Gorilla Guard release artifacts.

For a first installation, download `GorillaGuard.Bootstrap.exe` from the latest release. The bootstrapper downloads the release descriptor and detached signature fresh from this repository, verifies the pinned RSA signature and package SHA-256, safely extracts the package, and starts it only after its internal signed manifest also passes.

The launcher accepts an update only when the detached update descriptor signature matches its pinned release public key and the downloaded ZIP matches the SHA-256 recorded in that signed descriptor.

No Gorilla Tag game files, Discord webhook credentials, GitHub credentials, or release-signing private keys belong in this repository.
