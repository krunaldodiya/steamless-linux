# steamless — Linux wrapper for Steamless CLI

Runs `Steamless.CLI.exe` (a Windows .NET tool by [atom0s](https://github.com/atom0s/Steamless))
natively from Linux via Wine, with automatic backup, staging, and install-back of
unpacked executables.

---

## Quick start

```bash
steamless /path/to/your/game/game.exe
```

That's it. The wrapper handles everything else (see [Workflow](#workflow) below).

## Requirements

- **Wine** — already installed on this system (`wine-10.0`, CachyOS).
- **bash** — standard.
- Everything else (Steamless CLI + plugins) is bundled in this folder.

The command is available from anywhere because `~/.local/bin/steamless` symlinks to
`~/Downloads/steamless/steamless`. If you ever move this folder, re-create the link:

```bash
ln -sfn /new/path/to/steamless/steamless ~/.local/bin/steamless
```

---

## Workflow

When you give it a path **outside** the game folder
(`~/Downloads/steamless/game/`, changeable via `STEAMLESS_GAME_DIR`):

1. **Backup** — `game.exe` → `game.exe.bak` next to the original (created once,
   then reused on later runs).
2. **Stage** — a copy of the exe lands in the game folder: `game/game.exe`.
3. **Unpack** — Steamless.CLI runs on the staged copy.
4. **Install back** — on success, `game/game.exe.unpacked.exe` is moved back to the
   original path as `game.exe`.

After a successful run:

| Location | Contents |
|----------|----------|
| `/path/to/your/game/game.exe` | unpacked, DRM-free executable (playable) |
| `/path/to/your/game/game.exe.bak` | protected original (your safety net) |
| `~/Downloads/steamless/game/game.exe` | staged copy, still packed (+ any dumped files) |

### If the unpack fails

Nothing at the original location changes — the original is never moved or modified
until a successful unpack is installed back. The wrapper prints a summary with all
three paths, and **re-running the same command is always safe**.

### Safety guards

- An existing `.bak` or staged file is **never overwritten** if it differs from the
  current exe. The wrapper aborts and tells you which file is in the way.
- If it matches, it's reused silently — so retries after a failure are idempotent.
- If Steamless reports a file is not SteamStub-packed (e.g. you already unpacked it),
  it exits with "All unpackers failed to unpack file." and nothing is deployed.

---

## Options

### Wrapper options (consumed by the script)

| Option | Effect |
|--------|--------|
| `--inplace` | Skip backup/staging: run directly on the given file, output lands next to it. |
| `--deploy` | Install the unpacked file back at the original path (**default**). |
| `--no-deploy` | Keep `game.exe.unpacked.exe` in the game folder; place it yourself. |
| `-h`, `--help` | Show the Steamless usage text. |

### Steamless options (passed through to the CLI)

| Option | Effect |
|--------|--------|
| `--quiet` | Disables debug log messages. |
| `--keepbind` | Keeps the `.bind` section in the unpacked file. |
| `--keepstub` | Keeps the DOS stub in the unpacked file. |
| `--dumppayload` | Dumps the stub payload to disk. |
| `--dumpdrmp` | Dumps the `SteamDRMP.dll` to disk. |
| `--realign` | Realigns the unpacked file sections. |
| `--recalcchecksum` | Recalculates the unpacked file checksum. |
| `--exp` | Use experimental features. |

Example:

```bash
steamless --quiet --recalcchecksum ~/Games/SomeGame/SomeGame.exe
```

### Environment variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `STEAMLESS_HOME` | Steamless install directory | this script's folder |
| `STEAMLESS_GAME_DIR` | Staging folder | `~/Downloads/steamless/game` |
| `STEAMLESS_DEPLOY` | `0` = act like `--no-deploy` | `1` |
| `WINEPREFIX` | Wine prefix to use | `~/.wine` |
| `WINEDEBUG` | Wine debug channels | `-all` (quiet) |

---

## Restoring a game to its original state

```bash
cp /path/to/your/game/game.exe.bak /path/to/your/game/game.exe
```

The `.bak` is a byte-identical copy of the protected original.

---

## Troubleshooting

- **Weird/unexpected result?** Re-run with wine diagnostics enabled:

  ```bash
  WINEDEBUG=fixme-all steamless /path/to/game.exe
  ```

- **"backup exists and differs from the given file"** — you previously unpacked this
  game (or the exe changed since the backup). Either restore from the `.bak` first, or
  remove the `.bak` if you're sure you want a fresh one.
- **"a different file is already staged at ..."** — clear the staged file in the game
  folder, then re-run.
- **"All unpackers failed to unpack file."** — the exe is not SteamStub-protected
  (`--exp` won't change that) or it was already unpacked. Nothing was modified.

## Notes

- Outputs from `--dumppayload` / `--dumpdrmp` land next to the **staged** copy in the
  game folder.
- Exit codes mirror Steamless: `0` = unpacked successfully, `1` = failed.
- Under the hood the wrapper copies `Plugins/Steamless.API.dll` next to
  `Steamless.CLI.exe` if missing — Wine's Mono cannot resolve it from `Plugins/`
  at startup (it must sit next to the exe).
