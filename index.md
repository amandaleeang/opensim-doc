# OpenSimulator Manual

Welcome to the OpenSimulator manual. OpenSimulator ("OpenSim") is an open-source
virtual-world server platform written in C#. It can run as a single standalone
simulator on your own machine, or as part of a large distributed grid.

Pick the path that matches what you want to do:

| Path | Who it is for |
|------|---------------|
| [User / Resident](users/index.md) | People who want to log in with a viewer (Firestorm), explore worlds, and pick up tips. |
| [Grid & Sim Admin](admins/index.md) | People who run OpenSim: install .NET, build from source, configure INI files, run a standalone or a grid, and back things up. |
| [Developer](developers/index.md) | People who want to read, modify, or extend the OpenSim source code. |

## What is OpenSim?

- **Server** – the OpenSim process ("the simulator") hosts one or more *regions* and talks to viewers.
- **Grid services** – a separate program called **Robust** hosts the shared grid services (assets, inventory, grid map, login, etc.) over HTTP.
- **Viewer** – a client such as [Firestorm](https://www.firestormviewer.org/) that connects to a simulator to let you move around, build, and chat.

In **standalone** mode the simulator and the grid services run in the same process.
In **grid** mode the simulator talks to a separate Robust server (or to someone else's grid).

Continue to the [User path](users/index.md) to get started logging in.
