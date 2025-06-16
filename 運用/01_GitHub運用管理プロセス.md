

---

# 🚀 GitHub ブランチ・レビュー・マージ管理プロセス

## ✅ 目的

* ブランチ戦略を明確にし、チーム開発の効率と品質を高める
* レビュー・マージフローを統一し、ミス・コンフリクトを未然に防ぐ
* CI/CDと連携した継続的インテグレーションを可能にする

---

## 📁 ブランチ戦略と構成（親ブランチ含む）

| ブランチ種別      | 用途            | 命名規則             | 親ブランチ     | マージ先              |
| ----------- | ------------- | ---------------- | --------- | ----------------- |
| `main`      | 本番環境コード       | `main`           | なし        | なし（最終地点）          |
| `develop`   | 開発の統合         | `develop`        | `main`    | `release/*`       |
| `feature/*` | 新機能開発         | `feature/機能名`    | `develop` | `develop`         |
| `bugfix/*`  | 軽微な不具合修正      | `bugfix/内容`      | `develop` | `develop`         |
| `release/*` | リリース準備（QA、調整） | `release/v1.2.0` | `develop` | `main`, `develop` |
| `hotfix/*`  | 本番障害などの緊急修正   | `hotfix/障害名`     | `main`    | `main`, `develop` |

---

## 🔄 ブランチ間の従属関係（構成図）

```
        +---------------------+
        |        main         |
        +---------------------+
               ^      ^
               |      |
      +--------+      +--------+
      |                        |
+------------+         +---------------+
| release/*  | ←←←←←←  |  hotfix/*     |
+------------+         +---------------+
       ^                        |
       |                        |
+--------------+        +------------------+
|   develop    | ←←←←←←←←←←←←←←←←←←←←←←←←←+
+--------------+
     ^      ^
     |      |
+---------+ +-----------+
|feature/*| | bugfix/*  |
+---------+ +-----------+
```

---

## 🛠️ 開発・マージフロー（例：featureブランチ）

1. **作業開始**：`develop` ブランチから `feature/〇〇` を作成
2. **ローカル開発**：コード実装・単体テスト
3. **Push**：GitHubにプッシュ
4. **Pull Request 作成**：`develop` へのPRを提出
5. **レビューと承認**：最低1名以上のレビューを通す
6. **CI実行**：Lint, テストなど自動チェック
7. **マージ**：承認されたら `develop` にマージ
8. **ブランチ削除**：不要なブランチは削除（ローカル・リモート）

---

## 🚨 リリース・障害対応フロー

### 📦 リリースフロー（release/\*）

* `develop` → `release/vX.Y.Z` ブランチ作成
* QA実施、バグ修正（必要に応じて `bugfix/*` を取り込む）
* `main` にマージ → 本番反映
* `develop` にもマージして差分を保持

### 🔥 ホットフィックス（hotfix/\*）

* `main` から `hotfix/*` を作成
* 緊急修正を行い、本番テスト後に `main` にマージ
* `develop` にもマージして次回開発に反映

---

## ✅ マージルール（レビュー・保護ルール）

| 対象ブランチ      | レビュー数 | CIチェック | 直接Push | 備考               |
| ----------- | ----- | ------ | ------ | ---------------- |
| `main`      | 2人以上  | 必須     | 禁止     | 本番に直結、最も厳格な制限    |
| `develop`   | 1人以上  | 必須     | 禁止     | 開発統合の中核          |
| `release/*` | 2人以上  | 必須     | 禁止     | QA済み後に本番反映前確認が必要 |
| `feature/*` | 任意    | 任意     | 可      | 単独作業中は可、PR時にCI実施 |
| `hotfix/*`  | 1人以上  | 必須     | 原則禁止   | 緊急対応でもレビューを残す    |

---

## 🧷 補足：権限・保護ルールの設定（GitHub）

### 🔐 必須設定（リポジトリ > Settings > Branches）

| 設定項目                          | 推奨値              |
| ----------------------------- | ---------------- |
| Require pull request reviews  | 有効（`main`: 2人以上） |
| Require status checks to pass | 有効（CIチェック合格必須）   |
| Include administrators        | 有効（管理者にも適用）      |
| Restrict who can push         | 有効（特定チームのみ）      |
| Require signed commits        | 任意（署名付きコミット推奨）   |

### 👥 権限ロールの使い分け（GitHubチーム）

| ロール        | 操作可能範囲          |
| ---------- | --------------- |
| Owner      | 全権限             |
| Maintainer | ブランチ作成、設定変更、マージ |
| Developer  | PR作成、レビュー、マージ   |
| Reviewer   | レビュー専任者         |

---

## ⚙️ CI/CDとの連携（GitHub Actions例）

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm install
      - run: npm run lint
      - run: npm test
```

---

## 📝 運用ポイント

* 命名規則を統一し、履歴を追いやすくする（例：`feature/add-login`）
* 本番に関わる変更は必ずレビューとCIを通す
* 不要なブランチは削除して整理
* 障害対応でも必ず記録としてPRとレビュー履歴を残す

---

### ✅ **`hotfix/*` の修正を `release/*` にも反映したい場合**

以下のように **`main` → `release/*` へのマージ or cherry-pick が必要**です。

---

## 🔁 運用フロー（複数マージ先）

```text
main  ───┐
         ▼
     hotfix/123-bugfix
         ├───▶ main（本番修正）
         ├───▶ develop（差分を反映）
         └───▶ release/vX.Y.Z（← リリース準備中のバージョンにも反映）
```

---

## 🎯 マージ先を選ぶポイント

| マージ先        | 意図                | 方法                            |
| ----------- | ----------------- | ----------------------------- |
| `main`      | 本番環境に即時修正を反映      | 通常の Pull Request              |
| `develop`   | 修正を次回開発ブランチへ取り込む  | Pull Request or `cherry-pick` |
| `release/*` | リリース準備中のバージョンにも修正 | `cherry-pick` or Pull Request |

---

## 💡 注意点：`release/*` への反映が必要なケース

* リリースが目前なのに `develop` 側に修正があると未反映になる。
* `hotfix` で本番修正した内容を **`release/vX.Y.Z` に含めずにリリースすると再発のリスクがある。**

---

## 🛠 例：実際のGit操作イメージ

```bash
# hotfix/bug-fix-001 を main にマージ済みとする

# develop へ反映
git checkout develop
git cherry-pick <hotfix-commit>

# release/v1.2.0 へも反映
git checkout release/v1.2.0
git cherry-pick <hotfix-commit>
```

または、それぞれのブランチで `hotfix/` をマージする Pull Request を立てます。

---

## ✅ 結論：hotfix の反映先は3箇所になる場合がある

| 反映先         | 必須/任意 | 理由             |
| ----------- | ----- | -------------- |
| `main`      | ✅ 必須  | 本番反映           |
| `develop`   | ✅ 必須  | 差分を継続開発に含めるため  |
| `release/*` | ⭕ 任意  | リリース準備中なら反映すべき |

---

