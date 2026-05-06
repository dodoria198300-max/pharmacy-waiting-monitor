# 薬局待ち時間モニター

薬局待合室向けの待ち時間表示システムです。スタッフの操作を最小限に抑えつつ、患者への待ち状況のリアルタイム表示と蓄積データを活用した業務分析を実現します。

---

## 画面構成

| ファイル | 用途 | 想定端末 |
|---------|------|---------|
| `staff.html` | 受付番号発行・呼び出し・帰宅操作 | カウンター内PC・タブレット |
| `monitor.html` | 待ち人数・目安時間の表示 | 待合室TV・タブレット |
| `analytics.html` | 混雑・待ち時間の分析 | 管理者PC |

---

## 主な機能

- 受付番号の自動採番（毎日001からリセット・実績は永続蓄積）
- 一包化フラグによる待ち時間加算
- Supabase Realtimeによる即時反映（WebSocket）
- 待ち人数・目安待ち時間の自動計算
- 途中帰宅の記録（status: left）
- 設定値（基本分数・加算分数）のDB保存・端末間共有
- GASによる自動バックアップ（日次・月次）

---

## 技術構成

| 項目 | 内容 |
|------|------|
| フロントエンド | 単一HTMLファイル（フレームワーク不使用） |
| データベース | Supabase PostgreSQL |
| リアルタイム通信 | Supabase Realtime（WebSocket） |
| ホスティング | Vercel（GitHubと連携・自動デプロイ） |
| バックアップ | Google Apps Script + スプレッドシート |

---

## セットアップ

### 1. Supabase テーブル作成

SQL Editorで以下を実行：

```sql
-- 待ちキューテーブル
create table waiting_queue (
  id          uuid primary key default gen_random_uuid(),
  ticket_no   integer not null,
  status      text not null default 'waiting',
  is_ippoka   boolean not null default false,
  created_at  timestamptz not null default now(),
  called_at   timestamptz
);

-- 設定テーブル
create table settings (
  key   text primary key,
  value text not null
);
insert into settings values ('base_min', '10'), ('add_min', '5');

-- Realtime有効化
alter publication supabase_realtime add table waiting_queue;

-- アクセス権限
alter table waiting_queue disable row level security;
alter table settings disable row level security;
grant all on table waiting_queue to anon;
grant insert, update, select on table settings to anon;
```

### 2. 接続情報の書き換え

`staff.html`・`monitor.html`・`analytics.html` 内の以下を自環境の値に書き換え：

```js
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-key';
```

### 3. Vercelにデプロイ

GitHubリポジトリをVercelに連携してデプロイ。mainブランチへのpushで自動デプロイされます。

### 4. GASバックアップの設定

`backup_gas_v2.js` をGoogle Apps Scriptに貼り付け、以下のトリガーを設定：

| 関数 | タイプ | 実行タイミング |
|------|--------|--------------|
| `backupWaitingQueue` | 日付ベース | 毎日・午前6時 |
| `backupMonthlySummary` | 月ベース | 毎月1日・午前6時 |

---

## ステータス設計

| status | 意味 | 待ち人数への影響 |
|--------|------|---------------|
| `waiting` | 待ち中 | カウント対象 |
| `calling` | 呼び出し中 | カウント対象外・レコード保持 |
| `left` | 途中帰宅 | カウント対象外・レコード保持 |

---

## 待ち時間の計算ロジック

```
目安待ち時間 = Σ（基本分数 + 一包化フラグ × 加算分数）
```

- 基本分数・加算分数はstaff.htmlの設定パネルから変更可能
- 設定値はSupabase settingsテーブルに保存（端末間で共有）
- waitingステータスの患者のみ計算対象

---

## バックアップ設計

| 種別 | 内容 | 保存期間 |
|------|------|---------|
| 日次バックアップ | 当日分の生データ | 直近30日（自動削除） |
| 月次サマリ | 前月の集計データ（件数・待ち時間・一包化割合等） | 永久保存 |

- Supabase本体のデータは削除しない限り永続保存
- GASの日次実行によりSupabaseの7日間非アクティブ停止も防止

---

## Supabase 無料プランの制限

| 項目 | 上限 | 備考 |
|------|------|------|
| DBサイズ | 500MB/プロジェクト | 1日100件でも年間数MB程度 |
| アクティブプロジェクト | 2つまで | k-trackerと合わせて2つ |
| 非アクティブ停止 | 7日間でPause | GASの日次バックアップで回避 |

---

## アクセスURL

| 画面 | URL |
|------|-----|
| 受付操作 | `https://pharmacy-waiting-monitor.vercel.app/staff.html` |
| 待合室モニター | `https://pharmacy-waiting-monitor.vercel.app/monitor.html` |
| 分析 | `https://pharmacy-waiting-monitor.vercel.app/analytics.html` |

---

## 将来の拡張候補

- 複数店舗の一括監視画面（overview.html）
- analytics.htmlのPhase 2強化
- k-trackerとのデータ連携
- SaaS展開（store_idによる店舗分離・認証追加）
