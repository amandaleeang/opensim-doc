# Building OpenSim from Source

This covers getting the source and compiling it. If you just want to run it, you
still need to build (OpenSim does not ship prebuilt binaries in this repository).

## 1. Get the source

```bash
git clone git://opensimulator.org/git/opensim
cd opensim
```

Or clone from the GitHub mirror if you prefer. Use a fresh checkout – do not build
inside a directory with spaces in the path.

## 2. Generate project files

OpenSim uses `prebuild` to generate the `.csproj`/solution files from `prebuild.xml`.

**Windows:**

```cmd
runprebuild.bat
```

**Linux / macOS:**

```bash
./runprebuild.sh
```

This produces `OpenSim.sln` and the per-folder project files.

## 3. Build

```bash
dotnet build --configuration Release OpenSim.sln
```

The compiled output lands in the `bin/` directory. You can also open `OpenSim.sln`
in Visual Studio and build the solution there.

## 4. What ends up in `bin/`

After a build, `bin/` contains the executables and the default configuration files:

- `OpenSim` / `OpenSim.exe` – the simulator.
- `Robust` / `Robust.exe` – the grid services server (grid mode).
- `OpenSim.ini.example`, `OpenSimDefaults.ini` – configuration templates.
- `config-include/` – includes such as `Standalone.ini`, `Grid.ini`, and their
  `*Common.ini.example` templates.
- `Regions/` – region definitions (`Regions.ini`).
- `OpenSim.exe.config` / `Robust.exe.config` and `runtimeconfig.template.json`.

## 5. First run & configuration

You configure OpenSim by copying the `.example` files to real `.ini` files and
editing them. The two main choices are:

- **Standalone** – see [Standalone mode](standalone.md).
- **Grid** – see [Grid mode](grid-mode.md).

The build itself does not need configuration; configuration happens in `bin/` after
the build, as described in those pages.

## Troubleshooting the build

- **`dotnet: command not found`** – .NET 8 is not installed or not on `PATH`. See
  [Installing .NET](install-dotnet.md).
- **Missing `libgdiplus`** (Linux/macOS) – image-related features fail at runtime;
  install it as shown on the .NET page.
- **Build errors after a `git pull`** – remove stale build artifacts
  (`find . -name obj -type d -exec rm -rf {} +` and similar) and re-run
  `runprebuild.sh` then `dotnet build`.
