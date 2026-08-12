# Grid Mode

In **grid** mode the shared grid services run in a separate **Robust** server, and
one or more simulators connect to it. This lets many regions (possibly on different
machines) share assets, inventory, and a common map/login. Two common cases:

- **Run your own grid** – you run Robust plus one or more simulators.
- **Connect to an existing grid** – you only run a simulator and point it at
  someone else's Robust (the grid operator gives you the URLs).

## A. Run your own grid (Robust + simulator)

### 1. Start Robust (the services server)

From `bin/`:

```bash
cp Robust.ini.example Robust.ini
# Linux/macOS:
./robust.sh
# Windows:
Robust.exe
```

`Robust.ini` lists which service connectors to load (asset, grid, inventory, login,
…). By default it listens on port **8002** (login) and related ports. Edit the
`[Const]` section if you need to change hostnames/ports.

### 2. Configure the simulator for grid mode

```bash
cp OpenSim.ini.example OpenSim.ini
```

In `OpenSim.ini`, at the bottom `[Architecture]` section, uncomment the grid include
(**only one**):

```ini
[Architecture]
Include-Architecture = "config-include/Grid.ini"
```

Use `config-include/GridHypergrid.ini` if you want hypergrid enabled.

Then copy and edit the grid common file:

```bash
cp config-include/GridCommon.ini.example config-include/GridCommon.ini
```

In `config-include/GridCommon.ini`, set the **service URLs** to point at your Robust
server (default `http://localhost:8002` for a single-machine grid). This is what
swaps the local standalone services for remote connectors that talk to Robust.

### 3. Start the simulator

```bash
./opensim.sh      # Linux/macOS
OpenSim.exe       # Windows
```

Create your estate owner as described in [Standalone mode](standalone.md), then log
in with the viewer pointed at your Robust login URI (usually `http://host:8002`).

Add more regions/simulators by repeating step 2–3 on other machines, all pointing at
the same Robust.

## B. Connect a simulator to an existing grid

If a grid operator invites you to host a region on their grid:

1. Get from the operator:
   - The **grid/common INI settings** (the service URLs for asset, inventory, grid,
     login, etc.), or a ready-made `GridCommon.ini`.
   - The login URI users will use.
2. Use the `Grid.ini` (or `GridHypergrid.ini`) architecture as above, but paste the
   operator's `GridCommon.ini` contents so your simulator talks to **their** Robust.
3. Start your simulator and create your region; the grid's accounts/assets are shared.

> **Follow the grid's own instructions.** Each public grid may have specific
> requirements (allowed ports, asset conventions, banned content). The steps above are
> the general pattern.

## Ports cheat-sheet

| Service | Default port |
|---------|--------------|
| Simulator (viewer/login to local sim) | 9000 |
| Robust login | 8002 |
| Robust public/services | 8003, 8004, … (see `Robust.ini`) |

Make sure these are open in any firewall and reachable from the viewers/simulators.

## Next

- Fine-tune behaviour in [Advanced INI configuration](advanced-ini.md).
- Protect your data with [Console commands & backup](console-backup.md).
