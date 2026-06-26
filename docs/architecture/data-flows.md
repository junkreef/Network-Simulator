> 🇯🇵 日本語版はこちら → [data-flows.ja.md](./data-flows.ja.md)

# Data Flows

This document provides step-by-step walkthroughs of the four major data flows in the Network Simulator: topology deployment, router configuration, L2 switch configuration, and WebSocket terminal proxying. Understanding these flows is key to understanding how the system works end-to-end.

---

## Flow 1: Topology Deployment

This flow covers everything that happens when the user clicks the **Deploy** button in the UI. It starts in the browser, passes through the full backend stack, and ends with running Docker containers connected by virtual Ethernet pairs.

### Steps

**Step 1 — User clicks "Deploy" in the Header component**

The `Header` React component renders a Deploy button. Clicking it calls the `setTopology()` action in the Zustand store (passing in the current `nodes` and `edges` arrays), then calls `POST /api/v1/topology/deploy` with the serialized topology payload.

**Step 2 — `setTopology()` builds the topology payload**

Before sending to the backend, `setTopology()` also calls `GET /api/v1/topology/status` to check which containers are currently running, and merges the `"up"/"down"` status into each node's data. The payload sent to `/deploy` is a JSON object with three fields:

```json
{
  "name": "sim-network",
  "nodes": [
    {
      "name": "r1",
      "type": "router",
      "interfaces": ["eth1", "eth2"]
    },
    {
      "name": "sw1",
      "type": "switch",
      "interfaces": ["eth1", "eth2", "eth3"]
    }
  ],
  "links": [
    {
      "endpoints": ["r1:eth1", "sw1:eth1"]
    }
  ]
}
```

Each link endpoint is a `"node_name:interface_name"` string. The edge's `sourceInterface` and `targetInterface` from React Flow become the interface names on either end of the link.

**Step 3 — FastAPI validates the request**

The `POST /api/v1/topology/deploy` FastAPI route receives the request and validates it against a Pydantic `TopologyDeployRequest` model. If validation fails, FastAPI returns a `422 Unprocessable Entity` automatically.

**Step 4 — `Orchestrator.deploy_topology(topology_data)` is called**

The REST handler creates an `Orchestrator` instance and calls `deploy_topology()` passing the validated topology data as a dict.

**Step 5 — Router pre-setup: config directories and initial files**

For every node with `type == "router"`, the orchestrator creates the per-router config directory at `data/{topology_name}/{node_name}/` and writes three files into it:

```
data/sim-network/r1/
├── daemons       ← FRR daemon enable/disable list
├── vtysh.conf    ← vtysh integration config
└── frr.conf      ← initial (empty) routing config
```

**`daemons` file** — Written literally with these exact daemon states:

```
zebra=yes
bgpd=yes
ospfd=yes
ripd=yes
ospf6d=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```

**`vtysh.conf` file** — Enables integrated configuration mode:

```
service integrated-vtysh-config
```

**`frr.conf` file** — Written with a minimal initial stub (the real config is applied later by `configure_node()`):

```
log file /var/log/frr/frr.log
!
```

All directories and files are created with `chmod 777`. This is necessary because the FRR process inside the container runs as a non-root user and must be able to read these files through the bind mount.

> **Edge case**: If any of these paths already exists as a *directory* (from a broken previous run), the code uses `shutil.rmtree()` to remove it before creating the file. `os.fsync()` is called after every write to guarantee durability.

**Step 6 — Jinja2 renders `topology.clab.yml`**

The `topology.clab.yml.j2` template is rendered with the topology data. The output is a complete Containerlab YAML file. Here is what the template produces for each node type:

*Router nodes*:

```yaml
r1:
  image: alpine-frr:latest
  binds:
    - /abs/path/to/data/sim-network/r1/daemons:/etc/frr/daemons:ro
    - /abs/path/to/data/sim-network/r1/frr.conf:/etc/frr/frr.conf.import:ro
    - /abs/path/to/data/sim-network/r1/vtysh.conf:/etc/frr/vtysh.conf:ro
  exec:
    - ip link set dev eth1 up
    - ip link set dev eth2 up
```

The `exec` block runs these commands inside the container immediately after it starts. The `frr.conf` is mounted as `frr.conf.import` (not `frr.conf`) — FRR reads `.import` at startup and writes to the canonical `frr.conf` inside the container.

*Switch nodes*:

```yaml
sw1:
  image: alpine-switch:latest
  exec:
    - ip link set dev eth1 up
    - ip link set dev eth2 up
```

*Terminal nodes*:

```yaml
host1:
  image: alpine-terminal:latest
  exec:
    - ip link set dev eth1 up
```

*Links section*:

```yaml
links:
  - endpoints:
    - "r1:eth1"
    - "sw1:eth1"
```

Containerlab translates each link entry into a veth pair: a virtual Ethernet cable with one end attached to the network namespace of `r1` as `eth1`, and the other end attached to `sw1` as `eth1`.

**Step 7 — Diff check: skip if topology is unchanged**

Before writing the new YAML to disk, the orchestrator reads the existing `data/topology.clab.yml` (if it exists) and compares it byte-for-byte with the newly rendered YAML string. If they are identical, the deploy is skipped entirely:

```json
{
  "status": "skipped",
  "message": "Topology is identical, skipping deploy",
  "details": {
    "name": "sim-network",
    "container_count": 3
  }
}
```

This prevents unnecessary container recreation when the user clicks Deploy without having changed the topology structure.

**Step 8 — Write YAML to disk**

If the topology has changed, the rendered YAML is written to `data/topology.clab.yml` with `os.fsync()` to ensure it is flushed to disk before Containerlab reads it. A 1-second `time.sleep()` is added after the write to allow the filesystem to settle.

**Step 9 — Run `containerlab deploy --reconfigure`**

```bash
sudo containerlab deploy -t data/topology.clab.yml --reconfigure
```

The `--reconfigure` flag tells Containerlab to tear down and recreate containers even if they are already running. This ensures changes to binds, images, or exec commands take effect. Containerlab reads the YAML, pulls any needed images, creates Docker containers, and wires up the veth pairs between linked interfaces.

**Step 10 — Return result**

```json
{
  "status": "success",
  "message": "Topology deployed successfully",
  "details": {
    "name": "sim-network",
    "container_count": 3
  }
}
```

After the deploy succeeds, `saveState(deployed=true)` is called from the frontend, which snapshots the current React Flow state into `topology_deployed_state.json`. This snapshot is used to compute `hasChanges` — whether the current canvas differs from the last-deployed state.

---

## Flow 2: Router Configuration (Zero-Downtime)

This flow covers what happens when the user configures a router's interfaces, VLAN subinterfaces, or routing protocols and clicks **Apply**. No container is restarted — the configuration is diffed and applied live.

### Steps

**Step 1 — User edits router config in the `PropertyPanel`**

The `PropertyPanel` component (right side panel) lets users set IP addresses, netmasks, VLAN subinterfaces, OSPF areas, RIP networks, BGP neighbors, and static routes for the selected router node. All changes update the Zustand store immediately via `updateNodeData()`.

**Step 2 — User clicks "Apply" → frontend sends configuration**

The frontend builds a `RouterConfigureRequest` payload and sends it to:

```
POST /api/v1/nodes/{node_name}/configure
```

Example payload:

```json
{
  "interfaces": [
    { "name": "eth1", "ip_address": "10.0.1.1/24" },
    { "name": "eth2", "ip_address": "10.0.2.1/24" }
  ],
  "vlan_interfaces": [
    { "name": "eth1.10", "parent": "eth1", "vlan_id": 10, "ip_address": "192.168.10.1/24" }
  ],
  "routing": {
    "ospf": {
      "enabled": true,
      "router_id": "10.0.1.1",
      "areas": [
        { "area_id": "0", "networks": ["10.0.1.0/24", "10.0.2.0/24"] }
      ],
      "redistribute": { "connected": true }
    },
    "rip": { "enabled": false, "networks": [] },
    "bgp": { "enabled": false, "as_number": 65001, "router_id": "", "neighbors": [] }
  },
  "static_routes": [],
  "gateway": ""
}
```

**Step 3 — `Orchestrator.configure_node(node_name, config_data)` is called**

**Step 4 — Container lookup via `_get_container_by_name()`**

The orchestrator resolves the node name to a running Docker container by checking naming conventions in order:

1. **Exact match**: `container.name == node_name` (e.g., `"r1"`)
2. **Containerlab prefix** *(when the topology file exists and has a `name:` field)*: `container.name == f"clab-{topo_name}-{node_name}"` (e.g., `"clab-sim-network-r1"`) — the naming convention Containerlab uses by default
3. **Suffix fallback** *(only when the topology file is missing or has no `name:` field — replaces check 2 entirely)*: `container.name.endswith(f"-{node_name}")` — error-recovery path for unparseable topology files

Checks 2 and 3 are mutually exclusive: check 2 runs when `topo_name` is truthy; check 3 runs only when `topo_name` is falsy.

The topology name is read from the first `name:` line in `data/topology.clab.yml`.

**Step 5 — Determine node type**

The container's image name is inspected: if `"frr"` appears in the image name, or `"router"` appears in the node name, `is_router = True`. This check being OR-combined means a container named `"r1"` running any image will be treated as a router.

**Step 6 — VLAN subinterface sync (`_sync_vlan_interfaces()`)**

This function reconciles the Linux kernel VLAN subinterfaces inside the container with the target config:

1. Run `ip link show` inside the container, parse the output to find all existing interfaces whose name contains `"."` (e.g., `eth1.10`, `eth2.20`) — these are VLAN subinterfaces.

2. **Delete stale interfaces**: For each existing VLAN interface not present in the target config:
   ```bash
   ip link delete eth1.10
   ```

3. **Create new interfaces**: For each target VLAN interface not yet existing in the kernel:
   ```bash
   ip link add link eth1 name eth1.10 type vlan id 10
   ip link set dev eth1.10 up
   ```

The `parent` and `vlan_id` fields from the config are used to construct the `ip link add` command. VLAN ID is embedded in both the interface name (`eth1.10`) and the `vlan id` argument.

**Step 7 — Apply VLAN IP addresses at kernel level**

For each VLAN interface with an `ip_address`, the IP is applied directly to the Linux kernel interface (even for routers, where FRR will also manage it):

```bash
ip addr flush dev eth1.10
ip addr add 192.168.10.1/24 dev eth1.10
```

**Step 8 — Jinja2 renders `frr.conf`**

The `frr.conf.j2` template is rendered with the full configuration. The template supports:

- **Physical interfaces** with optional IP addresses
- **VLAN subinterfaces** with optional IP addresses
- **OSPF** with: router-id, areas (including stub, totally-stub, nssa, totally-nssa), per-area `network` statements, per-area ranges (route summarization), redistribution (connected/static/rip/bgp), and `default-information originate`
- **RIP** with: per-network announcements and redistribution (connected/static/ospf/bgp)
- **BGP** with: AS number, router-id, neighbors (ip + remote-as), `no bgp ebgp-requires-policy`, address-family ipv4 unicast block with neighbor activation and redistribution
- **Static routes**: `ip route {destination} {next_hop}`

**Gateway → static route conversion**: If a `gateway` field is provided in the config data and no `0.0.0.0/0` route already exists in `static_routes`, the orchestrator appends a default route automatically:

```
{"destination": "0.0.0.0/0", "next_hop": "<gateway_ip>"}
```

This becomes `ip route 0.0.0.0/0 <gateway_ip>` in the rendered `frr.conf`.

Example rendered `frr.conf` for an OSPF router:

```
log file /var/log/frr/frr.log
!
interface eth1
  ip address 10.0.1.1/24
!
interface eth2
  ip address 10.0.2.1/24
!
!
router ospf
  ospf router-id 10.0.1.1
  network 10.0.1.0/24 area 0
  network 10.0.2.0/24 area 0
  redistribute connected
!
!
```

**Step 9 — Wait for vtysh to become ready**

Before applying the config, the orchestrator polls until FRR is fully initialized inside the container. It runs `vtysh -c "write"` up to 30 times with 1-second intervals:

```bash
docker exec {container_id} vtysh -c "write"
```

If exit code is 0, FRR is ready. After a successful poll, an additional `time.sleep(2)` is added to allow any transient startup state to settle.

**Step 10 — Write new config to container**

The rendered config is written to `/etc/frr/frr.conf.new` inside the container using a heredoc via `docker exec sh -c`:

```bash
docker exec {container_id} sh -c "cat << 'EOF' > /etc/frr/frr.conf.new
log file /var/log/frr/frr.log
!
interface eth1
  ip address 10.0.1.1/24
...
EOF"
```

**Step 11 — Apply via `frr-reload.py` (zero-downtime)**

```bash
docker exec {container_id} /usr/lib/frr/frr-reload.py --reload /etc/frr/frr.conf.new
```

`frr-reload.py` is FRR's built-in live reload tool. It:
1. Reads the currently running FRR config (via `vtysh -c "show running-config"`)
2. Diffs it against the new config file
3. Generates a minimal sequence of `vtysh` commands that transform the old config into the new one
4. Applies those commands — **no daemon restart, no traffic interruption**

**Step 12 — Cleanup temp file**

```bash
docker exec {container_id} rm -f /etc/frr/frr.conf.new
```

**Step 13 — Persist config on the host**

The rendered `frr.conf` is written to the host-side file at `data/{topology_name}/{node_name}/frr.conf`. Because this file is bind-mounted into the container as `/etc/frr/frr.conf.import`, this persists the configuration across container restarts.

**Step 14 — Return result**

```json
{
  "status": "success",
  "output": "... frr-reload.py output ..."
}
```

---

## Flow 3: L2 Switch Configuration

L2 switch configuration uses the Linux kernel's bridge subsystem with VLAN filtering — no external switching software is required.

### Steps

**Step 1 — User configures the switch in `PropertyPanel` and clicks "Apply"**

The user sets each port's VLAN mode (access or trunk) and VLAN IDs in the PropertyPanel. The payload is sent to:

```
POST /api/v1/nodes/{switch_name}/configure
```

Example payload:

```json
{
  "interfaces": [
    { "name": "eth1", "vlan_mode": "access", "vlan_id": 10 },
    { "name": "eth2", "vlan_mode": "trunk", "vlan_ids": [10, 20] },
    { "name": "eth3", "vlan_mode": "access", "vlan_id": 20 }
  ]
}
```

**Step 2 — Container lookup and image check**

The same `_get_container_by_name()` lookup as in Flow 2. If `"switch"` appears in the image name or node name, `is_switch = True`.

**Step 3 — Create the Linux bridge with VLAN filtering**

```bash
ip link add name br0 type bridge vlan_filtering 1 forward_delay 0 stp_state 0
ip link set dev br0 up
```

Parameters explained:
- `vlan_filtering 1` — enables per-port VLAN control; without this, all ports share all VLANs
- `forward_delay 0` — disables the 15-second STP forwarding delay (not needed in a simulated environment)
- `stp_state 0` — disables Spanning Tree Protocol entirely, allowing immediate forwarding

**Step 4 — Add each interface to the bridge**

For each interface in the config:

```bash
ip link set dev eth1 master br0    # add to bridge
ip link set dev eth1 up            # bring up
bridge vlan del dev eth1 vid 1     # remove default VLAN 1 (added automatically by the kernel)
```

If `eth1` cannot be added to the bridge (e.g., already enslaved), a warning is logged and that interface is skipped.

**Step 5 — Configure Access mode ports**

For `vlan_mode == "access"` with `vlan_id = 10`:

```bash
bridge vlan add dev eth1 vid 10 pvid untagged
```

- `pvid` (Port VLAN ID) — untagged frames arriving on `eth1` are classified into VLAN 10
- `untagged` — frames leaving `eth1` have their VLAN tag stripped (the connected host sees untagged traffic)

This is the standard access port behavior: one VLAN, transparent to connected hosts.

**Step 6 — Configure Trunk mode ports**

For `vlan_mode == "trunk"` with `vlan_ids = [10, 20]`:

```bash
bridge vlan add dev eth2 vid 10
bridge vlan add dev eth2 vid 20
```

No `pvid` or `untagged` flags — tagged frames for VLAN 10 and VLAN 20 pass through with their 802.1Q tags intact. This allows multiple VLANs to traverse a single trunk link to a router or another switch.

**Step 7 — Return result**

```json
{
  "status": "success",
  "output": "L2 Switch configured successfully with VLAN filtering"
}
```

---

## Flow 4: WebSocket Terminal Proxy

This flow enables users to interact with a running container's shell through the browser. The backend acts as a transparent bidirectional pipe between the browser's Xterm.js client and the container's bash process.

### Sequence Diagram

```
Browser (Xterm.js)          FastAPI                Docker Container
      |                        |                         |
      |-- WS connect --------->|                         |
      |                        |-- exec_create /bin/bash |
      |                        |-- exec_start() -------->|
      |                        |<--- raw socket ---------|
      |<-- ws.accept() --------|                         |
      |                        |                         |
      |              [anyio task group starts]           |
      |                        |                         |
      |              [task 1: docker_to_ws]              |
      |                        |<- read 8-byte header ---|
      |                        |<- read payload bytes ---|
      |<-- send_text(output) --|                         |
      |                        |                         |
      |              [task 2: ws_to_docker]              |
      |-- send_text(input) --->|                         |
      |                        |-- sendall(bytes) ------>|
      |                        |                         |
      |-- send_text(resize) -->|                         |
      |  {"event":"resize",    |-- exec_resize() ------->|
      |   "cols":120,"rows":30}|                         |
      |                        |                         |
      |-- WS disconnect ------>|                         |
      |                        |-- sock.close() -------->|
      |                        |-- cleanup              |
```

### Steps

**Step 1 — User clicks the terminal icon on a node**

In the React frontend, each node card has a terminal icon button. Clicking it opens the `WebTerminal` component, which initializes an Xterm.js terminal in the browser.

**Step 2 — WebSocket connection is established**

The `WebTerminal` component creates a native WebSocket:

```javascript
new WebSocket("ws://localhost:8000/ws/terminal/r1")
```

**Step 3 — FastAPI accepts the connection**

The `@router.websocket("/ws/terminal/{node_name}")` handler is invoked. It immediately calls `await websocket.accept()` to complete the WebSocket handshake.

**Step 4 — Docker availability check**

An `Orchestrator` instance is created. If `orchestrator.docker_client` is `None` (Docker daemon unreachable), the WebSocket is closed immediately:

```python
await websocket.close(code=4001, reason="Docker daemon not available")
```

**Step 5 — Container lookup**

`orchestrator._get_container_by_name(node_name)` locates the running container using the same name resolution (exact match, then Containerlab prefix or suffix fallback depending on whether `topo_name` is available). If no container is found:

```python
await websocket.close(code=4004, reason=f"Node {node_name} not found")
```

**Step 6 — Create a Docker exec session**

A `docker exec` session is created against the running container, attaching to `/bin/bash` with a TTY:

```python
exec_inst = client.api.exec_create(
    container.id,
    cmd=["/bin/bash"],
    stdin=True,
    stdout=True,
    stderr=True,
    tty=True
)
```

The `tty=True` flag allocates a pseudo-terminal inside the container, enabling features like colored output, cursor movement, and interactive programs (vim, top, etc.).

**Step 7 — Start the exec and unwrap the socket**

```python
docker_socket = client.api.exec_start(exec_inst["Id"], socket=True)
real_sock = _unwrap_socket(docker_socket)
```

`exec_start(..., socket=True)` returns a socket-like object rather than streaming output. `_unwrap_socket()` iterates through common wrapper attributes (`_sock`, `socket`, `_socket`, `raw`) to extract the underlying OS-level socket object, which is required for raw `sendall()` calls.

**Step 8 — Spawn two concurrent anyio tasks**

Both tasks run in an `anyio` task group, meaning they run concurrently and the group exits when both tasks finish (or when one raises an unhandled exception):

```python
async with anyio.create_task_group() as tg:
    tg.start_soon(docker_to_ws)
    tg.start_soon(ws_to_docker)
```

**Task 1: `docker_to_ws` — Container output → Browser**

Reads the Docker exec socket output and forwards it to the WebSocket client. Docker multiplexes stdout and stderr with an 8-byte stream header:

- Bytes 0–3: stream type (1=stdout, 2=stderr)
- Bytes 4–7: payload size as a big-endian 32-bit integer

```python
header = await anyio.to_thread.run_sync(lambda: _read_exactly(8))
size = int.from_bytes(header[4:8], byteorder="big")
payload = await anyio.to_thread.run_sync(lambda: _read_exactly(size))
text = payload.decode("utf-8", errors="replace")
await websocket.send_text(text)
```

All socket reads are wrapped in `anyio.to_thread.run_sync()` to run the blocking socket I/O in a thread pool without blocking the async event loop.

**Task 2: `ws_to_docker` — Browser input → Container**

Receives messages from the WebSocket and writes them to the Docker exec socket. Messages can be one of three types:

*JSON control event — resize*:
```json
{ "event": "resize", "cols": 120, "rows": 30 }
```
→ calls `client.api.exec_resize(exec_id, height=rows, width=cols)` to resize the container's pseudo-terminal

*JSON control event — input*:
```json
{ "event": "input", "data": "ls -la\r" }
```
→ `real_sock.sendall(data.encode("utf-8"))` — writes the keystrokes directly to the container's bash stdin

*Raw text (non-JSON)*:
→ `real_sock.sendall(text_data.encode("utf-8"))` — legacy/fallback raw keystroke forwarding

*Binary data*:
→ `real_sock.sendall(bytes_data)` — direct binary passthrough

**Step 9 — Disconnect and cleanup**

When the user closes the terminal tab (or the container exits), either task encounters a `WebSocketDisconnect` or a socket read error and sets the shared `closed = True` flag. Both tasks terminate, the `anyio` task group exits, and the raw socket is closed:

```python
finally:
    closed = True
    real_sock.close()
```

### WebSocket Close Codes

| Code | Meaning |
|---|---|
| 4001 | Docker daemon is unavailable |
| 4004 | No running container found for the given node name |
| 4005 | Failed to start exec session inside the container |

---

*Navigation: [Architecture Index](./index.md) · [System Overview](./system-overview.md) · [State Management](./state-management.md)*
