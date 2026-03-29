# 大谷チェッカー — CLAUDE.md

## プロジェクト概要

大谷翔平の広告・CMを目撃したときに記録するWebアプリ。
テキスト入力または写真のメタデータ（EXIF）から日時・場所を自動取得して記録する。
Supabaseでクラウド保存・ユーザー認証対応。複数ユーザーが各自のデータを管理できる。

---

## 技術スタック

- フロントエンド: Vanilla HTML/CSS/JS（単一ファイル `index.html`）
- 認証・DB: **Supabase**（Google OAuth + PostgreSQL）
- EXIF解析: exifr full版（CDN: `https://cdn.jsdelivr.net/npm/exifr/dist/full.umd.js`）
- アイコン: Lucide Icons（CDN）
- フォント: Noto Sans JP（Google Fonts）

---

## Supabase設定

### 接続情報（index.html内に記述）

```javascript
const SUPABASE_URL = 'https://wkfxbgidlojjbtsuyoea.supabase.co';
const SUPABASE_ANON_KEY = '★ここにpublishable keyを入力★';
// publishable key は Supabase Dashboard → Settings → API → anon/publishable key
```

### Supabase Dashboardで行う作業（Claude Codeは実行できないので手動で行うこと）

#### 1. テーブル作成（SQL Editor で実行）

```sql
create table sightings (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade not null,
  title text,
  note text,
  datetime timestamptz not null,
  lat double precision,
  lng double precision,
  location_label text,
  source text check (source in ('text', 'photo')),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

#### 2. Row Level Security（RLS）を有効化（SQL Editor で実行）

```sql
-- RLS有効化
alter table sightings enable row level security;

-- 自分のデータだけ読める
create policy "自分のデータのみ読み取り可"
  on sightings for select
  using (auth.uid() = user_id);

-- 自分のデータだけ作成できる
create policy "自分のデータのみ作成可"
  on sightings for insert
  with check (auth.uid() = user_id);

-- 自分のデータだけ更新できる
create policy "自分のデータのみ更新可"
  on sightings for update
  using (auth.uid() = user_id);

-- 自分のデータだけ削除できる
create policy "自分のデータのみ削除可"
  on sightings for delete
  using (auth.uid() = user_id);
```

#### 3. Google OAuth設定（Supabase Dashboard → Authentication → Providers → Google）

- Google Cloud ConsoleでOAuthクライアントID作成
- 承認済みリダイレクトURIに `https://wkfxbgidlojjbtsuyoea.supabase.co/auth/v1/callback` を追加
- Client IDとClient SecretをSupabaseに登録

#### 4. リダイレクトURL許可（Supabase Dashboard → Authentication → URL Configuration）

```
https://madppp.github.io/ohtani-checker/
http://localhost:3000  ← ローカル開発用
```

---

## 実装タスク（Claude Codeが実装すること）

### 既存の index.html を以下の仕様で全面リライトする

---

### 1. Supabase初期化

```javascript
import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

---

### 2. 認証フロー

#### ログイン画面（未ログイン時に表示）

```
┌─────────────────────────────┐
│                             │
│      🏟️ 大谷チェッカー      │
│                             │
│   大谷翔平の広告を見つけたら│
│        記録しよう！         │
│                             │
│   [Googleでログイン]        │
│                             │
└─────────────────────────────┘
```

#### 認証処理

```javascript
// Googleログイン
async function signInWithGoogle() {
  const { error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: 'https://madppp.github.io/ohtani-checker/'
    }
  });
}

// ログアウト
async function signOut() {
  await supabase.auth.signOut();
}

// 認証状態の監視
supabase.auth.onAuthStateChange((event, session) => {
  if (session) {
    showMainApp(session.user);
  } else {
    showLoginScreen();
  }
});

// 初期化時にセッション確認
const { data: { session } } = await supabase.auth.getSession();
```

---

### 3. データ操作（IndexedDBの代わりにSupabaseを使う）

```javascript
// 全件取得（新しい順）
async function getAllSightings() {
  const { data, error } = await supabase
    .from('sightings')
    .select('*')
    .order('datetime', { ascending: false });
  if (error) throw error;
  return data;
}

// 新規作成
async function createSighting(sightingData) {
  const { data: { user } } = await supabase.auth.getUser();
  const { data, error } = await supabase
    .from('sightings')
    .insert({ ...sightingData, user_id: user.id })
    .select()
    .single();
  if (error) throw error;
  return data;
}

// 更新
async function updateSighting(id, updates) {
  const { data, error } = await supabase
    .from('sightings')
    .update({ ...updates, updated_at: new Date().toISOString() })
    .eq('id', id)
    .select()
    .single();
  if (error) throw error;
  return data;
}

// 削除
async function deleteSighting(id) {
  const { error } = await supabase
    .from('sightings')
    .delete()
    .eq('id', id);
  if (error) throw error;
}
```

---

### 4. EXIF解析（バグ修正済みコードを使うこと）

```javascript
async function extractPhotoMetadata(file) {
  try {
    const result = await exifr.parse(file, {
      gps: true,
      pick: ['DateTimeOriginal', 'GPSLatitude', 'GPSLongitude']
    });

    if (!result) return { datetime: null, location: null };

    let datetime = null;
    if (result.DateTimeOriginal) {
      const d = result.DateTimeOriginal instanceof Date
        ? result.DateTimeOriginal
        : new Date(result.DateTimeOriginal.replace(/^(\d{4}):(\d{2}):(\d{2})/, '$1-$2-$3'));
      if (!isNaN(d.getTime())) datetime = d.toISOString();
    }

    let location = null;
    if (result.latitude != null && result.longitude != null) {
      const label = await reverseGeocode(result.latitude, result.longitude);
      location = { lat: result.latitude, lng: result.longitude, label };
    }

    return { datetime, location };
  } catch (e) {
    console.warn('EXIF解析エラー:', e);
    return { datetime: null, location: null };
  }
}
```

---

### 5. UI仕様（DayOne風）

#### 一覧画面（ログイン後のメイン画面）

```
┌─────────────────────────────┐
│  大谷チェッカー  [検索] [👤]│  ← 右上にアカウントアイコン
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

- アカウントアイコンタップ → 「ログアウト」メニュー
- データ読み込み中はスケルトンスクリーン表示

#### 新規登録モーダル（ボトムシート）

```
┌─────────────────────────────┐
│          新しい記録     [×] │
├─────────────────────────────┤
│ タイトル [________________] │
│ メモ     [________________] │
│ 📅 2025年3月29日 14:30  [✏]│
│ 📍 渋谷駅前         [✏]   │
│ 📷 写真から情報を読み取る   │
│ [キャンセル]   [保存する]   │
└─────────────────────────────┘
```

#### デザイン
- 背景: ホワイト、アクセント: `#E63312`
- モバイルファースト（max-width: 480px）
- `env(safe-area-inset-*)` 対応
- CSS変数で全色・スペーシング管理

---

### 6. エラーハンドリング

- Supabase接続エラー → 「通信エラーが発生しました」トースト
- 未ログインでAPIアクセス → ログイン画面にリダイレクト
- Geolocation拒否 → 手動入力へ誘導
- EXIF情報なし → 警告トースト（クラッシュしない）

---

### 7. 動作確認チェックリスト

- [ ] Googleログインができる
- [ ] ログアウトができる
- [ ] 別ブラウザ・別デバイスで同じアカウントのデータが見える
- [ ] テキストで新規登録できる
- [ ] 写真EXIFから日時・場所が自動入力される
- [ ] 編集・削除が機能する
- [ ] 他のユーザーのデータが見えない（RLS確認）

---

### 8. 実装完了後にgit pushする

```bash
git add index.html
git commit -m "feat: Supabase認証・クラウド保存対応"
git push origin main
```

---

## 完成イメージ

- URL: `https://madppp.github.io/ohtani-checker/`
- データ: Supabase PostgreSQL（永続保存・デバイス間同期）
- 認証: Googleログイン（ユーザーごとにデータ分離）
- 写真: 保存なし（EXIFメタデータのみ）
