# Adding a Grid and Logging In

To connect to an OpenSim grid you must tell your viewer the grid's **login URI**,
usually a one-time setup. Once added, the grid stays in your viewer's list.

## Method 1 – Preferences → OpenSim (Grid Manager)

This is the standard method in Firestorm.

1. Open Firestorm.
2. From the top menu choose **Viewer → Preferences → OpenSim** (sometimes labelled
   *Grids* / *Grid Manager* depending on the viewer).
3. Click **Add Grid** (or *Add New Grid*).
4. In the **Login URI** box, type the grid's login URI, for example:
   - OSGrid: `login.osgrid.org`
   - A simulator on your own machine: `http://127.0.0.1:9000`
   - Another grid: whatever the grid operator gave you (it looks like
     `http://server.example.com:8002` or `https://grid.example.com:9002`).
5. Click **Get Grid Info** / **Apply**. The viewer contacts the grid and fills in the
   rest of the fields (asset, inventory, map, and other service URLs) automatically.
   The grid must be **online** at this moment.
6. Click **Apply**, then **OK**.
7. Back on the login screen, pick your grid from the **grid selector** dropdown.
8. Enter your avatar's **first name** and **last name** (e.g. `John Smith`) and your
   **password**, then click **Log In**.

> **Tip:** If "Get Grid Info" errors or freezes, you can leave the extra fields empty
> and still usually log in – only some advanced features may be unavailable until the
> missing URLs are filled in.

## Method 2 – One-click `hop://` link

Many grids publish a clickable link that adds the grid to Firestorm automatically. It
looks like:

```
hop:///app/gridmanager/addgrid/http%3A%2F%2Fyour-grid.com%3A8002
```

The last part is the URL-encoded login URI (`:` → `%3A`, `/` → `%2F`). Clicking it
launches Firestorm and adds the grid. The `hop://` protocol is registered when
Firestorm is installed.

## Method 3 – Command-line / viewer parameters

If your viewer has no grid selector, or adding the grid fails, launch the viewer with
the `--loginuri` parameter:

```
Firestorm.exe --loginuri http://login.osgrid.org/
```

## Common grids to try

- **OSGrid** – the largest public test grid. Login URI: `login.osgrid.org`.
  Create a free account at <https://www.osgrid.org/> first.

## Troubleshooting login

- **Wrong grid selected** – the most common mistake. Make sure the grid dropdown on the
  login screen shows the grid you added.
- **"Could not connect" / login URI wrong** – double-check the URI spelling and the
  port. Some grids require the port even when it is the default (e.g. `:8002`, `:9002`).
- **Grid offline** – "Get Grid Info" needs the grid to be reachable. Try again later.
- **Firewalled network** – some networks block the ports. Try a different network.
- **First/last name vs username** – OpenSim uses "First Last" (two words) by default,
  not a single account name. Enter them in the two name boxes.

Once you are in, see [Tips & tricks](tips-tricks.md) for moving around and building.
