# github-config

[gh-infra](https://github.com/babarot/gh-infra) を使って GitHub リポジトリの設定を YAML で宣言的に管理するリポジトリ。<br>
`repos/` の YAML が source of truth。変更は PR → plan → apply のフローで安全に反映される。

---

## 目次

- [仕組み](#仕組み)
- [ディレクトリ構成](#ディレクトリ構成)
- [クイックスタート](#クイックスタート)
- [GitHub Actions](#github-actions)
- [シークレット設定](#シークレット設定)

---

## 仕組み

```
YAML 編集
    │
    ▼
PR を作成  ──→  plan.yml 起動  ──→  gh infra plan  ──→  PR にコメント
    │                                                          │
    │                                              差分を確認してマージ
    ▼
main に push  ──→  apply.yml 起動  ──→  gh infra apply  ──→  GitHub に反映
```

---

## ディレクトリ構成

```
.
├── repos/                   # リポジトリごとの設定 YAML
└── .github/
    └── workflows/
        ├── plan.yml         # PR 時に差分確認
        └── apply.yml        # main push 時に自動適用
```

---

## クイックスタート

### リポジトリを追加する

```bash
# 既存リポジトリの設定を YAML に書き出す
gh infra import hirokisakabe/<repo-name> > repos/<repo-name>.yaml

# 差分を確認
gh infra plan repos/<repo-name>.yaml

# PR を作成 → CI の plan 結果を確認 → マージで自動 apply
```

### ローカルで全体を確認・適用する

```bash
gh infra plan ./repos/    # 差分確認
gh infra apply ./repos/   # 適用
```

> [!WARNING]
> `gh infra plan/apply` は GitHub REST API を大量に消費します。
> ローカルと CI は同じ quota (5,000 req/hour) を共有するため、実行前に残量を確認してください。

<details>
<summary>rate limit の確認方法</summary>

```bash
gh api rate_limit --jq .resources.core
# {"limit":5000,"remaining":4976,"reset":1779898027,"used":24}

# reset を人が読める形に変換 (macOS)
date -r <reset>
```

`remaining` が 1,000 を切っている場合は reset まで待ってから実行してください。
詳細な運用ルールは [AGENTS.md](./AGENTS.md) を参照。

</details>

---

## GitHub Actions

| Workflow | トリガー | 動作 |
|----------|---------|------|
| [`plan.yml`](.github/workflows/plan.yml) | PR (`repos/**` / 同 workflow 変更) / `workflow_dispatch` | YAML validate → rate limit check → `gh infra plan` → PR コメント |
| [`apply.yml`](.github/workflows/apply.yml) | `push` to `main` / `workflow_dispatch` | rate limit check → `gh infra apply --auto-approve` |

<details>
<summary>運用の詳細</summary>

- **PR plan の発火条件** — `repos/**` または `plan.yml` 自体が変更された PR のみ。README などの変更では plan は走らない。
- **手動実行** — Actions タブの `workflow_dispatch` から起動。`target` input を空にすると `./repos/` 全体、ファイル/ディレクトリ指定で対象を絞れる。
- **rate limit ログ** — 各 workflow の "Check rate limit" ステップで `remaining` / `used` / `reset` を notice として出力。
- **rate limit 到達時** — reset 時刻を error annotation に出して fail fast。自動リトライはしないので、reset 後に手動で再実行する。

</details>

---

## シークレット設定

| シークレット名 | 説明 |
|----------------|------|
| `GH_INFRA_TOKEN` | Fine-grained PAT（以下の権限を付与） |

<details>
<summary>Fine-grained PAT の作成手順</summary>

**Repository access**: All repositories

| 権限 | レベル |
|------|--------|
| Administration | Read and write |
| Contents | Read-only |
| Issues | Read-only |
| Secrets | Read-only |
| Variables | Read-only |

> [!CAUTION]
> classic PAT は権限が広すぎるため使用しないこと。

</details>
