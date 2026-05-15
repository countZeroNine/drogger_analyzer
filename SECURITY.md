# SECURITY 監査レポート

**監査日**: 2026-05-15
**対象**: `drogger_analyzer.html`, `aracer_lab.html`
**基準**: プロジェクト指示「セキュリティ設定自動組み込み」

---

## 概要

`drogger_analyzer.html` および `aracer_lab.html` は単一ファイル HTML アプリで、ビルドステップ・npm パッケージ依存なし。ブラウザ実行のみ。CDN 経由でライブラリ取得。

`npm install` を行わないため、Mini Shai-Hulud 等の install-time マルウェアによる credential 窃取リスクは構造的にゼロ。残るリスクは **CDN 自体の侵害**および **デプロイ後のレスポンスヘッダ未設定**。

---

## 監査結果と対処

### ✅ Mini Shai-Hulud 感染パッケージリストの照合

ユーザールール記載の感染グループ（`@tanstack/*`, `mistralai`, `@uipath/*`）および web_search で確認した他の感染パッケージ（`axios`, `@react-native-aria/*`, `react-native-international-phone-number`, `PyTorch Lightning 2.6.2-2.6.3`, `intercom-client@7.0.4` 等）について：

**結果**：いずれも未使用。安全。

| 使用ライブラリ | 感染リスト掲載 | 結果 |
|---|---|---|
| React 18 | × | OK |
| ReactDOM 18 | × | OK |
| Tailwind CSS | × | OK |
| Babel Standalone | × | OK |
| PapaParse | × | OK |

### ✅ 対処: CDN バージョン固定（重要・修正済み）

**Before**: `react@18`, `@babel/standalone`, `cdn.tailwindcss.com` 等が浮動指定で、CDN 側が侵害された場合に任意のバージョンが配信されるリスクあり。

**After**: 以下に明示的に固定。

```html
<!-- Pinned versions (audit: 2026-05-15) -->
<script src="https://cdn.tailwindcss.com/3.4.17"></script>
<script src="https://unpkg.com/react@18.3.1/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://unpkg.com/@babel/standalone@7.25.6/babel.min.js"></script>
```

バージョン更新は手動レビュー後のみ実施。

### ✅ 対処: `vercel.json` 新規作成（セキュリティヘッダ）

`vercel.json` を新規作成。以下を全レスポンスに適用：

| ヘッダ | 値 | 効果 |
|---|---|---|
| **Content-Security-Policy** | `default-src 'self'` + 必要な CDN のみ許可 | XSS・不正なリソース読込を防御 |
| **X-Content-Type-Options** | `nosniff` | MIME スニッフィング攻撃を防御 |
| **X-Frame-Options** | `DENY` | クリックジャッキングを防御 |
| **Referrer-Policy** | `strict-origin-when-cross-origin` | リファラ漏洩を最小化 |
| **Permissions-Policy** | カメラ・マイク・GPS 等を全拒否 | 不要な API アクセスをブロック |
| **Strict-Transport-Security** | `max-age=63072000; includeSubDomains; preload` | HTTPS 強制（2年） |

CSP の主要な許可ドメイン：

- `unpkg.com` — React, ReactDOM, Babel, PapaParse
- `cdn.tailwindcss.com` — Tailwind CSS
- `fonts.googleapis.com` + `fonts.gstatic.com` — aRacer Lab のフォント
- `va.vercel-scripts.com` — Vercel Analytics（drogger_analyzer のみ）

⚠️ **トレードオフ**：`'unsafe-eval'` は Babel Standalone がブラウザ内 JSX コンパイルで eval を使うため必須。`'unsafe-inline'` は Tailwind 動的スタイル注入で必須。これらは単一ファイル HTML アーキテクチャの制約。本番ビルドステップを導入すれば除去可能。

### ✅ 対処: `.gitignore` 新規作成

ユーザールール指定パターンを全て含めた `.gitignore` を作成：

```
.claude/setup.mjs
.vscode/setup.mjs
**/*token*  (大文字小文字バリエーション含む)
**/*secret* (同上)
```

加えて、標準的な credential file pattern（`.pem`, `.key`, `.env` 等）、node_modules、ビルド出力、IDE 一時ファイルも除外。

### ✅ 対処: SRI（Subresource Integrity）ハッシュ追加

**現状**: 完了済み（2026-05-15）

unpkg.com 配信の 4 ファイルすべてに SHA-384 ハッシュを `integrity` 属性として埋め込み済み。CDN 配信物が改ざんされた場合、ブラウザがハッシュ不一致を検出してロードを拒否する。

| パッケージ | SRI ハッシュ |
|---|---|
| `react@18.3.1` | `sha384-DGyLxAyjq0f9SPpVevD6IgztCFlnMF6oW/XQGmfe+IsZ8TqEiDrcHkMLKI6fiB/Z` |
| `react-dom@18.3.1` | `sha384-gTGxhz21lVGYNMcdJOyq01Edg0jhn/c22nsx0kyqP0TxaV5WVdsSH1fSDUf5YJj1` |
| `@babel/standalone@7.25.6` | `sha384-EIcs2MbARRLcLs7Gv2wqqhhfMlcLeW9yuJsWki1r5iveN1F5gpzHOwAKycJqSQOI` |
| `papaparse@5.4.1` (drogger のみ) | `sha384-D/t0ZMqQW31H3az8ktEiNb39wyKnS82iFY52QPACM+IjKW3jDUhyIgh2PApRqJZs` |

⚠️ **Tailwind CSS Play CDN（`cdn.tailwindcss.com/3.4.17`）は SRI 非適用**：Tailwind Play CDN は使用クラスに応じて動的にレスポンスを生成するため、SRI ハッシュが一意に定まらない（事前計算不可）。Tailwind Labs が運営する単一供給チェーンであることと、CSP の `style-src` 制限で代替防御。本番ビルドへ移行する際に Tailwind CLI 経由の事前ビルド済み CSS を採用すれば、その CSS にも SRI 適用可能になる。

**バージョン更新時の運用フロー**:
1. https://www.srihash.org/ で新バージョンのハッシュを生成
2. HTML の `integrity` 属性を新ハッシュに更新
3. ローカルで動作確認（ハッシュ不一致なら読み込み失敗するため即検知可能）
4. デプロイ

### ⚪ N/A: npm パッケージマネージャー関連

本プロジェクトは `package.json` を持たないため、以下のルールは N/A：

- `engines` 指定（Node.js >= 20）→ N/A
- `save-exact` でバージョン固定 → N/A
- `.npmrc` (`ignore-scripts=true`) → N/A
- pnpm 11 以降の推奨 → N/A

**将来 npm 導入する場合**：これらのルールに全部準拠し、別途 audit を実施する必要があります。

---

## 残存リスク

### CDN 自体の侵害

固定したバージョン (`react@18.3.1` 等) のファイルが unpkg.com 側で侵害された場合、**SRI ハッシュにより検出可能**（実装済）。改ざんされたファイルはブラウザがロードを拒否する。

唯一の例外は `cdn.tailwindcss.com` のみ（動的レスポンスのため SRI 非適用）。Tailwind Labs 単一の供給チェーンで、unpkg/npm より侵害確率は低いものの、ゼロではない。

### `'unsafe-eval'` の許可

Babel Standalone がブラウザ内で JSX を eval() でコンパイルするため、CSP で `'unsafe-eval'` を許可している。これは XSS 攻撃の効果を増幅する可能性がある。

**緩和策**：
- 本番化フェーズで Babel をビルド時コンパイルに移行 → `'unsafe-eval'` 除去可能
- 現状は単一ファイル HTML を維持するため受容

### Tailwind Play CDN のリスク

`cdn.tailwindcss.com` は Tailwind Labs が運営する CDN。供給チェーンが unpkg/npm より短い（Tailwind 単独）が、Tailwind Labs 自体が侵害された場合は影響を受ける。

**緩和策**：
- 本番化フェーズで Tailwind CLI による事前ビルド済み CSS を採用
- 現状は受容

---

## 監査済みファイル一覧

| ファイル | 内容 | 状態 |
|---|---|---|
| `drogger_analyzer.html` | CDN 固定、セキュリティコメント追加 | ✅ 修正済 |
| `aracer_lab.html` | CDN 固定、セキュリティコメント追加 | ✅ 修正済 |
| `vercel.json` | セキュリティヘッダ全 7 項目 | ✅ 新規作成 |
| `.gitignore` | ユーザー指定パターン + 標準除外 | ✅ 新規作成 |

---

## 次回 audit 推奨タイミング

- ピン留めしたパッケージのいずれかにメジャーアップデート（または重要セキュリティ修正）が出た時
- CDN 提供元（unpkg.com、cdn.tailwindcss.com）から侵害公表があった時
- 新たな依存パッケージ追加時
- 最低でも 3 ヶ月に 1 回
