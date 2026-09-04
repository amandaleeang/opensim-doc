# Where to Look

A quick index for "I want to change **X** → start in file **Y**". All paths are under
`OpenSim/`.

| I want to change… | Start here |
|-------------------|------------|
| Region startup / bootstrap | `Region/Application/OpenSimBase.cs`, `Region/Application/OpenSim.cs` |
| How a region is loaded from disk | `ApplicationPlugins/LoadRegions/` (`LoadRegionsPlugin.cs`, `RegionLoaderFileSystem.cs`) |
| First-run interactive config | `Framework/ConfigurationMember.cs` |
| Merged INI config behaviour | `Region/Application/ConfigurationLoader.cs`; `.ini` in `bin/config-include/` |
| A viewer command / packet | `Region/ClientStack/Linden/`; contract in `Framework/IClientAPI.cs` |
| Avatar / agent behaviour | `Region/Framework/Scenes/ScenePresence.cs`; `Region/CoreModules/Avatar/` |
| Prim / object behaviour | `Region/Framework/Scenes/SceneObjectGroup.cs`, `SceneObjectPart.cs`; `Region/CoreModules/World/Objects/` |
| Terrain | `Region/CoreModules/World/Terrain/` |
| Parcels / land / estate | `Region/CoreModules/World/Land/`, `Estate/` |
| Permissions | `Region/Framework/Scenes/Scene.Permissions.cs` |
| A grid service (asset/grid/inventory/…) | `Services/<Name>Service/` + its contract in `Services/Interfaces/I<Name>Service.cs` |
| Talking to a remote service | `Services/Connectors/` |
| **Wire messages** (sim ↔ sim, sim ↔ Robust, HG) | [Protocols](protocols.md) – request/response bodies, not the C# |
| Login | `Services/LLLoginService/LLLoginService.cs`; HTTP entry in `Server/Handlers/Login/` |
| Teleport / region crossing | `Region/CoreModules/Framework/EntityTransfer/EntityTransferModule.cs` |
| Hypergrid travel | `Services/HypergridService/`, `Region/CoreModules/Framework/EntityTransfer/HGEntityTransferModule.cs` |
| Physics | `Region/PhysicsModules/` (`BulletS`, `ubOde`, `POS`, `BasicPhysics`, `SharedBase`) |
| Scripting (LSL/XMR) | `Region/ScriptEngine/` (`YEngine/`, `Shared/`) |
| OAR / IAR import-export | `Region/CoreModules/World/Archiver/` |
| Asset cache (Flotsam) | `Region/CoreModules/Asset/FlotsamAssetCache.cs`; `config-include/FlotsamCache.ini` |
| Adding / editing a region module | Define interface in `Region/Framework/Interfaces/`, implement in `Region/CoreModules/` following `IRegionModuleBase` |
| Writing an **optional** region module | `Region/OptionalModules/` (examples in `Example/BareBones*`); hook `scene.EventManager` — see [Region module events](region-module-events.md) |
| Scene events a module can subscribe to | `Region/Framework/Scenes/EventManager.cs` |
| Per-client (viewer packet) events | `Framework/IClientAPI.cs`; subscribe from `EventManager.OnNewClient` |
| The Robust server itself | `Server/ServerMain.cs`, `Server/Base/`, `Server/Handlers/` |
| Database layer / a new store | `Data/` (`SQLite`, `MySQL`, `PGSQL`, `Null`) + `Data/Base` |
| The console / commands | `Framework/MainConsole.cs`, `Framework/ICommandConsole.cs`; commands registered via `ICommandConsole` |
| Auto-backup | `Region/OptionalModules/World/AutoBackup/AutoBackupModule.cs` |
| Groups / Offline IM / Voice add-ons | `Addons/` |

## Patterns to know

- **Discovery over `new`.** Features are wired via Mono.Addins extension points
  (`[Extension(...)]`, `[TypeExtensionPoint(...)]`), not hardcoded instantiation.
  Search for the interface name to find implementations.
- **Module interfaces as the API.** To use another subsystem, call
  `scene.RequestModuleInterface<T>()`; to provide one, call
  `scene.RegisterModuleInterface<T>(this)`. Same idea process-wide via
  `OpenSimBase.ApplicationRegistry` (`IRegistryCore`).
- **ReplaceableInterface.** A module declaring `ReplaceableInterface = typeof(IFoo)`
  can be swapped by another implementation of `IFoo`; only one loads. Use this when
  adding an alternative implementation of an existing feature.
- **Services/Interfaces vs Connectors.** If you change a grid service's behaviour,
  edit the implementation in `Services/<Name>Service/`; if you change how the
  simulator *talks* to it, edit `Services/Connectors/` or the `*ServiceConnectorsOut`
  module.
- **Partial `Scene`.** `Scene` is split across many files by concern
  (`Scene.Inventory.cs`, etc.). Edit the file matching the concern.
