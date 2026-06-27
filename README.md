# shared-workflows

複数リポジトリで共有する GitHub Actions の Reusable Workflows。

## ワークフロー一覧

| ファイル | 種別 | 用途 |
|---------|------|------|
| `.github/workflows/gemini-review.yml` | reusable (`workflow_call`) | Gemini による PR コードレビュー。指摘を CRITICAL / MAJOR / MINOR に分類して PR コメント投稿。`GEMINI_API_KEY` secret が必要 |
| `.github/workflows/dependabot-automerge.yml` | reusable (`workflow_call`) | Dependabot の patch / minor 更新を auto-merge（major は対象外） |
| `.github/workflows/pr-review.yml` | caller 例 | `pull_request` で `gemini-review.yml` を呼び出すサンプル |

## 利用方法（caller 側リポジトリ）

`.github/workflows/review.yml` を追加し、reusable workflow を呼び出す:

```yaml
name: PR Review
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
