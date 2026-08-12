# Standalone Mode

In **standalone** mode the simulator and all grid services run in a **single
process**. This is the simplest setup and is ideal for trying OpenSim on your own
machine, development, or a small personal world.

Data is stored in a local **SQLite** database by default – no external database
server required.

## Steps

1. **Build** OpenSim first – see [Building from source](build-from-source.md).

2. Go to the `bin/` directory:

   ```bash
   cd bin
   ```

3. Copy the main configuration template:

   ```bash
   cp OpenSim.ini.example OpenSim.ini
   ```

4. Edit `OpenSim.ini` and check the `[Const]` section (it usually needs no change
   for a first test). Then go to the bottom `[Architecture]` section and uncomment
   the standalone include – **only one** line:

   ```ini
   [Architecture]
   Include-Architecture = "config-include/Standalone.ini"
   ```

   Use `config-include/StandaloneHypergrid.ini` instead if you want hypergrid
   (travel to/from other grids) enabled.

5. Copy the standalone common template into `config-include/`:

   ```bash
   cp config-include/StandaloneCommon.ini.example config-include/StandaloneCommon.ini
   ```

   This file defines the database and backend services. The default uses SQLite,
   which needs no setup. (You can later switch to MySQL by editing the
   `[DatabaseService]` section here.)

6. Start OpenSim:

   - **Windows:** `OpenSim.exe`
   - **Linux/macOS:** `./opensim.sh` (or `dotnet OpenSim.dll`)

7. On the **first run** OpenSim asks a few questions on the console:

   - **Region name** – pick any name (default *OpenSim Test*).
   - **Grid location (x,y)** – accept the default.
   - **External host name** – use `127.0.0.1` for local testing, or your machine's
     IP/hostname if others connect over the network.
   - **Estate** – choose *not* to join an existing estate (default) and give an
     estate name.
   - **Estate owner** – create a first/last name, password, and (optional) email.
     This account can manage the region and is your first login.

8. When you see the prompt:

   ```
   Region (My region name) #
   ```

   OpenSim is running.

## Logging in

Point a viewer at `http://127.0.0.1:9000` (see the
[User guide – adding a grid](../users/add-grid.md)). Log in with the estate owner
account you just created, or type `create user` on the console to make more accounts.

## Where data is stored

- `OpenSim.db` (SQLite) in `bin/` – regions, assets, inventory, accounts, etc.
- `Regions/Regions.ini` – region definitions.
- `OpenSim.ini` and `config-include/StandaloneCommon.ini` – your configuration.

## Shutting down

Type `quit` (or `shutdown`) at the console, or press `Ctrl+C`.

## Next

Ready for something bigger? See [Grid mode](grid-mode.md). For tweaking settings, see
[Advanced INI configuration](advanced-ini.md).
