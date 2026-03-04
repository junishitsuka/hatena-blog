---
name: write-article
description: junishitsukaのはてなブログ記事を執筆する。outlines/配下のアウトラインを元に、guidelines/に従ったスタイルで記事を生成する。
---

# ブログ記事執筆

junishitsukaのスタイルと人格でブログ記事を執筆する。

## 事前確認（必須）

執筆前に以下を必ず読むこと：

- `guidelines/writing-style.md` — 文体・人格
- `guidelines/article-templates.md` — 構成テンプレート
- `guidelines/article-categories.md` — カテゴリ判定

## アウトラインの読み込み（必須）

`outlines/` 配下から対象のアウトラインファイルを特定して読み込む。

- ユーザーがファイルを指定した場合はそのファイルを使う
- 指定がない場合は `outlines/` 配下のファイルを一覧表示してユーザーに選ばせる

アウトラインのディレクトリ構造がそのまま投稿先ドメインを示す：
- `outlines/product.plex.co.jp/` → product.plex.co.jp に投稿
- `outlines/mitsuruya.hatenablog.com/` → mitsuruya.hatenablog.com に投稿

アウトラインのファイル名 `YYYYMMDD_[slug].md` から公開予定日とスラッグを読み取る。

## カテゴリ判定

| カテゴリ | 投稿先 | 判定基準 |
|---|---|---|
| 技術記事 | product.plex.co.jp | 実装・Tips・コード例を含む |
| 組織・採用記事 | product.plex.co.jp | チーム・イベント・採用 |
| 趣味・通常記事 | mitsuruya.hatenablog.com | 趣味・ライフスタイル |

## 執筆ルール

**文体（変えてはいけない）:**
- 一人称: 「自分」（「私」は使わない）
- 語尾: です・ます調
- 謙虚な表現: 「個人的には〜」「〜と感じています」
- 絵文字: 技術・組織記事では使用禁止

**構成（全カテゴリ共通）:**
1. フロントマター（title / published / updated / url / tags）
2. 「こんにちは、プレックスの[石塚](https://twitter.com/ij_spitz)です。」で始める
3. 導入（背景・動機を2〜3段落）
4. `[:contents]`
5. 本論
6. 「おわりに」または「終わりに」
7. 技術・組織記事のみ: 採用情報リンク `[https://dev.plex.co.jp/:embed:cite]`

**はてな記法:**
- リンク埋め込み: `[https://example.com/:embed:cite]`
- 画像: `[f:id:username:imageid]`
- 目次: `[:contents]`
- コードブロック: 言語名を必ず指定

## 出力先

```
completed/YYYYMMDD_[slug].md
```

アウトラインのファイル名（例: `20260331_heroku_use.md`）をそのまま使う。

## 執筆後チェックリスト

- [ ] 一人称が「自分」になっているか
- [ ] です・ます調で統一されているか
- [ ] 謙虚な表現が使われているか
- [ ] 具体例・コード例があるか（技術記事）
- [ ] 失敗・課題を正直に書いているか
- [ ] `[:contents]` が導入の後にあるか
- [ ] 技術・組織記事に採用情報があるか
- [ ] ファイルを正しいパスに保存したか
