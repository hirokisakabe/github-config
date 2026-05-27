# github-config

[gh-infra](https://github.com/babarot/gh-infra) を使って GitHub リポジトリの設定をコードで管理するリポジトリ。

## 構成

```
.
├── repos/          # リポジトリごとの設定 YAML
└── .github/
    └── workflows/
        ├── plan.yml   # PR 時に差分確認
        └── apply.yml  # main push 時に自動適用
```

## 使い方

### ローカルで確認

```bash
# 差分確認
gh infra plan ./repos/

# 適用
gh infra apply ./repos/
```

### 新しいリポジトリを追加

```bash
gh infra import hirokisakabe/<repo-name> > repos/<repo-name>.yaml
```

## GitHub Actions

| Workflow | Trigger | 動作 |
|----------|---------|------|
| `plan.yml` | PR (`repos/**` または同 workflow に変更) / `workflow_dispatch` | YAML validate → rate limit check → `gh infra plan` → PR コメント |
| `apply.yml` | `push` to `main` / `workflow_dispatch` | rate limit check → `gh infra apply --auto-approve` |

### plan / apply の運用

- **PR plan の発火条件**: `repos/**` または `.github/workflows/plan.yml` が変更された PR のみで plan が動く。README やそれ以外のファイルだけの PR では plan は走らない (rate limit の無駄打ちを避けるため)。`gh infra validate` (YAML 構文チェック、API 呼び出し無し) は plan ジョブの先頭で必ず実行する。
- **手動 plan / apply**: Actions タブから `workflow_dispatch` で起動できる。`target` input を空にすると `./repos/` 全体、ファイル/ディレクトリ指定で対象限定 plan / apply。
- **rate limit ログ**: 各 workflow の "Check rate limit (before/after)" で `remaining` / `used` / `reset` を `::notice::` として出す。
- **rate limit 到達時**: `gh infra plan/apply` の出力に `API rate limit exceeded` 等が含まれると、reset 時刻を `::error::` に出して fail fast する。自動リトライしないので、reset 後に手動で再実行する。
- **local と CI の quota 共有**: local の `gh` CLI と CI の `GH_INFRA_TOKEN` が同一 GitHub ユーザーで発行されている場合、両者は同じ 5,000 req/hour quota を食い合う。詳細・local 実行時の作業ルールは [`AGENTS.md`](./AGENTS.md) を参照。

### 必要なシークレット

| シークレット名 | 説明 |
|----------------|------|
| `GH_INFRA_TOKEN` | Fine-grained PAT（下記権限を付与） |

#### Fine-grained PAT の設定

- **Repository access**: All repositories
- **Repository permissions**:

| 権限 | レベル |
|------|--------|
| Administration | Read and write |
| Contents | Read-only |
| Issues | Read-only |
| Secrets | Read-only |
| Variables | Read-only |

> classic PAT は権限が広すぎるため使用しないこと。
