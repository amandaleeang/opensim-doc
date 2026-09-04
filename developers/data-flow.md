# Data Flow

Two end-to-end walkthroughs that connect the architecture pieces. Paths are under
`OpenSim/`.

## A) Viewer login (LLLogin)

1. The viewer hits Robust's login endpoint. HTTP entry point:
   `Server/Handlers/Login/LLLoginServiceInConnector.cs`.
2. That calls into the service:
   `Services/LLLoginService/LLLoginService.cs` → `LLLoginService.Login(...)`.
3. Inside `Login` (roughly):
   - Client allow/deny checks (regex on viewer name, MAC, ID0).
   - `m_UserAccountService.GetUserAccount(...)` – look up the account
     (`Services/UserAccountService/`).
   - `m_AuthenticationService.Authenticate(...)` – verify the password
     (`Services/AuthenticationService/`).
   - Duplicate-presence check via `m_GridUserService`.
   - Fetch inventory (`m_InventoryService`), friends/avatar data.
   - Ask `IGridService` for the destination region.
   - Build the `LoginResponse`: circuit code, seed capability, simulator IP/port.
4. The response returns to the viewer. The viewer then opens an **LLUDP** connection
   to the simulator and establishes an **agent circuit** (`AgentCircuitData`), handled
   by the client stack in `Region/ClientStack/Linden/`.

The simulator side of "an agent arrived" is a **child or root agent** represented by
`ScenePresence` (`Region/Framework/Scenes/ScenePresence.cs`).

## B) Agent transfer between regions (teleport / border cross)

1. `EntityTransferModule`
   (`Region/CoreModules/Framework/EntityTransfer/EntityTransferModule.cs`,
   `: INonSharedRegionModule, IEntityTransferModule`) handles teleport
   (`TransferAgent`) and border cross (`CrossToNeighbour`). It builds an
   `AgentCircuitData` and calls:
   - `m_scene.SimulationService.CreateAgent(source, destination, agentCircuit, flags, ctx, out reason)`
   - `m_scene.SimulationService.UpdateAgent(destination, agent, ctx)`
2. `m_scene.SimulationService` is an `ISimulationService` provided by a region module:
   - **Local** destination → `LocalSimulationConnectorModule`
     (`CoreModules/ServiceConnectorsOut/Simulation/LocalSimulationConnector.cs`)
     routes to the in-process `Scene`.
   - **Remote** destination → `RemoteSimulationConnectorModule`
     (`.../RemoteSimulationConnector.cs`) delegates to `SimulationServiceConnector`
     (`Services/Connectors/Simulation/SimulationServiceConnector.cs`), which issues
     HTTP `agent/` REST calls to the destination simulator.
3. At the destination, Robust-style in-handlers
   (`Server/Handlers/Simulation/AgentHandlers.cs` +
   `SimulationServiceInConnector.cs`) call the destination `Scene`'s
   `SimulationService.CreateAgent`, creating a **child agent** (`ScenePresence`) there.
4. When the viewer connects to the destination and reports in,
   `ReleaseAgent`/`CloseAgent` finalize the move. Hypergrid crossings extend this via
   `HGEntityTransferModule.cs`.

## Mental summary

- **Login** = viewer → Robust (`LLLoginService`) → back to viewer → LLUDP to simulator
  → `ScenePresence`.
- **Teleport/cross** = source `Scene` → `EntityTransferModule` →
  `ISimulationService` (local or HTTP connector) → destination `Scene`
  (`ScenePresence` child agent) → viewer reconnects.

Both flows show the same design: shared logic lives in `Services/*Service`, remote
access is a `Connectors/*` proxy, and region-side behaviour is a `CoreModules` region
module. See [Where to look](where-to-look.md) to jump straight to the code, and
[Protocols](protocols.md) for the HTTP bodies those connectors send and receive
(local grid and Hypergrid).
