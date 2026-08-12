# Console Commands & Backup

This page covers the most useful OpenSim **console commands** and how to **back up**
your world. Commands are typed at the simulator (`OpenSim`) or Robust console prompt.

## Useful console commands

| Command | What it does |
|---------|--------------|
| `help` | List available commands (context-sensitive). |
| `quit` / `shutdown` | Stop OpenSim/Robust cleanly. |
| `restart` | Restart the simulator in place. |
| `config show` | Print the effective merged configuration. |
| `show regions` | List configured regions and their status. |
| `show users` | List avatars currently connected. |
| `show assets` | Show asset cache/storage statistics. |
| `show stats` | Show simulator performance statistics. |
| `show threads` | List running threads (debugging). |
| `create user` | Create a new user account interactively. |
| `reset user password` | Reset an account's password. |
| `delete user` | Remove a user account. |
| `force update` | Force the simulator to resend prim/terrain data to viewers. |
| `backup` | Force a scene save of all changed objects to the database now. |
| `save oar` / `load oar` | Export/import a whole region to/from an `.oar` archive. |
| `save iar` / `load iar` | Export/import an avatar's inventory to/from an `.iar` archive. |
| `save prims` / `load prims` | Export/import prims (objects) to/from XML. |
| `export map` / `import map` | Save/load the region terrain heightmap. |
| `change region <name>` | Switch the console's focus to a specific region (multi-region). |
| `region restart` | Restart a single region. |
| `alert <message>` | Send an in-world alert to all connected users. |

Commands may vary slightly by version; `help` is always authoritative. Some commands
accept extra arguments (e.g. `save oar <region> <file.oar>`).

## Backing up your world

There are three layers of backup; use the ones appropriate to your situation.

### 1. Database backup (authoritative)

All persistent data (regions, assets, inventory, accounts) lives in the database.
- **SQLite:** stop OpenSim and copy `OpenSim.db` (and any `-journal` file).
- **MySQL:** use `mysqldump` (or your host's snapshot) on the OpenSim database.

This is the primary backup. The archives below are convenient but the database is the
source of truth.

### 2. Region archives (OAR)

An **OAR** is a single-file snapshot of a region (terrain, objects, parcels, scripts).

```text
Region (My region) # save oar MyRegion-2026-01-01.oar
```

Restore it later (to the same or another region):

```text
Region (My region) # load oar MyRegion-2026-01-01.oar
```

OARs are great for moving a region between installations or keeping point-in-time
snapshots.

### 3. Inventory archives (IAR)

An **IAR** snapshots an avatar's inventory. Useful before making risky changes or to
move an inventory between grids (where permitted).

```text
Region (My region) # save iar <first> <last> MyInventory.iar
```

Restore with `load iar <first> <last> MyInventory.iar`.

## Automatic backups with the AutoBackup module

OpenSim can save OARs on a schedule via the **AutoBackup** module
(`OpenSim/Region/OptionalModules/World/AutoBackup`). Enable it in `OpenSim.ini`:

```ini
[AutoBackupModule]
    AutoBackupModuleEnabled = True
    AutoBackup = True
    ; Interval in minutes between backups (default 720 = 12h)
    AutoBackupInterval = 720
    ; Directory to write OARs to (default ".")
    AutoBackupDir = "backups"
    ; Keep files for this many days (0 = keep forever)
    AutoBackupKeepFilesForDays = 7
    ; Naming: Time | Overwrite | Sequential
    AutoBackupNaming = Time
    ; Skip bundling assets into the OAR (smaller files, assets stay in DB)
    AutoBackupSkipAssets = False
    ; Optionally run a script after each backup
    ; AutoBackupScript = "backup_done.sh"
```

You can also trigger a one-off auto-backup manually on the console:

```text
Region (My region) # dooarbackup
```

This writes OARs for the region (or all regions) using the AutoBackup settings and
resets the timer.

## Backup best practice

- Schedule `AutoBackup` OARs **and** take regular database dumps.
- Store backups off the server (another disk, or offsite).
- Test a restore occasionally – an untested backup is a hope, not a backup.
- Stop the simulator (or use `backup`) before copying `OpenSim.db` to avoid a
  half-written file.
