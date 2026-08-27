## Distribution

This repository distributes an importable Pterodactyl egg:

- `7-days-to-die-undead-legacy.json`

The Undead Legacy game files are downloaded from their original upstream sources during server installation. No Undead Legacy assets are distributed with this egg.

## How the egg makes Undead Legacy work

Undead Legacy is not just a folder placed in a normal 7 Days to Die server. It needs a specific, compatible game version, the complete mod payload, BepInEx (the Unity mod loader), and a Linux loader that starts BepInEx before the game. The installation script in this egg prepares all of those pieces, and the launcher it creates checks that they really start.

### During installation

1. It installs the **7 Days to Die dedicated server** through SteamCMD, pinned to the Steam `v2.6` beta branch. This matters because the included Undead Legacy release is built for that game version. The installer validates the Steam files and retries SteamCMD once if the first download attempt fails.

2. It downloads the two upstream Undead Legacy archives that make up the supported 2.7.17 release: Experimental Part 1 and Experimental Part 2. The official `2.7.15–2.7.17` patch is optional and is skipped by default for fresh 2.7.17 installs; set `UL_APPLY_PATCH=1` when updating an existing 2.7.15 or 2.7.16 installation. Each source revision and SHA-256 checksum is pinned in the egg. In plain terms, it will only install the exact files this egg was tested with; a partial download, changed file, or corrupted archive stops the installation instead of producing a subtly broken server.

3. It extracts those archives into a temporary staging area and overlays the complete result onto the server directory. Before doing that, it confirms that the essential Undead Legacy mod metadata and the `BepInEx` directory are present.

4. It records the files and directories owned by that Undead Legacy release in `.ul-state`. On a later reinstall, it removes only files listed in that record before copying the release again. This lets a reinstall repair or replace the mod payload without blindly deleting unrelated server files, saves, or user-created directories.

5. It installs the matching Linux native **Doorstop** library from BepInEx `5.4.23.5`, verifies its checksum, and places it at `doorstop_libs/libdoorstop_x64.so`. Doorstop is the native bridge that loads BepInEx into the Linux 7 Days to Die server process. Without a working Linux Doorstop library, having the Undead Legacy and BepInEx files on disk alone would not make the mod load.

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

Use **Reinstall** to repair this exact supported combination of 7 Days to Die `v2.6`, Undead Legacy `2.7.17`, and BepInEx/Doorstop. The egg intentionally disables changing the game's Steam branch through normal update settings: changing it independently could make the game and mod incompatible.

By default, downloaded Undead Legacy archives are deleted after they are verified and installed. Set `UL_KEEP_CACHE=1` only if you want to retain the verified, release-specific archives for faster reinstalls; it consumes about 6.5 GiB of persistent disk space. Plan for at least 40 GiB of server disk space and roughly 20 GiB free space before a reinstall, because the installer needs room for the game, downloads, and a temporary staged copy of the mod. `UL_APPLY_PATCH` defaults to `0`; enable it only when updating from 2.7.15 or 2.7.16 with the 2.7.15-to-2.7.17 incremental patch. The 2.7.17 release requires a new game when saves are from 2.7.15 or older.

Players must also select the Steam `v2.6` beta branch in their 7 Days to Die client so that their game version matches the server.

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
