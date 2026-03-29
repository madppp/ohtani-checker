# 🏟️ 大谷チェッカー

大谷翔平の広告・CMを見つけたら記録するWebアプリ。

**👉 https://madppp.github.io/ohtani-checker/**

---

## 📱 機能

- **テキスト登録** — タイトル・メモ・日時・場所を入力して記録
- **写真から登録** — 写真のEXIFデータ（撮影日時・GPS）を自動読み取り（写真本体は保存しない）
- **現在地自動取得** — フォームを開くと日時・場所を自動入力
- **クラウド同期** — スマホ・PCどこからでも同じデータにアクセス
- **Googleログイン** — アカウントごとにデータを分離管理

---

## 🛠 技術スタック

| カテゴリ | 技術 |
|---|---|
| フロントエンド | Vanilla HTML / CSS / JavaScript（単一ファイル） |
| 認証・DB | Supabase（Google OAuth + PostgreSQL） |
| セキュリティ | Row Level Security（ユーザーごとのデータ分離） |
| EXIF解析 | exifr（GPS・撮影日時の抽出） |
| 住所変換 | Nominatim / OpenStreetMap（逆ジオコーディング） |
| 位置情報 | Geolocation API |
| ホスティング | GitHub Pages |
| 開発 | Claude Code（サブエージェント形式） |

---

## 🏗 アーキテクチャ

```
ブラウザ（index.html 単一ファイル）
  ├── Supabase Auth     → Googleログイン
  ├── Supabase DB       → データの保存・取得（RLS付き）
  ├── Geolocation API   → 現在地取得
  ├── Nominatim API     → 緯度経度 → 住所変換
  └── exifr             → 写真EXIFから日時・GPS抽出
```

データはブラウザではなく**Supabaseクラウドに保存**されるため、端末を変えても消えない。

---

## 📂 ファイル構成

```
大谷/
  ├── index.html          # アプリ本体（全コードを1ファイルに集約）
  ├── ohtani-checker/
  │     └── index.html    # Claude Codeの作業ディレクトリ
  ├── CLAUDE.md           # Claude Code向け設計書・サブエージェント仕様
  └── .gitignore
```

---

## 🚀 ローカルで動かす

```bash
git clone https://github.com/madppp/ohtani-checker.git
cd ohtani-checker

# index.html をブラウザで開くだけ（サーバー不要）
open index.html
```

---

## 🔧 開発メモ（ハマりポイント集）

**写真の時刻が9時間ずれる**
EXIFの `DateTimeOriginal` はローカル時刻（JST）で記録されているが `new Date()` がUTCとして解釈するため発生。タイムゾーンオフセットを手動補正することで解決。

**iOSでフォーム入力時に画面が拡大される**
iOS Safariはfont-sizeが16px未満のinputで自動ズームする仕様。`font-size: 16px` と `maximum-scale=1.0` で解決。

**exifr lite版ではGPSが取れない**
`lite.umd.js` はGPS情報を省略している。`full.umd.js` を使うこと。

**GitHub Pagesで404になる**
リポジトリルートに `index.html` がないと404。`ohtani-checker/index.html` をルートにコピーしてpushすること。

---

## 📝 ライセンス

MIT
