# Region Module Events

This page is the developer reference for **writing an optional region module**
and the **events it can hook**. Paths are under `OpenSim/` of the source tree.

Optional modules live in `Region/OptionalModules/` (in-tree) or in
`addon-modules/` (out-of-tree). They are ordinary region modules: implement
`ISharedRegionModule` or `INonSharedRegionModule`, get discovered by Mono.Addins,
and then subscribe to scene and client events.

Worked examples in the tree:

- `Region/OptionalModules/Example/BareBonesNonShared/BareBonesNonSharedModule.cs`
- `Region/OptionalModules/Example/BareBonesShared/BareBonesSharedModule.cs`
- `Region/OptionalModules/Aura/AuraLoginChatModule.cs` (subscribe / unsubscribe)
- `Region/OptionalModules/World/MoneyModule/SampleMoneyModule.cs` (many scene + client events)

Source of truth for scene events:
`Region/Framework/Scenes/EventManager.cs`.

---

## 1. Module lifecycle methods

These are the methods **you implement** on the module. The loader
(`ApplicationPlugins/RegionModulesController/RegionModulesControllerPlugin.cs`)
calls them. They are the only "event methods" on the module class itself.

Defined on `IRegionModuleBase`
(`Region/Framework/Interfaces/IRegionModuleBase.cs`):

| Method | When it runs | Typical work |
|--------|----------------|--------------|
| `Initialise(IConfigSource source)` | Shared: once, after the single instance is created. Non-shared: once per region instance, after construction. | Read INI (`source.Configs["MyModule"]`), set an enabled flag. Do **not** touch a `Scene` yet. |
| `PostInitialise()` | Shared modules only (`ISharedRegionModule`). Once, after every shared module has been `Initialise`d. | Cross-module setup that does not need a scene. |
| `AddRegion(Scene scene)` | Shared: once per region this simulator hosts. Non-shared: once, for this instance's scene. | `scene.RegisterModuleInterface<T>(this)`, subscribe to `scene.EventManager`. |
| `RegionLoaded(Scene scene)` | After **every** module has had `AddRegion` for this scene. | `scene.RequestModuleInterface<T>()` — other modules' interfaces are now registered. Safe place to hook events that depend on another module. |
| `RemoveRegion(Scene scene)` | When the region is taken down. Shared: may happen several times. Non-shared: once, then `Close()`. | Unsubscribe from `EventManager`, `UnregisterModuleInterface`. |
| `Close()` | Shared: simulator shutdown. Non-shared: after `RemoveRegion`. The instance is not usable after this. | Dispose timers, HTTP handlers, leftover state. |

Also implement:

- `string Name { get; }` — identifier used in logs and `scene.AddRegionModule`.
- `Type ReplaceableInterface { get; }` — return `null` unless this is a stub meant to be swapped for another implementation of the same interface. See [Architecture](architecture.md).

### Shared vs non-shared

| | `ISharedRegionModule` | `INonSharedRegionModule` |
|--|------------------------|---------------------------|
| Instances | One for the whole simulator | One per region |
| `AddRegion` / `RegionLoaded` / `RemoveRegion` | Called once per scene; keep a `List<Scene>` or `Dictionary` if you need per-region state | Called once; you can store `m_scene` on the instance |
| Extra method | `PostInitialise()` | none |

### Load order for one scene

1. Shared modules: `AddRegion(scene)` (replaceable modules deferred until last).
2. Non-shared modules: construct → `Initialise` → `AddRegion(scene)`.
3. All loaded modules: `RegionLoaded(scene)`.
4. `scene.AllModulesLoaded()`.

On region removal: `RemoveRegion(scene)` for every module, then `Close()` on
non-shared instances.

### Skeleton

```csharp
[Extension(Path = "/OpenSim/RegionModules", NodeName = "RegionModule", Id = "MyOptionalModule")]
public class MyOptionalModule : INonSharedRegionModule
{
    private Scene m_scene;
    private bool m_enabled;

    public string Name { get { return "MyOptionalModule"; } }
    public Type ReplaceableInterface { get { return null; } }

    public void Initialise(IConfigSource source)
    {
        IConfig cfg = source.Configs["MyOptionalModule"];
        m_enabled = cfg != null && cfg.GetBoolean("Enabled", false);
    }

    public void AddRegion(Scene scene)
    {
        if (!m_enabled)
            return;

        m_scene = scene;
        scene.RegisterModuleInterface<IMyOptionalModule>(this);

        scene.EventManager.OnNewClient += OnNewClient;
        scene.EventManager.OnMakeRootAgent += OnMakeRootAgent;
        scene.EventManager.OnClientClosed += OnClientClosed;
    }

    public void RegionLoaded(Scene scene)
    {
        if (!m_enabled)
            return;

        // Other modules have now registered their interfaces.
        IDialogModule dialog = scene.RequestModuleInterface<IDialogModule>();
    }

    public void RemoveRegion(Scene scene)
    {
        if (!m_enabled)
            return;

        scene.EventManager.OnNewClient -= OnNewClient;
        scene.EventManager.OnMakeRootAgent -= OnMakeRootAgent;
        scene.EventManager.OnClientClosed -= OnClientClosed;
        scene.UnregisterModuleInterface<IMyOptionalModule>(this);
    }

    public void Close() { }

    private void OnNewClient(IClientAPI client) { /* ... */ }
    private void OnMakeRootAgent(ScenePresence sp) { /* ... */ }
    private void OnClientClosed(UUID agentId, Scene scene) { /* ... */ }
}
```

Uncomment the `[Extension(...)]` line (it is commented out on the BareBones
examples so they stay inactive). For a **new assembly** that is not already an
addin, also add:

```csharp
[assembly: Addin("MyModule", "1.0")]
[assembly: AddinDependency("OpenSim", "0.8.1")]
```

Enable/disable from INI with `Setup_<Id> = disabled` in `[Modules]`, or with
your own `[MyOptionalModule] Enabled = true` flag as in the skeleton.

---

## 2. How to use scene events

Every `Scene` has an `EventManager`. Subscribe in `AddRegion` (or
`RegionLoaded`), unsubscribe in `RemoveRegion`:

```csharp
scene.EventManager.OnNewClient += OnNewClient;
// ...
scene.EventManager.OnNewClient -= OnNewClient;
```

`EventManager` invokes each handler independently and logs + continues if one
throws. A failing module does not take down the others.

### Rules of thumb

- **Unsubscribe.** Shared modules especially: if you skip `-=` in
  `RemoveRegion`, the handler keeps a reference to a dead scene.
- **Keep handlers short.** Several events (`OnNewClient`, `OnClientLogin`,
  `OnRemovePresence`, `OnClientClosed`) run **under a per-agent lock**. Do
  long work on another thread (`Util.FireAndForget`, a queue, etc.).
- **`OnMakeRootAgent` is on the teleport critical path.** Do as little as
  possible, or defer. See `AuraLoginChatModule` for waiting until
  `IClientAPI.OnRegionHandShakeReply` before talking to the viewer.
- **NPCs.** `OnNewClient` / `OnClientConnect` fire for viewer connections.
  `OnClientConnect` is **not** raised for NPCs (they are not `IClientCore`).
  `OnNewPresence` / `OnRemovePresence` fire for both users and NPCs.
- **Child vs root.** `OnNewClient` fires for both child and root agents.
  `OnClientLogin` fires only when the arrival is a fresh login.
  `OnMakeRootAgent` fires when the presence becomes the root agent in this
  scene (login, teleport in, region cross in).
- **Veto events.** A few handlers return `bool` (`OnSceneGroupMove`,
  `OnSceneGroupSpinStart`, `OnSceneGroupSpin`, `OnDeRezRequested`). Returning
  `false` can cancel the action — check the trigger site before relying on that.
- **Do not subscribe to `OnFrame` / heartbeat unless you must.** They fire
  every sim frame.

---

## 3. `EventManager` catalogue

All of these live on `scene.EventManager`. Signatures are the C# delegate /
`Action` types from `EventManager.cs`.

### Region lifecycle and heartbeat

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnFrame` | `void ()` | Each sim frame (`Scene.Update`). Used by sun/wind/clouds. Expensive if misused. |
| `OnBackup` | `void (ISimulationDataService datastore, bool forceBackup)` | Just before the region is persisted. |
| `OnShutdown` | `Action` | The **whole simulator** is shutting down. |
| `OnSceneShuttingDown` | `Action<Scene>` | This **one scene** is shutting down (not necessarily the process). |
| `OnRegionStarted` | `void (Scene scene)` | The region has started. |
| `OnRegionUp` | `void (GridRegion region)` | A neighbouring / known region is up. |
| `OnRegionHeartbeatStart` | `void (Scene scene)` | Start of a region heartbeat. |
| `OnRegionHeartbeatEnd` | `void (Scene scene)` | End of a region heartbeat. |
| `OnRegionLoginsStatusChange` | `void (IScene scene)` | Logins to the region were enabled or disabled. |
| `OnRegionReadyStatusChange` | `Action<IScene>` | Region is considered ready (startup work such as existing scripts has finished). |
| `OnPrimsLoaded` | `void (Scene s)` | In-world prims have been loaded from storage. |
| `OnPluginConsole` | `void (string[] args)` | A console command was dispatched to plugin modules. |
| `OnExtraSettingChanged` | `void (Scene scene, string name, string value)` | A region extra setting changed. |

### Clients, presence, teleport

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnNewClient` | `void (IClientAPI client)` | A client (child or root) is added. **Before** `OnClientLogin`. Under per-agent lock. Typical place to subscribe to `IClientAPI` events. |
| `OnClientConnect` | `void (IClientCore client)` | Same moment as `OnNewClient`, but only if the client implements `IClientCore` (not NPCs). |
| `OnClientLogin` | `Action<IClientAPI>` | The client is entering this sim as a **new login**. Under per-agent lock. |
| `OnNewPresence` | `void (ScenePresence presence)` | A presence was added (user **or** NPC). |
| `OnRemovePresence` | `void (UUID agentId)` | A presence was removed. Under per-agent lock. |
| `OnClientClosed` | `void (UUID clientID, Scene scene)` | Client removed, child or root. The scene **still contains** the presence. Under per-agent lock. |
| `OnSetRootAgentScene` | `void (UUID agentID, Scene scene)` | **Before** root-agent grunt work (attachments, physics, animations). Before `OnMakeRootAgent`. |
| `OnMakeRootAgent` | `Action<ScenePresence>` | **After** that grunt work. Teleport critical path — keep it cheap. |
| `OnMakeChildAgent` | `void (ScenePresence presence)` | The agent was demoted to a child agent (teleport/cross out). |
| `OnClientMovement` | `void (ScenePresence client)` | Agent movement update, before `OnScenePresenceUpdated`. |
| `OnSignificantClientMovement` | `Action<ScenePresence>` | Agent moved a significant distance (parcel/land checks, etc.). |
| `OnScenePresenceUpdated` | `void (ScenePresence sp)` | Presence properties were updated. |
| `OnAvatarAppearanceChange` | `void (ScenePresence avatar)` | Avatar appearance changed. |
| `OnAvatarEnteringNewParcel` | `void (ScenePresence avatar, int localLandID, UUID regionID)` | Avatar entered a different parcel. |
| `OnAvatarKilled` | `void (uint KillerLocalID, ScenePresence avatar)` | Avatar health reached zero. |
| `OnThrottleUpdate` | `void (ScenePresence scenePresence)` | Client throttle changed. |
| `OnCrossAgentToNewRegion` | `void (ScenePresence sp, bool isFlying, GridRegion newRegion)` | Agent is crossing into another region. |
| `OnTeleportStart` | `void (IClientAPI client, GridRegion destination, GridRegion finalDestination, uint teleportFlags, bool gridLogout)` | A teleport is starting. |
| `OnTeleportFail` | `void (IClientAPI client, bool gridLogout)` | A teleport failed. |
| `OnRegisterCaps` | `void (UUID agentID, Caps caps)` | Caps object exists; **add your capability handlers here**. |
| `OnDeregisterCaps` | `void (UUID agentID, Caps caps)` | Caps for this agent are being removed. |

Typical arrival order for a logging-in avatar:

`OnNewClient` → `OnClientConnect` (if `IClientCore`) → `OnClientLogin` →
`OnNewPresence` → `OnSetRootAgentScene` → `OnMakeRootAgent` →
`OnRegisterCaps` (caps also register around connection setup; do not assume
strict order against presence events — subscribe to the event you actually need).

### Objects, grab, physics

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnObjectAddedToScene` | `Action<SceneObjectGroup>` | Object added (`Scene.AddNewSceneObject`, duplicate, …). |
| `OnIncomingSceneObject` | `void (SceneObjectGroup so)` | Object or attachment entering the scene (e.g. from another region). |
| `OnObjectBeingRemovedFromScene` | `void (SceneObjectGroup obj)` | Object deleted from the scene. |
| `OnDeRezRequested` | `bool (IClientAPI remoteClient, List<SceneObjectGroup> objs, DeRezAction action)` | Client asked to derez, **before** delete. Return value can cancel. `remoteClient` may be null. |
| `OnObjectAddedToPhysicalScene` | `Action<SceneObjectPart>` | Physics actor created. |
| `OnObjectRemovedFromPhysicalScene` | `Action<SceneObjectPart>` | Just **before** the physics actor is destroyed — last chance to clean up. |
| `OnSceneObjectLoaded` | `void (SceneObjectGroup so)` | Object just loaded from storage. |
| `OnSceneObjectPreSave` | `void (SceneObjectGroup persistingSo, SceneObjectGroup originalSo)` | About to persist. Mutations on `persistingSo` are saved; mutations on `originalSo` stay in memory only. |
| `OnSceneObjectPartCopy` | `void (SceneObjectPart copy, SceneObjectPart original, bool userExposed)` | A part was cloned. `userExposed` is true if the copy is immediately in-world. |
| `OnSceneObjectPartUpdated` | `void (SceneObjectPart sop, bool full)` | A part was updated (`full` = full update vs. terse). |
| `OnObjectGrab` | `void (uint localID, uint originalID, Vector3 offsetPos, IClientAPI remoteClient, SurfaceTouchEventArgs surfaceArgs)` | Touch/grab start. `localID` is the **root**; `originalID` is the part actually touched. |
| `OnObjectGrabbing` | same as `OnObjectGrab` | Continuous grab updates. |
| `OnObjectDeGrab` | `void (uint localID, uint originalID, IClientAPI remoteClient, SurfaceTouchEventArgs surfaceArgs)` | Touch/grab end. |
| `OnSceneGroupMove` | `bool (UUID groupID, Vector3 delta)` | Linkset moved. |
| `OnSceneGroupGrab` | `void (UUID groupID, Vector3 offset, UUID userID)` | Linkset grabbed. |
| `OnSceneGroupSpinStart` | `bool (UUID groupID)` | Linkset started spinning. |
| `OnSceneGroupSpin` | `bool (UUID groupID, Quaternion rotation)` | Linkset is being spun. |
| `OnAttach` | `void (uint localID, UUID itemID, UUID avatarID)` | Object attached or detached. On **detach**, `avatarID` is `UUID.Zero` (historical). |
| `OnPermissionError` | `void (UUID user, string reason)` | A permissions check failed. |

### Scripts

Most of these exist so the script engine (and modules that track script
lifecycle) can react. Optional modules use them to clean per-script state
(see `JsonStoreScriptModule`).

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnNewScript` | `void (UUID clientID, SceneObjectPart part, UUID itemID)` | Script created. **Before** `OnRezScript`. |
| `OnRezScript` | `void (uint localID, UUID itemID, string script, int startParam, bool postOnRez, string engine, int stateSource)` | Script instance created / run. After `OnNewScript`. |
| `OnUpdateScript` | `void (UUID clientID, UUID itemId, UUID primId, bool isScriptRunning, UUID newAssetID)` | Client uploaded a new script asset. |
| `OnRemoveScript` | `void (uint localID, UUID itemID)` | Script removed from an object. |
| `OnStartScript` | `void (uint localID, UUID itemID)` | Script started. |
| `OnStopScript` | `void (uint localID, UUID itemID)` | Script stopped. |
| `OnScriptReset` | `void (uint localID, UUID itemID)` | Script reset. |
| `OnGetScriptRunning` | `void (IClientAPI controllingClient, UUID objectID, UUID itemID)` | Viewer asked whether a script is running. |
| `OnScriptChangedEvent` | `void (uint localID, uint change, object data)` | Object property a script might care about changed (colour, scale, inventory, …). This is the LSL `changed` event, **not** "the script source changed" (`OnUpdateScript`). |
| `OnScriptControlEvent` | `void (UUID item, UUID avatarID, uint held, uint changed)` | Agent control input forwarded to a script (`llTakeControls`). |
| `OnScriptListenEvent` | `void (UUID scriptID, int channel, string name, UUID id, string message)` | `llListen` matched a chat/message. |
| `OnScriptMovingStartEvent` | `void (uint localID)` | Intended: physical object started moving (see source TODO). |
| `OnScriptMovingEndEvent` | `void (uint localID)` | Intended: physical object stopped moving (see source TODO). |
| `OnScriptAtTargetEvent` | `void (UUID scriptID, uint handle, Vector3 targetpos, Vector3 atpos)` | Object reached an `llTarget`. |
| `OnScriptNotAtTargetEvent` | `void (UUID scriptID)` | Object has a target but is not within tolerance. |
| `OnScriptAtRotTargetEvent` | `void (UUID scriptID, uint handle, Quaternion targetrot, Quaternion atrot)` | Object reached an `llRotTarget`. |
| `OnScriptNotAtRotTargetEvent` | `void (UUID scriptID)` | Object has a rotation target but is not within tolerance. |
| `OnScriptColliderStart` | `void (uint localID, ColliderArgs colliders)` | Collision **start** with something other than terrain. |
| `OnScriptColliding` | same | Collision **ongoing**. |
| `OnScriptCollidingEnd` | same | Collision **end**. |
| `OnScriptLandColliderStart` | `void (uint localID, ColliderArgs colliders)` | Collision start with terrain. |
| `OnScriptLandColliding` | same | Collision ongoing with terrain. |
| `OnScriptLandColliderEnd` | same | Collision end with terrain. |
| `OnEmptyScriptCompileQueue` | `void (int numScriptsFailed, string message)` | Script compile queue drained. `numScriptsFailed` is how many did not start. |

### Terrain, parcels, estate

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnTerrainTainted` | `void ()` | Terrain was edited. |
| `OnTerrainTick` | `void ()` | Terrain update tick (core uses this to push terrain into physics). |
| `OnTerrainCheckUpdates` | `void ()` | Check whether terrain needs sending. |
| `OnTerrainUpdate` | `void ()` | Terrain was updated. |
| `OnRequestChangeWaterHeight` | `void (float height)` | Water height change requested. |
| `OnEstateToolsSunUpdate` | `void (ulong regionHandle)` | Estate sun settings updated. |
| `OnGetCurrentTimeAsLindenSunHour` | `float ()` | Something asked for the current Linden sun hour. |
| `OnLandObjectAdded` | `void (ILandObject newParcel)` | Parcel added. |
| `OnLandObjectRemoved` | `void (UUID globalID)` | Parcel removed. |
| `OnParcelPropertiesUpdateRequest` | `void (LandUpdateArgs args, int local_id, IClientAPI remote_client)` | Parcel properties updated. |
| `OnParcelPrimCountUpdate` | `void ()` | Prim count may have changed, or an accurate count is required. |
| `OnParcelPrimCountAdd` | `void (SceneObjectGroup obj)` | Response to the above for objects that count against the parcel. |
| `OnParcelPrimCountTainted` | `void ()` | Prim count is dirty (object moved, attached, deleted, …). |
| `OnRequestParcelPrimCountUpdate` | `void ()` | Request to recompute parcel prim counts. |
| `OnNoticeNoLandDataFromStorage` | `void ()` | No parcel data in storage (fresh region). |
| `OnIncomingLandDataFromStorage` | `void (List<LandData> data)` | Parcel data loaded from storage. |
| `OnSetAllowForcefulBan` | `void (bool allow)` | Estate "allow forceful ban" flag changed. |
| `OnValidateLandBuy` | `void (Object sender, LandBuyArgs e)` | **Before** a parcel purchase; set flags on `e` to allow/deny. |
| `OnLandBuy` | `void (Object sender, LandBuyArgs e)` | After validation, process the purchase. |

`LandBuyArgs` fields (on `EventManager`): `agentId`, `groupId`,
`parcelOwnerID`, `final`, `groupOwned`, `removeContribution`, `parcelLocalID`,
`parcelArea`, `parcelPrice`, `authenticated`, `landValidated`,
`economyValidated`, `transactionID`, `amountDebited`.

### Chat, IM, money, inventory, OAR

| Event | Signature | Fires when |
|-------|-----------|------------|
| `OnChatFromClient` | `void (Object sender, OSChatMessage chat)` | Chat from a viewer (via ChatModule or a replacement). |
| `OnChatFromWorld` | `void (Object sender, OSChatMessage chat)` | Chat originating in-world (objects, NPC, system). |
| `OnChatBroadcast` | `void (Object sender, OSChatMessage chat)` | Region-wide broadcast chat. |
| `OnIncomingInstantMessage` | `void (GridInstantMessage message)` | IM arrived at this scene. |
| `OnUnhandledInstantMessage` | `void (GridInstantMessage message)` | IM was not handled by anyone else. |
| `OnMoneyTransfer` | `void (Object sender, MoneyTransferArgs e)` | Grid-currency transfer requested. Args: `sender`, `receiver`, `amount`, `transactiontype`, `description`, `authenticated`. |
| `OnNewInventoryItemUploadComplete` | `void (InventoryItemBase item, int userlevel)` | Inventory item finished uploading. |
| `OnOarFileLoaded` | `void (Guid guid, List<UUID> loadedScenes, string message)` | OAR load finished (scripts may still be starting). `message` is non-empty on problems. |
| `OnOarFileSaved` | `void (Guid guid, string message)` | OAR save finished. `guid` is the id passed to the save request, or `Guid.Empty`. |

---

## 4. Per-client events (`IClientAPI`)

Scene events tell you that **something happened in the region**. Viewer
packets (chat typed by the user, object buy, appearance, inventory, …) arrive
as events on that agent's `IClientAPI`.

Pattern used throughout OptionalModules (Money, DataSnapshot, DynamicFloater):

```csharp
private void OnNewClient(IClientAPI client)
{
    client.OnChatFromClient += OnChatFromClient;
    client.OnMoneyBalanceRequest += SendMoneyBalance;
    client.OnLogout += ClientLoggedOut;
}
```

The full list is in `Framework/IClientAPI.cs` (well over 150 events). Groups
you will actually use from an optional module:

| Group | Examples |
|-------|----------|
| Connection / movement | `OnRegionHandShakeReply`, `OnCompleteMovementToRegion`, `OnAgentUpdate`, `OnPreAgentUpdate`, `OnLogout`, `OnConnectionClosed` |
| Chat / IM | `OnChatFromClient`, `OnInstantMessage`, `OnGenericMessage` |
| Appearance / attachments | `OnSetAppearance`, `OnAvatarNowWearing`, `OnRezSingleAttachmentFromInv`, `OnObjectAttach`, `OnObjectDetach`, `OnObjectDrop` |
| Objects | `OnAddPrim`, `OnRezObject`, `OnDeRezObject`, `OnGrabObject`, `OnGrabUpdate`, `OnDeGrabObject`, `OnLinkObjects`, `OnObjectDuplicate`, `OnObjectPermissions` |
| Money / buy | `OnMoneyTransferRequest`, `OnMoneyBalanceRequest`, `OnEconomyDataRequest`, `OnObjectBuy`, `OnRequestPayPrice`, `OnParcelBuy` |
| Land / estate | `OnParcelPropertiesUpdateRequest`, `OnParcelDivideRequest`, `OnParcelJoinRequest`, `OnModifyTerrain` |
| Inventory / scripts | `OnRezScript`, `OnScriptReset`, `OnSetScriptRunning`, `OnUpdateInventoryItem` |
| Teleport | `OnTeleportLocationRequest`, `OnTeleportLandmarkRequest`, `OnTeleportHomeRequest`, `OnTeleportCancel` |

Unsubscribe on `OnClientClosed` / `OnLogout` if you added per-client handlers
that capture the module (or a large object) — otherwise the client can pin
your module after the avatar has left.

`IClientAPI` is the contract; the Linden implementation is
`Region/ClientStack/Linden/` (`LLClientView`).

---

## 5. Which hook should I use?

| I want to… | Hook |
|------------|------|
| Run once when my module starts | `Initialise` / `PostInitialise` |
| Bind this module to a region | `AddRegion` — register interface, subscribe to `EventManager` |
| Call another module | `RegionLoaded` + `scene.RequestModuleInterface<T>()` |
| See every new viewer | `EventManager.OnNewClient` |
| See login only (not child-agent or NPC) | `OnClientLogin`, and/or `OnMakeRootAgent` plus `sp.TeleportFlags` |
| Greet / act after the avatar is fully in-world | `OnMakeRootAgent`, then often `client.OnRegionHandShakeReply` |
| Add a capability | `OnRegisterCaps` |
| Watch objects rez / delete | `OnObjectAddedToScene` / `OnObjectBeingRemovedFromScene` |
| Watch chat | `OnChatFromClient` / `OnChatFromWorld` (scene) or `client.OnChatFromClient` (that viewer only) |
| Handle money or land sale | `OnMoneyTransfer`, `OnValidateLandBuy`, `OnLandBuy`, plus `IClientAPI` money events |
| Track script create/reset/remove | `OnNewScript`, `OnRezScript`, `OnScriptReset`, `OnRemoveScript` |
| React to region backup / OAR | `OnBackup`, `OnOarFileLoaded`, `OnOarFileSaved` |
| Clean up when the region goes away | `RemoveRegion` (unsubscribe) then `Close` |

---

## 6. Publishing your own events

If other modules should react to **your** module, do not add events to
`EventManager`. Define an interface in `Region/Framework/Interfaces/` (or
next to your module), register it, and put C# events on that interface.

```csharp
// On your module, in AddRegion:
scene.RegisterModuleInterface<IMyOptionalModule>(this);

// In another module, in RegionLoaded:
IMyOptionalModule mine = scene.RequestModuleInterface<IMyOptionalModule>();
```

That is the same pattern `IMoneyModule.OnObjectPaid` uses. Process-wide
services go on `OpenSimBase.ApplicationRegistry` (`IRegistryCore`) instead of
the scene.

---

## See also

- [Architecture overview](architecture.md) — module loader, shared vs non-shared, replaceable interfaces
- [Where to look](where-to-look.md) — file index
- `Region/Framework/Scenes/EventManager.cs` — event declarations and `Trigger*` methods
- `Framework/IClientAPI.cs` — per-client viewer events
- `Region/Framework/Interfaces/IRegionModuleBase.cs` — lifecycle contract
