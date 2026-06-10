# DungeonStudy — CLAUDE.md

このドキュメントは DungeonStudy プロジェクトで作業する AI アシスタントへの指示書です。
作業前に必ず全体を読んでください。

---

## 1. プロジェクト概要

**DungeonStudy** は中学3年生向けのローグライク型学習ゲームです。
ダンジョンを探索して敵（問題）を倒し、経験値を得てレベルアップするゲームシステムに、
**アクティブリコール（能動的想起）** と **SRS（間隔反復）** を組み込んだ教育アプリです。

- **ターゲット**: 中学3年生（12〜15歳）、高校受験対策
- **プラットフォーム**: スマートフォン優先のブラウザアプリ（GitHub Pages 公開）
- **外部依存**: Google Fonts（Noto Sans JP）のみ

### ファイル構成

```
learning-rogue-game-app/
├── dungeon_study.html        ← 唯一のアプリファイル（約1552行）
├── README.md
├── DESIGN.md                 ← 設計思想ドキュメント
├── DESIGN_DETAILS.md         ← 詳細設計・背景
└── HANDOVER-DungeonStudy.md  ← 引き継ぎドキュメント
```

**このプロジェクトは単一HTMLファイル設計が意図的な制約です。**
複数ファイルへの分割や外部ライブラリ（React/Vue等）の導入は原則禁止です。

---

## 2. アーキテクチャガイド

### 単一ファイル設計

すべてのコード（HTML/CSS/JavaScript）が `dungeon_study.html` の1ファイルに収まっています。
ファイル名は `dungeon_study.html` に固定です（GitHub Pages の URL 変更を避けるため）。
バージョン管理はアプリ内の `APP_VERSION` 定数で行います。

### コード構造（dungeon_study.html）

| 行番号目安 | セクション |
|---|---|
| 1〜373 | HTML構造 + CSS（`:root`変数、全スタイル） |
| 374〜619 | CONFIG & DATA（定数・DEFAULT_DUNGEONS 19個・アバター定義） |
| 620〜650 | STATE（グローバル状態オブジェクト `S`）・タイマー変数 |
| 651〜677 | localStorage ヘルパー（`saveDungeons`, `loadDungeons`） |
| 678〜718 | レベル・EXP計算（`syncLevel`, `addExp`, `triggerLvUp`） |
| 719〜779 | MAP生成（Recursive Backtracking）+ 霧の管理 |
| 780〜950 | Canvas 描画（アイソメトリック）+ タイル描画関数群 |
| 951〜989 | アバター SVG 生成（`avHUD`, `avFull`, `enSVG`） |
| 990〜1095 | 移動・衝突判定（`tryMove`, `pickItem`, `fleeBattle`） |
| 996〜1095 | 戦闘システム（`startBattle`, `startRecall`, `onChoice`） |
| 1096〜1190 | バトル描画（`renderBov`） |
| 1191〜1282 | HUD・討伐図鑑（`updHUD`, `zukanRecord`, `renderZukan`） |
| 1283〜1436 | 管理画面（`renderAdminList`, `openEdit`, `saveEdit`, `importDungeon`） |
| 1437〜1552 | シーン管理・イベントハンドラ・BOOT（`window.addEventListener("load",...)` ） |

### グローバル状態オブジェクト S

すべてのゲーム状態は単一のオブジェクト `S` で管理されています。

```javascript
let S = {
  scene: "title",          // "title"|"dungeon"|"battle"|"clear"|"gameover"
  player: {
    name: "冒険者",
    key: "warrior",        // アバターキー
    hp: 5, maxHp: 5,
    totalExp: 0, level: 1,
    combo: 0,              // 連続正解数（3でHP+1）
  },
  dungeon: null,           // 現在のダンジョンオブジェクト
  map: null,               // 9×9 タイル配列
  fog: null,               // 9×9 霧配列
  pos: {x:1, y:1},        // プレイヤー位置
  prevPos: null,           // 直前位置（逃げた時の戻り先）
  enemies: [],             // 敵配置リスト
  items: [],               // アイテム配置リスト
  exitPos: null,           // 出口位置
  curEnemy: null,          // 戦闘中の敵エンティティ
  prevScene: null,         // 呪い戦闘前のシーン（戻り先）
  chosen: null,            // 選択した答え
  judged: false,           // 判定済みフラグ
  isOk: false,             // 正解フラグ
  hintUsed: false,         // ヒント使用フラグ（EXP半減）
  timeLeft: 20,            // 制限時間（秒）
  rcPhase: true,           // 想起フェーズ中フラグ
  rcCount: 3,              // 想起カウントダウン
  expGain: null,           // 獲得EXP（アニメーション用）
  healGain: false,         // HP回復フラグ（アニメーション用）
  exp2x: false,            // EXP2倍フラグ
  log: [],                 // ダンジョン内ログ（クリア画面表示用）
};
```

### 主要定数

```javascript
APP_VERSION = "1.0.1"                     // タイトル画面に表示。更新時はここのみ変更
MSIZE = 9                                  // マップサイズ（9×9グリッド）
LS_KEY = "dungeon_study_dungeons_v4"       // localStorage キー
DIF_EXP = {1:20, 2:28, 3:38, 4:50}       // difficulty → EXP 変換
DIF_TIER = {1:0,  2:1,  3:2,  4:2}       // difficulty → tier 変換
```

---

## 3. 開発ガイドライン

### 変更禁止の設計決定（8項目）

以下は意図的な設計決定です。バグではありません。絶対に変更しないでください。

| # | 項目 | 理由 |
|---|---|---|
| 1 | **早期レベルアップ優先（Lv1→2で20EXP）** | 敵1体で達成できる設計。序盤の達成感が継続率に直結するため |
| 2 | **HP回復は3種類のみ** | コンボ3連正解・ボス撃破・呪い解除。意図的なバランス設計 |
| 3 | **ボス封印（雑魚全滅が前提条件）** | 雑魚でアクティブリコールを練習してからボスへ進む学習フロー |
| 4 | **ボスは記述式（4択ではなく自由テキスト入力）** | 4択の運任せを排除し、真の定着度を測定するため |
| 5 | **カスタムモーダル（`window.confirm/alert/prompt` 使用禁止）** | iframe/WebView 環境で動作しないため |
| 6 | **ファイル名固定（`dungeon_study.html`）** | GitHub Pages の URL を変えないため |
| 7 | **呪いリスト（SRS実装）** | 不正解問題を記録し間を置いて再想起させる設計 |
| 8 | **テンプレートリテラルの3重ネスト禁止** | 構文エラーのリスクがあるため文字列連結方式を使用 |

### テンプレートリテラルの制約（重要）

バッククォート（`` ` ``）の3重ネストは構文エラーになります。
HTML文字列を生成する際は必ず**文字列連結方式**を使用してください。

```javascript
// NG: バッククォートの3重ネスト
const html = `<div class="${cls}">${`<span>${text}</span>`}</div>`;

// OK: 文字列連結方式
const html = '<div class="' + cls + '"><span>' + text + '</span></div>';

// OK: バッククォートは2重まで（変数展開のみ）
const html = `<div class="${cls}">` + inner + '</div>';
```

### コーディングルール

- **グローバル変数**: 新しいゲーム状態は必ず `S` オブジェクトに追加する
- **タイマー変数**: `tID`（バトルタイマー）、`rID`（想起タイマー）、`tTimer`（トースト）、`lvTimer`（レベルアップバナー）。`clearTs()` で `tID` と `rID` を一括クリア
- **UI更新**: `innerHTML` への文字列代入で UI を更新する（現在のパターンを踏襲）
- **イベント登録**: `renderBov()` など描画関数内で `onclick` を直接代入する（現在のパターン）
- **マップサイズ**: `MSIZE=9` より大きくする場合は Recursive Backtracking をスタックオーバーフロー対策の反復版に変更が必要

---

## 4. 問題データ形式

### インポート用 JSON（管理画面入力形式）

```json
{
  "name": "ダンジョン名",
  "subject": "科目名",
  "icon": "⚗️",
  "color": "#00809E",
  "questions": [
    {
      "difficulty": 1,
      "monster_name": "数式ゴブリン",
      "question": "問題文",
      "choices": ["選択肢A", "選択肢B", "選択肢C", "選択肢D"],
      "answer": "選択肢A",
      "explanation": "解説文"
    },
    {
      "difficulty": 4,
      "type": "boss",
      "monster_name": "代数ドラゴン",
      "question": "記述式の問題文",
      "answer": "記述式の正解",
      "explanation": "解説文"
    }
  ]
}
```

- `difficulty 1〜3`: 4択問題（雑魚モンスター）
- `difficulty 4` + `type:"boss"`: 記述式（ボス、`choices` なし）
- `answer` は `choices` のいずれかと**文字列完全一致**にすること（ボスを除く）
- `monster_name`、`hint` は省略可能

### 内部データ構造（`DUNGEONS` 配列の各要素）

```javascript
// ダンジョン
{
  id: "sci_ion",           // 文字列ID（DEFAULT_DUNGEONS）またはタイムスタンプ生成ID
  name: "イオンの洞窟",
  subject: "理科",
  icon: "⚗️",
  color: "#9B59B6",
  enemies: [ /* 敵配列 */ ]
}

// 敵（問題）
{
  id: "si1",               // 敵のユニークID
  name: "陽イオンスライム",
  tier: 0,                 // 0(易)・1(中)・2(難)。外見に影響
  exp: 20,                 // 獲得EXP
  difficulty: 1,           // 1〜4
  isBoss: false,           // ボスフラグ
  q: "問題文",
  ans: "正解",
  choices: ["A","B","C","D"],  // ボスは undefined
  hint: "ヒント文",
  explanation: "解説文",
}
```

### `buildEnemies()` 変換ロジック

管理画面でインポートした JSON は `buildEnemies()` 関数で内部形式に変換されます。
`choices` が省略されたボス問題は `undefined` になります（`isBoss` チェックで分岐）。

---

## 5. ゲームシステムの主要ロジック

### EXP・レベル計算

```javascript
// 次のレベルに必要な累積EXP
expNeeded(lv) = lv <= 1 ? 20 : Math.floor(15 * Math.log(lv + 1) + 10)

// 参考値: Lv1→2: 20EXP, Lv10→11: 51EXP, Lv100→101: 100EXP

// difficulty → EXP 変換
DIF_EXP = {1:20, 2:28, 3:38, 4:50}
```

### HP 回復条件（3種類のみ）

| 条件 | 効果 |
|---|---|
| 3連続正解（コンボ） | HP +1、コンボリセット |
| ボス撃破 | HP 全回復 |
| 呪い問題に正解 | 失ったHP分（`hpLost`）を回復 |

マップ上の💊アイテムは設計バランス外のボーナスです。上記3種類が「設計した回復手段」です。

### 戦闘フロー

```
enterDungeon() → 敵マスに踏み込む
  → startBattle()
    → startRecall()        // 3秒カウントダウン（想起フェーズ）
      → renderBov()        // 問題表示
        → startBTimer()    // 20秒タイマー開始
          → onChoice(c)    // 選択 / テキスト入力
            → 正解: addExp() + HP回復チェック + zukanRecord()
            → 不正解: HP-1 + addCurse() + zukanRecord()
          → onTimeout()    // タイムオーバー: HP-2 + addCurse()
            → HP=0 → goGO()（ゲームオーバー）
```

### ボス封印ロジック

`tryMove()` 内でボスマスへの移動時に雑魚残存数をチェックしています。

```javascript
const normLeft = S.enemies.filter(e => !e.enemy.isBoss && !e.done).length;
if (normLeft > 0) { showToast("⛔ ボスは封印中！雑魚を" + normLeft + "体倒せ"); return; }
```

### 呪いリスト（SRS）

`CURSE_LIST` はメモリ上のみの配列です（**ページリロードでリセット**されます）。

```javascript
// 呪い追加（不正解・タイムオーバー時）
addCurse(enemy, hpLost)  // hpLost: 不正解=1, タイムオーバー=2

// 呪い解除（正解時）
CURSE_LIST.splice(ci, 1)  // 該当問題を除去し、hpLost 分のHP回復

// ダンジョン入場時の自動出題
if (CURSE_LIST.length > 0) startCurseBattle(CURSE_LIST[0]);
```

### シーン管理

```javascript
// シーンID一覧
SCRS = ["scr-title","scr-avatar","scr-select","scr-zukan","scr-admin",
        "scr-admin-add","scr-admin-edit","scr-admin-prompt","scr-clear","scr-go"]

showScr(id)   // 指定シーンのみ表示、他を非表示
showGame()    // HUD + ゲームエリアを表示（ダンジョン探索中）
hideGame()    // HUD + ゲームエリアを非表示（選択画面等）
```

### ボス記述式の正答判定

現在は `c.trim() === en.ans.trim()` の**完全一致**のみです（既知の制限）。
`ans` に設定する文字列は表記揺れが発生しないよう統一してください。

---

## 6. 既知の制約・課題

| 項目 | 内容 | 優先度 |
|---|---|---|
| **CURSE_LIST 非永続化** | ページリロードでリセット。SRS効果が限定的 | 高 |
| **ボス記述式の正答判定** | 完全一致のみ。全角・スペースの表記揺れ非対応 | 高 |
| **localStorage 5MB上限** | 問題数が数百問規模になると不足 | 中 |
| **単一ファイル1500行超** | 保守性の低下。将来的なコンポーネント分割が課題 | 低 |
| **ZUKAN 非永続化** | 討伐図鑑もページリロードでリセット | 低 |
| **Recursive Backtracking** | `MSIZE=9` は問題なし。13以上にする場合は反復版に変更が必要 | 低 |

---

## 7. 作業時の注意事項

### ファイルを壊さないために

1. **バッククォートのネスト確認**: 文字列を追加する際は3重ネストになっていないか必ず確認
2. **`window.confirm/alert/prompt` を追加しない**: カスタムモーダル（`del-modal` 等）を使う
3. **`S.judged` フラグの確認**: 戦闘中は `S.judged` フラグを必ず確認してから状態変更
4. **`clearTs()` の呼び忘れ注意**: 複数タイマーが走ると二重動作します
5. **`innerHTML` の文字列生成**: XSS リスクはあるが現在の設計方針。ユーザー入力は `sanitizeJSON()` で処理

### バージョンアップ時

`APP_VERSION` 定数のみ更新します（コード先頭付近 `const APP_VERSION="1.0.1"` の行）。
ファイル名は変更しません。

### 問題データの追加

`DEFAULT_DUNGEONS` 配列に直接追記するか、管理画面のJSONインポート機能を使います。
`DEFAULT_DUNGEONS` に追記した場合、`localStorage` にデータが既存の場合は上書きされません
（`loadDungeons()` は `localStorage` にデータがあればそちらを優先します）。
既存ユーザーに新ダンジョンを配布したい場合は `LS_KEY` を変更する必要があります（現在 `v4`）。

### 管理画面の JSON サニタイズ

`sanitizeJSON()` 関数で以下を自動処理しています：
- Markdown コードブロック（` ```json ` 等）の除去
- 全角スペース（`　`）→ 半角スペース
- 全角クォート → 半角クォート
- trailing comma の除去
- BOM の除去

AI が生成した JSON をそのまま貼り付けても動作する設計です。

---

## 8. 将来の改善優先項目（参考）

1. 問題DBのバックエンド化（localStorage → API）
2. SRS永続化（CURSE_LIST をユーザーアカウントに紐付け）
3. 記述式ボスの正答判定改善（全角半角正規化・スペース除去）
4. コンポーネント分割（React/Vue 等への移行）
5. 問題コンテンツ拡充（AIプロンプト生成機能はすでに実装済み）
