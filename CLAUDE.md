# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## デプロイ

```bash
# 編集後、GitHub Pages に反映（このリポジトリ内のファイルを直接編集して push）
git add shift_schedule.html
git commit -m "変更内容の説明"
git push origin main
```

公開URL: https://tsukasa-jpg.github.io/shift-schedule/shift_schedule.html  
（push 後、GitHub Pages に反映されるまで1〜2分かかる）

`C:\Users\user\workspace\shift_schedule.html` は古い作業コピー。**編集はこのリポジトリの `shift_schedule.html` を直接行う**。編集後に workspace 側へも同期する場合は `Copy-Item` で上書きコピーする。

## アーキテクチャ概要

**単一ファイル構成**：`shift_schedule.html` 1ファイルに HTML／CSS／JavaScript が全て含まれる。サーバー不要でブラウザのみで動作。ビルド手順なし。

### データ構造（グローバル変数）

```javascript
staffList     // string[]  スタッフ名一覧（表示順）
shiftData     // { periodKey: { staffName: { "YYYY-MM-DD": shiftCode } } }
staffSkills   // { staffName: {
              //     type: "パート" | "社員",
              //     shifts: string[],              // 担当可能シフトコード（誕・有給は除外）
              //     workSat: bool,                 // 土曜出勤フラグ
              //     workSkills: string[],          // 作業スキル一覧
              //     regularDaysOff: number[],      // 定休曜日（0=日〜6=土）毎週繰り返し
              //     requestedDaysOff: string[],    // 希望休（"YYYY-MM-DD"）特定日
              //     hourlyWage: number,            // 時給
              //     maxConsecutive: number,        // 最大連勤日数（0=制限なし）
              //     dateConstraints: [             // 日付別制約
              //       { dk: "YYYY-MM-DD", type: "endBy"|"startFrom"|"memo",
              //         value?: number,  // 時 (endBy/startFrom)
              //         text?: string   // (memo)
              //       }
              //     ]
              //   } }
workSkillReqs // { wsName: { dow(0-6): required_count } }
bikoNotes     // { periodKey: string }
paidLeaveData // { staffName: { "YYYY": { granted: N } } }  YYYY = 年度開始年
```

作業スキル一覧（`WORK_SKILLS`定数）: `['調整作業', 'プライス貼り', 'EC', '西友', '配送', '営業', '総務経理']`

### 期間の仕組み（重要）

表示単位は「1ヶ月」ではなく **16日〜翌月15日** の固定期間。

```javascript
function getPeriodDays()  // 当月16日〜末日 + 翌月1日〜15日 を返す
function periodKey()      // `${currentYear}-${pad2(currentMonth)}-16`
// currentYear/currentMonth は期間の「前半月」を指す
// 例: 2026年1月期 = "2026-01-16" → 2026/1/16〜2026/2/15
```

### データ保存・同期フロー

```
シフト変更
  → setShift() → saveData()
    → _saveLocal()        → localStorage['shiftAppV4']（即時・オフライン対応）
    → saveDataToSupabase() → Supabase DB テーブル shift_app_data（id='main'）
```

起動時は `loadDataFromSupabase()` で Supabase の `updated_at` とローカルを比較し、新しい方を使う。複数PC間のデータ同期はこの仕組みで実現。アプリ本体（HTML）は GitHub Pages で配布。

### 自動作成ロジック（`runAutoCreate()`）

Phase 1 → Phase 2 の2段階:

1. **Phase 1**: 各スタッフのシフトスキルをローテーションで割り当て。以下の優先順位で除外・制約を適用:
   - 社員の会社カレンダー休業日 → 強制「休」
   - `optSunRest` が ON かつ日曜・祝日 → 「休」
   - `optSatRest` が ON かつ土曜（workSat=false の場合）→ 「休」
   - `isRegularDayOff` / `isRequestedDayOff` → 「休」
   - `getValidShiftsForDate()` で日付制約（endBy/startFrom）を適用しシフトを絞り込み
   - 連勤制限（`maxConsecutive`）: Phase 1 後に連続勤務をスキャンして超過分を「休」に変更

2. **Phase 2**: 作業要件が未達の日について、休み中の該当スキル保持者を出勤に変更（定休日・社員の会社休日は上書きしない）

**重要な制約**: `誕`（誕生日休暇）はシフトスキルチップに表示しない・ローテーションから除外。手動入力専用。

### 主要関数

| 関数 | 役割 |
|---|---|
| `renderTable()` | メインのシフト表を全列再生成 |
| `openPopup(td, si, di)` | セルクリック時のシフト選択ポップアップ |
| `buildHalfTableHTML(half)` | 印刷用テーブルHTML生成（'first'=1-15日 / 'second'=16-末日） |
| `doHalfPrint(html)` | `#half-print-area` に HTML を注入して `body.printing-halves` で印刷 |
| `getValidShiftsForDate(name, dk, skills)` | 日付制約を考慮した使用可能シフト一覧を返す |
| `getShiftHours(code)` | シフトコードの開始・終了時刻を `{start, end}` で返す |

### 印刷の仕組み

- **A4横・A3横**（通常月間印刷）: `printWithPageSize(size)` でページサイズを動的注入して `window.print()`
- **両面印刷（A4横）**: `printBothHalves()` → `doHalfPrint()` で `#half-print-area` に前半・後半の独立テーブルを生成。`body.printing-halves` クラスで既存コンテンツを非表示にして印刷。`@page { size: A4 landscape; margin: 4mm; }` を動的設定。

### 祝日・会社休日

- `HOLIDAYS` オブジェクト: 国民の祝日（2025〜2026年分ハードコード）
- `COMPANY_HOLIDAYS` Set: 会社年間休日（2026/4〜2027/3分ハードコード）
- 年度追加が必要な場合は両定数を直接編集する

### シフト定義の変更

`SHIFTS` 配列（ファイル上部）を編集する。`h` は拘束時間（実労働時間は `getPaidH(h)` で休憩控除済みに変換）。`誕`・`有給`・`休` はシフトスキルの選択対象外。

```javascript
{ code: '早③', label: '早番③', time: '9:00～15:30', h: 6.5, bg: '#f4ee9c', fg: '#585800' }
```

### Supabase 設定

`SUPABASE_URL` と `SUPABASE_ANON_KEY` はファイル上部にハードコード済み。テーブル名は `shift_app_data`、レコードIDは `'main'`（全データを1レコードで管理）。
