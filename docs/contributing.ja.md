# コントリビューション

> 🇬🇧 English version available → [contributing.md](./contributing.md)

Network Simulator へのコントリビューションにご関心をお持ちいただきありがとうございます！このガイドでは、リポジトリ構成、開発ワークフロー、テスト要件、コントリビューション基準について説明します。

---

## リポジトリ構成

このプロジェクトは **2 つの Git サブモジュールを持つメインリポジトリ**として管理されています:

```
network-simulator/                    ← メインリポジトリ
├── frontend/                         ← サブモジュール (git@github.com:junkreef/Network-Simulator-frontend.git)
├── backend/                          ← サブモジュール (git@github.com:junkreef/Network-Simulator-backend.git)
├── examples/                         ← トポロジー設定のサンプル
├── docs/                             ← プロジェクト全体のドキュメント
├── README.md
├── README.ja.md
└── .gitmodules
```

各サブモジュールは独立した Git リポジトリです:

- **フロントエンドサブモジュール** (`frontend/`) — React 18、TypeScript、React Flow、Xterm.js、Zustand、Vite、Vitest、Playwright。`git@github.com:junkreef/Network-Simulator-frontend.git` でホスト。
- **バックエンドサブモジュール** (`backend/`) — FastAPI、Pydantic、Docker SDK、Containerlab CLI、FRR、Jinja2。`git@github.com:junkreef/Network-Simulator-backend.git` でホスト。

> `frontend/` または `backend/` への変更は、まずサブモジュール内でコミットし、その後メインリポジトリのサブモジュール参照を更新する必要があります。

---

## 開発セットアップ

サブモジュールを含めてクローン:

```bash
git clone --recurse-submodules <repo-url>
cd network-simulator
```

**フロントエンド:**

```bash
cd frontend
npm install
npm run dev
```

**バックエンド:**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
.venv/bin/uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
```

詳細なセットアップ手順とトラブルシューティングは [はじめに](./getting-started.ja.md) を参照してください。

---

## テストファースト要件

**コードを変更する前に、必ずテスト検証手順を定義してください。**

必須のワークフロー:

```
テスト手順の定義 → 変更を実施 → テストを実行 → 失敗した場合は修正 → 繰り返す
```

具体的な手順:

1. 既存のカバレッジを把握するため、関連するテストファイルを読む。
2. 変更内容を検証するテストを作成または特定する。
3. 変更前にテストが失敗（レッド）することを確認する。
4. 変更を実装する。
5. テストが成功（グリーン）することを確認する。

「些細な変更だから」といってこのプロセスをスキップしないでください。このルールがリグレッションを防ぎ、テストスイートの信頼性を保ちます。

---

## テストの実行

### フロントエンド — ユニット / インテグレーション（Vitest）

```bash
cd frontend
npm test
```

### フロントエンド — E2E テスト（Playwright）

```bash
cd frontend
npm run test:e2e
```

> Playwright の E2E テストにはフロントエンドとバックエンドの両方が起動している必要があります。E2E テスト実行前に両サーバーを起動してください。

### バックエンド — ユニット / インテグレーション（pytest）

```bash
cd backend
.venv/bin/pytest
```

詳細出力:

```bash
.venv/bin/pytest -v
```

特定のテストファイルを指定:

```bash
.venv/bin/pytest tests/test_topology.py -v
```

---

## コードスタイル

### フロントエンド（TypeScript）

- **TypeScript strict モード**が有効です。暗黙的な `any` は禁止、すべての型を明示的に記述してください。
- **ESLint** が設定されています。コミット前に実行してください:
  ```bash
  cd frontend
  npx eslint src/
  ```

### バックエンド（Python）

- 標準的な Python の慣習（PEP 8）に従ってください。
- **pylint** でリントできます:
  ```bash
  cd backend
  .venv/bin/pylint src/
  ```

---

## コミットガイドライン

- **明確で簡潔なコミットメッセージ**を書き、「何を」「なぜ」変更したかを記述してください。
- 件名行は命令形を使用してください: `Add OSPF area type support`（`Added OSPF area types` ではなく）。
- 件名行は 72 文字以内に収め、追加のコンテキストが必要な場合は本文を追加してください。
- コミットメッセージに `Co-authored-by:` トレーラーや自動生成のフッターを**追加しないでください**。

### サブモジュールへのコミット

`frontend/` または `backend/` への変更がある場合:

1. サブモジュール内で変更をコミット:
   ```bash
   cd frontend   # または backend
   git add .
   git commit -m "コミットメッセージ"
   git push
   ```

2. メインリポジトリのサブモジュール参照を更新:
   ```bash
   cd ..   # network-simulator/ に戻る
   git add frontend   # または backend
   git commit -m "Update frontend submodule to include <説明>"
   git push
   ```

---

## プルリクエストチェックリスト

プルリクエストを開く前に、以下をすべて確認してください:

- [ ] 関連するすべてのテストが通過している（`npm test`、`npm run test:e2e`、`.venv/bin/pytest`）
- [ ] 新しい動作がテストでカバーされている
- [ ] ユーザー向けの動作や開発者セットアップに影響する変更は、ドキュメントも更新されている
- [ ] シークレット、認証情報、API キーがコミットされていない
- [ ] TypeScript の型が明示的（正当な理由のない `any` 抑制はしない）
- [ ] コミットメッセージが明確で、自動生成のトレーラーが含まれていない
- [ ] サブモジュールに変更がある場合、メインリポジトリのサブモジュール参照が更新されている

---

## ナビゲーション

- ← [README に戻る](../README.ja.md)
- ← [はじめに](./getting-started.ja.md)
- ← [ユーザーガイド](./user-guide.ja.md)
