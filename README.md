# shared-workflows

複数リポジトリ共有の GitHub Actions Reusable Workflows

## 内容

### `.github/workflows/gemini-review.yml`

Gemini 2.5 Flash による PR コードレビューワークフロー。

PR の diff を Gemini に送信し、指摘事項を CRITICAL / MAJOR / MINOR に分類して PR コメントとして投稿する。

### 機能

- PR diff の自動取得（`fetch-depth: 0`）
- Gemini 2.5 Flash によるコードレビュー（日本語）
- 指摘の重要度分類: CRITICAL（バグ・脆弱性）/ MAJOR（ロジックエラー）/ MINOR（スタイル）
- diff の大規模対応（100K 文字超は truncate）
- レビュー結果を PR コメントとして自動投稿

## 利用方法

caller 側リポジトリに以下のワークフローを追加する（`.github/workflows/review.yml`）:

```yaml
name: Code Review
on:
  pull_request:

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    uses: yushin0319/shared-workflows/.github/workflows/gemini-review.yml@main
    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
```

### 注意事項

- `permissions` は **caller 側**に設定すること。reusable workflow 側に設定すると `startup_failure` になる。
- `GEMINI_API_KEY` は caller 側リポジトリの Secrets に登録が必要。

## 技術スタック

- Python 3.12
- [google-genai](https://pypi.org/project/google-genai/) SDK
