# 大谷チェッカー — CLAUDE.md

## プロジェクト概要

大谷翔平の広告・CMを目撃したときに記録するWebアプリ。
テキスト入力または写真のメタデータ（EXIF）から日時・場所を自動取得して記録する。

---

## 🚨 既知のバグ修正（最優先で対応）

### Bug: 写真からの位置情報・日時が取得できない

**原因**: `exifr` の `lite` 版はGPS情報を含まない。

**修正内容**:

1. CDNのURLを `lite` → `full` に変更する

```html
<!-- ❌ 修正前 -->
<script src="https://cdn.jsdelivr.net/npm/exifr/dist/lite.umd.js"></script>

<!-- ✅ 修正後 -->
<script src="https://cdn.jsdelivr.net/npm/exifr/dist/full.umd.js"></script>
```

2. EXIF解析コードを以下に差し替える

```javascript
async function extractPhotoMetadata(file) {
  try {
    const result = await exifr.parse(file, {
      gps: true,
      pick: ['DateTimeOriginal', 'GPSLatitude', 'GPSLongitude']
    });

    if (!result) {
      return { datetime: null, location: null };
    }

    // 日時: DateTimeOriginal は Date オブジェクトまたは文字列 "2025:03:29 14:30:00"
    let datetime = null;
    if (result.DateTimeOriginal) {
      const d = result.DateTimeOriginal instanceof Date
        ? result.DateTimeOriginal
        : new Date(result.DateTimeOriginal.replace(/^(\d{4}):(\d{2}):(\d{2})/, '$1-$2-$3'));
      if (!isNaN(d.getTime())) {
        datetime = d.toISOString();
      }
    }

    // GPS: exifr.parse with gps:true で lat/lng が直接返る
    let location = null;
    if (result.latitude != null && result.longitude != null) {
      const lat = result.latitude;
      const lng = result.longitude;
      const label = await reverseGeocode(lat, lng);
      location = { lat, lng, label };
    }

    return { datetime, location };

  } catch (e) {
    console.warn('EXIF解析エラー:', e);
    return { datetime: null, location: null };
  }
}
```

3. 写真選択イベントのフォーム反映ロジック

```javascript
photoInput.addEventListener('change', async (e) => {
  const file = e.target.files[0];
  if (!file) return;

  showLoading('写真から情報を読み取っています...');
  const { datetime, location } = await extractPhotoMetadata(file);

  if (datetime) {
    datetimeInput.value = datetime.slice(0, 16); // "2025-03-29T14:30"
    showToast('📅 写真の撮影日時を取得しました');
  } else {
    showToast('⚠️ 日時情報が見つかりません。手動で入力してください');
  }

  if (location) {
    locationInput.value = location.label;
    latInput.value = location.lat;
    lngInput.value = location.lng;
    showToast('📍 写真の位置情報を取得しました');
  } else {
    showToast('⚠️ 位置情報が見つかりません。手動で入力してください');
  }

  hideLoading();
});
```

**修正完了の確認**:
- [ ] GPS情報付きの写真を選択したとき、場所フィールドが書き換わる
- [ ] 撮影日時フィールドが現在時刻ではなく写真の日時になる
- [ ] GPS情報なし写真でクラッシュしない（警告トーストのみ）

---

## Git管理セットアップ

プロジェクトルートで以下を実行する：

```bash
# 1. Gitリポジトリ初期化
git init

# 2. .gitignore作成
cat > .gitignore << 'EOF'
.DS_Store
*.log
node_modules/
.env
EOF

# 3. 初回コミット
git add .
git commit -m "feat: 大谷チェッカー 初期実装"

# 4. GitHubにリポジトリを作成後、リモート登録してpush
git remote add origin https://github.com/<YOUR_USERNAME>/ohtani-checker.git
git branch -M main
git push -u origin main
```

---

## デプロイ（GitHub Pages）

このアプリはサーバー不要の静的HTMLなので **GitHub Pages が最適**（無料・即時反映）。

### 初回セットアップ

GitHubリポジトリ作成後、Settings → Pages → Source を `main` ブランチ / `/ (root)` に設定するだけでOK。

### 公開URL

```
https://<YOUR_USERNAME>.github.io/ohtani-checker/
```

### 以降の更新手順

```bash
git add .
git commit -m "fix: EXIFバグ修正"
git push origin main
# → GitHub Pagesに自動反映（数十秒〜1分）
```

---

## サブエージェント構成（初回実装用）

> ⚠️ **すでに初回実装済みの場合はこのセクションは不要。上記バグ修正のみ実行すること。**

---

### Agent 1: Architect（設計）

**タスク**:
1. 技術スタック決定
   - フレームワーク: **Vanilla HTML/CSS/JS（単一ファイル `index.html`）**
   - データ永続化: **IndexedDB**
   - EXIF解析: **exifr full版**（`https://cdn.jsdelivr.net/npm/exifr/dist/full.umd.js`）
   - アイコン: **Lucide Icons**（CDN）
   - フォント: **Noto Sans JP**（Google Fonts）
2. IndexedDBスキーマ定義

```javascript
// DB: OhtaniCheckerDB v1 / store: sightings
{
  id: string,           // crypto.randomUUID()
  title: string,
  note: string,
  datetime: string,     // ISO 8601
  location: { lat: number|null, lng: number|null, label: string },
  source: "text"|"photo",
  createdAt: string,
  updatedAt: string
}
```

3. `index.html` の骨格（HTML構造のみ）を作成

**出力**: 骨格のみの `index.html`

---

### Agent 2: DataLayer（データ層）

**前提**: Agent 1の `index.html` を受け取る

**タスク**:
1. IndexedDB初期化（`openDB()`）
2. CRUD関数実装

```javascript
async function getAllSightings()     // 全件取得（datetime降順）
async function getSighting(id)
async function createSighting(data)
async function updateSighting(id, data)
async function deleteSighting(id)
```

3. ユーティリティ（🚨バグ修正セクションのコードを使うこと）
   - `extractPhotoMetadata(file)` — exifr full版でGPS+日時取得
   - `getCurrentPosition()` — Geolocation API
   - `reverseGeocode(lat, lng)` — Nominatim API
4. 日時フォーマット関数

```javascript
function formatDatetime(isoString)  // "2025年3月29日（土）14:30"
function formatRelative(isoString)  // "3時間前" / "昨日"
```

**出力**: データ層実装済みの `index.html`

---

### Agent 3: UI（画面実装）

**前提**: Agent 2の `index.html` を受け取る

**デザイン方針（DayOne参考）**:
- 背景: ホワイト、アクセント: **#E63312**
- フォント: Noto Sans JP、数字は等幅
- カード型リスト、角丸、グレーサブテキスト
- アニメーション: カードはフェードイン、モーダルはスライドアップ

**実装する画面**:

#### 一覧画面
```
┌─────────────────────────────┐
│  大谷チェッカー        [検索]│
├─────────────────────────────┤
│ 2025年3月                   │  ← 月セクション（スティッキー）
│ ┌─────────────────────────┐ │
│ │ 29（土）14:30           │ │
│ │ 渋谷駅前                │ │
│ │ ドコモのCM看板          │ │
│ │ [text]                  │ │
│ └─────────────────────────┘ │
│                         [＋]│  ← FAB
└─────────────────────────────┘
```

#### 新規登録モーダル
```
┌─────────────────────────────┐
│          新しい記録     [×] │
├─────────────────────────────┤
│ タイトル [________________] │
│ メモ     [________________] │
│ 📅 2025年3月29日 14:30  [✏]│  ← 自動入力＋編集可
│ 📍 渋谷駅前         [✏]   │  ← 自動入力＋編集可
│ 📷 写真から情報を読み取る   │  ← EXIFのみ（写真保存なし）
│ [キャンセル]   [保存する]   │
└─────────────────────────────┘
```

- モーダルを開いた瞬間にGeolocation + 現在時刻を自動入力
- 写真選択時はEXIF解析で日時・場所を上書き

#### 詳細・編集モーダル
- 詳細表示 → 「編集」ボタンで編集モードへ
- 「削除」ボタン（確認ダイアログあり）

**CSSルール**:
- CSS変数で色・スペーシング管理
- モバイルファースト（max-width: 480px）
- `env(safe-area-inset-*)` 対応

**出力**: UI実装済みの `index.html`

---

### Agent 4: Integration & Polish（統合・仕上げ）

**前提**: Agent 3の `index.html` を受け取る

**タスク**:
1. UIとデータ層の結合
2. エラーハンドリング
   - Geolocation拒否 → トースト＋手動入力誘導
   - EXIF情報なし → トースト（クラッシュしない）
   - IndexedDB失敗 → トースト
3. UX細部
   - 保存成功: 「✅ 記録しました！」トースト（2秒）
   - FABタップ: `navigator.vibrate(10)`
   - ロード中: スケルトンスクリーン
   - 検索: タイトル・場所・メモでフィルタリング
4. 動作確認チェックリスト
   - [ ] テキストで新規登録できる
   - [ ] 写真EXIFから日時・場所が正しく自動入力される（現在時刻に戻らない）
   - [ ] GPS情報なし写真でクラッシュしない
   - [ ] 編集・削除が機能する
   - [ ] ページリロード後もデータが保持される
5. Git初期化

```bash
git init
echo '.DS_Store\n*.log\nnode_modules/' > .gitignore
git add .
git commit -m "feat: 大谷チェッカー 初期実装"
```

**出力**: 完成版 `ohtani-checker/index.html` + `.gitignore`

---

## オーケストレーター向け指示

```
以下の順番でサブエージェントを呼び出してください:

1. Agent 1 (Architect)   → index.html（骨格）
2. Agent 2 (DataLayer)   → index.html（データ層追加）
3. Agent 3 (UI)          → index.html（UI実装済み）
4. Agent 4 (Integration) → ohtani-checker/index.html（完成版）+ git init

各エージェントは前のエージェントの出力ファイルを読み込んでから作業を開始すること。
```

---

## 技術メモ

### exifr full版（GPS対応）

```javascript
// full版でグローバルに exifr が展開される
const result = await exifr.parse(file, { gps: true });
// result.latitude, result.longitude で直接取得可能
```

### Nominatim Reverse Geocoding

```javascript
const res = await fetch(
  `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`,
  { headers: { 'Accept-Language': 'ja' } }
);
const data = await res.json();
const label = data.address?.city || data.address?.town || data.display_name;
```

### Geolocation

```javascript
const pos = await new Promise((resolve, reject) =>
  navigator.geolocation.getCurrentPosition(resolve, reject, {
    enableHighAccuracy: true, timeout: 10000
  })
);
const { latitude: lat, longitude: lng } = pos.coords;
```

---

## 完成イメージ

- ファイル: `ohtani-checker/index.html`（1ファイルで完結）
- ホスティング: GitHub Pages（`https://<username>.github.io/ohtani-checker/`）
- ストレージ: IndexedDB（ブラウザ内、サーバー不要）
- 写真: 保存なし（EXIFメタデータのみ抽出）
