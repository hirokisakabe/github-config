# AGENTS.md

このリポジトリで AI agent (Claude Code / Codex CLI / Cursor / Gemini 等) が作業する際の運用ガイド。

## このリポジトリの概要

`gh infra` を使って GitHub リポジトリの設定を YAML で宣言的に管理するリポジトリ。`repos/` 配下の YAML が source of truth、GitHub 本体が state、差分は `gh infra plan` / `gh infra apply` で扱う。

## `gh infra plan/apply` を local で実行する際の方針

`gh infra plan/apply` は管理対象リポジトリごとに settings / Actions permissions / secrets / variables / release immutability / branch protection など複数 endpoint を取得するため、1 回の full plan/apply でも GitHub REST API の primary rate limit (authenticated user で 5,000 req/hour) を相応に消費する。**local の `gh` と CI の `GH_INFRA_TOKEN` は同じ GitHub ユーザーの quota を共有する** (詳細は後述「local と CI の quota 関係」)。

以下の方針を守ること:

1. **full `plan/apply` 前後で rate limit を確認する**

    ```bash
    gh api rate_limit --jq .resources.core
    # {"limit":5000,"remaining":4976,"reset":1779898027,"used":24}
    # reset を人が読める形に: macOS なら `date -r <reset>`、Linux なら `date -d @<reset>`
    ```

    `remaining` が 1,000 を切っている状態で full `plan/apply` を始めると途中で 403 が連発する。`reset` 時刻を過ぎるまで待つ。

2. **full `plan/apply` を何度も連続実行しない**

    試行錯誤中は `gh infra plan repos/<file>.yaml` のように対象を絞る。同じ full plan を 5 分間に複数回回すと quota を急速に消費する。

3. **large RepositorySet 変更では、実行コマンドと rate limit 残量を作業ログ / PR description に残す**

    `gh infra apply` の直前直後で `gh api rate_limit` を取り、PR description やコミットメッセージに「apply 前: remaining=4500 / apply 後: remaining=3800」のような粒度で残しておく。CI 側の plan が rate limit を引いた原因を後から追跡できるようにするため。

4. **rate limit に当たったらリトライ連打せず、reset を待つ**

    CI workflow は rate limit を検知すると reset 時刻を出して fail fast する設計 (`plan.yml` / `apply.yml` の "Check rate limit" ステップ参照)。local でも同様に振る舞うこと。

## local と CI の quota 関係

GitHub REST API の primary rate limit は **token を発行した user 単位** で集計される ([GitHub Docs: REST API rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28))。本リポジトリでは:

- **local の `gh` CLI**: 各自の GitHub ユーザートークン (`gh auth login` で発行) で動く。
- **CI (`plan.yml` / `apply.yml`)**: secrets `GH_INFRA_TOKEN` (= リポジトリオーナーが発行した fine-grained PAT) で動く。

local の `gh` と CI の `GH_INFRA_TOKEN` が **同一の GitHub ユーザーで発行されている場合、両者は同じ 5,000 req/hour quota を食い合う**。たとえば:

- local で `gh infra plan` を full で 1 回 → ~数千 req
- 続けて CI の Plan workflow が PR で起動 → 同じ user quota から引かれる
- 個別に `gh api repos/...` で確認 → さらに引かれる

→ 1 時間以内にすべてが積み上がり `remaining=0` になる。

CI と local の quota を分離したい場合は、CI 用 PAT を別の machine user (bot アカウント) で発行する、GitHub App installation token に切り替える、などの選択肢がある (本リポジトリでは未対応。必要になった時点で検討)。

## CI workflow の運用

| Workflow | Trigger | 用途 |
|----------|---------|------|
| `plan.yml` | PR (paths: `repos/**` または同 workflow) / `workflow_dispatch` | YAML validate → rate limit check → `gh infra plan` → PR コメント |
| `apply.yml` | `push` to `main` / `workflow_dispatch` | rate limit check → `gh infra apply --auto-approve` |

- **PR plan の発火**: `repos/**` か `plan.yml` 自体に変更がある PR のみ。それ以外 (README 更新だけ等) では plan は走らない。
- **手動 plan / apply**: Actions タブから `workflow_dispatch` で起動できる。`target` input を空にすると `./repos/` 全体、ファイル/ディレクトリ指定で対象限定 plan/apply。
- **rate limit ログ**: 各 workflow の "Check rate limit (before/after)" ステップで `remaining` / `used` / `reset` を notice として出力する。
- **rate limit 到達時の挙動**: `gh infra plan/apply` の出力に `API rate limit exceeded` などが含まれると、reset 時刻を error annotation に出して非 0 終了する。CI 自動リトライはせず、手動で reset 後に再実行すること。

## 不明点

実装方針に揺らぎや迷いが出た場合は、勝手に拡張せず issue / PR でユーザーに確認すること。
