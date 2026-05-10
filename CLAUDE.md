# 3D Graph Notes - Claude Code Memory

このファイルは Claude Code セッション間で書籍検索改善プロジェクトの文脈を引き継ぐためのもの。

## プロジェクト基本情報

- **アーキテクチャ**: PWA + Electron 配布 (単一の `index.html` ~950 行に CSS+HTML+JS 全部入り)
- **GitHub**: https://github.com/staka9161-gif/3d-graph-notes
- **デプロイ**: GitHub Pages (https://staka9161-gif.github.io/3d-graph-notes/)
- **データストア**: `localStorage` のみ (DB なし、サーバーなし)
- **Electron 固有ファイル** (`main.js` / `preload.js` / `package.json`) は GitHub 外。`index.html` のみ編集すれば PWA / デスクトップ両方に反映

## 開発ワークフロー

- ソース: `C:/Users/staka/Saved Games/3d-graph-notes/`
- master 直接 push (PR / branch 不要)
- 既存の minify スタイル (1関数1行詰め) を維持
- 各 Phase = 1 コミット (レビュー可能サイズ)
- パターン: 調査 → 計画 → GO → 実装 → diff → コミット

### 編集規則
- `index.html` 以外は変更しない
- 新規関数は `recentBooks` 関数群の近く (L650 付近) に配置
- 「ついで」修正禁止 (意図しない変更ゼロ)
- 実装前に必ず計画を提示し、ユーザーの GO サインを待つ

## 書籍検索の現状 (Phase 6 完了)

### API 構成
`Promise.allSettled` で 5 本並列:
- CiNii Books: `title=` + `author=`
- Google Books: `q=` + `intitle:` + `inauthor:`
- + openBD で ISBN 補完 (CiNii / Google それぞれ別呼出)

### スコアリング (`sortByRelevance` @ L757)
| 軸 | スコア |
|---|---|
| タイトル完全一致 | +2.0 |
| タイトル前方一致 | +1.0 |
| タイトル部分一致 | +0.5 |
| 著者完全一致 | +3.0 |
| 著者部分一致 | +2.0 |
| 著者不明ペナルティ | -0.5 |
| 全集系ペナルティ | -1.5 |
| 映像メディアペナルティ | -1.0 |
| 人気度ボーナス (Google Books のみ) | 0 〜 +1.5 |

正規表現:
- 全集系: `/(全作品|全集|選集|傑作集|作品集|大全|アンソロジー)/`
- 映像系: `/監督|脚本|主演|製作|出演/`
- 人気度: `Math.min(1.5, Math.log10(ratingsCount + 1) * 0.5) * (averageRating / 5)`

### 文字列正規化 (`norm` @ L756)
1. 全角 ASCII → 半角 ASCII (`Ａ-Ｚａ-ｚ０-９` → `A-Za-z0-9`)
2. `toLowerCase`
3. 区切り文字除去 (`[,、・\s　]+`)

### キャッシュ
- localStorage キー: `bookSearchCache:v6`
- TTL: 60 分 / 最大 50 件 / LRU 風 eviction
- スコアリング変更時はキー version を上げて旧キャッシュを cleanup する慣習
- 関連関数: `getSearchCache`, `setSearchCache`, `normalizeBookQuery` @ L656-658

### 主要関数の場所
| 関数 | 行 | 役割 |
|---|---|---|
| `parseCiniiItems(cj)` | L686 | CiNii レスポンスパース |
| `isbnToIsbn13(isbn)` | L685 | ISBN-10 → 13 変換 |
| `normalizeBookQuery(q)` | L656 | キャッシュキー正規化 |
| `getSearchCache(q)` | L657 | キャッシュ読取 |
| `setSearchCache(q, ...)` | L658 | キャッシュ書込 |
| `searchBooks(q)` | L687 | メインエントリ |
| `sortByRelevance(a, b)` | L757 | スコアリング (ローカル) |
| `norm(s)` | L756 | 文字列正規化 (ローカル) |

### book オブジェクトのスキーマ
parseCiniiItems と Google Books push の両方で生成される共通形状:
- 表示用: `title`, `authors`, `publisher`, `series`, `date`, `isbn`,
  `cover`, `price`, `pages`, `categories`, `lang`, `desc`, `rating` (★文字列)
- スコアリング用: `ratingNum` (number), `ratingsCount` (number)
  - CiNii はどちらも 0 固定 (rating 情報なし)
  - Google Books は volumeInfo.averageRating / ratingsCount から取得
  - Phase 7 (楽天) で reviewAverage / reviewCount を同じフィールドに書く想定

## Phase 履歴

| Phase | Hash | 内容 |
|---|---|---|
| 2a | `d176edd` | 著者検索追加 (CiNii author=, Google inauthor:) |
| 2b | `35e9b4b` | localStorage キャッシュ |
| 2c | `376d897` | sortByRelevance (多軸スコアリング) |
| 3 | `839d5f6` | ISBN-13 正規化 |
| 4 | `6b47271` | 全集ペナルティ + 全角半角正規化 |
| 5 | `c1324fa` | 映像メディアペナルティ |
| 6 | `011a0e4` | 人気度ボーナス (Google Books rating 活用) |

累計 +1 行 (948 → 949)、7 機能追加。

## 検証済みクエリ (Phase 5 後)

- 夏目漱石 → 漱石本人著作 (夢十夜・道草・硝子戸の中等) が上位 ✓
- 東野圭吾 → 容疑者X・名探偵の掟・パラレルワールド等の本人著作のみ ✓
- 村上春樹 → 単行本 (アンダーグラウンド・レキシントンの幽霊) が上位、全作品集は下位 ✓
- 容疑者Xの献身 → 本人 1 位 (回帰なし) ✓
- ノルウェイの森 → Phase 5 で映画版を蹴落とし、単行本浮上 ✓

## 残課題候補 (未着手)

| 候補 | 効果 | 規模 |
|---|---|---|
| 楽天 API 追加 (ユーザー提供キー方式) | 新刊カバレッジ向上 | 中〜大 |
| 出版年加点 | 古い版より新版優先 | 小 |
| シリーズグルーピング表示 | UX 改善 | 中 |

### 楽天 API について
Electron 配布では `accessKey` が asar 抽出で漏洩可能。ユーザー提供キー方式 (設定 UI で入力) が現実的。詳細は将来検討。

## デバッグ Tips

### Service Worker キャッシュ問題
PWA の `sw.js` は network-first 戦略。古い `index.html` が見える場合:
- DevTools → Application → Service Workers → Unregister
- `Ctrl+Shift+R` でハードリロード
- またはシークレットウィンドウでテスト (推奨)

### localStorage 確認
DevTools → Application → Local Storage → `https://staka9161-gif.github.io`

| キー | 用途 |
|---|---|
| `bookSearchCache:v6` | 検索結果キャッシュ (現行) |
| `recentBooks` | 最近使った書籍 (最大 10 件) |
| `3d-graph-notes` | ノート本体 |
| `3d-graph-notes-gcid` | Google OAuth Client ID |

### ローカル動作確認
```bash
cd "/c/Users/staka/Saved Games/3d-graph-notes"
python -m http.server 8080
# → http://localhost:8080/ をブラウザで開く
```
