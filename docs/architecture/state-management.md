> 🇯🇵 日本語版はこちら → [state-management.ja.md](./state-management.ja.md)

# State Management & Stateless Design

## Core Principle: The Frontend Owns State

The Network Simulator is built around a deliberately **stateless backend**. The FastAPI backend never holds topology state in memory between requests. Every REST call carries the full context needed to complete it, and the backend's only durable artifacts are files on disk (`topology.clab.yml`, `frr.conf`, `topology_state.json`).

The React frontend — specifically the **Zustand store** in `topologyStore.ts` — is the single source of truth for all topology configuration. This includes node positions, interface IP addresses, VLAN configurations, routing protocol settings, and edge connections.

## Why Stateless?

This design choice enables several important capabilities:

- **Partial reconfiguration without redeployment**: A single node can be reconfigured at any time by sending only that node's config to `/api/v1/nodes/{name}/configure`. The backend needs no memory of previous state because the frontend sends everything it needs.
- **Page reload resilience**: If the browser tab is refreshed, `loadState()` fetches the last-saved state from the backend's JSON files and the live container status from Containerlab, then re-hydrates the Zustand store seamlessly.
- **Authoritative diff tracking**: Because the frontend always has the complete topology, it can compute whether the canvas has diverged from the last-deployed state by comparing `nodes[]` + `edges[]` against `deployedNodes[]` + `deployedEdges[]`.
- **Simple backend**: The backend has no session management, no in-memory caches, no topology state machines. Each `Orchestrator` instance is created fresh per request.

## `topologyStore.ts` — The Zustand Store

### State Interface

```typescript
interface TopologyState {
  // Live canvas state (what the user is currently drawing)
  nodes: Node[];              // React Flow nodes (router, switch, host)
  edges: Edge[];              // React Flow edges (connections between ports)

  // Deployed snapshot (what was last successfully deployed)
  deployedNodes: Node[];      // Deep copy of nodes[] at last deploy
  deployedEdges: Edge[];      // Deep copy of edges[] at last deploy

  // Selection state
  selectedNodeId: string | null;
  selectedEdgeId: string | null;

  // UI flags
  hasChanges: boolean;        // true when nodes/edges differ from deployedNodes/deployedEdges
  isSaving: boolean;          // true while saveState() HTTP call is in flight
}
```

### `hasChanges` Computation

`hasChanges` is recomputed automatically on every state update via a custom `set()` wrapper. The comparison uses the `checkHasChanges()` function, which:

1. Strips position data (`position`, `positionAbsolute`, `width`, `height`, `selected`, `dragging`) from each node — position changes alone should not trigger a "has changes" warning
2. Strips the `status` field from node data — container status is runtime state, not config state
3. Sorts both arrays by node/edge `id`
4. Does a deep JSON string comparison

```typescript
function checkHasChanges(nodes, edges, deployedNodes, deployedEdges): boolean {
  const cleanNodes     = nodes.map(cleanNodeForComparison).sort(...);
  const cleanDeployed  = deployedNodes.map(cleanNodeForComparison).sort(...);
  const cleanEdges     = edges.map(cleanEdgeForComparison).sort(...);
  const cleanDEdges    = deployedEdges.map(cleanEdgeForComparison).sort(...);
  return JSON.stringify(cleanNodes)  !== JSON.stringify(cleanDeployed) ||
         JSON.stringify(cleanEdges)  !== JSON.stringify(cleanDEdges);
}
```

### Auto-Save (Debounced)

Any state change that modifies `nodes` or `edges` triggers a debounced auto-save with a 2-second delay:

```typescript
function triggerAutoSave() {
  clearTimeout(autoSaveTimeoutId);
  autoSaveTimeoutId = setTimeout(() => {
    useTopologyStore.getState().saveState(false);  // save without marking as deployed
  }, 2000);
}
```

This means that 2 seconds after the user stops editing (dragging nodes, changing properties), the canvas state is automatically persisted to `topology_state.json` on the backend — without the user needing to manually save.

### Key Actions

| Action | Description |
|---|---|
| `addNode(type)` | Creates a new node of type `"router"`, `"host"`, or `"switch"` with default data, auto-assigns the next available alphabetical label (Router-A, Router-B, ...) |
| `deleteNode(id)` | Removes the node and all edges connected to it; clears `selectedNodeId` if it was the deleted node |
| `updateNodeData(nodeId, data)` | Merges partial data into the node's `data` field; triggers auto-save |
| `deleteEdge(id)` | Removes the edge and clears the `connectedTo` reference on both endpoint nodes |
| `addPort(nodeId)` | Adds a new interface to a router or switch; names it `eth{N}` where N is one higher than the current max |
| `deletePort(nodeId, portName)` | Removes an interface from a router or switch; also removes any edges using that port |
| `onConnect(connection)` | Called by React Flow when the user drags a connection handle; creates a new edge and updates `connectedTo` on both endpoint nodes |
| `setTopology(nodes, edges)` | Fetches current container status, merges `"up"/"down"` into node data, then sets `nodes` and `edges` in the store |
| `setDeployedState(nodes, edges)` | Deep-copies the given nodes/edges into `deployedNodes`/`deployedEdges`; used to mark a deploy as complete |
| `saveState(deployed?)` | Persists current `nodes` + `edges` to backend; if `deployed=true`, also updates `deployedNodes`/`deployedEdges` and resets `hasChanges` |
| `loadState()` | Fetches `topology_state.json` and `topology_deployed_state.json` from backend, merges live container status, re-hydrates the store |
| `resetTopologyState()` | Deletes both JSON files on backend; resets store to the default two-router + one-host canvas |

### Default Initial State

When the store is first created (or after `resetTopologyState()`), the canvas shows:

```
┌─────────────┐                    ┌─────────────┐
│  Router-A   │                    │   Router-B  │
│  (router-1) │                    │  (router-2) │
└─────────────┘                    └─────────────┘
       │ eth1
       │
┌─────────────┐
│   Host-A    │
│   (host-1)  │
└─────────────┘
```

- **Router-A** (`id: "router-1"`): 4 interfaces (eth1–eth4), all routing protocols disabled
- **Router-B** (`id: "router-2"`): 4 interfaces (eth1–eth4), all routing protocols disabled
- **Host-A** (`id: "host-1"`): no IP address, no gateway
- **Edge** connecting `router-1:eth1` → `host-1:eth1`

Routers are initialized by `initialRouterData()`:

```typescript
{
  label: "Router-A",
  status: "down",
  interfaces: [
    { id: "eth1", name: "eth1", ipAddress: "", netmask: "" },
    { id: "eth2", name: "eth2", ipAddress: "", netmask: "" },
    { id: "eth3", name: "eth3", ipAddress: "", netmask: "" },
    { id: "eth4", name: "eth4", ipAddress: "", netmask: "" },
  ],
  vlanInterfaces: [],
  routing: {
    ospf: { enabled: false, routerId: "", areas: [],
            redistribute: { connected: false, static: false, rip: false, bgp: false },
            defaultInformationOriginate: { enabled: false, always: false } },
    rip:  { enabled: false, networks: [],
            redistribute: { connected: false, static: false, ospf: false, bgp: false } },
    bgp:  { enabled: false, asNumber: 65001, routerId: "", neighbors: [],
            redistribute: { connected: false, static: false, ospf: false, rip: false } },
  },
  staticRoutes: [],
}
```

Switches are initialized with 4 interfaces in access mode, all on VLAN 1:

```typescript
interfaces: [
  { id: "eth1", name: "eth1", vlanMode: "access", vlanId: 1, vlanIds: [] },
  ...
]
```

## Node Data Types (`types/topology.ts`)

### `RouterNodeData`

```typescript
interface RouterNodeData {
  label: string;
  status: 'up' | 'down';
  interfaces: InterfaceData[];       // physical ports (eth1, eth2, ...)
  vlanInterfaces: VlanInterfaceData[]; // VLAN subinterfaces (eth1.10, eth2.20, ...)
  routing: {
    ospf: OspfConfig;
    rip: RipConfig;
    bgp: BgpConfig;
  };
  staticRoutes: StaticRoute[];
}

interface InterfaceData {
  id: string;         // same as name (e.g. "eth1")
  name: string;
  ipAddress: string;  // e.g. "10.0.1.1"
  netmask: string;    // e.g. "255.255.255.0" or "24"
  connectedTo?: string; // target node id
  adminState?: 'up' | 'down';
}

interface VlanInterfaceData {
  name: string;           // e.g. "eth1.10"
  parentInterface: string; // e.g. "eth1"
  vlanId: number;         // e.g. 10
  ipAddress: string;      // CIDR format, e.g. "10.10.10.1/24"
}
```

OSPF configuration supports areas with sub-types (normal/stub/totally-stub/nssa/totally-nssa), per-area ranges (route summarization), redistribution from connected/static/rip/bgp, and `default-information originate` with optional `always` flag and metric.

BGP configuration supports AS number, router-id, multiple neighbors (each with IP and remote-AS), and redistribution.

### `HostNodeData`

```typescript
interface HostNodeData {
  label: string;
  status: 'up' | 'down';
  ipAddress: string;    // CIDR format, e.g. "192.168.1.10/24"
  gateway: string;      // e.g. "192.168.1.1"
  connectedTo?: string; // connected node id
  vlanInterfaces: VlanInterfaceData[];
  eth1AdminState?: 'up' | 'down';
}
```

### `SwitchNodeData`

```typescript
interface SwitchNodeData {
  label: string;
  status: 'up' | 'down';
  interfaces: SwitchInterfaceData[];
}

interface SwitchInterfaceData {
  id: string;
  name: string;
  vlanMode: 'none' | 'access' | 'trunk';
  vlanId?: number;    // used when vlanMode == "access" (1–4094)
  vlanIds?: number[]; // used when vlanMode == "trunk" (e.g. [10, 20])
  connectedTo?: string;
  adminState?: 'up' | 'down';
}
```

### `NetworkEdgeData`

```typescript
interface NetworkEdgeData {
  bandwidth?: string;        // e.g. "1Gbps" (informational only)
  delay?: string;            // e.g. "10ms"  (informational only)
  cost?: number;             // e.g. 10      (informational only)
  sourceInterface?: string;  // e.g. "eth1"
  targetInterface?: string;  // e.g. "eth2"
  status?: 'up' | 'down';
}
```

## State Persistence

### Saving

```
saveState(deployed = false)
    │
    ├─ reads current nodes + edges from store
    ├─ POST /api/v1/topology/state
    │     body: { nodes, edges }
    │     query: ?deployed=false
    │
    ▼  backend writes:
    topology_state.json        ← always written
    topology_deployed_state.json ← only when deployed=true

    If deployed=true:
    ├─ deep-copies nodes/edges into deployedNodes/deployedEdges
    └─ resets hasChanges = false
```

### Loading

```
loadState()
    │
    ├─ GET /api/v1/topology/state             → topology_state.json
    │     (falls back to default_topology.json if not found)
    ├─ GET /api/v1/topology/state?deployed=true → topology_deployed_state.json
    │     (returns { nodes: [], edges: [] } if not found)
    ├─ GET /api/v1/topology/status            → containerlab inspect output
    │
    ├─ merges container running status into each node's data.status field
    ├─ recomputes hasChanges
    └─ sets nodes, edges, deployedNodes, deployedEdges in store
```

### Resetting

```
resetTopologyState()
    │
    ├─ DELETE /api/v1/topology/state
    │     backend deletes topology_state.json AND topology_deployed_state.json
    │
    └─ resets store to default two-router + one-host canvas
         deployedNodes = [], deployedEdges = [], hasChanges = true
```

## State Flow Diagram

```
User Action
    │
    ▼
React Flow UI
(nodes[], edges[])
    │
    ├── onNodesChange / onEdgesChange  ← React Flow built-in event handlers
    │        │
    │        └── triggers triggerAutoSave() → debounced 2s → saveState(false)
    │
    ├── updateNodeData()               ← PropertyPanel edits
    │        │
    │        └── triggers triggerAutoSave()
    │
    └── onConnect()                    ← user drags a connection handle
             │
             └── creates edge, updates connectedTo on both nodes
                  └── triggers triggerAutoSave()

Deploy Button
    │
    ▼
setTopology(nodes, edges)
    │
    ├── fetch /topology/status → merge container status into nodes
    ├── set nodes + edges in store
    ├── POST /topology/deploy → Orchestrator → containerlab deploy
    └── saveState(deployed=true)
             │
             ├── POST /topology/state?deployed=true → topology_deployed_state.json
             ├── deep-copy nodes → deployedNodes
             └── hasChanges = false

Page Load
    │
    ▼
loadState()
    │
    ├── GET /topology/state → topology_state.json
    ├── GET /topology/state?deployed=true → topology_deployed_state.json
    ├── GET /topology/status → merge live status
    └── hydrate store (nodes, edges, deployedNodes, deployedEdges, hasChanges)
```

---

*Navigation: [Architecture Index](./index.md) · [System Overview](./system-overview.md) · [Data Flows](./data-flows.md)*
