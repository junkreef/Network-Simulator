# はじめに

> 🇬🇧 English version available → [getting-started.md](./getting-started.md)

このガイドでは、前提条件のインストールからリポジトリのクローン、フロントエンド・バックエンドのローカル起動までを順を追って説明します。

---

## 前提条件

以下のツールをすべてインストールしてから作業を進めてください:

| ツール | バージョン | インストール |
|---|---|---|
| Docker | 最新版 | [Docker Desktop](https://docs.docker.com/get-docker/) |
| Containerlab | 最新版 | [Containerlab ドキュメント](https://containerlab.dev/install/) |
| Node.js | ≥ 18 | [nodejs.org](https://nodejs.org/en/download) |
| Python | ≥ 3.10 | [python.org](https://www.python.org/downloads/) |
| Git | 任意 | [git-scm.com](https://git-scm.com/) |

インストール確認:

```bash
docker --version
containerlab version
node --version
python3 --version
git --version
```

---

## リポジトリのクローン

このプロジェクトは **Git サブモジュール** を使用しており、フロントエンドとバックエンドはそれぞれ独立したリポジトリとして管理されています。まとめてクローンするには `--recurse-submodules` オプションが必要です:

```bash
git clone --recurse-submodules <repo-url>
cd network-simulator
```

> **Git サブモジュールとは？** サブモジュールは、別のリポジトリの特定コミットへの参照です。`frontend/` と `backend/` ディレクトリは独立したリポジトリとして、このメインリポジトリに固定バージョンで含まれています。

`--recurse-submodules` なしでクローンした場合は、以下のコマンドで初期化できます:

```bash
git submodule update --init --recursive
```

---

## フロントエンドのセットアップ

```bash
cd frontend
npm install
npm run dev
```

正常に起動した場合の出力例:

```
  VITE v5.x.x  ready in XXX ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
```

React 開発サーバーが **http://localhost:5173** で起動します。

---

## バックエンドのセットアップ

```bash
cd backend
python -m venv .venv
```

仮想環境をアクティベート:

- **Linux / macOS:**
  ```bash
  source .venv/bin/activate
  ```
- **Windows:**
  ```bat
  .venv\Scripts\activate
  ```

依存関係をインストールしてサーバーを起動:

```bash
pip install -r requirements.txt
.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
```

正常に起動した場合の出力例:

```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process
```

FastAPI サーバーが **http://localhost:8000** で起動します。

---

## 動作確認

両方のサーバーが起動したら、以下の手順で動作を確認します:

1. **UI を開く** → [http://localhost:5173](http://localhost:5173)
   左側にノードパレットが表示されたトポロジーキャンバスが確認できます。

2. **バックエンド API を確認:**
   ```bash
   curl http://localhost:8000/api/v1/topology/status
   ```
   期待されるレスポンス:
   ```json
   {"status": "stopped"}
   ```

---

## トラブルシューティング

### Docker が起動していない

**症状:** バックエンドは起動するが、デプロイ時に Docker 接続エラーが発生する。

**対処法:** Docker Desktop（または Docker デーモン）が起動していることを確認:
```bash
docker ps
```
エラーが出る場合は Docker を起動してから再試行してください。

---

### ポート競合（5173 または 8000 が使用中）

**症状:** 起動時に「アドレスが既に使用されています」というエラーが発生する。

**対処法:** 競合しているプロセスを特定して停止:
```bash
# ポート 5173 を使用しているプロセスを確認
lsof -i :5173

# ポート 8000 を使用しているプロセスを確認
lsof -i :8000
```

または、使用するポートを変更することもできます:
- フロントエンド: `npm run dev -- --port 3000`
- バックエンド: uvicorn コマンドの `--port 8000` を変更

---

### サブモジュールが初期化されていない

**症状:** `frontend/` または `backend/` ディレクトリが空になっている。

**対処法:**
```bash
git submodule update --init --recursive
```

---

### `pip install` が失敗する

**症状:** 依存関係の解決エラーやシステムライブラリの不足。

**対処法:** Python ≥ 3.10 を使用し、`pip install` を実行する前に仮想環境がアクティベートされていることを確認してください。

---

## 次のステップ

- ← [README に戻る](../README.ja.md)
- → [ユーザーガイド](./user-guide.ja.md) — トポロジーの構築・デプロイ方法を学ぶ
