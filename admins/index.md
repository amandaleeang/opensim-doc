# Grid & Sim Admin Guide

This guide is for people who **run** OpenSimulator: installing the runtime,
building from source, configuring it, and keeping it running and backed up.

There are two programs you will deal with:

- **OpenSim** – the simulator that hosts regions and talks to viewers.
- **Robust** – the grid services server (assets, inventory, grid map, login, …) used in grid mode.

## Prerequisites

- [.NET 8.0 runtime/SDK](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) – see [Installing .NET](install-dotnet.md).
- `libgdiplus` on Linux/macOS (for image handling).
- A supported OS: Windows 10/11, Linux, or macOS.

## Sections

1. [Installing .NET](install-dotnet.md)
2. [Building OpenSim from source](build-from-source.md)
3. [Standalone mode](standalone.md) – one process, everything local (great for testing).
4. [Grid mode](grid-mode.md) – running Robust, or connecting to an existing grid.
5. [Advanced INI configuration](advanced-ini.md)
6. [Console commands & backup](console-backup.md)

New admins should go in order: install .NET, build, then set up standalone first.
