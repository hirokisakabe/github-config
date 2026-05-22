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
| `GH_INFRA_TOKEN` | リポジトリ管理権限を持つ PAT (classic) |

PAT に必要なスコープ: `repo`, `admin:org` (必要に応じて)
