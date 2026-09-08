# shared-workflows

複数リポジトリで共有する GitHub Actions の Reusable Workflows。

## ワークフロー一覧

| ファイル | 種別 | 用途 |
|---------|------|------|
| `.github/workflows/gemini-review.yml` | reusable (`workflow_call`) | Gemini による PR コードレビュー。指摘を CRITICAL / MAJOR / MINOR に分類して PR コメント投稿。`GEMINI_API_KEY` secret が必要 |
| `.github/workflows/dependabot-automerge.yml` | reusable (`workflow_call`) | Dependabot の patch / minor 更新を auto-merge（major は対象外） |
| `.github/workflows/pr-review.yml` | caller 例 | `pull_request` で `gemini-review.yml` を呼び出すサンプル |

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
- 「エラーの握りつぶし」を検出した場合は MAJOR として報告する

### dependabot-automerge.yml

- `github.actor == 'dependabot[bot]'` のときのみ実行
- `dependabot/fetch-metadata@v3` で更新種別を判定し、`version-update:semver-major` 以外を `gh pr merge --auto --merge`
