# Architecture Overview

This page maps the OpenSim source tree to concepts. Paths are under `OpenSim/`.

## Two executables, two entry points

| Program | `Main()` location | Output | Role |
|---------|-------------------|--------|------|
| **OpenSim** (simulator) | `Region/Application/Application.cs` → `Application.Main()` | `bin/OpenSim` (`OpenSim.dll`) | Hosts regions/scenes, talks to viewers |
| **Robust** (grid services) | `Server/ServerMain.cs` → `OpenSimServer.Main()` | `bin/Robust` (`Robust.dll`) | Hosts grid services over HTTP |

### Simulator startup

`Application.Main()` parses command-line switches into a Nini config source, then
constructs `OpenSim` / `OpenSimBackground` and calls `.Startup()`. The real bootstrap
lives in:

- `Region/Application/OpenSim.cs`
- `Region/Application/OpenSimBase.cs` (base class, drives plugin/module loading)
- `Region/Application/ConfigurationLoader.cs` (builds the merged config – see
  [Advanced INI](../admins/advanced-ini.md))

### Robust startup

`ServerMain.Main()` creates `HttpServerBase`, reads the `[Startup] ServiceConnectors`
list from `Robust.ini`, and for each connector string calls
`ServerUtils.LoadPlugin<IServiceConnector>(...)`. Each connector registers HTTP
handlers. Robust is a thin host; the logic is in the service connectors under
`Server/Handlers/`.

## The central `Scene`

A *region* is represented by the `Scene` class:

- `Region/Framework/Scenes/Scene.cs` – `public partial class Scene : SceneBase`.
  It is a **partial class** split across many files (`Scene.Inventory.cs`,
  `Scene.Permissions.cs`, `Scene.PacketHandlers.cs`, …).
- `Region/Framework/Scenes/SceneBase.cs` – `abstract class SceneBase : IScene`.
  `IScene` (`Region/Framework/IScene.cs`) is the public contract for a region.
- Nearby classes in the same folder:
  - `SceneManager.cs` – singleton managing all scenes (`SceneManager.Instance`).
  - `SceneGraph.cs` – entities/objects within a scene.
  - `ScenePresence.cs` – an avatar/agent in the scene (`EntityBase, IScenePresence`).
  - `SceneObjectGroup.cs` / `SceneObjectPart.cs` – prims and linksets.

## The module system (region behaviour)

Almost all region behaviour is implemented as **region modules**. The interfaces live
in `Region/Framework/Interfaces/`:

- `IRegionModuleBase` – base of every region module. Members: `Name`,
  `ReplaceableInterface` (a `Type` that lets one implementation replace another),
  `Initialise(IConfigSource)`, `Close()`, `AddRegion(Scene)`, `RemoveRegion(Scene)`,
  `RegionLoaded(Scene)`.
- `ISharedRegionModule` – `: IRegionModuleBase` + `PostInitialise()`. **One instance
  shared by all scenes.**
- `INonSharedRegionModule` – `: IRegionModuleBase`. **One new instance per scene.**

### Loading & registration

`ApplicationPlugins/RegionModulesController/RegionModulesControllerPlugin.cs`
(`RegionModulesControllerPlugin`) is itself an *application plugin* (see below). Its
`AddRegionToModules(Scene)` is the heart of the loader: it adds shared module
instances, creates a fresh non-shared instance per scene, handles
`ReplaceableInterface` de-duplication, and finally calls `RegionLoaded` on all modules
(so modules can safely fetch each other's interfaces).

Modules publish capabilities with `scene.RegisterModuleInterface<T>(this)` and consume
them with `scene.RequestModuleInterface<T>()`. Process-wide services go on
`OpenSimBase.ApplicationRegistry` (an `IRegistryCore`).

### Where features live

`Region/CoreModules/` holds most region behaviour, grouped by area:

- `Agent/` – asset transactions, texture sender, xfer, IPBan.
- `Avatar/` – attachments, avatar factory, chat, friends, groups, IM, inventory,
  profile, gods.
- `World/` – terrain, land/parcels, estate, objects, permissions, OAR archiver,
  wind, map, sound.
- `Scripting/` – XMLRPC, HTTP request, LSLHttp, dynamic texture, world comm.
- `Framework/` – entity transfer, caps, inventory access, library, user management,
  service throttle, monitoring.
- `ServiceConnectorsIn/` and `ServiceConnectorsOut/` – see next section.

## Services vs Connectors

Folder `Services/` separates **interfaces**, **local implementations**, and **remote
connectors**:

- `Services/Interfaces/` – pure contracts: `IAssetService`, `IGridService`,
  `IInventoryService`, `IUserAccountService`, `IAvatarService`, `IPresenceService`,
  `ILoginService`, `ISimulationService`, `IAuthenticationService`, etc. Both the
  simulator and Robust reference only these.
- `Services/<Name>Service/` – **local, authoritative** implementations that persist to
  the database via `Data/`. Example: `Services/AssetService/AssetService.cs`,
  `Services/LLLoginService/LLLoginService.cs`. These run **inside Robust** in grid
  mode, or **inside the simulator** in standalone mode.
- `Services/Connectors/` – **remote proxies** implementing the *same interfaces*,
  marshalling calls over HTTP to a remote service (e.g.
  `Connectors/Asset/AssetServicesConnector.cs`,
  `Connectors/Simulation/SimulationServiceConnector.cs`). Base class:
  `Connectors/BaseServiceConnector.cs`.

### How standalone vs grid swaps local for remote

This is driven by **INI config**, not code changes: on the simulator, the
`ServiceConnectorsOut` modules choose in-process vs remote. For simulation:

- `CoreModules/ServiceConnectorsOut/Simulation/LocalSimulationConnector.cs`
  (`LocalSimulationConnectorModule`) – implements `ISimulationService` for regions
  **in this process**. Enabled in standalone.
- `CoreModules/ServiceConnectorsOut/Simulation/RemoteSimulationConnector.cs`
  (`RemoteSimulationConnectorModule`) – used in grid mode; delegates local regions to
  the local connector and remote regions to the HTTP `SimulationServiceConnector`.

On Robust, the matching HTTP endpoint is exposed by an *in-connector*, e.g.
`Server/Handlers/Simulation/SimulationServiceInConnector.cs` and
`Server/Handlers/Login/LLLoginServiceInConnector.cs`.

## Application plugins (bootstrap)

Implement `IApplicationPlugin` (`Region/Application/IApplicationPlugin.cs`), discovered
via Mono.Addins at extension path `/OpenSim/Startup`, and loaded by
`OpenSimBase.LoadPlugins()`. Key plugins in `ApplicationPlugins/`:

- **RegionModulesController** – loads/manages all region modules.
- **LoadRegions** (`LoadRegions/LoadRegionsPlugin.cs`) – picks a region loader
  (`RegionLoaderFileSystem` or `RegionLoaderWebServer`), loads estates, and creates
  each region. Registers `IRegionCreator`.
- **RemoteController** – exposes console/control over REST for remote administration.

### Bootstrap order (simulator)

`Application.Main` → `OpenSim.Startup()` →
`RegionApplicationBase.StartupSpecific()` (starts HTTP servers, `SceneManager`) →
`OpenSimBase.StartupSpecific()` (loads data services, then all `IApplicationPlugin`s:
`Initialise` then `PostInitialise`) → `LoadRegionsPlugin.PostInitialise()` creates
each region → `OpenSimBase.CreateRegion()` → `Scene` →
`RegionModulesController.AddRegionToModules(scene)`.

## Framework essentials (`Framework/`)

- `IScene` – a region. `IClientAPI` – a connected viewer (dozens of delegates for
  viewer actions; implemented by the Linden client stack).
- `IPlugin` – generic plugin contract. `IRegistryCore` / `RegistryCore` – the
  get/register interface registry (DI).
- `Util.cs` – general helpers. `RegionInfo.cs` – region metadata.
- `ConfigurationMember.cs`, `ConfigSettings.cs`, `IGenericConfig.cs` – config loading.
- `PluginLoader.cs` / `PluginManager.cs` – Mono.Addins plugin loading.

## Other notable folders

- `Region/ClientStack/Linden/` – the LLUDP (viewer protocol) and LLHTTP capabilities
  stacks that produce `IClientAPI` objects.
- `Region/PhysicsModules/` – physics engines: `BasicPhysics`, `BulletS`, `POS`,
  `ubOde`, plus `SharedBase` (common physics interfaces).
- `Region/ScriptEngine/` – in-world scripting: `YEngine/` (the LSL/XMR engine),
  `Shared/` (LSL types/API base classes), `Interfaces/`.
- `Region/OptionalModules/` – optional/experimental modules (AutoBackup, NPC, money,
  voice, IRC, …). See [Admin – AutoBackup](../admins/console-backup.md).
- `Data/` – database providers: `SQLite`, `MySQL`, `PGSQL`, `Null`.
- `Addons/` – separate add-on modules (e.g. Groups, OfflineIM, WebRTC voice).
- `Tools/` – utilities (`pCampBot` load tester, `Configger`, `LaunchSLClient`).
