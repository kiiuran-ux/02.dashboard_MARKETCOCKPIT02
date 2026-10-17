# mama_indicators.html メモキー仕様書

> **運用ルール**: HTMLに変更を加えた際は必ずこのファイルも同時に更新すること。
> クロは mama_indicators.html のメモ更新前に必ずこのファイルを読んでから作業すること。

---

## ⚠️ 最重要警告：2種類のメモ更新の仕組みが混在している

このページには構造の異なる2種類のメモ更新の仕組みが同居している。
**どちらの仕組みに属するかを必ず確認してから作業すること。**

---

### 仕組みA：`data-val` 属性（読み込み専用・一方向）

**対象キー：** `mama_val_*` 系10キー（VAL_MAP定数で定義）

**動作：** ページロード時に `loadPortfolioValues()` が `/api/memos_mama` を一度だけ fetch し、
`data-val` 属性を持つ DOM 要素の `textContent` に直接代入するだけ。
ブラウザ側からの書き戻しは発生しない（ユーザー操作では上書きされない）。

**クロからの更新：** GET → マージ → POST → ハードリロード確認 で安全に更新できる。

---

### 仕組みB：`memo-wrap[data-key]` 属性（双方向・memosCache経由）

**対象キー：** `mama_monthly_*` 4キー、`mama_sp500` 等銘柄メモ5キー、`mama_boj_rate`、`mama_total`

**動作：** ページロード時に `loadMemosFromServer()` が全データを `memosCache`（JSオブジェクト）
に読み込む。以後このページが「信じている唯一の真実」は **ロード時点の memosCache** になる。
ユーザーがどこか1つのメモ欄を編集してフォーカスを外す（blur）と、`persistMemos()` が発火し、
「ページが起動時に読み込んだ古い memosCache 全体」を `/api/memos_mama` に丸ごと上書き POST する。

---

### ⚠️ 上書き消滅のリスク（2026-06-20 発生事故の根本原因）

クロが API を直接 GET → マージ → POST してこのグループのキーを更新しても、
**その後ブラウザ側でユーザーがどこか別のメモ欄を1つでも編集・blur すると、
クロが書いた内容は「ページロード時点の古い memosCache」で丸ごと上書きされて消える。**

これが 2026-06-20 に発生した事故（米CPI・米PCE・米雇用統計の内容が消えた）の根本原因。

#### 必須対処手順

仕組みB のキーをクロが更新する場合は、以下のいずれかを必ず行うこと：

**(a) 推奨** — API直叩き後に必ずブラウザをリロードする（`location.reload()`）。
　これにより memosCache がクロの書いた最新内容で再同期される。

**(b) 代替** — ブラウザ JS 内で `memosCache` 変数そのものを更新してから
　`persistMemos()` を呼ぶ（API直叩きを避ける）。

**(c) 確認必須** — 更新後は必ずハードリロードして全項目（評価額系・月次指標系・
　銘柄メモ系）が保持されているかを目視確認する。確認を怠らない。

---

## メモ関連 主要関数一覧

コード上の関数名が変わった場合はここを最初に更新すること。

| 関数名 | 役割 | 変更禁止度 |
|---|---|---|
| `loadPortfolioValues()` | 起動時に `/api/memos_mama` を独立 fetch し、`data-val` 要素に直接代入する（仕組みA） | ⛔ ロジック変更禁止 |
| `loadMemosFromServer(cb)` | 起動時に `/api/memos_mama` を fetch して `memosCache` に格納し callback を呼ぶ（仕組みB） | ⛔ ロジック変更禁止 |
| `buildMemoWidget(container, key, opts?)` | 1つのメモ欄をDOMに生成・注入する汎用部品。`opts.useInput=true` で `<input>` 1行モード、`opts.fallback` で空欄時のプレースホルダを指定 | ⚠️ UI変更時のみ編集可 |
| `initMemos()` | `.memo-wrap[data-key]` を走査して `buildMemoWidget` を呼ぶ。`data-inline` 属性があれば `useInput:true` + boj-rate用フォールバック、`data-badge` 属性があれば `useInput:true` + `¥--` フォールバックを渡す | ⛔ ロジック変更禁止 |
| `persistMemos()` | `memosCache` 全体を `/api/memos_mama` に POST する（300ms デバウンス） | ⛔ 変更禁止 |
| `refreshAllTimestamps()` | 全タイムスタンプ表示を再描画する（1分ごとに setInterval で呼ばれる） | ✅ 変更可 |
| `updateTimeEl(key)` | `#memo-time-<key>` 要素にフォーマット済みタイムスタンプを反映する | ✅ 変更可 |
| `tsKey(key)` | `'memo_ts_' + key` を返す（タイムスタンプキー生成） | ⛔ 変更禁止 |
| `fmtTimestamp(iso)` | ISO文字列を `M/D HH:MM` 形式にフォーマットする | ✅ 変更可 |
| `ageClass(iso)` | 経過時間から `fresh` / `recent` / `old` を返す | ✅ 変更可 |

> `initWeeklyMemo()` と `initTotalBadge()` は **2026-06-20 のリファクタリングで削除**。
> `mama_boj_rate` と `mama_total` は `buildMemoWidget` に統合され、通常の `memo-wrap[data-key]` パターンで処理される。

**呼び出し順序（起動時）：**
```
initPage()
  ├─ loadPortfolioValues()       ← 独立fetch（仕組みA、loadMemosFromServerと並行）
  └─ loadMemosFromServer(cb)
       ├─ initMemos()            ← .memo-wrap[data-key] 全件にウィジェット注入
       │                            （mama_boj_rate: data-inline → input+fallback付き）
       │                            （mama_total: data-badge → input+¥--付き）
       ├─ initTooltips()
       ├─ setInterval(refreshAllTimestamps, 60000)
       └─ fetchAll()             ← ライブ価格取得開始
```

> ⚠️ `loadPortfolioValues()` と `loadMemosFromServer()` は**独立した並行 fetch** であり、
> どちらが先に完了するかは保証されない。起動直後は評価額が表示される前に memosCache が
> 読み込まれることも、その逆もある。

---

## タイムスタンプキーのプレフィックス

| ダッシュボード | プレフィックス | 例 |
|---|---|---|
| 旦那用（investment_indicators.html） | `memo_updated_` | `memo_updated_memo_monthly_FOMC結果` |
| 奥様用（mama_indicators.html） | `memo_ts_` | `memo_ts_mama_monthly_fomc` |

**両者は形式が異なる。コードを流用するときは必ず確認すること。**

---

## メモキー一覧

### 仕組みA：`data-val` 系（ポートフォリオ評価額）

`loadPortfolioValues()` が処理。ユーザーは直接編集不可。

| `data-val` 属性値 | API キー | 表示場所 |
|---|---|---|
| `sp500_hyoka` | `mama_val_sp500_hyoka` | 新NISA成長枠 S&P500 評価額 |
| `sp500_son` | `mama_val_sp500_son` | 新NISA成長枠 S&P500 含み損益 |
| `senshin_hyoka` | `mama_val_senshin_hyoka` | 特定口座 eMAXIS先進国 評価額 |
| `nissei_hyoka` | `mama_val_nissei_hyoka` | 特定口座 ニッセイ外国株 評価額 |
| `tokutei_total` | `mama_val_tokutei_total` | 特定口座 合計評価額＋含み益 |
| `tokutei_zei` | `mama_val_tokutei_zei` | 特定口座 税金概算 |
| `oldnisa_senshin_hyoka` | `mama_val_oldnisa_senshin_hyoka` | 旧NISA① eMAXIS先進国 評価額 |
| `oldnisa_senshin_son` | `mama_val_oldnisa_senshin_son` | 旧NISA① 含み損益 |
| `oldnisa_orcan_hyoka` | `mama_val_oldnisa_orcan_hyoka` | 旧NISA② オルカン 評価額 |
| `oldnisa_orcan_son` | `mama_val_oldnisa_orcan_son` | 旧NISA② 含み損益 |

---

### 資産サマリー（CSV取込メタ・2026-10-17追加）

ヘッダーの「総評価額」付近と特定口座カードに表示される、CSV取込日・前回比の管理キー。`renderAssetMeta()` が `loadMemosFromServer` のコールバック内で描画する（表示専用。カード上での直接編集は不可）。

| キー | 用途 | 更新方法 | フォーマット |
|---|---|---|---|
| `mama_val_updated` | CSV取込日（表示用。ヘッダーに「CSV：10/17」のように表示。7日超でグレー＋⚠️） | CSV取込時に手動更新 | 文字列 `'YYYY-MM-DD'`（未設定なら「CSV：--」） |
| `mama_total_prev` | 前回CSVの総評価額（`mama_total` との前回比の基準） | 更新時に旧 `mama_total` を退避 | 円・カンマ区切り文字列（初回は空） |
| `mama_val_tokutei_total_prev` | 前回CSVの特定口座合計（`mama_val_tokutei_total` との前回比の基準） | 更新時に旧 `mama_val_tokutei_total` を退避 | 円・カンマ区切り文字列（初回は空） |
| `mama_val_prev_date` | 前回CSVの取込日（前回比の「（前回 10/15）」表示に使用。両方の前回比で共通） | 更新時に旧 `mama_val_updated` を退避 | 文字列 `'YYYY-MM-DD'`（初回は空） |

**前回比の運用（2本立てスナップショット・旦那用の `val_*_prev` と同じ考え方）：**
CSV取込で新しい額に更新するときは、①現在キー（`mama_total`・`mama_val_tokutei_total`）の値を対応する `_prev` キーへ退避 → ②新しい額を現在キーへ書き込む → ③ `mama_val_updated` を当日日付に（このとき旧 `mama_val_updated` を `mama_val_prev_date` へ退避）。
前回比の表示値は `¥`・`約`・カンマ等の記号を除いて数値化してから差分計算する（`parseYen()`）。旧NISA売却中は売却分が現金化されて評価額から外れるため、前回比がマイナスに見える場合がある旨をヘッダー直下に注記表示している（`.prev-note-bar`）。

---

### 仕組みB：`memo-wrap[data-key]` 系（テキストメモ）

`initMemos()` が処理。ユーザーが編集 → blur → `persistMemos()` が発火して全体上書き。

#### 月次経済指標（4件）

| `data-key` 属性値（= API キー） | 表示場所 |
|---|---|
| `mama_monthly_cpi` | 米CPI |
| `mama_monthly_pce` | 米PCE |
| `mama_monthly_koyo` | 米雇用統計 |
| `mama_monthly_fomc` | FOMC・日銀結果 |

#### 次回発表予定（2026-10-17追加・旦那用 `renderMonthlyIndicators()` と同じ仕組み）

「📅 次回発表予定」グリッド（`monthlyIndicators` 配列）はメモ管理下。配列の `nextDate` はメモ未設定時のフォールバック値としてのみ機能する。`renderMonthlyIndicators()` は `loadMemosFromServer` のコールバック内で再実行され、`memosCache` の値があればそちらを優先表示する。対応するタイムスタンプ（`memo_ts_<キー>`）の古さから `.monthly-item` に `fresh`/`recent`/`old` の枠線色が自動で付く（旦那用と同じ `ageClass()` 基準）。

| 指標 | 実際のAPIキー | フォールバック値（2026-10-17時点シード） |
|---|---|---|
| 米雇用統計 | `mama_monthly_indicator_雇用統計` | `次回 11/6（10月分）` |
| 米CPI | `mama_monthly_indicator_CPI` | `次回 11/10（10月分）` |
| 米PCE | `mama_monthly_indicator_PCE` | `次回 10月末ごろ（9月分）` |
| 米PPI | `mama_monthly_indicator_PPI` | `次回 11月中旬（10月分）` |

> 対応するタイムスタンプキーは `mama_monthly_*` 系と同様 `memo_ts_<メモキー>` 形式（例：`memo_ts_mama_monthly_indicator_CPI`）。

#### 銘柄カードメモ（5件）

各カードに `.memo-time` タイムスタンプ要素（`id="memo-time-<key>"`）が対応する。

| `data-key` 属性値（= API キー） | タイムスタンプ要素ID | 表示場所 |
|---|---|---|
| `mama_sp500` | `memo-time-mama_sp500` | 新NISA成長枠 S&P500カード |
| `mama_orcan_nisa` | `memo-time-mama_orcan_nisa` | 新NISA成長枠 オルカンカード |
| `mama_tokutei` | `memo-time-mama_tokutei` | 特定口座カード |
| `mama_oldnisa_senshin` | `memo-time-mama_oldnisa_senshin` | 旧NISA① eMAXIS先進国カード |
| `mama_oldnisa_orcan` | `memo-time-mama_oldnisa_orcan` | 旧NISA② オルカンカード |

---

### 仕組みB（特殊バリアント）：インライン編集

通常の `textarea` ではなく `<input>` 1行モードを使うキー。`initMemos()` が `data-inline` / `data-badge` 属性を検出して自動適用する。

| API キー | DOM 要素 | 属性 | 編集方法 | フォールバック値 |
|---|---|---|---|---|
| `mama_boj_rate` | `.memo-wrap[data-key="mama_boj_rate"]` | `data-inline="1"` | クリック → 1行 input（Enter/blur で確定、Esc でキャンセル） | `⚠️ 要確認（クリックして更新）` |
| `mama_total` | `.memo-wrap[data-key="mama_total"]`（ヘッダーバッジ内） | `data-badge="1"` | クリック → 1行 input（Enter/blur で確定、Esc でキャンセル） | `¥--`（空欄時） |

> どちらも `buildMemoWidget(container, key, opts)` の共通ロジックで動作する（blur → commit → persistMemos）。
> `mama_boj_rate` はタイムスタンプキー `memo_ts_mama_boj_rate` も保存される。
> `mama_total` はタイムスタンプキー `memo_ts_mama_total` も保存されるが、`updateTimeEl` 対応の DOM 要素はない（updateTimeEl は早期リターン）。

---

### タイムスタンプキー一覧

タイムスタンプキーは `'memo_ts_' + メモキー` の形式。値は ISO 8601 文字列。

| タイムスタンプキー | 対応するメモキー |
|---|---|
| `memo_ts_mama_monthly_cpi` | `mama_monthly_cpi` |
| `memo_ts_mama_monthly_pce` | `mama_monthly_pce` |
| `memo_ts_mama_monthly_koyo` | `mama_monthly_koyo` |
| `memo_ts_mama_monthly_fomc` | `mama_monthly_fomc` |
| `memo_ts_mama_monthly_indicator_雇用統計` | `mama_monthly_indicator_雇用統計` |
| `memo_ts_mama_monthly_indicator_CPI` | `mama_monthly_indicator_CPI` |
| `memo_ts_mama_monthly_indicator_PCE` | `mama_monthly_indicator_PCE` |
| `memo_ts_mama_monthly_indicator_PPI` | `mama_monthly_indicator_PPI` |
| `memo_ts_mama_sp500` | `mama_sp500` |
| `memo_ts_mama_orcan_nisa` | `mama_orcan_nisa` |
| `memo_ts_mama_tokutei` | `mama_tokutei` |
| `memo_ts_mama_oldnisa_senshin` | `mama_oldnisa_senshin` |
| `memo_ts_mama_oldnisa_orcan` | `mama_oldnisa_orcan` |
| `memo_ts_mama_boj_rate` | `mama_boj_rate` |
| `memo_ts_mama_total` | `mama_total` |
| `memo_ts_mama_val_updated` | `mama_val_updated` |
| `memo_ts_mama_total_prev` | `mama_total_prev` |
| `memo_ts_mama_val_tokutei_total_prev` | `mama_val_tokutei_total_prev` |
| `memo_ts_mama_val_prev_date` | `mama_val_prev_date` |

> 下3つは資産サマリーのメタ情報キー。タイムスタンプ自体はUI表示に使用していないが、サーバー側自動タイムスタンプ（下記）の対象にはなる。

---

### サーバー側自動タイムスタンプ（2026-10-17実装）

`server.py` の `_save_json()` が `.memos_mama.json`（`MEMOS_MAMA_FILE`）への書き込み時に `_auto_timestamp_memos()` を経由するようになった（旦那用 `MEMOS_FILE` と同じ関数を共用・プレフィックスのみ切り替え）。
POSTされた `mama_*` キー（`memo_ts_*` 自身は対象外）の値が、保存済みの旧値と異なる場合、対応する `memo_ts_<キー>` を**サーバー側の現在時刻（UTC ISO8601）で強制上書き**する。

- 旦那用との違い：旦那用はタイムスタンプキーが `memo_updated_` プレフィックス・対象キーが `memo_` プレフィックス。奥様用はタイムスタンプキーが `memo_ts_` プレフィックス・対象キーが `mama_` プレフィックス（本仕様書冒頭「タイムスタンプキーのプレフィックス」参照）。
- **⚠️ ただしブラウザ側の上書き消滅リスク（本仕様書冒頭参照）は解消しない。** サーバーは「内容が変わったPOSTに対して正しい時刻を付与する」だけで、`persistMemos()` が古い `memosCache` ごとPOSTしてしまう問題自体への対策ではない。クロがAPI直叩きで更新する場合は、引き続き「POST後は必ず `location.reload()`」のルールを守ること。
- 適用範囲：`MEMOS_MAMA_FILE`（奥様用ダッシュボード）のみ。`MEMOS_SHARED_FILE` は対象外。

---

## API 仕様

| 項目 | 内容 |
|---|---|
| エンドポイント | `GET / POST http://localhost:8765/api/memos_mama` |
| データ形式 | JSON フラットオブジェクト `{ "キー": "値", ... }` |
| 保存先 | `/Users/shu/開発/.memos_mama.json` |
| タイムスタンプキー | `memo_ts_<メモキー>`（値は ISO 8601 文字列） |
| POST 方式 | オブジェクト全体を毎回上書き（部分更新ではない） |
| `persistMemos()` のデバウンス | 300ms（最後の編集から300ms後にPOST） |

---

## クロからの更新手順

### fetch() について

奥様チャット側では **`fetch()` を直接使用できる**（`[BLOCKED]` エラーは発生しない）。
旦那用仕様書にある XMLHttpRequest 同期モードは奥様側では不要。

---

### 仕組みA：`mama_val_*` キー（評価額）の更新

ページの memosCache に影響されにくいが、**更新後のリロード確認を推奨**。

```javascript
// Step 1: 現在の内容を取得
const res = await fetch('/api/memos_mama');
const data = await res.json();

// Step 2: 更新するキーだけ差し替え
Object.assign(data, {
  "mama_val_sp500_hyoka": "¥2,500,000",
  "mama_val_sp500_son":   "+¥100,000（+4.2%）",
  "memo_ts_mama_val_sp500_hyoka": new Date().toISOString(),
  // 更新するキーのみ列挙
});

// Step 3: POST
const postRes = await fetch('/api/memos_mama', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(data)
});
console.log('status:', postRes.status); // 200 なら成功

// Step 4: ページをリロードして反映確認
location.reload();
```

---

### 仕組みB：`mama_monthly_*` / 銘柄メモ等の更新

**⚠️ 上書き消滅リスクあり。必ずリロードまで行うこと。**

```javascript
// Step 1: 現在の内容を取得
const res = await fetch('/api/memos_mama');
const data = await res.json();
console.log('keys:', Object.keys(data).join(', ')); // 既存キー確認

// Step 2: 更新するキーだけ差し替え
const now = new Date().toISOString();
Object.assign(data, {
  "mama_monthly_fomc":         "【6/18確定】据え置き。ドットプロットはタカ派。",
  "memo_ts_mama_monthly_fomc": now,
  // 更新するキーのみ列挙
});

// Step 3: POST
const postRes = await fetch('/api/memos_mama', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(data)
});
console.log('status:', postRes.status); // 200 なら成功

// Step 4: ★必須★ ブラウザをリロードして memosCache を再同期
location.reload();
```

---

### Step 5：保存確認

リロード後に目視で以下を確認する：

- 月次指標メモが正しく表示されているか
- 銘柄カードのメモが消えていないか
- 評価額（`data-val`）が正しく表示されているか
- タイムスタンプが今日の日付になっているか

---

## よくあるエラーと対処

| エラー・現象 | 原因 | 対処 |
|---|---|---|
| クロが書いた内容が消えた | ブラウザ側でメモ欄を編集→blur → 古い memosCache で上書きされた | **次回からは必ずリロードする。** 今回消えた内容は再度POSTする |
| `location.reload()` 後もメモが消えている | POST が失敗していた（status != 200） | `postRes.status` を確認。サーバー起動中か `launchctl list | grep investment` で確認 |
| `fetch` で `Failed to fetch` エラー | サーバー（port 8765）が停止している | `launchctl start com.shu.investment-server` で起動 |
| POST したのに値が変わらない | GET→修正→POST の途中でキーを間違えた | `Object.keys(data)` でキー名を確認してから再送 |
| 評価額（`¥--`）が表示のまま | `loadPortfolioValues()` の fetch 失敗 or キー不一致 | VAL_MAP の `data-val` 属性値と API キーの対応を確認 |
| `mama_boj_rate` が消えた | `mama_boj_rate` は 仕組みB なのに POST後リロードしなかった | 必ず POST → `location.reload()` の順で行う |

---

## カウントダウン優先度ルール（2026-10-17変更・旦那用に統一）

`updateCountdowns()` が `#countdown-grid` に出すカードの優先度・件数ルール。旦那用 `investment_indicators.html` の実装に合わせた。

| 残日数 | クラス（カード・cd-days共通） | 優先度バッジ | 表示スタイル |
|---|---|---|---|
| 1〜14日・当日 | `urgent` | `URGENT`（赤） | 赤強調 |
| 15〜60日 | `soon` | `SOON`（オレンジ） | オレンジ |
| 61日以上 | `future` | `FUTURE`（グレー） | 薄く表示 |
| 経過後 | `done` | なし | グレー・小フォント |

表示件数のルール（旦那用と同一）：

| category | 表示件数 | 条件 |
|---|---|---|
| `task`（奥様のアクションが必要なもの） | 件数制限なし・常に表示 | ただし**90日以内**のイベントのみ |
| `market`（定例の市場イベント） | 直近から最大4件 | 従来どおり |

> 並び順は常に日付順。「📋 全予定を見る」ポップアップ（`openAllEventsModal()`）はこのルールの対象外で、件数制限なし・未来のイベント全件を表示する（変更なし）。
> 優先度バッジは `.cd-priority-badge`（カード右上）・クラス名 `badge-urgent`/`badge-soon`/`badge-future`。既存の `.cd-badge-dyn`（イベント説明文）とは別物で、両方同時に表示される。

---

## 変更履歴

| 日付 | 変更内容 | APIキーへの影響 |
|---|---|---|
| 2026-06-17 | `mama_boj_rate` キー追加。日銀政策金利表示を `mama_weekly` → `mama_boj_rate` に変更 | `mama_weekly` 廃止・`mama_boj_rate` 新設 |
| 2026-06-20 | 本仕様書を新規作成（2種類のメモ仕組みの明文化・上書き事故の記録） | — |
| 2026-06-20 | `#boj-rate` のHTML初期値とJSフォールバック値を `⚠️ 要確認（クリックして更新）` に変更（古い数値固定化の防止・案C適用） | なし |
| 2026-06-20 | `initWeeklyMemo()` と `initTotalBadge()` を削除し、`buildMemoWidget(container, key, opts)` に統合。`mama_boj_rate` → `data-inline` 属性付き `memo-wrap`、`mama_total` → `data-badge` 属性付き `memo-wrap`（ヘッダーバッジ内）に移行。Tab キーバグ・prompt() ダイアログ廃止 | なし |
| 2026-10-17 | ①SYMBOLS `lv-jgb` の `yf: '^TNX'` フォールバックを削除（日本長期金利に米10年債の値が代用表示されるバグの修正）。stooq取得失敗時は「取得失敗」と表示し米国債で代用しない。②`countdownEvents` を全面更新（旧NISA先進国株式の3分割売却スケジュール・10月FOMC/日銀・米中間選挙・12月FOMC/SQ/日銀に差し替え）。③カウントダウン優先度ルールを旦那用に統一（14日/60日の閾値・`cd-priority-badge`・task90日以内無制限／market最大4件）。④「次回発表予定」（`monthlyIndicators`）をメモ管理下へ移行（旦那用 `renderMonthlyIndicators()` と同パターン）。⑤資産サマリーにCSV取込日・前回比表示を追加（`renderAssetMeta()`）。⑥`server.py` に奥様用の自動タイムスタンプ（`memo_ts_` プレフィックス）を追加。⑦奥様用カード内容を更新（旧NISA先進国株式カードの3分割売却状況・2026年スケジュール行・見込み金額への注記）。⑧評価額の初期値プレースホルダを全て `¥--` に変更（古い数値がメモ未読み込み時に本物に見えないよう対策） | **`mama_monthly_indicator_雇用統計`・`mama_monthly_indicator_CPI`・`mama_monthly_indicator_PCE`・`mama_monthly_indicator_PPI`・`mama_val_updated`・`mama_total_prev`・`mama_val_tokutei_total_prev`・`mama_val_prev_date` の8キーを新規追加** |
| 2026-10-17 | `server.py` の `/api/price`（`_serve_price()`）のキャッシュ汚染バグを追加修正。①の `yf` フォールバック削除だけでは解消しなかった原因として、stooq失敗時にyfで代用取得した値を従来 `PRICE_CACHE[sym]` に保存していたため、一度でも代用取得が発生した記号は以後stooqが正常化しても代用値のキャッシュが残り続けていたことが判明（`10yjpy.b` に `^TNX` の値 5.098% が出続けていた真因）。代用値は `PRICE_CACHE` へ保存しない仕様に変更（`fallback: true` を付与して返すのみ）。stooq失敗かつyf未指定／yfも失敗の場合は従来通り502エラーを返す。奥様用は `yf` 指定自体を削除済みのため常に502→「取得失敗」表示、旦那用は `yf:'^TNX'` を残しているため、stooqが失敗した場合は引き続き米10年債の値で代用表示される（キャッシュ汚染のみ解消・代用自体の是非は投資信託チャット側で別途検討） | なし |

---

## この仕様書について（作成意図と旦那用との差分）

### 特に重点を置いた点

**① 仕組みAと仕組みBの明確な分離（冒頭に配置）**

mama_indicators.html には「読み込み専用の data-val 系（仕組みA）」と
「双方向書き換えが起きる memo-wrap 系（仕組みB）」の2系統が混在している。
この区別を理解しないまま作業すると上書き事故が再発するため、
仕様書の最冒頭に警告として記載した。

**② 上書き消滅リスクの記録（2026-06-20 事故の根本原因）**

クロが API を直叩きで更新した後、ユーザーが別のメモ欄を編集・blur するだけで
`persistMemos()` が発火し、ページロード時点の古い `memosCache` 全体で
サーバーが上書きされる。この事故を防ぐために「API POST 後は必ずリロード」を
必須手順として明記した。

**③ 旦那用仕様書との実装差分を明示**

コードを流用する際の混乱を防ぐため、以下の差異を本文中に都度注記した。

| 項目 | 旦那用（investment_indicators.html） | 奥様用（mama_indicators.html） |
|---|---|---|
| タイムスタンプ prefix | `memo_updated_` | `memo_ts_` |
| fetch 方式 | XMLHttpRequest 同期モード必須（fetch が BLOCKED） | 通常の `fetch()` で問題なし |
| localStorage | あり（`patchStorage` フックで書き込みを検知） | **なし** |
| memo 入力要素 | `<input type="text">` | `<textarea rows="2">` |
| initMemos の注入対象 | `.live-val.stale` と `.r-label` を走査 | `.memo-wrap[data-key]` を走査 |
| 特殊キー | なし | `mama_boj_rate`（`data-inline` 付き memo-wrap）/ `mama_total`（`data-badge` 付き memo-wrap、ヘッダーバッジ内） |
| 評価額キー | なし | `mama_val_*` 10キー（仕組みA・loadPortfolioValues が担当） |
| slugify 関数 | あり（キー名を動的生成） | **なし**（キー名は HTML の `data-key` に直書き） |
