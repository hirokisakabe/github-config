# github-config

[gh-infra](https://github.com/babarot/gh-infra) を使って GitHub リポジトリの設定を YAML で宣言的に管理するリポジトリ。<br>
`repos/` の YAML が source of truth。変更は PR → plan → apply のフローで安全に反映される。

## ローカルで確認・適用する

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
