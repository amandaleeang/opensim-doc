# Installing Firestorm

[Firestorm](https://www.firestormviewer.org/) is a free, open-source viewer based on
the Second Life viewer. It is the most widely used viewer for OpenSim grids.

> **Important:** download the **OpenSim** build of Firestorm, not the Second Life-only
> build. The SL-only build will not let you log in to OpenSim grids. On the Firestorm
> download page the OpenSim releases are clearly labelled (e.g. *"Firestorm for OpenSim"*
> / `Phoenix-FirestormOS-...`).

## Where to download

Always download Firestorm from the official site:

- Windows / macOS / Linux: <https://www.firestormviewer.org/>

Choose the release (not the beta) unless you have a reason to test pre-release code.
Firestorm will never ask for your login credentials to download the viewer – if a site
does, it is not official.

## Install on Windows

1. Run the downloaded `Phoenix-FirestormOS-...-Setup.exe`.
2. Follow the installer. The default options are fine.
3. Launch **Firestorm** from the Start menu or desktop shortcut.

## Install on macOS

1. Open the downloaded `.dmg`.
2. Drag **Firestorm** to the Applications folder.
3. Launch it from Applications. If macOS blocks it because of an unidentified
   developer, right-click the app and choose *Open* the first time.

## Install on Linux

1. Extract the downloaded `.tar.xz` archive:

   ```bash
   tar xf Phoenix-FirestormOS-*.tar.xz
   cd Phoenix-FirestormOS-*/
   ./install.sh
   ```

2. The installer places the viewer under `~/firestorm` (or similar) and adds a
   menu entry. Run it from your applications menu, or directly:

   ```bash
   ~/firestorm/firestorm
   ```

> **Note for Linux:** Firestorm needs some 32/64-bit compatibility libraries
> (e.g. `libGLU`, `libuuid`, `alsa`, `gtk`). If the viewer fails to start,
> install your distribution's "viewer/OpenGL" compatibility packages. The
> exact package names vary by distro (Debian/Ubuntu, Fedora, Arch, etc.).

## First launch

When Firestorm opens you will see the **login screen**. Before you can log in to an
OpenSim grid you must add it – see [Adding a grid and logging in](add-grid.md).

If you do not see a **grid selector** (a dropdown to pick the grid) at the bottom of
the login screen, press **Ctrl+Shift+G** to show/hide it.
