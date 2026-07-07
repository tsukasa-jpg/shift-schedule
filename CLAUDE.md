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
              //     ],
              //     birthMonth: number,             // 誕生月（1-12、0=未設定）
              //     birthDay: number,               // 誕生日（1-31、0=未設定）
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
   - `isBirthday()`（`birthMonth`/`birthDay` が当日と一致）→ 強制「誕」（最優先）
   - 社員の会社カレンダー休業日 → 強制「休」
   - `optSunRest` が ON かつ日曜・祝日 → 「休」
   - `optSatRest` が ON かつ土曜（workSat=false の場合）→ 「休」
   - `isRegularDayOff` / `isRequestedDayOff` → 「休」
   - `getValidShiftsForDate()` で日付制約（endBy/startFrom）を適用しシフトを絞り込み
   - 連勤制限（`maxConsecutive`）: Phase 1 後に連続勤務をスキャンして超過分を「休」に変更

2. **Phase 2**: 作業要件が未達の日について、休み中の該当スキル保持者を出勤に変更。定休日・希望休・社員の会社休日は上書きしない。割り当てる際も `getValidShiftsForDate()` で日付制約（endBy/startFrom）を適用する（Phase 1 と同様、無視しないこと）。

**重要な制約**: `誕`（誕生日休暇）はシフトスキルチップに表示しない・ローテーションから除外。`birthMonth`/`birthDay` が設定されていれば自動作成時に自動セットされる（スタッフ管理画面で設定）。それ以外の日は従来通り手動入力専用。

### 主要関数

| 関数 | 役割 |
|---|---|
| `renderTable()` | メインのシフト表を全列再生成 |
| `openPopup(td, si, di)` | セルクリック時のシフト選択ポップアップ |
| `buildHalfTableHTML(half)` | 印刷用テーブルHTML生成（'first'=1-15日 / 'second'=16-末日） |
| `halfBikoHtml()` | 半月印刷用の備考欄HTML文字列を生成（`buildHalfTableHTML` の末尾に連結して使う） |
| `doHalfPrint(html)` | `#half-print-area` に HTML を注入して `body.printing-halves` で印刷 |
| `getValidShiftsForDate(name, dk, skills)` | 日付制約を考慮した使用可能シフト一覧を返す |
| `getShiftHours(code)` | シフトコードの開始・終了時刻を `{start, end}` で返す |
| `onAuthChange(session)` | Supabase Auth セッション変化時に UI 表示を切り替える |
| `doLogin()` | ログインフォームの値で `signInWithPassword` を呼ぶ（async） |

### 印刷の仕組み

- **A4横・A3横**（通常月間印刷）: `printWithPageSize(size)` でページサイズを動的注入して `window.print()`。`#biko-section` は通常の DOM 要素として `@media print` CSS で表示される。
- **前半のみ・後半のみ・両面印刷（A4横）**: `printHalfMonth(half)` / `printBothHalves()` → `doHalfPrint()` で `#half-print-area` に前半・後半の独立テーブルを生成。`body.printing-halves` クラスで `#half-print-area` 以外を `display: none !important` にして印刷。備考欄は `halfBikoHtml()` で HTML 文字列として生成し、最後の `half-print-section` の末尾に連結して埋め込む（`#biko-section` DOM 要素は `body.printing-halves` で非表示になるため）。
- **背景色の印刷**: セルの背景色はブラウザのデフォルト設定では印刷されない。`print-color-adjust: exact` と `-webkit-print-color-adjust: exact` を `@media print` 内のセルセレクタに付与して強制出力している。

### 祝日・会社休日

- `HOLIDAYS` オブジェクト: 国民の祝日（2025〜2026年分ハードコード）
- `COMPANY_HOLIDAYS` Set: 会社年間休日（2026/4〜2027/3分ハードコード）
- 年度追加が必要な場合は両定数を直接編集する

### シフト定義の変更

`SHIFTS` 配列（ファイル上部）を編集する。`h` は拘束時間（実労働時間は `getPaidH(h)` で休憩控除済みに変換）。`誕`・`有給`・`休` はシフトスキルの選択対象外。

```javascript
{ code: '早③', label: '早番③', time: '9:00～15:30', h: 6.5, bg: '#f4ee9c', fg: '#585800' }
```

**重要**: `SKILL_SHIFTS = SHIFTS.slice(0, -1)` により **配列の最後の要素は必ずスキル選択対象外**になる。現在の最後の要素は `'休'`。新しいシフトを追加する場合は `'休'` の前に挿入すること。

### 土日カラーの CSS ルール（重要）

土日セルの色制御は以下の仕組みで機能している:

- **ヘッダー行**（`.dh-num`, `.dh-dow`, `.hn`）: `.d-sat`/`.d-sun` クラスを直接付与 → 青/赤で着色
- **出勤シフトセル**（メイン表）: `const cellDcCls = shift ? '' : dc` により `.d-sat`/`.d-sun` クラスを**付与しない**。インラインスタイルでシフト色を設定
- **出勤シフトセル**（半月印刷テーブル）: `const dcCls = s ? '' : dc` により同様にクラスを**付与しない**
- **非出勤セル**（休み・空白など）: `.d-sat`/`.d-sun` クラスが付与されるが、`.shift-cell.d-sat { background-color: #fff !important; }` および `.half-tbl td.d-sat:not(.hn) { background-color: #fff !important; }` で白背景に上書き

この仕組みにより「日付・曜日ヘッダーは土日色あり、出勤セルはシフト色、それ以外は白」が実現されている。

### Supabase 設定

`SUPABASE_URL` と `SUPABASE_ANON_KEY` はファイル上部にハードコード済み。テーブル名は `shift_app_data`、レコードIDは `'main'`（全データを1レコードで管理）。

### 認証（Supabase Auth）

**アーキテクチャ**: 閲覧（SELECT）は anonymous でも可、書き込み（INSERT/UPDATE/DELETE）は authenticated のみ許可する RLS ポリシーを Supabase 側に設定済み。

**状態変数**: `currentUser`（null = 未ログイン、User オブジェクト = ログイン中）

**UI 制御**:
- `#editor-actions` div: ログイン時のみ `display:flex`（作業要件設定・自動作成・スタッフ管理・有給管理ボタンを内包）
- `#bikoText` textarea: ログイン時のみ `readOnly = false`
- シフトセルの click / contextmenu: `currentUser` が null なら即 `return`（UI は動作するが何も起きない）

**セッション維持**: Supabase JS v2 がセッションを localStorage に自動保存。ページリロード時は `getSession()` で復元し `onAuthChange()` を呼ぶ。

```javascript
// セッション状態が変わるたびに呼ばれる（ログイン・ログアウト・リロード時）
function onAuthChange(session) { ... }

// signInWithPassword を呼んでログインする
async function doLogin() { ... }

// signOut を呼ぶ（onAuthStateChange 経由で onAuthChange が呼ばれる）
logoutBtn.addEventListener('click', async () => { ... })
```

**ユーザー管理**: Supabase ダッシュボード → Authentication → Users でユーザーを追加する（アプリ内にサインアップ画面はない）。
