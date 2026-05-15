# 薬局待ち時間モニター

薬局待合室向けの待ち時間表示システムです。スタッフの操作を最小限に抑えつつ、患者への待ち状況のリアルタイム表示と蓄積データを活用した業務分析を実現します。

---

## 画面構成

| ファイル | 用途 | 想定端末 |
|---------|------|---------|
| `staff.html` | 受付番号発行・投薬終了・帰宅操作 | カウンター内PC・タブレット |
| `monitor.html` | 待ち人数・目安時間・調剤中番号の表示 | 待合室TV・タブレット |
| `analytics.html` | 混雑・待ち時間の分析 | 管理者PC |

---

## 主な機能

### staff.html
- 受付番号の自動採番（毎日001からリセット・実績は永続蓄積）
- 一包化フラグによる待ち時間加算
- 一旦退出ボタン（受取待ちステータスへ移行）
- 投薬終了ボタン（会計完了として記録）
- 帰宅ボタン（途中帰宅・後日来局として記録）
- ソートタブ（待ち中 / 受取待ち / 投薬済 / 全て）
- 修正ボタン（誤操作時にステータスを選択して修正・確認ダイアログあり）
- 設定値（基本分数・加算分数）のDB保存・端末間共有

### monitor.html
- 待ち人数・目安待ち時間のリアルタイム表示
- 調剤中番号リスト（waitingの全件表示・投薬終了時に自動消滅）
- お受け取り可能エリア（readyの番号を表示・オレンジ系）
- 一包化バッジ表示
- 件数可変サイズ（1件:特大 / 2〜3件:大 / 4件以上:通常）
- Supabase Realtimeによる即時反映（リロード不要）
- FireTV Stick + Silkブラウザ対応

### analytics.html
- 日次サマリ（受付件数・一包化率・帰宅率・平均待ち時間）
- 時間帯別混雑グラフ
- 曜日別・日次・月次推移
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

-- CHECK制約（ステータス制限）
alter table waiting_queue
add constraint waiting_queue_status_check
check (status in ('waiting', 'calling', 'left', 'ready'));

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

| status | 意味 | 待ち人数への影響 | 追加時期 |
|--------|------|---------------|---------|
| `waiting` | 調剤中 | カウント対象 | 初期 |
| `calling` | 投薬終了（会計完了） | カウント対象外・レコード保持 | 初期 |
| `left` | 途中帰宅・後日来局 | カウント対象外・レコード保持 | 初期 |
| `ready` | 受取待ち（一旦退出中） | カウント対象外・モニター表示 | V1.6〜 |

### 患者フロー

```
パターン①（大多数）
受付 → waiting（調剤中）→ calling（投薬終了）

パターン②（一旦退出）
受付 → waiting（調剤中）→ ready（受取待ち）→ calling（投薬終了）

パターン③（帰宅）
受付 → waiting（調剤中）→ left（帰宅・後日）
```

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

## バージョン履歴

詳細は [CHANGELOG.md](./CHANGELOG.md) を参照。

| ファイル | 現在のバージョン |
|---------|---------------|
| staff.html | V1.8 |
| monitor.html | V2.1 |
| analytics.html | V2.0 |

---

## 将来の拡張候補

| 機能 | 概要 | 優先度 |
|------|------|--------|
| 複数店舗監視画面 | 全店舗の待ち状況を1画面に集約 | 中 |
| SaaS展開 | store_idによる店舗分離・認証追加・RLS有効化 | 中 |
| セルフ受付端末（kiosk.html） | 患者自身が受付操作 | 低 |
| k-trackerとの連携 | 処方内容×待ち時間の相関分析 | 低 |
| 当日戻り／後日来局の区別 | leftステータスをさらに細分化 | 低 |
