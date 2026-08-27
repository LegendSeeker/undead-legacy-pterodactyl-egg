## Distribution

This repository distributes an importable Pterodactyl egg:

- `7-days-to-die-undead-legacy.json`

The Undead Legacy game files are downloaded from their original upstream sources during server installation. No Undead Legacy assets are distributed with this egg.

<!-- BEGIN GENERATED RELEASE -->
## Current pinned release

- Undead Legacy: `2.7.18`
- 7 Days to Die Steam branch: `v2.6`
- BepInEx/Doorstop: `5.4.23.5`
- Optional patch: `2.7.15-2.7.18` for existing 2.7.15, 2.7.16, or 2.7.17 installations
- Compatibility: A new game is required if your saves are from 2.7.15 or older.
- Storage: at least 40 GiB allocated, 20 GiB free before reinstalling, plus 6.5 GiB when archive caching is enabled
<!-- END GENERATED RELEASE -->

## How the egg makes Undead Legacy work

Undead Legacy is not just a folder placed in a normal 7 Days to Die server. It needs a specific, compatible game version, the complete mod payload, BepInEx (the Unity mod loader), and a Linux loader that starts BepInEx before the game. The installation script in this egg prepares all of those pieces, and the launcher it creates checks that they really start.

### During installation

1. It installs the **7 Days to Die dedicated server** through SteamCMD, pinned to the Steam beta branch listed above. This matters because the included Undead Legacy release is built for that game version. The installer validates the Steam files and retries SteamCMD once if the first download attempt fails.

2. It downloads the two upstream Undead Legacy archives that make up the current pinned release: Experimental Part 1 and Experimental Part 2. The incremental patch listed above is optional and skipped by default for fresh installs; set `UL_APPLY_PATCH=1` only for an eligible existing installation. Each source revision and SHA-256 checksum is pinned in the egg. In plain terms, it will only install the exact files this egg was tested with; a partial download, changed file, or corrupted archive stops the installation instead of producing a subtly broken server.

3. It extracts those archives into a temporary staging area and overlays the complete result onto the server directory. Before doing that, it confirms that the essential Undead Legacy mod metadata and the `BepInEx` directory are present.

4. It records the files and directories owned by that Undead Legacy release in `.ul-state`. On a later reinstall, it removes only files listed in that record before copying the release again. This lets a reinstall repair or replace the mod payload without blindly deleting unrelated server files, saves, or user-created directories.

5. It installs the matching Linux native **Doorstop** library from the pinned BepInEx release, verifies its checksum, and places it at `doorstop_libs/libdoorstop_x64.so`. Doorstop is the native bridge that loads BepInEx into the Linux 7 Days to Die server process. Without a working Linux Doorstop library, having the Undead Legacy and BepInEx files on disk alone would not make the mod load.

6. It creates `_serverconfig.xml` from the game default on a new server, then uses that underscore-prefixed copy for future starts. This keeps the server configuration separate from Steam's default file so it survives normal game-file overlays. It also sets `EACEnabled` to `false`, because Undead Legacy requires Easy Anti-Cheat to be disabled.

7. It adds the required Linux Mono `libdl.so.2` mapping when the game's Mono configuration is present. This is a compatibility adjustment for the server's managed runtime.

8. It configures BepInEx to send important messages to the process console and disables BepInEx's separate disk logger. That gives Pterodactyl one useful log stream while retaining the important loader errors and warnings.

### When the server starts

The egg does not start `7DaysToDieServer.x86_64` directly. It runs the generated `.ul-bin/start-undead-legacy.sh` launcher, which:

- Clears inherited loader settings, so a previous or container-provided preload setting cannot interfere with this start.
- Checks that the game binary, BepInEx preloader, and verified Doorstop library exist and are readable. It also checks Doorstop's Linux shared-library dependencies before launch.
- Starts the game in dedicated, headless mode with EAC disabled, while setting the environment variables that preload Doorstop and point it at `BepInEx.Preloader.dll`. This is the step that injects BepInEx into the game process so Undead Legacy can run.
- Keeps the Pterodactyl console usable through a loopback-only Telnet/RCON connection. The Telnet port is internal to the container and should not be exposed publicly.
- Writes the combined launcher, BepInEx, and game output to `logs/latest.log`, while also showing it in the Pterodactyl console. It filters only known noisy headless Unity/Mono messages; other output, including warnings and errors, remains visible.
- Confirms that the native Doorstop library is actually mapped into the running server process, then waits for BepInEx to report `Chainloader startup complete`. Finally, it waits for 7 Days to Die to report `StartGame done` before printing `[UL-LOADER] Undead Legacy server ready`. If those checks fail or time out, it terminates the server rather than leaving a server running without a confirmed mod loader.

### Reinstalling, updating, and disk space

Use **Reinstall** to repair the exact supported combination listed above. The egg intentionally disables changing the game's Steam branch through normal update settings: changing it independently could make the game and mod incompatible.

By default, downloaded Undead Legacy archives are deleted after they are verified and installed. Set `UL_KEEP_CACHE=1` only if you want to retain the verified, release-specific archives for faster reinstalls; the current storage requirements are listed above. `UL_APPLY_PATCH` defaults to `0`; enable it only for one of the eligible source versions listed above. Follow the current compatibility note before reusing an existing save.

Players must also select the Steam beta branch listed above in their 7 Days to Die client so that their game version matches the server.

## Maintaining the egg

Release-specific values live in `release-manifest.json`; the import-ready egg and the release summary above are generated from it. To start an update, run:

```text
node src/update-release.js prepare <version>
```

Copy the exact upstream archives into the generated `.ul-release-input/<version>/` directory, complete its `release-input.json`, and apply it with:

```text
node src/update-release.js apply .ul-release-input/<version>/release-input.json
```

The updater inspects the archive layouts, calculates SHA-256 values directly from the local files, regenerates the egg and README, and validates the result. Local release inputs remain ignored by Git. In Codex, `$update-undead-legacy-egg` provides the complete repository-specific workflow.

## Disclaimer

This is a community-maintained Pterodactyl egg. It is not affiliated with or endorsed by The Fun Pimps, Subquake, Undead Legacy, BepInEx, or Pterodactyl.

Undead Legacy and its assets belong to Subquake and remain subject to the mod author’s terms of use.

This egg automates installation from upstream sources. It does not grant permission to redistribute Undead Legacy assets.

Use this egg at your own risk. Always back up important server data before updating or reinstalling.

## Credits

- [The Fun Pimps](https://7daystodie.com/) — 7 Days to Die
- [Subquake](https://community.7daystodie.com/profile/7396-subquake/) — Undead Legacy
- [BepInEx](https://github.com/BepInEx/BepInEx) — Unity mod framework
- [Pterodactyl](https://pterodactyl.io/) — game server management panel
- [ptero-eggs](https://github.com/ptero-eggs) — container ecosystem and base egg references
