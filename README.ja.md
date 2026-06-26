# Network Simulator

> 🇬🇧 English version available → [README.md](./README.md)

**ブラウザ上でネットワークトポロジーを設計・設定し、実際のコンテナ環境にデプロイできる、Web ベースのネットワークシミュレーション環境です。**

![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-required-2496ED?logo=docker&logoColor=white)
![Containerlab](https://img.shields.io/badge/Containerlab-latest-00BFA5)
![FRR](https://img.shields.io/badge/FRR-Free_Range_Routing-4CAF50)

> 📸 スクリーンショット / デモは近日公開予定

---

## 前提条件

- [Docker](https://docs.docker.com/get-docker/) — コンテナランタイム
- [Containerlab](https://containerlab.dev/install/) — ネットワークラボオーケストレーター
- [Node.js](https://nodejs.org/) ≥ 18 — フロントエンドビルドツールチェーン
- [Python](https://www.python.org/downloads/) ≥ 3.10 — バックエンドランタイム
- [Git](https://git-scm.com/) — サブモジュール対応

---

## クイックスタート

1. **リポジトリをクローン**（サブモジュールを含む）:

   ```bash
   git clone --recurse-submodules <repo-url> && cd network-simulator
   ```

2. **ターミナル 1 — フロントエンドを起動:**

   ```bash
   cd frontend && npm install && npm run dev
   ```

3. **ターミナル 2 — バックエンドを起動:**

   ```bash
   cd backend && python -m venv .venv && pip install -r requirements.txt && .venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
   ```

4. **ブラウザで開く** → [http://localhost:5173](http://localhost:5173)

---

## ドキュメント

| ドキュメント | 内容 |
|---|---|
| [はじめに](./docs/getting-started.ja.md) | セットアップガイドとトラブルシューティング |
| [アーキテクチャ](./docs/architecture/index.md) | システム設計とデータフロー図 |
| [ユーザーガイド](./docs/user-guide.ja.md) | トポロジーの構築・実行方法 |
| [コントリビューション](./docs/contributing.ja.md) | 開発ワークフローとコントリビューションガイド |

---

## リポジトリ構成

このリポジトリは Git サブモジュールを使用して、フロントエンドとバックエンドを分離管理しています:

```
network-simulator/          ← メインリポジトリ（このリポジトリ）
├── frontend/               ← サブモジュール: React + TypeScript + React Flow + Xterm.js
├── backend/                ← サブモジュール: FastAPI + Containerlab + FRR
├── examples/               ← トポロジー設定のサンプル
├── docs/                   ← プロジェクト全体のドキュメント
├── README.md
├── README.ja.md
└── .gitmodules
```

- **フロントエンド** ([`frontend/`](./frontend/)): React 18、TypeScript、React Flow（トポロジーキャンバス）、Xterm.js（Web ターミナル）、Zustand（状態管理）、Vite、Vitest、Playwright。
- **バックエンド** ([`backend/`](./backend/)): FastAPI、Pydantic、Docker SDK、Containerlab CLI、FRR（Free Range Routing）、Jinja2 テンプレート。

---

## ライセンス

ライセンス情報は近日公開予定。
