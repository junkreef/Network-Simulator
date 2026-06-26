> 🇬🇧 English version available → [system-overview.md](./system-overview.md)

# システムコンポーネントと技術スタック

## コンポーネント図

```
┌─────────────────────────────────────────────────────────┐
│                      ブラウザ                             │
│  ┌──────────────────────────────────────────────────┐   │
│  │         React フロントエンド (:5173)              │   │
│  │  ┌──────────────┐  ┌──────────┐  ┌───────────┐  │   │
│  │  │  React Flow  │  │ Zustand  │  │ Xterm.js  │  │   │
│  │  │  （キャンバス）│  │  ストア  │  │（ターミナル）│  │   │
│  │  └──────┬───────┘  └────┬─────┘  └─────┬─────┘  │   │
│  └─────────┼───────────────┼──────────────┼─────────┘   │
│            │ REST/JSON     │              │ WebSocket    │
└────────────┼───────────────┼──────────────┼─────────────┘
             ▼               ▼              ▼
┌─────────────────────────────────────────────────────────┐
│           FastAPI バックエンド (:8000)                    │
│  ┌──────────────────────┐   ┌────────────────────────┐  │
│  │   REST API ルーター  │   │  WebSocket ルーター     │  │
│  │  /api/v1/topology/*  │   │  /ws/terminal/{node}   │  │
│  │  /api/v1/nodes/*     │   └──────────┬─────────────┘  │
│  └──────────┬───────────┘              │                 │
│             ▼                          ▼                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │            オーケストレーター                     │   │
│  │  deploy_topology() │ configure_node()             │   │
│  │  get_runtime_info()│ get_topology_status()        │   │
│  └─────┬──────────────┴──────────────────┬──────────┘   │
│        │                                  │              │
│        ▼ Jinja2                           ▼ Docker SDK   │
│  ┌──────────────┐              ┌──────────────────────┐  │
│  │テンプレート  │              │   Docker デーモン     │  │
│  │frr.conf.j2   │              └──────────┬───────────┘  │
│  │topology.clab │                         │              │
│  │  .yml.j2     │              ┌──────────▼───────────┐  │
│  └──────────────┘              │  Containerlab CLI    │  │
│        │                       └──────────┬───────────┘  │
└────────┼───────────────────────────────────┼─────────────┘
         │ ファイル書き込み                   │ 管理
         ▼                                  ▼
┌──────────────────────┐   ┌─────────────────────────────┐
│  data/ ディレクトリ   │   │    Docker コンテナ群        │
│  topology.clab.yml   │   │  ┌─────────┐ ┌──────────┐  │
│  sim-network/        │   │  │alpine-  │ │alpine-   │  │
│    r1/frr.conf       │   │  │frr      │ │switch    │  │
│    r1/daemons        │   │  │（ルーター）│ │（スイッチ）│  │
│    r1/vtysh.conf     │   │  └────┬────┘ └────┬─────┘  │
│  topology_state.json │   │       │veth ペア   │         │
└──────────────────────┘   │  ┌───▼──────┐     │         │
                            │  │alpine-   │     │         │
                            │  │terminal  │◄────┘         │
                            │  │（ホスト PC）│             │
                            │  └──────────┘               │
                            └─────────────────────────────┘
```

## 技術スタック

| レイヤー | 技術 | 目的 |
|---|---|---|
| フロントエンド UI | React 18 + TypeScript | コンポーネントフレームワーク |
| トポロジーキャンバス | React Flow | ノード・エッジのビジュアルグラフ（ドラッグ・接続・パン・ズーム） |
| 状態管理 | Zustand | サブスクリプション対応の軽量クライアント状態ストア |
| Web ターミナル | Xterm.js | 高機能なブラウザ組み込みターミナルエミュレーター |
| ビルドツール | Vite | HMR 対応フロントエンド開発サーバー・バンドラー |
| テスト（フロントエンド） | Vitest + Playwright | ユニット/統合テスト + E2E ブラウザテスト |
| API サーバー | FastAPI（Python） | REST エンドポイントと WebSocket ハンドラー |
| データバリデーション | Pydantic | リクエスト/レスポンスのスキーマ検証とシリアライズ |
| コンテナランタイム | Docker | Python SDK 経由のコンテナライフサイクル管理 |
| ネットワークトポロジー | Containerlab | 宣言的なコンテナベースネットワークトポロジーオーケストレーション |
| ルーティングソフトウェア | FRR（Free Range Routing） | コンテナ内で動作する OSPF・RIP・BGP ルーティングデーモン |
| テンプレーティング | Jinja2 | `topology.clab.yml` および `frr.conf` のテンプレートからの生成 |
| テスト（バックエンド） | pytest | バックエンドロジックのユニット/統合テスト |

## ポートと URL マッピング

| サービス | URL | 備考 |
|---|---|---|
| フロントエンド開発サーバー | http://localhost:5173 | Vite HMR — 変更はページリロードなしに即反映 |
| バックエンド API（REST） | http://localhost:8000/api/v1 | FastAPI — すべての REST エンドポイントはこのプレフィックス以下 |
| WebSocket ターミナル | ws://localhost:8000/ws/terminal/{node_name} | ノードごとに 1 接続 |
| バックエンド API ドキュメント | http://localhost:8000/docs | FastAPI 自動生成 Swagger UI |

## コンテナイメージ

シミュレーター内の各ノードタイプは、専用に構築された Docker イメージで動作します：

| イメージ | ノードタイプ | 主要ソフトウェア |
|---|---|---|
| `alpine-frr:latest` | ルーター | FRR デーモン（zebra, bgpd, ospfd, ripd）、vtysh、bash |
| `alpine-switch:latest` | L2 スイッチ | iproute2（bridge, ip）、bash — カーネルブリッジで VLAN スイッチング |
| `alpine-terminal:latest` | ターミナル / ホスト PC | bash、iproute2、ping、curl、traceroute |

すべてのイメージは Alpine Linux ベースで最小限の構成です。WebSocket ターミナルプロキシが `/bin/bash` を起動するため、すべてのイメージに bash がインストールされています。

## 設定ディレクトリ構造（`data/`）

`data/` ディレクトリはリポジトリルートに位置し、バックエンドと実行中のコンテナの間の共有永続化レイヤーとして機能します。ルーターコンテナは起動時に `data/` のサブディレクトリをマウントし、初期 FRR 設定を読み込みます。

```
data/
├── topology.clab.yml              # レンダリングされた Containerlab トポロジー定義
│                                  # 利用元: containerlab deploy / inspect / destroy
│
├── topology_state.json            # 保存済み React Flow UI 状態（ノード + エッジ）
│                                  # 書き込み: saveState(deployed=false)
│                                  # 読み込み: ページロード時の loadState()
│
├── topology_deployed_state.json   # 最後のデプロイ成功時の UI 状態スナップショット
│                                  # 書き込み: デプロイ後の saveState(deployed=true)
│                                  # hasChanges の差分計算に使用
│
└── sim-network/                   # トポロジーごとの設定ディレクトリ（名前 = topology name フィールド）
    ├── r1/                        # ルーターごとの設定ディレクトリ（名前 = ノードラベル）
    │   ├── daemons                # FRR デーモンの有効/無効リスト — コンテナ起動時に読み込み
    │   ├── frr.conf               # FRR ルーティング設定 — configure_node() が永続化
    │   │                          # （frr.conf.import マウント経由で起動設定としても使用）
    │   └── vtysh.conf             # vtysh 統合設定（service integrated-vtysh-config）
    ├── r2/
    │   └── ...
    └── r3/
        └── ...
```

### ルーターコンテナのマウントポイント

各ルーターコンテナ（`alpine-frr:latest`）には、`topology.clab.yml` に設定された 3 つの読み取り専用バインドマウントがあります：

| ホスト側パス | コンテナ内パス | 目的 |
|---|---|---|
| `data/{topo}/{node}/daemons` | `/etc/frr/daemons` | FRR が起動するデーモンを指定 |
| `data/{topo}/{node}/frr.conf` | `/etc/frr/frr.conf.import` | 起動時に読み込まれる初期ルーティング設定 |
| `data/{topo}/{node}/vtysh.conf` | `/etc/frr/vtysh.conf` | `vtysh` 統合設定モードを有効化 |

> **注意**: ホスト側の `frr.conf` は起動時に `/etc/frr/frr.conf.import`（`/etc/frr/frr.conf` ではない）としてマウントされます。コンテナ起動後、`configure_node()` は `docker exec` 経由でコンテナ内の `/etc/frr/frr.conf` に直接新しい設定を書き込み、さらにコンテナ再起動時の永続性のためホスト側の `frr.conf` も更新します。

## FRR デーモン設定

すべてのルーターノードは、`deploy_topology()` 実行時に `daemons` ファイルへ書き込まれる以下の FRR デーモン状態で事前設定されます：

| デーモン | 状態 | プロトコル |
|---|---|---|
| `zebra` | **有効** | カーネルルーティングテーブルマネージャー（他のすべてのデーモンが必要とする） |
| `bgpd` | **有効** | BGP（Border Gateway Protocol） |
| `ospfd` | **有効** | OSPFv2（IPv4 向け Open Shortest Path First） |
| `ripd` | **有効** | RIPv2（Routing Information Protocol） |
| `ospf6d` | 無効 | OSPFv3（IPv6） |
| `ripngd` | 無効 | RIPng（IPv6） |
| `isisd` | 無効 | IS-IS |
| `pimd` | 無効 | PIM（マルチキャスト） |
| `ldpd` | 無効 | LDP（MPLS ラベル配布） |
| `nhrpd` | 無効 | NHRP（Next Hop Resolution） |
| `eigrpd` | 無効 | EIGRP |
| `babeld` | 無効 | Babel ルーティングプロトコル |
| `sharpd` | 無効 | SHARP（テスト用デーモン） |
| `pbrd` | 無効 | PBR（ポリシーベースルーティング） |
| `bfdd` | 無効 | BFD（Bidirectional Forwarding Detection） |
| `fabricd` | 無効 | OpenFabric |
| `vrrpd` | 無効 | VRRP（Virtual Router Redundancy Protocol） |
| `pathd` | 無効 | SRTE パス管理 |

`data/` 以下のすべてのファイルとディレクトリは `chmod 777` で作成されます。これは、Alpine コンテナ内で非 root ユーザーとして動作するプロセスが読み取れるようにするためです。

---

*ナビゲーション: [アーキテクチャインデックス](./index.ja.md) · [データフロー](./data-flows.ja.md) · [状態管理](./state-management.ja.md)*
