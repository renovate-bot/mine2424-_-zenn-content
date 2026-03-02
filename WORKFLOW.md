# Zenn 記事自動生成・投稿ワークフロー

## 概要

AI（Claude）が毎日3回、4軸の技術記事を自動生成して `mine2424/zenn-content` にpushします。
Zennはこのリポジトリと連携しており、pushされると自動でdraft記事として同期されます。

---

## 自動生成スケジュール

| 時刻（JST） | 生成される記事 |
|---|---|
| 09:00 | 4軸 × 4記事（`YYYYMMDD-0900-*.md`） |
| 13:00 | 4軸 × 4記事（`YYYYMMDD-1300-*.md`） |
| 19:00 | 4軸 × 4記事（`YYYYMMDD-1900-*.md`） |

**1日合計: 12記事（すべてdraft状態）**

---

## 記事の4軸

| 軸 | テーマ例 |
|---|---|
| 🤖 AI | LLM・プロンプトエンジニアリング・AIエージェント |
| 🧪 QA | テスト自動化・E2E・品質保証・TDD |
| ⚡ Next.js | App Router・Server Components・パフォーマンス |
| 🦋 Flutter | Dart・状態管理・ウィジェット・Riverpod |

---

## ファイル命名規則

### Zennのslug要件（必須）

Zennはファイル名（拡張子を除く）をslugとして使用します。以下の条件を**すべて満たす**必要があります：

- **文字数**: 12文字以上・50文字以下
- **使用可能文字**: 半角英数字（a-z0-9）、ハイフン（-）、アンダースコア（_）のみ
- 大文字・日本語・スペース・その他記号は使用不可

### 自動生成ファイル名の形式

```
articles/YYYYMMDD-HHMM-{軸}-{suffix}.md

例:
articles/20260302-0900-ai-tech.md       （20文字 ✅）
articles/20260302-0900-qa-testing.md    （24文字 ✅）
articles/20260302-0900-nextjs-tech.md   （25文字 ✅）
articles/20260302-0900-flutter-dev.md   （25文字 ✅）
```

### NG例

```
articles/20260302-ai.md   ❌ 11文字（12文字未満）
articles/20260302-qa.md   ❌ 11文字（12文字未満）
```

---

## 記事スタイル

- **Zennスタイル**：説明・考え方・概念中心
- コードは最小限（1〜2ブロック程度）
- 3000〜4000字の読み応えある説明文
- すべてdraft（`published: false`）で生成

## ⚠️ 自動公開は絶対にしない

自動生成スクリプトは **`published: false`（draft）のみ** で動作します。

- cronジョブは記事の**生成とGitHub pushのみ**を行う
- 公開（`published: true`）への変更は**必ず手動**で行う
- 記事の品質確認・編集後に、Ryotaが判断して公開する

---

## Frontmatter形式

```yaml
---
title: "記事タイトル"
emoji: "🤖"
type: "tech"
topics: ["ai", "llm", "promptengineering"]
published: false
---
```

---

## 記事を公開するフロー

1. [zenn.dev/dashboard](https://zenn.dev/dashboard) を開く
2. 対象のdraft記事を選択
3. 内容を確認・編集
4. `published: true` に変更してpush、またはZennの管理画面から公開

### GitHubから直接公開する場合

```bash
# 対象ファイルのpublishedをtrueに変更
sed -i 's/published: false/published: true/' articles/YYYYMMDD-HHMM-{軸}.md

git add articles/YYYYMMDD-HHMM-{軸}.md
git commit -m "publish: YYYY-MM-DD {軸} 記事"
git push origin main
```

---

## リポジトリ構成

```
zenn-content/
├── articles/       # 記事（自動生成 + 手動編集）
├── books/          # 本（手動作成）
├── WORKFLOW.md     # このファイル
└── README.md
```

---

## 手動で記事を追加する場合

Zenn CLIを使ってローカルで記事を作成できます。

```bash
# Zenn CLIインストール
npm install zenn-cli

# 新規記事作成
npx zenn new:article --slug my-article --title "タイトル" --type tech

# プレビュー
npx zenn preview
```

---

## トラブルシューティング

### Zennに記事が反映されない

- GitHubリポジトリとZennの連携設定を確認
- `published: false` の記事はZennのダッシュボードの「下書き」に表示される
- pushから反映まで数分かかる場合がある

### 記事のテーマが重複している

- 自動生成AIは過去記事のタイトルを参照してテーマを選ぶ設計になっていない
- 気になる場合は手動でタイトル・内容を編集してからpushすればOK

---

## 関連リンク

- [Zennダッシュボード](https://zenn.dev/dashboard)
- [mine2424/zenn-content](https://github.com/mine2424/zenn-content)
- [Zenn CLIドキュメント](https://zenn.dev/zenn/articles/zenn-cli-guide)
