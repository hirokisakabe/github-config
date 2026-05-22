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

| イベント | 動作 |
|----------|------|
| PR 作成・更新 | `gh infra plan` を実行し結果をコメント |
| main への push | `gh infra apply` を自動実行 |

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
