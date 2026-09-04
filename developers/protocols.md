# Sim-to-sim and sim-to-Robust protocols

This page is a **message-level** reference: what the simulator sends, what comes
back, and how that differs on a **local grid** versus **Hypergrid (HG)**. It does
not walk through C# classes. For where the code lives, see
[Architecture](architecture.md) and [Where to look](where-to-look.md). For the
login / teleport call sequence, see [Data flow](data-flow.md).

Examples below use made-up hosts and UUIDs. Bodies are shortened; `…` means
"more of the same kind of field".

## Who talks to whom

OpenSim is two programs talking HTTP:

| Role | Process | Typical URL |
|------|---------|-------------|
| Simulator (region) | `OpenSim` | `http://sim-west.example:9000/` (the region's `ServerURI`) |
| Grid services | `Robust` | private `http://robust.example:8003`, public `http://robust.example:8002` |

On a **local grid** (no HG):

```text
Viewer ── LLUDP / CAPS ──► Simulator A
                              │
                              ├── form POST / XML ──► Robust (private port)
                              │     grid, assets, inventory, presence, …
                              │
                              └── JSON OSD ─────────► Simulator B
                                    /agent, /object, /region
```

On **Hypergrid**, the same local messages still happen. Extra public-port
messages go to the *other* grid's Robust (gatekeeper, user-agent, HG assets,
HG inventory, HG friends).

Same-process regions (two regions in one OpenSim, or standalone) skip HTTP and
call the local simulation connector. The **payload is the same OSD map**; there
is just no wire.

Default Robust ports from `Robust.ini` / `Robust.HG.ini`: **8003** private
(sim-to-Robust), **8002** public (viewer login, HG, map tiles). Optional HTTP
Basic auth can protect the private port (`AuthType = BasicHttpAuthentication`
in `[Network]`).

---

## Four encodings

Almost every inter-process call is one of these.

### 1. Form POST + XML `ServerResponse` (most Robust services)

`Content-Type: application/x-www-form-urlencoded`. One field is always
`METHOD=…`. Nested objects come back as XML with `type="List"`.

```http
POST /grid HTTP/1.1
Host: robust.example:8003
Content-Type: application/x-www-form-urlencoded

METHOD=get_region_by_name&SCOPEID=00000000-0000-0000-0000-000000000000&NAME=East
```

```xml
<?xml version="1.0"?>
<ServerResponse>
  <result type="List">
    <uuid>33333333-3333-3333-3333-333333333333</uuid>
    <regionName>East</regionName>
    <serverURI>http://sim-east.example:9000/</serverURI>
    …
  </result>
</ServerResponse>
```

Success/failure for register-style calls:

```xml
<ServerResponse><Result>Success</Result></ServerResponse>
```

```xml
<ServerResponse>
  <Result>Failure</Result>
  <Message>Region already exists</Message>
</ServerResponse>
```

Inventory uses `RESULT` (boolean) instead of `Result`.

### 2. JSON OSD maps (sim-to-sim agents and objects)

`Content-Type: application/json`. Large agent posts are **gzip-compressed**
with header `X-Content-Encoding: gzip` (not `Content-Encoding`, so older
sims do not reject the request). The JSON itself is an OSD map: strings,
UUIDs, nested maps/arrays. Vector3 values travel as strings:
`"<x, y, z>"`.

The HTTP method is sometimes a made-up verb: **`QUERYACCESS`**, not GET or POST.

### 3. XML-serialized `AssetBase` (asset service)

`GET`/`POST` `/assets` uses .NET XML serialization of `AssetBase` (metadata +
base64 `Data`). `POST /get_assets_exist` sends a `string[]` of UUIDs and
returns a `bool[]`.

### 4. XML-RPC (Hypergrid gatekeeper and user-agent)

Posted to the **public** Robust URL (no extra path). Method name in the XML-RPC
envelope, parameters as a single struct of strings.

---

## Local grid: sim → Robust

Simulator config (`config-include/GridCommon.ini`) points each connector at
`${Const|PrivURL}:${Const|PrivatePort}` (default `http://robust.example:8003`).
The connector appends a path.

| Service | Path | Typical `METHOD` values |
|---------|------|-------------------------|
| Grid | `POST /grid` | `register`, `deregister`, `get_neighbours`, `get_region_by_uuid`, `get_region_by_position`, `get_region_by_name`, `get_regions_by_name`, `get_region_range`, `get_default_regions`, `get_online_regions`, `get_hyperlinks`, `get_region_flags` |
| Presence | `POST /presence` | `login`, `logout`, `logoutregion`, `report`, `getagent`, `getagents` |
| Grid user | `POST /griduser` | `loggedin`, `loggedout`, `sethome`, `setposition`, `getgriduserinfo` |
| User accounts | `POST /accounts` | `getaccount`, `getaccounts`, `setaccount`, `createuser` |
| Friends | `POST /friends` | `getfriends`, `storefriend`, `deletefriend_string` |
| Avatar appearance | `POST /avatar` | `getavatar`, `setavatar`, `resetavatar`, `setitems`, `removeitems` |
| Auth | `POST /auth/plain` | `authenticate`, `verify`, `release` |
| Inventory | `POST /xinventory` | `GETITEM`, `GETFOLDER`, `GETROOTFOLDER`, `GETFOLDERCONTENT`, `UPDATEFOLDER`, `MOVEFOLDER`, … |
| Mute list | `POST /mutelist` | `get`, `update`, `delete` |

Assets are REST, not `METHOD=` forms (next section).

`VERSIONMIN` / `VERSIONMAX` on many calls are the **connector protocol**
integers, not the simulation-service version used in `QUERYACCESS`.

### Register this region (startup)

**Send** `POST http://robust.example:8003/grid`

```
METHOD=register
&SCOPEID=00000000-0000-0000-0000-000000000000
&VERSIONMIN=0
&VERSIONMAX=0
&uuid=22222222-2222-2222-2222-222222222222
&locX=256000
&locY=256000
&sizeX=256
&sizeY=256
&regionName=West
&serverIP=sim-west.example
&serverHttpPort=9000
&serverURI=http://sim-west.example:9000/
&serverPort=9000
&regionMapTexture=aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
&access=21
&owner_uuid=11111111-1111-1111-1111-111111111111
&Token=
```

`locX` / `locY` are **meters** (map cell × 256), not region-index cells.

**Receive**

```xml
<ServerResponse><Result>Success</Result></ServerResponse>
```

### Look up a destination region (before teleport)

**Send** `POST /grid`

```
METHOD=get_region_by_name
&SCOPEID=00000000-0000-0000-0000-000000000000
&NAME=East
```

**Receive** — one region as a nested list (or `result` = `null` if missing):

```xml
<ServerResponse>
  <result type="List">
    <uuid>33333333-3333-3333-3333-333333333333</uuid>
    <locX>256256</locX>
    <locY>256000</locY>
    <sizeX>256</sizeX>
    <sizeY>256</sizeY>
    <regionName>East</regionName>
    <serverIP>sim-east.example</serverIP>
    <serverHttpPort>9000</serverHttpPort>
    <serverURI>http://sim-east.example:9000/</serverURI>
    <serverPort>9000</serverPort>
    <access>21</access>
    <owner_uuid>11111111-1111-1111-1111-111111111111</owner_uuid>
  </result>
</ServerResponse>
```

`get_neighbours` returns `region0`, `region1`, … each `type="List"` with the
same fields.

### Presence: this avatar is in this region

**Send** `POST http://robust.example:8003/presence`

```
METHOD=report
&VERSIONMIN=0
&VERSIONMAX=0
&SessionID=44444444-4444-4444-4444-444444444444
&RegionID=22222222-2222-2222-2222-222222222222
```

**Receive**

```xml
<ServerResponse><result>Success</result></ServerResponse>
```

Login/logout of a session use `METHOD=login` / `logout` with `UserID`,
`SessionID`, `SecureSessionID`.

### Grid-user last position

**Send** `POST /griduser`

```
METHOD=setposition
&VERSIONMIN=0
&VERSIONMAX=0
&UserID=11111111-1111-1111-1111-111111111111
&RegionID=22222222-2222-2222-2222-222222222222
&Position=%3C128,%20128,%2025%3E
&LookAt=%3C1,%200,%200%3E
```

Vector3 values are the usual `<x, y, z>` strings; on a form POST the angle
brackets and spaces are URL-encoded (`%3C`, `%3E`, `%20`).

**Receive** success/failure `ServerResponse` as above.

`METHOD=getgriduserinfo` comes back with HomeRegion, LastRegion, Position,
LookAt, Online, etc.

### User account

**Send** `POST /accounts`

```
METHOD=getaccount
&ScopeID=00000000-0000-0000-0000-000000000000
&UserID=11111111-1111-1111-1111-111111111111
&VERSIONMIN=0
&VERSIONMAX=0
```

**Receive**

```xml
<ServerResponse>
  <result type="List">
    <FirstName>Alice</FirstName>
    <LastName>Resident</LastName>
    <Email></Email>
    <PrincipalID>11111111-1111-1111-1111-111111111111</PrincipalID>
    <ScopeID>00000000-0000-0000-0000-000000000000</ScopeID>
    <UserLevel>0</UserLevel>
    <UserTitle></UserTitle>
    <DisplayName></DisplayName>
    <LocalToGrid>True</LocalToGrid>
    <ServiceURLs>HomeURI*http://robust.example:8002/;AssetServerURI*http://robust.example:8002/;InventoryServerURI*http://robust.example:8002/;</ServiceURLs>
  </result>
</ServerResponse>
```

On a local grid those `ServiceURLs` are usually unused. On HG they are how a
foreign sim finds the visitor's **home** asset/inventory/friends servers.

### Friends list

**Send** `POST /friends`

```
METHOD=getfriends
&PRINCIPALID=11111111-1111-1111-1111-111111111111
```

**Receive** — `friend0`, `friend1`, … or `result` = `null`:

```xml
<ServerResponse>
  <friend0 type="List">
    <PrincipalID>11111111-1111-1111-1111-111111111111</PrincipalID>
    <Friend>55555555-5555-5555-5555-555555555555</Friend>
    <MyFlags>1</MyFlags>
    <TheirFlags>1</TheirFlags>
  </friend0>
</ServerResponse>
```

### Inventory item

**Send** `POST http://robust.example:8003/xinventory`

```
METHOD=GETITEM
&ID=66666666-6666-6666-6666-666666666666
&PRINCIPAL=11111111-1111-1111-1111-111111111111
```

**Receive**

```xml
<ServerResponse>
  <item type="List">
    <ID>66666666-6666-6666-6666-666666666666</ID>
    <AssetID>77777777-7777-7777-7777-777777777777</AssetID>
    <AssetType>6</AssetType>
    <InvType>6</InvType>
    <Name>My Box</Name>
    <Owner>11111111-1111-1111-1111-111111111111</Owner>
    <Folder>88888888-8888-8888-8888-888888888888</Folder>
    <CreatorId>11111111-1111-1111-1111-111111111111</CreatorId>
    <BasePermissions>581632</BasePermissions>
    …
  </item>
</ServerResponse>
```

Boolean inventory ops return `<RESULT>True</RESULT>` / `<RESULT>False</RESULT>`.

### Assets

**Get one asset**

```http
GET /assets/77777777-7777-7777-7777-777777777777 HTTP/1.1
Host: robust.example:8003
```

**Receive** `Content-Type: text/xml` — serialized `AssetBase`. `Data` is the
raw bytes as XML base64.

```xml
<?xml version="1.0"?>
<AssetBase>
  <Data>iVBORw0KGgoAAAANSUhEUgAA…</Data>
  <FullID>
    <Guid>77777777-7777-7777-7777-777777777777</Guid>
  </FullID>
  <ID>77777777-7777-7777-7777-777777777777</ID>
  <Name>box texture</Name>
  <Description></Description>
  <Type>0</Type>
  <Local>false</Local>
  <Temporary>false</Temporary>
  <CreatorID>11111111-1111-1111-1111-111111111111</CreatorID>
</AssetBase>
```

404 if missing. Variants:

| Request | Body |
|---------|------|
| `GET /assets/{id}/data` | raw bytes, `application/octet-stream` |
| `GET /assets/{id}/metadata` | XML `AssetMetadata` only |
| `POST /assets` | XML `AssetBase` in; XML string UUID out |
| `POST /assets/{id}` | XML `AssetBase`; updates content; XML `bool` out |
| `POST /get_assets_exist` | XML `string[]` of UUIDs in; XML `bool[]` out |

There is **no** bulk "give me these N asset bodies" call. Existence is bulk;
fetch is one UUID at a time.

### Auth token (used by some private-port calls)

**Send** `POST /auth/plain`

```
METHOD=authenticate
&PRINCIPAL=11111111-1111-1111-1111-111111111111
&PASSWORD=$1$hexmd5ofpassword
&LIFETIME=30
```

**Receive**

```xml
<ServerResponse>
  <Result>Success</Result>
  <Token>random-session-token</Token>
</ServerResponse>
```

---

## Local grid: sim → sim

Once Robust has returned `serverURI` for the destination, the source sim talks
**straight to that sim**. Robust is not in the agent-transfer path.

Base URL is the destination `ServerURI`, always with a trailing slash:
`http://sim-east.example:9000/`.

| Action | HTTP | Path |
|--------|------|------|
| Can this agent enter? | `QUERYACCESS` | `/agent/{agentID}/{regionID}/` |
| Create child/root agent | `POST` | `/agent/{agentID}/` |
| Full agent snapshot | `PUT` | `/agent/{agentID}/` (`message_type=AgentData`) |
| Position-only update | `PUT` | `/agent/{agentID}/` (`message_type=AgentPosition`) |
| Viewer arrived; release source | `DELETE` | opaque callback URI from the fat pack (`…/agent/{id}/{regionID}/release`) |
| Close leftover agent | `DELETE` | `/agent/{agentID}/{regionID}/?auth={token}` |
| Object / prim crossing | `POST` | `/object/{objectID}/` |
| Neighbour hello | `POST` | `/region/{sourceRegionID}/` |
| Friends IM-style notify | `POST` | `{ServerURI}friends` |

Simulation protocol versions currently advertised: **0.3–0.8**
(`SIMULATION/0.3` legacy string). `QUERYACCESS` negotiates the version used
for the later POST/PUT.

### Neighbour hello (region coming online)

West tells East "I exist next to you".

**Send** `POST http://sim-east.example:9000/region/22222222-2222-2222-2222-222222222222/`

```json
{
  "region_id": "22222222-2222-2222-2222-222222222222",
  "region_name": "West",
  "external_host_name": "sim-west.example",
  "http_port": "9000",
  "server_uri": "http://sim-west.example:9000/",
  "region_xloc": "256000",
  "region_yloc": "256000",
  "region_size_x": "256",
  "region_size_y": "256",
  "internal_ep_address": "0.0.0.0",
  "internal_ep_port": "9000",
  "destination_handle": "1099511627520000"
}
```

**Receive** JSON OSD of East's own `RegionInfo` (same field names), or HTTP
error. The connector treats a 2xx as success.

### QueryAccess (teleport / border-cross preflight)

**Send** `QUERYACCESS http://sim-east.example:9000/agent/11111111-1111-1111-1111-111111111111/33333333-3333-3333-3333-333333333333/`

```json
{
  "viaTeleport": true,
  "position": "<128, 20, 25>",
  "my_version": "SIMULATION/0.3",
  "simulation_service_supported_min": 0.3,
  "simulation_service_supported_max": 0.8,
  "simulation_service_accepted_min": 0.3,
  "simulation_service_accepted_max": 0.8,
  "context": {
    "InboundVersion": 0.8,
    "OutboundVersion": 0.8,
    "WearablesCount": -1
  },
  "features": [],
  "agent_home_uri": "http://robust.example:8002/"
}
```

On a purely local grid `agent_home_uri` may be omitted. HG always sends it.

**Receive**

```json
{
  "success": true,
  "reason": "",
  "version": "SIMULATION/0.8",
  "negotiated_inbound_version": 0.8,
  "negotiated_outbound_version": 0.8,
  "features": []
}
```

Denied:

```json
{
  "success": false,
  "reason": "You are not allowed on this region",
  "version": "SIMULATION/0.8",
  "negotiated_inbound_version": 0.8,
  "negotiated_outbound_version": 0.8,
  "features": []
}
```

If versions do not overlap, `reason` explains the range mismatch and the
teleport stops here.

### CreateAgent (open a circuit on the destination)

**Send** `POST http://sim-east.example:9000/agent/11111111-1111-1111-1111-111111111111/`  
usually gzip JSON.

```json
{
  "agent_id": "11111111-1111-1111-1111-111111111111",
  "first_name": "Alice",
  "last_name": "Resident",
  "circuit_code": "123456789",
  "session_id": "44444444-4444-4444-4444-444444444444",
  "secure_session_id": "99999999-9999-9999-9999-999999999999",
  "service_session_id": "",
  "caps_path": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "child": true,
  "start_pos": "<128, 20, 25>",
  "client_ip": "203.0.113.10",
  "viewer": "Firestorm-Releasex64 7.1.11.76496",
  "channel": "Firestorm-Releasex64",
  "mac": "aa:bb:cc:dd:ee:ff",
  "id0": "0123456789abcdef",
  "inventory_folder": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
  "base_folder": "00000000-0000-0000-0000-000000000000",
  "appearance_serial": 12,
  "packed_appearance": { "serial": 12, "wearables": [ "…" ], "textures": [ "…" ] },
  "serviceurls": {
    "HomeURI": "http://robust.example:8002/",
    "AssetServerURI": "http://robust.example:8002/",
    "InventoryServerURI": "http://robust.example:8002/"
  },
  "service_urls": [
    "HomeURI", "http://robust.example:8002/",
    "AssetServerURI", "http://robust.example:8002/"
  ],
  "source_x": "256000",
  "source_y": "256000",
  "source_name": "West",
  "source_uuid": "22222222-2222-2222-2222-222222222222",
  "source_server_uri": "http://sim-west.example:9000/",
  "destination_x": "256256",
  "destination_y": "256000",
  "destination_name": "East",
  "destination_uuid": "33333333-3333-3333-3333-333333333333",
  "teleport_flags": "4",
  "context": { "InboundVersion": 0.8, "OutboundVersion": 0.8, "WearablesCount": -1 }
}
```

`service_urls` (flat array) is the legacy twin of `serviceurls` (map). Both
are sent. `teleport_flags` is the numeric `TeleportFlags` bitmask (`ViaLogin`,
`ViaHGLogin`, etc.).

**Receive**

```json
{
  "success": true,
  "reason": "",
  "your_ip": "198.51.100.20"
}
```

`your_ip` is the source sim's address as seen by the destination (used by HG
1.5). On failure `success` is false and `reason` is a viewer-visible string.

The HTTP wrapper around this JSON also has a boolean `success` for "the HTTP
call worked". The inner map is the real CreateAgent result.

### UpdateAgent — full snapshot (`AgentData`)

Sent after CreateAgent, and whenever the child needs a fat update (appearance,
attachments, animations). Timeout is long (up to 200s) because the body is
large.

**Send** `PUT http://sim-east.example:9000/agent/11111111-1111-1111-1111-111111111111/`  
gzip JSON.

```json
{
  "message_type": "AgentData",
  "agent_uuid": "11111111-1111-1111-1111-111111111111",
  "session_uuid": "44444444-4444-4444-4444-444444444444",
  "circuit_code": "123456789",
  "region_id": "22222222-2222-2222-2222-222222222222",
  "position": "<250, 128, 25>",
  "velocity": "<1.2, 0, 0>",
  "center": "<250, 128, 26>",
  "size": "<0.45, 0.6, 1.8>",
  "at_axis": "<1, 0, 0>",
  "left_axis": "<0, 1, 0>",
  "up_axis": "<0, 0, 1>",
  "far": 128.0,
  "aspect": 1.333,
  "head_rotation": "<0, 0, 0, 1>",
  "body_rotation": "<0, 0, 0, 1>",
  "control_flags": "0",
  "always_run": false,
  "wait_for_root": true,
  "changed_grid": true,
  "active_group_id": "00000000-0000-0000-0000-000000000000",
  "active_group_name": "",
  "packed_appearance": { "…" : "wearables, textures, visual params" },
  "attach_objects": [
    {
      "sog": "<SceneObjectGroup>…XML2 of the attachment…</SceneObjectGroup>",
      "extra": "<extra>…</extra>",
      "modified": true,
      "state": "<script state xml or empty>"
    }
  ],
  "cb_uri": "http://sim-west.example:9000/agent/11111111-1111-1111-1111-111111111111/22222222-2222-2222-2222-222222222222/release",
  "destination_x": "256256",
  "destination_y": "256000",
  "destination_name": "East",
  "destination_uuid": "33333333-3333-3333-3333-333333333333",
  "context": { "InboundVersion": 0.8, "OutboundVersion": 0.8, "WearablesCount": -1 }
}
```

Other optional blocks: `animations`, `default_animation`, `controllers`
(scripted vehicle controls), `children_seeds`, `throttles` (binary),
`god_data`, `cfonline` (cached online friends).

**Receive** plain text, not JSON:

```
True
```

or `False`. HTTP 200 in both cases; the body is the boolean.

### UpdateAgent — position only (`AgentPosition`)

Sent often while the avatar moves near a border. Same URL, smaller body.
The sender coalesces updates per destination URI and only ships the latest.

**Send** `PUT /agent/{agentID}/`

```json
{
  "message_type": "AgentPosition",
  "agent_uuid": "11111111-1111-1111-1111-111111111111",
  "session_uuid": "44444444-4444-4444-4444-444444444444",
  "circuit_code": "123456789",
  "region_handle": "1099511627776000",
  "position": "<250, 128, 25>",
  "velocity": "<2, 0, 0>",
  "center": "<250, 128, 26>",
  "size": "<0.45, 0.6, 1.8>",
  "at_axis": "<1, 0, 0>",
  "left_axis": "<0, 1, 0>",
  "up_axis": "<0, 0, 1>",
  "far": 128.0,
  "changed_grid": false,
  "destination_x": "256256",
  "destination_y": "256000",
  "destination_name": "East",
  "destination_uuid": "33333333-3333-3333-3333-333333333333",
  "context": { "InboundVersion": 0.8, "OutboundVersion": 0.8, "WearablesCount": -1 }
}
```

**Receive** `True` / `False` as text. Repeated failures blacklist that
`ServerURI` for two minutes.

### ReleaseAgent / CloseAgent

When the viewer has connected to East, East calls the callback URI from
`cb_uri` so West can drop the root:

**Send** `DELETE http://sim-west.example:9000/agent/11111111-1111-1111-1111-111111111111/22222222-2222-2222-2222-222222222222/release`

Empty body.

**Receive**

```
OpenSim agent 11111111-1111-1111-1111-111111111111
```

To tear down a leftover child without the release dance:

**Send** `DELETE http://sim-east.example:9000/agent/11111111-1111-1111-1111-111111111111/33333333-3333-3333-3333-333333333333/?auth=circuit-auth-token`

Same text response.

### Object crossing

**Send** `POST http://sim-east.example:9000/object/{objectUUID}/`

```json
{
  "sog": "<SceneObjectGroup>…XML2 of the linkset…</SceneObjectGroup>",
  "extra": "<extra>…</extra>",
  "modified": true,
  "new_position": "<10, 128, 25>",
  "state": "<script state xml>",
  "destination_x": "256256",
  "destination_y": "256000",
  "destination_name": "East",
  "destination_uuid": "33333333-3333-3333-3333-333333333333"
}
```

`state` is only applied if the destination allows script crossings.

**Receive** `True` / `False` as text.

### Friends notify (sim → other sim)

Not Robust: West posts to East's friends handler so a local friend sees online
status or an offer.

**Send** `POST http://sim-east.example:9000/friends`

```
METHOD=status
&FromID=11111111-1111-1111-1111-111111111111
&ToID=55555555-5555-5555-5555-555555555555
&Online=true
```

Other methods: `friendship_offered`, `friendship_approved`, `friendship_denied`,
`friendship_terminated`, `grant_rights`.

**Receive** XML `ServerResponse` with `RESULT` true/false.

### Local-grid teleport, message by message

```text
West sim                         Robust                         East sim
   |                                |                               |
   | POST /grid METHOD=get_region_by_name "East"                    |
   |------------------------------->|                               |
   |  region East + serverURI       |                               |
   |<-------------------------------|                               |
   |                                                                |
   | QUERYACCESS /agent/{alice}/{east}/                             |
   |--------------------------------------------------------------->|
   |  { success, negotiated versions }                              |
   |<---------------------------------------------------------------|
   |                                                                |
   | POST /agent/{alice}/   (circuit + appearance + service URLs)   |
   |--------------------------------------------------------------->|
   |  { success, reason, your_ip }                                  |
   |<---------------------------------------------------------------|
   |                                                                |
   | PUT  /agent/{alice}/   message_type=AgentData  (fat pack)      |
   |--------------------------------------------------------------->|
   |  True                                                          |
   |<---------------------------------------------------------------|
   |                                                                |
   |                    (viewer connects to East over LLUDP/CAPS)   |
   |                                                                |
   | DELETE …/release                                               |
   |<---------------------------------------------------------------|
   |  OpenSim agent {alice}                                         |
   |--------------------------------------------------------------->|
```

Border walk is the same, except `viaTeleport` is false, CreateAgent is a
child, and position PUTs keep firing until the cross.

---

## Hypergrid: extra messages

HG does **not** replace the local-grid messages. Neighbours on the same grid
still use `/agent` as above. What HG adds:

1. **Public Robust** endpoints on port 8002 (gatekeeper, user-agent, HELO,
   HG assets, HG inventory, HG friends, IM).
2. The visitor's **home service URLs** inside CreateAgent (`serviceurls`).
3. Two extra agent paths: `/foreignagent/` (on the destination gatekeeper)
   and `/homeagent/` (on the visitor's home user-agent).
4. XML-RPC to resolve a hyperlink and to verify the agent.

Simulator architecture is `config-include/GridHypergrid.ini`:
`EntityTransferModule = HGEntityTransferModule`, plus HG inventory/asset
brokers. Robust is started with `Robust.HG.ini`.

### HELO (is this an OpenSim Robust?)

**Send** `GET` (or `HEAD`) `http://dest-robust.example:8002/helo/`

Empty body.

**Receive** HTTP 200, header:

```
X-Handlers-Provided: opensim-robust
```

Anything else (missing header, connection fail) means "not a usable HG
target".

### link_region (resolve a hyperlink name)

Posted to the **destination gatekeeper** public URL.

**Send** XML-RPC `link_region` to `http://dest-robust.example:8002/`

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>link_region</methodName>
  <params>
    <param>
      <value><struct>
        <member><name>region_name</name>
          <value><string>Lbsa Plaza</string></value></member>
      </struct></value>
    </param>
  </params>
</methodCall>
```

Empty `region_name` means "default region".

**Receive**

```xml
<methodResponse>
  <params><param><value><struct>
    <member><name>result</name><value><string>True</string></value></member>
    <member><name>uuid</name>
      <value><string>33333333-3333-3333-3333-333333333333</string></value></member>
    <member><name>handle</name>
      <value><string>1099511627776000</string></value></member>
    <member><name>size_x</name><value><string>256</string></value></member>
    <member><name>size_y</name><value><string>256</string></value></member>
    <member><name>region_image</name>
      <value><string>http://dest-robust.example:8002/map-1-1000-1000-objects.jpg</string></value></member>
    <member><name>external_name</name>
      <value><string>dest.example:8002:Lbsa Plaza</string></value></member>
  </struct></value></param></params>
</methodResponse>
```

`result` is the string `"True"` / `"False"`, not a boolean type.

### get_region (real simulator address)

**Send** XML-RPC `get_region` to the same gatekeeper:

```xml
<methodName>get_region</methodName>
<!-- struct: -->
<member><name>region_uuid</name>
  <value><string>33333333-3333-3333-3333-333333333333</string></value></member>
<member><name>agent_id</name>
  <value><string>11111111-1111-1111-1111-111111111111</string></value></member>
<member><name>agent_home_uri</name>
  <value><string>http://home-robust.example:8002</string></value></member>
```

**Receive** on success (`result` = `"true"`):

| Field | Meaning |
|-------|---------|
| `uuid` | region UUID |
| `x`, `y` | location meters |
| `size_x`, `size_y` | region size |
| `region_name` | name |
| `hostname` | sim host |
| `http_port` | sim HTTP |
| `internal_port` | LLUDP port |
| `server_uri` | sim `ServerURI` (this is where `/agent` goes) |
| `message` | optional human text |

On denial (`DisallowForeigners`, etc.) `result` is `"false"` and `message`
explains why.

### Home user-agent: "this avatar is leaving"

Before the destination will accept a foreigner, the **home** user-agent is
told. This is a CreateAgent-shaped POST, but to `/homeagent/` on home Robust.

**Send** `POST http://home-robust.example:8002/homeagent/11111111-1111-1111-1111-111111111111/`

Same JSON as local CreateAgent, **plus**:

```json
{
  "gatekeeper_serveruri": "http://dest-robust.example:8002/",
  "gatekeeper_host": "dest-robust.example",
  "gatekeeper_port": "8002",
  "destination_serveruri": "http://sim-east.example:9000/",
  "teleport_flags": "8"
}
```

(`ViaHome` / `ViaLogin` flags; the handler also checks the caller's IP.)

**Receive** same as CreateAgent: `{ "success", "reason", "your_ip" }`.

### Foreign agent: gatekeeper → destination sim

The destination **gatekeeper** (not the home sim) then creates the agent on
the destination sim. Callers POST to `/foreignagent/` on dest Robust; Robust
forwards a normal `/agent` CreateAgent to the sim.

**Send** `POST http://dest-robust.example:8002/foreignagent/11111111-1111-1111-1111-111111111111/`

Body is the same CreateAgent OSD (circuit, appearance, `serviceurls` pointing
at **home**).

**Receive** `{ "success", "reason", "your_ip" }`.

After that, fat `PUT /agent/…` and viewer connect happen **directly to the
destination sim**, same as local grid.

### Verify the visitor (destination → home)

When dest needs to prove this circuit is real:

**Send** XML-RPC `verify_agent` to `http://home-robust.example:8002/`

```xml
<member><name>sessionID</name>
  <value><string>44444444-4444-4444-4444-444444444444</string></value></member>
<member><name>token</name>
  <value><string>service-session-token</string></value></member>
```

**Receive** `{ "result": "true" }` or `"false"`.

Related XML-RPC on the home user-agent:

| Method | Request fields | Reply |
|--------|----------------|-------|
| `verify_client` | `sessionID`, `token` | `result` true/false |
| `get_home_region` | `userID` | `result`, `uuid`, `x`, `y`, `size_x`, `size_y`, `region_name`, `hostname`, `http_port`, `internal_port`, `server_uri`, `position`, `lookAt` |
| `get_server_urls` | `userID` | `SRV_HomeURI`, `SRV_AssetServerURI`, `SRV_InventoryServerURI`, `SRV_FriendsServerURI`, `SRV_IMServerURI`, … |
| `get_user_info` | `userID` | first/last name, home, … |
| `logout_agent` | `userID`, `sessionID` | `result` |
| `agent_is_coming_home` | `sessionID`, `externalName` | `result` |
| `locate_user` / `get_uui` / `get_uuid` | identity lookup | UUI / UUID strings |

### HG assets (destination sim → visitor's home)

The visitor's CreateAgent carried `serviceurls.AssetServerURI` =
`http://home-robust.example:8002/` (public HG asset connector, not the
private 8003 store).

Fetch is the **same** asset REST as local, pointed at home:

```http
GET /assets/77777777-7777-7777-7777-777777777777 HTTP/1.1
Host: home-robust.example:8002
```

**Receive** the same `AssetBase` XML. Home's HG asset service may refuse
export (`DisallowExport` in `[HGAssetService]`); then the GET is 404 / empty
and the attachment or texture never appears.

`POST /get_assets_exist` is used as a preflight. There is still no bulk body
fetch: appearance gather does many parallel single GETs.

When a local user takes a foreign object home, the sim **POSTs** `AssetBase`
XML to home `/assets` (import filter `DisallowImport` applies).

### HG inventory (suitcase)

Same `POST /xinventory` form protocol as local, but to the **public**
inventory connector (`HGInventoryService` on port 8002). `GETROOTFOLDER` for
a visitor returns the **suitcase** folder, not "My Inventory". Deletes are
ignored. Item XML looks like the local `GETITEM` example.

### HG friends

Form POST to the **public** `/hgfriends` (not `/friends`):

**Send** `POST http://home-robust.example:8002/hgfriends`

```
METHOD=getfriendperms
&PRINCIPALID=11111111-1111-1111-1111-111111111111
&FRIENDID=55555555-5555-5555-5555-555555555555
&KEY=service-session-key
&SESSIONID=44444444-4444-4444-4444-444444444444
```

Other methods: `newfriendship`, `deletefriendship`,
`validate_friendship_offered`, `statusnotification`.

**Receive** XML `ServerResponse` with a `Value` (permissions integer) or
boolean `RESULT`.

Sim-to-sim friend notifies to a **foreign** sim use the same form body as
local `/friends`, posted to `{foreignServerURI}hgfriends`.

### HG teleport, message by message

Alice on Home grid teleports to `dest.example:8002:Lbsa Plaza`.

```text
Home sim                    Home Robust                 Dest Robust                 Dest sim
   |                             |                           |                          |
   | GET dest/helo/              |                           |                          |
   |-------------------------------------------------------->|                          |
   |  X-Handlers-Provided: opensim-robust                    |                          |
   |<--------------------------------------------------------|                          |
   |                             |                           |                          |
   | XML-RPC link_region "Lbsa Plaza"                        |                          |
   |-------------------------------------------------------->|                          |
   |  uuid, handle, external_name, map image                 |                          |
   |<--------------------------------------------------------|                          |
   |                             |                           |                          |
   | XML-RPC get_region {uuid, agent_id, agent_home_uri}     |                          |
   |-------------------------------------------------------->|                          |
   |  hostname, http_port, server_uri of dest sim            |                          |
   |<--------------------------------------------------------|                          |
   |                             |                           |                          |
   | QUERYACCESS dest-sim /agent/{alice}/{region}/           |                          |
   |  (includes agent_home_uri)                              |                          |
   |----------------------------------------------------------------------------------->|
   |  success + versions                                                                |
   |<-----------------------------------------------------------------------------------|
   |                             |                           |                          |
   | POST home-robust /homeagent/{alice}/                    |                          |
   |  + gatekeeper_* + destination_serveruri                 |                          |
   |---------------------------->|                           |                          |
   |  { success }                |                           |                          |
   |<----------------------------|                           |                          |
   |                             |                           |                          |
   | POST dest-robust /foreignagent/{alice}/                 |                          |
   |  circuit + serviceurls pointing at HOME                 |                          |
   |-------------------------------------------------------->|                          |
   |                             |                           | POST dest-sim /agent/    |
   |                             |                           |------------------------->|
   |                             |                           |  { success }             |
   |                             |                           |<-------------------------|
   |  { success, your_ip }                                   |                          |
   |<--------------------------------------------------------|                          |
   |                             |                           |                          |
   | PUT dest-sim /agent/{alice}/  AgentData                 |                          |
   |----------------------------------------------------------------------------------->|
   |  True                                                                              |
   |<-----------------------------------------------------------------------------------|
   |                             |                           |                          |
   |              viewer LLUDP/CAPS to dest sim              |                          |
   |                             |                           |                          |
   |                             |     GET home-robust:8002/assets/{uuid}  (gather)     |
   |                             |<-----------------------------------------------------|
   |                             |     AssetBase XML                                    |
   |                             |----------------------------------------------------->|
```

Coming home is the reverse: dest calls home `/homeagent/` (or
`agent_is_coming_home` XML-RPC), then a normal CreateAgent on the home sim.

---

## Local grid vs HG — what actually changes

| Message | Local grid | Hypergrid |
|---------|------------|-----------|
| `POST /grid`, `/presence`, `/xinventory`, private `/assets` | sim → home Robust **:8003** | **unchanged** for local users / local regions |
| `QUERYACCESS` / `POST` / `PUT` `/agent` | sim → dest sim | same **after** gatekeeper has accepted the visitor |
| Neighbour `/region` | same-grid neighbours only | same; foreign regions are not UDP neighbours |
| CreateAgent `serviceurls` | often unused | **required**: home Asset/Inventory/Friends/IM URLs |
| Gatekeeper XML-RPC `link_region` / `get_region` | not used | first hop to a foreign name |
| `POST /homeagent/` | not used | home Robust, public port |
| `POST /foreignagent/` | not used | dest Robust, public port |
| `GET /helo/` | not used | dest Robust |
| `verify_agent` XML-RPC | not used | dest → home |
| Asset GET during appearance | private `:8003/assets` | visitor's **home public** `/assets` |
| Inventory while abroad | blocked or suitcase | public `/xinventory` on home, suitcase root |
| Friends | `POST /friends` on Robust and sims | `/hgfriends` on home public Robust + foreign sim |

Standalone Hypergrid is the same **messages** as grid HG; the "Robust"
connectors just listen on the sim's own public port instead of a separate
process.

---

## Source pointers (if a field is missing)

| Wire piece | Pack / send | Receive |
|------------|-------------|---------|
| Agent circuit (CreateAgent body) | `Framework/AgentCircuitData.cs` `PackAgentCircuitData` | `Server/Handlers/Simulation/AgentHandlers.cs` |
| Fat / position update | `Framework/ChildAgentDataUpdate.cs` `AgentData` / `AgentPosition` | same handler, `PUT` |
| HTTP verbs and paths | `Services/Connectors/Simulation/SimulationServiceConnector.cs` | `AgentHandlers.cs`, `ObjectHandlers.cs` |
| Grid / presence / inventory forms | `Services/Connectors/*` | `Server/Handlers/*` |
| HG gatekeeper XML-RPC | `Services/Connectors/Hypergrid/GatekeeperServiceConnector.cs` | `Server/Handlers/Hypergrid/HypergridHandlers.cs` |
| HG user-agent XML-RPC + `/homeagent` | `UserAgentServiceConnector.cs` | `UserAgentServerConnector.cs`, `HomeAgentHandlers.cs` |
| `/foreignagent` | `GatekeeperServiceConnector` (`AgentPath` = `foreignagent/`) | `Server/Handlers/Hypergrid/AgentHandlers.cs` |
| Compressed JSON helper | `Framework/WebUtil.cs` `PostToServiceCompressed` | `Handlers/Simulation/Utils.cs` `DeserializeJSONOSMap` |
