# shared-workflows

複数リポジトリで共有する GitHub Actions の Reusable Workflows。

## ワークフロー一覧

| ファイル | 種別 | 用途 |
|---------|------|------|
| `.github/workflows/gemini-review.yml` | reusable (`workflow_call`) | Gemini による PR コードレビュー。指摘を CRITICAL / MAJOR / MINOR に分類して PR コメント（見出し `# Code Review by Gemini`）を 1 件投稿。`GEMINI_API_KEY` secret が必要 |
| `.github/workflows/dependabot-automerge.yml` | reusable (`workflow_call`) | Dependabot の patch / minor 更新を auto-merge（major は対象外） |
| `.github/workflows/pr-review.yml` | caller（このリポジトリ用） | このリポジトリ自身の PR を、PR 側の `./.github/workflows/gemini-review.yml`（`@main` ではない）でレビューする。変更をその PR で試せるようにするため |
| `.github/dependabot.yml` | Dependabot 設定 | このリポジトリの GitHub Actions のバージョンを毎週確認（PR は最大 5 件） |

## 利用方法（caller 側リポジトリ）

### Gemini レビュー

`.github/workflows/gemini-review.yml` を追加し、reusable workflow を呼び出す:

```yaml
name: Gemini Code Review
on:
  pull_request:
    types: [opened, synchronize]
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    uses: yushin0319/shared-workflows/.github/workflows/gemini-review.yml@main
    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
```

### Dependabot auto-merge

`.github/workflows/dependabot-automerge.yml` を追加する。secrets は不要（`github.token` を使う）が、
`contents: write` + `pull-requests: write` と `pull_request_target` トリガーが必要:

```yaml
name: Dependabot Auto-Merge
on: pull_request_target
permissions:
  contents: write
  pull-requests: write
jobs:
  automerge:
    uses: yushin0319/shared-workflows/.github/workflows/dependabot-automerge.yml@main
```

## 実装メモ

### gemini-review.yml

- ランナー: ubuntu-latest / `actions/checkout@v7`（`fetch-depth: 0`）/ `actions/setup-python@v7`（Python 3.12）
- SDK: `google-genai>=1.0,<2.0`、モデルは **`gemini-2.5-flash`**
- レビュー対象は `git diff -U10 BASE_SHA HEAD_SHA`。`package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` / `*.lock` は除外する
- diff が 100,000 文字を超えたら先頭だけ渡し、コメント末尾に「Diff was truncated」の注記を付ける
- 「エラーの握りつぶし」を検出した場合は MAJOR として報告する
- Gemini API が 429 / 500 / 502 / 503 / 504 を返したら 20 / 40 / 80 秒待って最大 3 回送り直す。それでも失敗したら見出しを `# Gemini Review Failed` にしてコメントし、ジョブも失敗（exit 1）にする。見出しを変えるのは、マージガード（`Code Review by Gemini` のコメントがあるかでマージ可否を見る hook）にレビュー済みと読まれないようにするため

### dependabot-automerge.yml

- `github.actor == 'dependabot[bot]'` のときのみ実行
- `dependabot/fetch-metadata@v3` で更新種別を判定し、`version-update:semver-major` 以外を `gh pr merge --auto --merge`
