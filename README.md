# 薬局待ち時間モニター

薬局待合室向けの待ち時間表示システムです。

## 画面構成

| ファイル | 用途 | 想定端末 |
|---------|------|---------|
| `staff.html` | 受付番号発行・呼び出し操作 | カウンター内PC・タブレット |
| `monitor.html` | 待ち人数・目安時間の表示 | 待合室TV・タブレット |

## 主な機能

- 受付番号の自動採番（毎日001からリセット）
- 一包化フラグによる待ち時間加算
- Supabase Realtimeによる即時反映
- 待ち人数・目安待ち時間の自動計算

## 技術構成

- フロントエンド：単一HTMLファイル（フレームワーク不使用）
- データベース：Supabase PostgreSQL
- リアルタイム通信：Supabase Realtime（WebSocket）
- ホスティング：Vercel

## セットアップ

### 1. Supabase テーブル作成

SQL Editorで以下を実行：

```sql
create table waiting_queue (
  id          uuid primary key default gen_random_uuid(),
  ticket_no   integer not null,
  status      text not null default 'waiting',
  is_ippoka   boolean not null default false,
  created_at  timestamptz not null default now(),
  called_at   timestamptz
);

alter publication supabase_realtime add table waiting_queue;
```

### 2. 環境変数の設定

`staff.html` と `monitor.html` 内の以下を自環境の値に書き換え：

```js
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-key';
```

### 3. Vercelにデプロイ

GitHubリポジトリをVercelに連携してデプロイ。

## 待ち時間の計算ロジック

```
目安待ち時間 = Σ（基本分数 + 一包化フラグ × 加算分数）
```

- 基本分数・加算分数はstaff.htmlの設定から変更可能
- localStorageに保存されるため端末ごとに保持
- waitingステータスの患者のみ計算対象

## ステータス設計

| status | 意味 |
|--------|------|
| `waiting` | 待ち中（待ち人数・時間の計算対象） |
| `calling` | 呼び出し中（計算対象外・レコードは保持） |
