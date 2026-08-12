# Installing .NET

OpenSimulator requires **.NET 8.0**. The exact package you need depends on what you
are doing:

- **Running** a prebuilt or self-built OpenSim → the **.NET 8.0 Runtime**.
- **Building** OpenSim from source → the **.NET 8.0 SDK** (which includes the runtime).

> OpenSim no longer runs on Mono. Use the official .NET 8.0 from Microsoft.

## Windows

1. Download the .NET 8.0 SDK (or Runtime) from
   <https://dotnet.microsoft.com/en-us/download/dotnet/8.0>.
2. Run the installer and follow the prompts.
3. Verify:

   ```powershell
   dotnet --version
   ```

   It should print something like `8.0.xxx`.

Optionally install [Visual Studio 2022 or later](https://visualstudio.microsoft.com/)
if you want to build/debug using the IDE.

## Linux

Install the .NET 8.0 SDK. On Debian/Ubuntu the quick route is the Microsoft package
feed; alternatively use your distro's packages if they provide .NET 8.

You also need `libgdiplus` (for image handling). If you previously had Mono 6.x
complete, you already have it.

```bash
# Debian/Ubuntu example
sudo apt-get update
sudo apt-get install -y apt-utils libgdiplus libc6-dev
```

Then install the .NET 8 SDK following Microsoft's instructions for your distro
(<https://learn.microsoft.com/dotnet/core/install/linux>), then verify:

```bash
dotnet --version
```

## macOS

1. Download the .NET 8.0 SDK for macOS from
   <https://dotnet.microsoft.com/en-us/download/dotnet/8.0> (the `.pkg` installer).
2. Run it and follow the prompts.
3. Install `libgdiplus` via [Homebrew](https://brew.sh/) (or MacPorts):

   ```bash
   brew install mono-libgdiplus
   ```

4. Verify:

   ```bash
   dotnet --version
   ```

## Next

Once `dotnet --version` reports 8.0.x, continue to
[Building OpenSim from source](build-from-source.md).
