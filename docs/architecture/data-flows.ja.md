> 🇬🇧 English version available → [data-flows.md](./data-flows.md)

# データフロー

このドキュメントでは、Network Simulator における 4 つの主要なデータフローをステップバイステップで詳しく解説します：トポロジーデプロイ・ルーター設定・L2 スイッチ設定・WebSocket ターミナルプロキシ。これらのフローを理解することが、システム全体の動作を深く理解するカギとなります。

---

## フロー 1：トポロジーデプロイ

このフローは、ユーザーが UI の **Deploy ボタン** をクリックしたときに起きるすべての処理をカバーします。ブラウザから始まり、バックエンドスタック全体を経て、仮想イーサネットペアで接続された実行中の Docker コンテナが完成するまでの流れです。

### ステップ

**ステップ 1 — ユーザーが Header コンポーネントの "Deploy" をクリック**

`Header` React コンポーネントに Deploy ボタンがあります。クリックすると、Zustand ストアの `setTopology()` アクション（現在の `nodes` と `edges` 配列を渡す）が呼ばれ、その後シリアライズされたトポロジーペイロードで `POST /api/v1/topology/deploy` が送信されます。

**ステップ 2 — `setTopology()` がトポロジーペイロードを構築**

バックエンドへ送信する前に、`setTopology()` は `GET /api/v1/topology/status` を呼び出して現在実行中のコンテナを確認し、各ノードデータに `"up"/"down"` ステータスをマージします。`/deploy` へ送信される JSON ペイロードは次の 3 フィールドを持ちます：

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

各リンクのエンドポイントは `"node_name:interface_name"` 形式の文字列です。React Flow のエッジの `sourceInterface` と `targetInterface` がリンク両端のインターフェース名になります。

**ステップ 3 — FastAPI がリクエストを検証**

`POST /api/v1/topology/deploy` FastAPI ルートがリクエストを受け取り、Pydantic の `TopologyDeployRequest` モデルに対してバリデーションを行います。バリデーション失敗時は FastAPI が自動的に `422 Unprocessable Entity` を返します。

**ステップ 4 — `Orchestrator.deploy_topology(topology_data)` が呼び出される**

REST ハンドラーが `Orchestrator` インスタンスを作成し、バリデーション済みのトポロジーデータをディクショナリとして `deploy_topology()` に渡します。

**ステップ 5 — ルーター事前セットアップ：設定ディレクトリと初期ファイルの作成**

`type == "router"` のノードごとに、オーケストレーターは `data/{topology_name}/{node_name}/` にルーターごとの設定ディレクトリを作成し、3 つのファイルを書き込みます：

```
data/sim-network/r1/
├── daemons       ← FRR デーモンの有効/無効リスト
├── vtysh.conf    ← vtysh 統合設定
└── frr.conf      ← 初期（空）ルーティング設定
```

**`daemons` ファイル** — 以下の内容がそのまま書き込まれます：

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

**`vtysh.conf` ファイル** — 統合設定モードを有効化：

```
service integrated-vtysh-config
```

**`frr.conf` ファイル** — 最小限のスタブで初期化（実際の設定は後で `configure_node()` が適用）：

```
log file /var/log/frr/frr.log
!
```

すべてのディレクトリとファイルは `chmod 777` で作成されます。これは、コンテナ内で非 root ユーザーとして動作する FRR プロセスが、バインドマウント経由でこれらのファイルを読み取れるようにするためです。

> **エッジケース**: これらのパスのいずれかが（破損した以前の実行により）*ディレクトリ* として存在している場合、コードはファイル作成前に `shutil.rmtree()` でそれを削除します。すべての書き込み後に `os.fsync()` を呼び出してデータの永続性を保証します。

**ステップ 6 — Jinja2 が `topology.clab.yml` をレンダリング**

`topology.clab.yml.j2` テンプレートがトポロジーデータでレンダリングされ、完全な Containerlab YAML ファイルが生成されます。各ノードタイプに対してテンプレートが生成する内容：

*ルーターノード*:

```yaml
r1:
  image: alpine-frr:latest
  binds:
    - /絶対パス/data/sim-network/r1/daemons:/etc/frr/daemons:ro
    - /絶対パス/data/sim-network/r1/frr.conf:/etc/frr/frr.conf.import:ro
    - /絶対パス/data/sim-network/r1/vtysh.conf:/etc/frr/vtysh.conf:ro
  exec:
    - ip link set dev eth1 up
    - ip link set dev eth2 up
```

`exec` ブロックはコンテナ起動直後にコンテナ内でこれらのコマンドを実行します。`frr.conf` は `/etc/frr/frr.conf.import`（`/etc/frr/frr.conf` ではない）としてマウントされます。FRR は起動時に `.import` ファイルを読み込み、標準の `frr.conf` に書き込みます。

*スイッチノード*:

```yaml
sw1:
  image: alpine-switch:latest
  exec:
    - ip link set dev eth1 up
    - ip link set dev eth2 up
```

*ターミナルノード*:

```yaml
host1:
  image: alpine-terminal:latest
  exec:
    - ip link set dev eth1 up
```

*リンクセクション*:

```yaml
links:
  - endpoints:
    - "r1:eth1"
    - "sw1:eth1"
```

Containerlab は各リンクエントリを veth ペアに変換します：一方の端が `r1` のネットワーク名前空間に `eth1` として、もう一方が `sw1` に `eth1` として接続された仮想イーサネットケーブルです。

**ステップ 7 — 差分チェック：トポロジーが変わっていなければスキップ**

新しい YAML をディスクに書き込む前に、オーケストレーターは既存の `data/topology.clab.yml`（存在する場合）を読み込み、新たにレンダリングされた YAML 文字列とバイト単位で比較します。一致する場合、デプロイは完全にスキップされます：

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

これにより、トポロジー構造を変更せずに Deploy をクリックした場合の不要なコンテナ再作成を防ぎます。

**ステップ 8 — YAML をディスクに書き込み**

トポロジーが変更されている場合、レンダリングされた YAML が `data/topology.clab.yml` に書き込まれ、`os.fsync()` で Containerlab が読み込む前にディスクにフラッシュされることを保証します。書き込み後に `time.sleep(1)` を追加してファイルシステムを安定させます。

**ステップ 9 — `containerlab deploy --reconfigure` を実行**

```bash
sudo containerlab deploy -t data/topology.clab.yml --reconfigure
```

`--reconfigure` フラグは、すでに実行中のコンテナでも強制的に再作成するよう Containerlab に指示します。Containerlab は YAML を読み込み、必要なイメージをプルし、Docker コンテナを作成し、リンクされたインターフェース間に veth ペアを配線します。

**ステップ 10 — 結果を返す**

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

デプロイ成功後、フロントエンドから `saveState(deployed=true)` が呼ばれ、現在の React Flow 状態を `topology_deployed_state.json` にスナップショットとして保存します。このスナップショットが `hasChanges`（現在のキャンバスが最後にデプロイされた状態と異なるか）の計算に使用されます。

---

## フロー 2：ルーター設定（ゼロダウンタイム）

このフローは、ユーザーがルーターのインターフェース・VLAN サブインターフェース・ルーティングプロトコルを設定して **Apply** をクリックしたときに起きることをカバーします。コンテナは再起動されず、設定は差分比較されてライブ適用されます。

### ステップ

**ステップ 1 — ユーザーが `PropertyPanel` でルーター設定を編集**

`PropertyPanel` コンポーネント（右側パネル）では、選択されたルーターノードの IP アドレス・ネットマスク・VLAN サブインターフェース・OSPF エリア・RIP ネットワーク・BGP ネイバー・スタティックルートを設定できます。すべての変更は `updateNodeData()` 経由で即座に Zustand ストアに反映されます。

**ステップ 2 — "Apply" クリック → フロントエンドが設定を送信**

フロントエンドは `RouterConfigureRequest` ペイロードを構築し、以下へ送信します：

```
POST /api/v1/nodes/{node_name}/configure
```

ペイロード例：

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

**ステップ 3 — `Orchestrator.configure_node(node_name, config_data)` が呼び出される**

**ステップ 4 — `_get_container_by_name()` によるコンテナ検索**

オーケストレーターは以下の順で実行中の Docker コンテナを特定します：

1. **完全一致**: `container.name == node_name`（例: `"r1"`）
2. **Containerlab プレフィックス** *（トポロジーファイルが存在し `name:` フィールドがある場合）*: `container.name == f"clab-{topo_name}-{node_name}"`（例: `"clab-sim-network-r1"`）— Containerlab がデフォルトで使用する命名規則
3. **サフィックスフォールバック** *（トポロジーファイルが存在しないか `name:` フィールドがない場合のみ — チェック 2 の代替として実行）*: `container.name.endswith(f"-{node_name}")` — トポロジーファイルが解析不能な場合のエラー回復パス

チェック 2 とチェック 3 は相互排他です：`topo_name` が真値のときはチェック 2 が実行され、`topo_name` が偽値のときのみチェック 3 が実行されます。

トポロジー名は `data/topology.clab.yml` の最初の `name:` 行から読み込まれます。

**ステップ 5 — ノードタイプの判定**

コンテナのイメージ名を検査します：イメージ名に `"frr"` が含まれる、またはノード名に `"router"` が含まれる場合、`is_router = True`。この条件が OR で結合されているため、任意のイメージで動作する `"r1"` という名前のコンテナもルーターとして扱われます。

**ステップ 6 — VLAN サブインターフェース同期（`_sync_vlan_interfaces()`）**

この関数は、コンテナ内の Linux カーネル VLAN サブインターフェースをターゲット設定と突き合わせます：

1. コンテナ内で `ip link show` を実行し、出力を解析して名前に `"."` を含む既存インターフェース（例: `eth1.10`、`eth2.20`）を探します — これらが VLAN サブインターフェースです。

2. **不要なインターフェースを削除**: ターゲット設定に存在しない既存の VLAN インターフェースごとに：
   ```bash
   ip link delete eth1.10
   ```

3. **新しいインターフェースを作成**: カーネルにまだ存在しないターゲット VLAN インターフェースごとに：
   ```bash
   ip link add link eth1 name eth1.10 type vlan id 10
   ip link set dev eth1.10 up
   ```

設定の `parent` と `vlan_id` フィールドが `ip link add` コマンドの構築に使われます。VLAN ID はインターフェース名（`eth1.10`）と `vlan id` 引数の両方に埋め込まれます。

**ステップ 7 — カーネルレベルで VLAN IP アドレスを適用**

`ip_address` を持つ VLAN インターフェースごとに、IP を Linux カーネルインターフェースに直接適用します（FRR が管理するルーターでも同様に実行）：

```bash
ip addr flush dev eth1.10
ip addr add 192.168.10.1/24 dev eth1.10
```

**ステップ 8 — Jinja2 が `frr.conf` をレンダリング**

`frr.conf.j2` テンプレートが完全な設定でレンダリングされます。テンプレートは以下をサポートします：

- **物理インターフェース** — オプションの IP アドレス付き
- **VLAN サブインターフェース** — オプションの IP アドレス付き
- **OSPF** — ルーター ID・エリア（stub、totally-stub、nssa、totally-nssa 対応）・エリアごとの `network` 文・エリアごとのレンジ（ルート集約）・再配送（connected/static/rip/bgp）・`default-information originate`
- **RIP** — ネットワークごとのアナウンスと再配送（connected/static/ospf/bgp）
- **BGP** — AS 番号・ルーター ID・ネイバー（ip + remote-as）・`no bgp ebgp-requires-policy`・address-family ipv4 unicast ブロック（ネイバーアクティベーションと再配送）
- **スタティックルート**: `ip route {destination} {next_hop}`

**ゲートウェイ → スタティックルート変換**: 設定データに `gateway` フィールドが存在し、`static_routes` に `0.0.0.0/0` ルートがまだない場合、オーケストレーターは自動的にデフォルトルートを追加します：

```
{"destination": "0.0.0.0/0", "next_hop": "<gateway_ip>"}
```

これにより `frr.conf` に `ip route 0.0.0.0/0 <gateway_ip>` が生成されます。

OSPF ルーターのレンダリング済み `frr.conf` 例：

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

**ステップ 9 — vtysh の準備完了を待機**

設定を適用する前に、オーケストレーターはコンテナ内で FRR が完全に初期化されるまでポーリングします。`vtysh -c "write"` を最大 30 回、1 秒間隔で実行します：

```bash
docker exec {container_id} vtysh -c "write"
```

終了コードが 0 なら FRR は準備完了です。ポーリング成功後、過渡的な起動状態が落ち着くよう追加で `time.sleep(2)` が入ります。

**ステップ 10 — コンテナに新しい設定を書き込む**

レンダリングされた設定を `docker exec sh -c` 経由のヒアドキュメントでコンテナ内の `/etc/frr/frr.conf.new` に書き込みます：

```bash
docker exec {container_id} sh -c "cat << 'EOF' > /etc/frr/frr.conf.new
log file /var/log/frr/frr.log
!
interface eth1
  ip address 10.0.1.1/24
...
EOF"
```

**ステップ 11 — `frr-reload.py` 経由でゼロダウンタイム適用**

```bash
docker exec {container_id} /usr/lib/frr/frr-reload.py --reload /etc/frr/frr.conf.new
```

`frr-reload.py` は FRR 組み込みのライブリロードツールです。動作の流れ：
1. 現在の実行中 FRR 設定を読み込む（`vtysh -c "show running-config"` 経由）
2. 新しい設定ファイルと差分比較する
3. 旧設定を新設定に変換する最小限の `vtysh` コマンド列を生成する
4. そのコマンドを適用する — **デーモン再起動なし、トラフィック断なし**

**ステップ 12 — 一時ファイルを削除**

```bash
docker exec {container_id} rm -f /etc/frr/frr.conf.new
```

**ステップ 13 — ホスト側に設定を永続化**

レンダリングされた `frr.conf` がホスト側のファイル `data/{topology_name}/{node_name}/frr.conf` に書き込まれます。このファイルはコンテナに `/etc/frr/frr.conf.import` としてバインドマウントされているため、コンテナ再起動をまたいで設定が永続化されます。

**ステップ 14 — 結果を返す**

```json
{
  "status": "success",
  "output": "... frr-reload.py の出力 ..."
}
```

---

## フロー 3：L2 スイッチ設定

L2 スイッチ設定は、外部スイッチングソフトウェアを使わず、Linux カーネルのブリッジサブシステムと VLAN フィルタリングを使用します。

### ステップ

**ステップ 1 — ユーザーが `PropertyPanel` でスイッチを設定して "Apply" をクリック**

ユーザーが PropertyPanel で各ポートの VLAN モード（access または trunk）と VLAN ID を設定します。ペイロードは以下へ送信されます：

```
POST /api/v1/nodes/{switch_name}/configure
```

ペイロード例：

```json
{
  "interfaces": [
    { "name": "eth1", "vlan_mode": "access", "vlan_id": 10 },
    { "name": "eth2", "vlan_mode": "trunk", "vlan_ids": [10, 20] },
    { "name": "eth3", "vlan_mode": "access", "vlan_id": 20 }
  ]
}
```

**ステップ 2 — コンテナ検索とイメージ確認**

フロー 2 と同じ `_get_container_by_name()` によるコンテナ検索。イメージ名またはノード名に `"switch"` が含まれる場合、`is_switch = True`。

**ステップ 3 — VLAN フィルタリング付き Linux ブリッジを作成**

```bash
ip link add name br0 type bridge vlan_filtering 1 forward_delay 0 stp_state 0
ip link set dev br0 up
```

各パラメーターの説明：
- `vlan_filtering 1` — ポートごとの VLAN 制御を有効化。これがないとすべてのポートがすべての VLAN を共有する
- `forward_delay 0` — 15 秒の STP 転送遅延を無効化（シミュレーション環境では不要）
- `stp_state 0` — スパニングツリープロトコルを完全に無効化し、即時転送を可能にする

**ステップ 4 — 各インターフェースをブリッジに追加**

設定内の各インターフェースに対して：

```bash
ip link set dev eth1 master br0    # ブリッジに追加
ip link set dev eth1 up            # リンクアップ
bridge vlan del dev eth1 vid 1     # デフォルト VLAN 1 を削除（カーネルが自動追加）
```

`eth1` がブリッジに追加できない場合（例: すでに他のブリッジに所属）、警告をログに記録してそのインターフェースをスキップします。

**ステップ 5 — Access モードポートを設定**

`vlan_mode == "access"` で `vlan_id = 10` の場合：

```bash
bridge vlan add dev eth1 vid 10 pvid untagged
```

- `pvid`（Port VLAN ID）— `eth1` に届くタグなしフレームを VLAN 10 として分類
- `untagged` — `eth1` から出るフレームの VLAN タグを除去（接続先ホストはタグなしトラフィックを受け取る）

これが標準的な Access ポートの動作：1 つの VLAN、接続先ホストには透明。

**ステップ 6 — Trunk モードポートを設定**

`vlan_mode == "trunk"` で `vlan_ids = [10, 20]` の場合：

```bash
bridge vlan add dev eth2 vid 10
bridge vlan add dev eth2 vid 20
```

`pvid` や `untagged` フラグなし — VLAN 10 と VLAN 20 のタグ付きフレームは 802.1Q タグを保持したまま通過します。これにより、1 本のトランクリンクでルーターや別のスイッチへ複数の VLAN を通すことができます。

**ステップ 7 — 結果を返す**

```json
{
  "status": "success",
  "output": "L2 Switch configured successfully with VLAN filtering"
}
```

---

## フロー 4：WebSocket ターミナルプロキシ

このフローにより、ユーザーがブラウザから実行中のコンテナのシェルと対話できます。バックエンドはブラウザの Xterm.js クライアントとコンテナの bash プロセスの間の透過的な双方向パイプとして機能します。

### シーケンス図

```
ブラウザ (Xterm.js)       FastAPI               Docker コンテナ
      |                        |                         |
      |-- WS 接続 ------------>|                         |
      |                        |-- exec_create /bin/bash |
      |                        |-- exec_start() -------->|
      |                        |<--- raw socket ---------|
      |<-- ws.accept() --------|                         |
      |                        |                         |
      |           [anyio タスクグループ開始]              |
      |                        |                         |
      |           [タスク 1: docker_to_ws]               |
      |                        |<- 8 バイトヘッダー読み取り|
      |                        |<- ペイロードバイト読み取り|
      |<-- send_text(出力) ----|                         |
      |                        |                         |
      |           [タスク 2: ws_to_docker]               |
      |-- send_text(入力) ---->|                         |
      |                        |-- sendall(バイト) ------>|
      |                        |                         |
      |-- send_text(resize) -->|                         |
      |  {"event":"resize",    |-- exec_resize() ------->|
      |   "cols":120,"rows":30}|                         |
      |                        |                         |
      |-- WS 切断 ------------>|                         |
      |                        |-- sock.close() -------->|
      |                        |-- クリーンアップ         |
```

### ステップ

**ステップ 1 — ユーザーがノードのターミナルアイコンをクリック**

React フロントエンドでは、各ノードカードにターミナルアイコンボタンがあります。クリックすると `WebTerminal` コンポーネントが開き、ブラウザ内で Xterm.js ターミナルを初期化します。

**ステップ 2 — WebSocket 接続が確立される**

`WebTerminal` コンポーネントがネイティブ WebSocket を作成します：

```javascript
new WebSocket("ws://localhost:8000/ws/terminal/r1")
```

**ステップ 3 — FastAPI が接続を受け入れる**

`@router.websocket("/ws/terminal/{node_name}")` ハンドラーが呼び出され、即座に `await websocket.accept()` を呼んで WebSocket ハンドシェイクを完了します。

**ステップ 4 — Docker 利用可能チェック**

`Orchestrator` インスタンスが作成されます。`orchestrator.docker_client` が `None`（Docker デーモン到達不可）の場合、WebSocket を即座にクローズします：

```python
await websocket.close(code=4001, reason="Docker daemon not available")
```

**ステップ 5 — コンテナの検索**

`orchestrator._get_container_by_name(node_name)` が同じ名前解決（完全一致、次に `topo_name` の有無に応じて Containerlab プレフィックスまたはサフィックスフォールバック）で実行中のコンテナを特定します。コンテナが見つからない場合：

```python
await websocket.close(code=4004, reason=f"Node {node_name} not found")
```

**ステップ 6 — Docker exec セッションを作成**

TTY 付きで `/bin/bash` にアタッチした `docker exec` セッションを作成します：

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

`tty=True` フラグはコンテナ内に擬似端末（pseudo-terminal）を割り当て、カラー出力・カーソル移動・インタラクティブプログラム（vim、top など）を有効にします。

**ステップ 7 — exec を開始してソケットをアンラップ**

```python
docker_socket = client.api.exec_start(exec_inst["Id"], socket=True)
real_sock = _unwrap_socket(docker_socket)
```

`exec_start(..., socket=True)` はストリーム出力ではなくソケット状オブジェクトを返します。`_unwrap_socket()` は一般的なラッパー属性（`_sock`、`socket`、`_socket`、`raw`）を順に確認し、生の `sendall()` 呼び出しに必要な OS レベルのソケットオブジェクトを取り出します。

**ステップ 8 — 2 つの並行 anyio タスクを起動**

両タスクは `anyio` タスクグループで実行され、両タスクが終了する（または一方がハンドルされない例外を発生させる）までグループは継続します：

```python
async with anyio.create_task_group() as tg:
    tg.start_soon(docker_to_ws)
    tg.start_soon(ws_to_docker)
```

**タスク 1: `docker_to_ws` — コンテナ出力 → ブラウザ**

Docker exec ソケットの出力を読み込み、WebSocket クライアントに転送します。Docker は stdout と stderr を 8 バイトのストリームヘッダーで多重化します：

- バイト 0–3: ストリームタイプ（1=stdout、2=stderr）
- バイト 4–7: ペイロードサイズ（ビッグエンディアン 32 ビット整数）

```python
header = await anyio.to_thread.run_sync(lambda: _read_exactly(8))
size = int.from_bytes(header[4:8], byteorder="big")
payload = await anyio.to_thread.run_sync(lambda: _read_exactly(size))
text = payload.decode("utf-8", errors="replace")
await websocket.send_text(text)
```

すべてのソケット読み取りは `anyio.to_thread.run_sync()` でラップされ、ブロッキングなソケット I/O を非同期イベントループをブロックせずにスレッドプールで実行します。

**タスク 2: `ws_to_docker` — ブラウザ入力 → コンテナ**

WebSocket からメッセージを受け取り、Docker exec ソケットに書き込みます。メッセージは 3 種類あります：

*JSON 制御イベント — resize（ターミナルサイズ変更）*:
```json
{ "event": "resize", "cols": 120, "rows": 30 }
```
→ `client.api.exec_resize(exec_id, height=rows, width=cols)` を呼び出してコンテナの擬似端末をリサイズ

*JSON 制御イベント — input（キー入力）*:
```json
{ "event": "input", "data": "ls -la\r" }
```
→ `real_sock.sendall(data.encode("utf-8"))` — キーストロークをコンテナの bash stdin に直接書き込む

*Raw テキスト（非 JSON）*:
→ `real_sock.sendall(text_data.encode("utf-8"))` — レガシー/フォールバックの生キーストロー転送

*バイナリデータ*:
→ `real_sock.sendall(bytes_data)` — 直接バイナリパススルー

**ステップ 9 — 切断とクリーンアップ**

ユーザーがターミナルタブを閉じた（またはコンテナが終了した）とき、どちらかのタスクが `WebSocketDisconnect` またはソケット読み取りエラーに遭遇し、共有の `closed = True` フラグを設定します。両タスクが終了し、`anyio` タスクグループが終了し、生のソケットがクローズされます：

```python
finally:
    closed = True
    real_sock.close()
```

### WebSocket クローズコード

| コード | 意味 |
|---|---|
| 4001 | Docker デーモンが利用不可 |
| 4004 | 指定したノード名の実行中コンテナが見つからない |
| 4005 | コンテナ内で exec セッションの開始に失敗 |

---

*ナビゲーション: [アーキテクチャインデックス](./index.ja.md) · [システム概要](./system-overview.ja.md) · [状態管理](./state-management.ja.md)*
