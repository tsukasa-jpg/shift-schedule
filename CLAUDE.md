# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## デプロイ

```bash
# ローカル編集後、GitHub Pages に反映
cd C:\Users\user\shift-schedule-repo
cp C:\Users\user\workspace\shift_schedule.html shift_schedule.html
git add shift_schedule.html
git commit -m "変更内容の説明"
git push origin main
```

公開URL: https://tsukasa-jpg.github.io/shift-schedule/shift_schedule.html  
（push 後、GitHub Pages に反映されるまで1〜2分かかる）

編集作業は `C:\Users\user\workspace\shift_schedule.html` で行い、完成後にこのリポジトリへコピーして push する運用。

## アーキテクチャ概要

**単一ファイル構成**：`shift_schedule.html` 1ファイルに HTML／CSS／JavaScript が全て含まれる。サーバー不要でブラウザだけで動作。

### データ構造（グローバル変数）

```javascript
staffList     // string[]  スタッフ名一覧（表示順）
shiftData     // { periodKey: { staffName: { "YYYY-MM-DD": shiftCode } } }
staffSkills   // { staffName: { shifts: string[], workSat: bool, workSkills: string[] } }
workSkillReqs // { wsName: { dow(0-6): required_count } }
bikoNotes     // { periodKey: string }
paidLeaveData // { staffName: { "YYYY": { granted: N } } }  YYYY = 年度開始年
```

### 期間の仕組み（重要）

表示単位は「1ヶ月」ではなく **16日〜翌月15日** の固定期間。

```javascript
function getPeriodDays() {
  // 当月16日〜末日 + 翌月1日〜15日 を返す
}
function periodKey() { return `${currentYear}-${currentMonth}`; }
// currentYear/currentMonth は期間の「前半月」を指す
```

### データ保存・同期フロー

```
シフト変更
  → setShift() → saveData()
    → _saveLocal()  → localStorage['shiftAppV4']（即時・オフライン対応）
    → saveDataToSupabase()  → Supabase DB テーブル shift_app_data（id='main'）
```

起動時は `loadDataFromSupabase()` で Supabase の `updated_at` とローカルを比較し、新しい方を使う。複数PC間の同期はこの仕組みで実現。

### 主要関数

| 関数 | 役割 |
|---|---|
| `renderTable()` | メインのシフト表を描画（全列再生成） |
| `openPopup()` | セルクリック時のシフト選択ポップアップ |
| `buildHalfTableHTML(half)` | 印刷用テーブルHTML生成（'first'=1-15日 / 'second'=16-末日） |
| `printBothHalves()` | 両面印刷（A4縦2ページ） |
| `changeMonth(delta)` | 月ナビゲーション |

### 印刷の仕組み

通常印刷（A4横・A3横）は `printWithPageSize(size)` でページサイズを動的に注入して `window.print()`。

半月印刷は `#half-print-area` div に独立したテーブルを生成し、`body.printing-halves` クラスで既存コンテンツを非表示にして印刷。これにより colspan の問題を回避。

### シフト定義の変更

`SHIFTS` 配列（ファイル上部）を編集する。`h` は拘束時間（実労働時間は `getPaidH(h)` で休憩控除済みに変換される）。

```javascript
{ code: '早③', label: '早番③', time: '9:00～15:30', h: 6.5, bg: '#f4ee9c', fg: '#585800' }
```

### Supabase 設定

`SUPABASE_URL` と `SUPABASE_ANON_KEY` はファイル上部にハードコード済み。テーブル名は `shift_app_data`、レコードIDは `'main'`（全データを1レコードで管理）。
