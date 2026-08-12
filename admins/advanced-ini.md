# Advanced INI Configuration

OpenSim's configuration is a layered set of `.ini` files merged at startup. This page
explains how they fit together and which knobs matter most.

## How configuration is loaded

When OpenSim (or Robust) starts, it builds one merged configuration from, in order:

1. **`OpenSimDefaults.ini`** – shipped defaults. Don't edit this; it is overwritten
   on upgrade. Your settings override it.
2. **`OpenSim.ini`** (or `Robust.ini`) – your main file. Most things are commented out
   here, meaning "use the default". Uncomment a line to override it.
3. **`config-include/`** – included files referenced from `OpenSim.ini` via
   `Include-Architecture` and other `Include-*` keys (e.g. `StandaloneCommon.ini`,
   `GridCommon.ini`, `FlotsamCache.ini`, `osslEnable.ini`). These are where the
   database, services, and feature toggles live.
4. **Environment variables** and **command-line arguments** – merged on top.

To see the *effective* configuration after merging, type `config show` on the
console.

## The `[Const]` section

`OpenSim.ini` has an optional `[Const]` section that defines variables you can reuse
elsewhere with `${VarName}`. A common one is the base hostname/port used by
`config-include` files. Check it matches your machine.

## Choosing an architecture

At the bottom of `OpenSim.ini`, the `[Architecture]` section's
`Include-Architecture` line picks the topology. **Uncomment exactly one:**

```ini
Include-Architecture = "config-include/Standalone.ini"
Include-Architecture = "config-include/StandaloneHypergrid.ini"
Include-Architecture = "config-include/Grid.ini"
Include-Architecture = "config-include/GridHypergrid.ini"
```

Each pulls in the matching `*Common.ini` via `config-include/`.

## Database

In `config-include/StandaloneCommon.ini` (standalone) or `GridCommon.ini` (grid),
the `[DatabaseService]` section chooses the storage backend:

- **SQLite** (default) – file-based, zero setup, fine for testing/small use.
- **MySQL / MariaDB** – set `StorageProvider = OpenSim.Data.MySQL.dll` and fill in
  `ConnectionString`. Needed for larger grids. The MySQL driver is built in.

## Modules & features

The `[Modules]` section enables or disables optional region modules, for example:

```ini
[Modules]
AssetServices = "AssetServices"          ; local asset service (standalone)
; ScriptEngine = "YEngine"               ; LSL/XMR scripting engine
```

Many optional features live in `OpenSim/Region/OptionalModules` and are turned on by
adding the relevant module/section. Search the `.example` files for the feature name.

### OSSL (OpenSim Scripting Language) permissions

`config-include/osslEnable.ini` (and `osslDefaultEnable.ini`) control which OSSL
functions scripts may call. By default most OSSL functions are blocked for safety;
enable them per-function with `Allow_` / `Allow_All` settings.

### Asset cache (Flotsam)

`config-include/FlotsamCache.ini` configures the Flotsam asset cache, which keeps
frequently used assets in memory/on disk so regions load faster. Tune
`CacheDirectory`, `CacheWarnAt`, and `CacheSize` for your RAM.

## Networking & ports

- `[Network]` in `OpenSim.ini` sets the simulator's `http_listener_port` (default
  **9000**) and `default_location` for regions.
- Each region in `Regions/Regions.ini` has its own `InternalPort` and
  `ExternalHostName`. For viewers on other machines, `ExternalHostName` must be the
  public IP/hostname, not `127.0.0.1`.
- Robust ports are set in `Robust.ini` (`[Startup]` / service connectors), typically
  8002 (login) and 8003+ (services).

## Hypergrid

Use the `*Hypergrid.ini` architecture and the corresponding `*Common` file. Hypergrid
lets avatars teleport between independent grids. It requires the grid's `Gatekeeper`
and `HG` services enabled and proper DNS/port exposure.

## Tips

- Change **one thing at a time** and restart, so you know what each setting does.
- Keep your edited files (`OpenSim.ini`, `*Common.ini`) out of version control if you
  share the source tree; they contain your topology.
- After editing, `config show` on the console confirms what OpenSim actually loaded.
