> 🇬🇧 English version available → [state-management.md](./state-management.md)

# 状態管理とステートレス設計

## コア原則：フロントエンドが状態を所有する

Network Simulator は意図的に **ステートレスなバックエンド** を中心に設計されています。FastAPI バックエンドはリクエスト間でトポロジー状態をメモリに保持しません。すべての REST 呼び出しは処理を完了するために必要な完全なコンテキストを含み、バックエンドの唯一の永続的な成果物はディスク上のファイル（`topology.clab.yml`、`frr.conf`、`topology_state.json`）です。

React フロントエンド、具体的には `topologyStore.ts` の **Zustand ストア** が、すべてのトポロジー設定の唯一の信頼できる情報源です。これにはノードの位置・インターフェース IP アドレス・VLAN 設定・ルーティングプロトコル設定・エッジの接続情報がすべて含まれます。

## なぜステートレスか？

この設計上の選択により、いくつかの重要な機能が実現されます：

- **再デプロイなしの部分的な再設定**: 単一ノードはいつでも `/api/v1/nodes/{name}/configure` にそのノードの設定だけを送ることで再設定できます。バックエンドはフロントエンドが必要なものをすべて送信するため、以前の状態を記憶する必要がありません。
- **ページリロード耐性**: ブラウザタブをリフレッシュしても、`loadState()` がバックエンドの JSON ファイルから最後に保存された状態を取得し、Containerlab からライブのコンテナステータスを取得して、Zustand ストアをシームレスに再水和します。
- **信頼できる差分追跡**: フロントエンドが常にトポロジー全体を持っているため、`nodes[]` + `edges[]` と `deployedNodes[]` + `deployedEdges[]` を比較することで、キャンバスが最後にデプロイされた状態から変化したかどうかを計算できます。
- **シンプルなバックエンド**: バックエンドにはセッション管理もメモリ内キャッシュもトポロジーステートマシンもありません。各 `Orchestrator` インスタンスはリクエストごとに新しく作成されます。

## `topologyStore.ts` — Zustand ストア

### 状態インターフェース

```typescript
interface TopologyState {
  // ライブキャンバス状態（ユーザーが現在描いているもの）
  nodes: Node[];              // React Flow ノード（router, switch, host）
  edges: Edge[];              // React Flow エッジ（ポート間の接続）

  // デプロイ済みスナップショット（最後に正常にデプロイされたもの）
  deployedNodes: Node[];      // 最後のデプロイ時の nodes[] のディープコピー
  deployedEdges: Edge[];      // 最後のデプロイ時の edges[] のディープコピー

  // 選択状態
  selectedNodeId: string | null;
  selectedEdgeId: string | null;

  // UI フラグ
  hasChanges: boolean;        // nodes/edges が deployedNodes/deployedEdges と異なる場合 true
  isSaving: boolean;          // saveState() の HTTP 呼び出し中は true
}
```

### `hasChanges` の計算

`hasChanges` はカスタムの `set()` ラッパーによってすべての状態更新時に自動的に再計算されます。比較には `checkHasChanges()` 関数が使用されており：

1. 各ノードから位置データ（`position`、`positionAbsolute`、`width`、`height`、`selected`、`dragging`）を除去します — 位置変更だけでは「変更あり」の警告を出すべきではないため
2. ノードデータから `status` フィールドを除去します — コンテナステータスはランタイム状態であり設定状態ではないため
3. 両方の配列をノード/エッジの `id` でソートします
4. JSON 文字列の深い比較を行います

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

### 自動保存（デバウンス）

`nodes` または `edges` を変更する状態変化は、2 秒のデバウンスで自動保存をトリガーします：

```typescript
function triggerAutoSave() {
  clearTimeout(autoSaveTimeoutId);
  autoSaveTimeoutId = setTimeout(() => {
    useTopologyStore.getState().saveState(false);  // デプロイ済みとしてマークせずに保存
  }, 2000);
}
```

つまり、ユーザーが編集（ノードのドラッグ、プロパティの変更）を止めてから 2 秒後に、キャンバスの状態が自動的にバックエンドの `topology_state.json` に永続化されます — ユーザーが手動で保存する必要はありません。

### 主要なアクション

| アクション | 説明 |
|---|---|
| `addNode(type)` | `"router"`、`"host"`、`"switch"` タイプのノードをデフォルトデータで新規作成。次に利用可能なアルファベットのラベルを自動付与（Router-A、Router-B、...） |
| `deleteNode(id)` | ノードとそれに接続されたすべてのエッジを削除。削除されたノードが選択中だった場合は `selectedNodeId` をクリア |
| `updateNodeData(nodeId, data)` | 部分的なデータをノードの `data` フィールドにマージ。自動保存をトリガー |
| `deleteEdge(id)` | エッジを削除し、両端のノードの `connectedTo` 参照をクリア |
| `addPort(nodeId)` | ルーターまたはスイッチに新しいインターフェースを追加。現在の最大番号より 1 大きい `eth{N}` という名前を付ける |
| `deletePort(nodeId, portName)` | ルーターまたはスイッチからインターフェースを削除。そのポートを使用するエッジも削除 |
| `onConnect(connection)` | ユーザーが接続ハンドルをドラッグしたとき React Flow から呼ばれる。新しいエッジを作成し、両端のノードの `connectedTo` を更新 |
| `setTopology(nodes, edges)` | 現在のコンテナステータスを取得し、ノードデータに `"up"/"down"` をマージして、ストアの `nodes` と `edges` をセット |
| `setDeployedState(nodes, edges)` | 指定されたノード/エッジを `deployedNodes`/`deployedEdges` にディープコピー。デプロイ完了のマークに使用 |
| `saveState(deployed?)` | 現在の `nodes` + `edges` をバックエンドに永続化。`deployed=true` の場合は `deployedNodes`/`deployedEdges` も更新して `hasChanges` をリセット |
| `loadState()` | バックエンドから `topology_state.json` と `topology_deployed_state.json` を取得、ライブコンテナステータスをマージ、ストアを再水和 |
| `resetTopologyState()` | バックエンドの両 JSON ファイルを削除。ストアをデフォルトの 2 ルーター + 1 ホストキャンバスにリセット |

### デフォルト初期状態

ストアが最初に作成されるとき（または `resetTopologyState()` の後）、キャンバスには以下が表示されます：

```
┌─────────────┐                    ┌─────────────┐
│  Router-A   │──── eth1──eth1 ────│   Router-B  │
│  (router-1) │                    │  (router-2) │
└─────────────┘                    └─────────────┘
       │ eth1
       │
┌─────────────┐
│   Host-A    │
│   (host-1)  │
└─────────────┘
```

- **Router-A** (`id: "router-1"`): 4 インターフェース（eth1–eth4）、すべてのルーティングプロトコル無効
- **Router-B** (`id: "router-2"`): 4 インターフェース（eth1–eth4）、すべてのルーティングプロトコル無効
- **Host-A** (`id: "host-1"`): IP アドレスなし、ゲートウェイなし
- **エッジ**: `router-1:eth1` → `host-1:eth1` を接続

ルーターは `initialRouterData()` で初期化されます：

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

スイッチは 4 インターフェース、すべて VLAN 1 の Access モードで初期化されます：

```typescript
interfaces: [
  { id: "eth1", name: "eth1", vlanMode: "access", vlanId: 1, vlanIds: [] },
  ...
]
```

## ノードデータ型（`types/topology.ts`）

### `RouterNodeData`

```typescript
interface RouterNodeData {
  label: string;
  status: 'up' | 'down';
  interfaces: InterfaceData[];         // 物理ポート（eth1, eth2, ...）
  vlanInterfaces: VlanInterfaceData[]; // VLAN サブインターフェース（eth1.10, eth2.20, ...）
  routing: {
    ospf: OspfConfig;
    rip: RipConfig;
    bgp: BgpConfig;
  };
  staticRoutes: StaticRoute[];
}

interface InterfaceData {
  id: string;         // name と同じ（例: "eth1"）
  name: string;
  ipAddress: string;  // 例: "10.0.1.1"
  netmask: string;    // 例: "255.255.255.0" または "24"
  connectedTo?: string; // 接続先ノード ID
  adminState?: 'up' | 'down';
}

interface VlanInterfaceData {
  name: string;            // 例: "eth1.10"
  parentInterface: string; // 例: "eth1"
  vlanId: number;          // 例: 10
  ipAddress: string;       // CIDR 形式、例: "10.10.10.1/24"
}
```

OSPF 設定はエリアのサブタイプ（normal/stub/totally-stub/nssa/totally-nssa）、エリアごとのレンジ（ルート集約）、connected/static/rip/bgp からの再配送、オプションの `always` フラグとメトリックを持つ `default-information originate` をサポートします。

BGP 設定は AS 番号・ルーター ID・複数のネイバー（IP と remote-AS の組み合わせ）・再配送をサポートします。

### `HostNodeData`

```typescript
interface HostNodeData {
  label: string;
  status: 'up' | 'down';
  ipAddress: string;    // CIDR 形式、例: "192.168.1.10/24"
  gateway: string;      // 例: "192.168.1.1"
  connectedTo?: string; // 接続先ノード ID
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
  vlanId?: number;    // vlanMode == "access" のとき使用（1–4094）
  vlanIds?: number[]; // vlanMode == "trunk" のとき使用（例: [10, 20]）
  connectedTo?: string;
  adminState?: 'up' | 'down';
}
```

### `NetworkEdgeData`

```typescript
interface NetworkEdgeData {
  bandwidth?: string;        // 例: "1Gbps"（情報提供のみ）
  delay?: string;            // 例: "10ms"（情報提供のみ）
  cost?: number;             // 例: 10（情報提供のみ）
  sourceInterface?: string;  // 例: "eth1"
  targetInterface?: string;  // 例: "eth2"
  status?: 'up' | 'down';
}
```

## 状態の永続化

### 保存

```
saveState(deployed = false)
    │
    ├─ ストアから現在の nodes + edges を読み取る
    ├─ POST /api/v1/topology/state
    │     ボディ: { nodes, edges }
    │     クエリ: ?deployed=false
    │
    ▼  バックエンドが書き込む:
    topology_state.json           ← 常に書き込まれる
    topology_deployed_state.json  ← deployed=true のときのみ

    deployed=true の場合:
    ├─ nodes/edges を deployedNodes/deployedEdges にディープコピー
    └─ hasChanges = false にリセット
```

### 読み込み

```
loadState()
    │
    ├─ GET /api/v1/topology/state               → topology_state.json
    │     （見つからない場合は default_topology.json にフォールバック）
    ├─ GET /api/v1/topology/state?deployed=true → topology_deployed_state.json
    │     （見つからない場合は { nodes: [], edges: [] } を返す）
    ├─ GET /api/v1/topology/status              → containerlab inspect の出力
    │
    ├─ コンテナの実行ステータスを各ノードの data.status フィールドにマージ
    ├─ hasChanges を再計算
    └─ ストアに nodes, edges, deployedNodes, deployedEdges をセット
```

### リセット

```
resetTopologyState()
    │
    ├─ DELETE /api/v1/topology/state
    │     バックエンドが topology_state.json と topology_deployed_state.json を両方削除
    │
    └─ ストアをデフォルトの 2 ルーター + 1 ホストキャンバスにリセット
         deployedNodes = [], deployedEdges = [], hasChanges = true
```

## 状態フロー図

```
ユーザーアクション
    │
    ▼
React Flow UI
(nodes[], edges[])
    │
    ├── onNodesChange / onEdgesChange  ← React Flow 組み込みイベントハンドラー
    │        │
    │        └── triggerAutoSave() をトリガー → 2 秒デバウンス → saveState(false)
    │
    ├── updateNodeData()               ← PropertyPanel での編集
    │        │
    │        └── triggerAutoSave() をトリガー
    │
    └── onConnect()                    ← ユーザーが接続ハンドルをドラッグ
             │
             └── エッジを作成、両ノードの connectedTo を更新
                  └── triggerAutoSave() をトリガー

Deploy ボタン
    │
    ▼
setTopology(nodes, edges)
    │
    ├── /topology/status を取得 → コンテナステータスをノードにマージ
    ├── ストアに nodes + edges をセット
    ├── POST /topology/deploy → オーケストレーター → containerlab deploy
    └── saveState(deployed=true)
             │
             ├── POST /topology/state?deployed=true → topology_deployed_state.json
             ├── nodes を deployedNodes にディープコピー
             └── hasChanges = false

ページロード
    │
    ▼
loadState()
    │
    ├── GET /topology/state → topology_state.json
    ├── GET /topology/state?deployed=true → topology_deployed_state.json
    ├── GET /topology/status → ライブステータスをマージ
    └── ストアを水和（nodes, edges, deployedNodes, deployedEdges, hasChanges）
```

---

*ナビゲーション: [アーキテクチャインデックス](./index.ja.md) · [システム概要](./system-overview.ja.md) · [データフロー](./data-flows.ja.md)*
