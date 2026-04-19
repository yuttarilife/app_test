# Spirit Ascend — 実装計画

作成日: 2026-04-18

---

## Context

`apps/spirit-ascend/index.html` を新規作成する。設計仕様は `plans/plan-functional-meteor.md` に定義済み。既存の `apps/neko-punch/index.html`（IIFE モジュールパターン）を踏襲して、外部依存ゼロ・単一HTMLファイル・ブラウザ直接起動で動作するゲームを実装する。

---

## 対象ファイル

| 役割 | パス |
|---|---|
| 参照元（パターン流用） | `apps/neko-punch/index.html` |
| 設計仕様書 | `plans/plan-functional-meteor.md` |
| 新規作成 | `apps/spirit-ascend/index.html` |

---

## モジュール構成

```
SpiritAscend.Config    — ステージ定義・難易度・セリフデータ・障害物定義
SpiritAscend.Storage   — localStorage読み書き・バリデーション
SpiritAscend.Sound     — Web Audio API サウンドエンジン
SpiritAscend.Render    — SVG精霊描画・進化切替・障害物SVG生成
SpiritAscend.Game      — ゲームループ・衝突判定・スコア・ステージ管理
SpiritAscend.UI        — 画面遷移・イベント・SNSシェアカード生成
SpiritAscend.init()    — 初期化エントリーポイント
```

---

## 実装ステップ

### Step 1: HTMLスケルトン + CSS基盤

- 全画面を `.screen` クラスで管理し、`display:none / flex` で切り替える
- 画面ID一覧: `screen-title`, `screen-gender`, `screen-stage-select`, `screen-countdown`, `screen-game`, `screen-result`, `screen-all-clear`, `modal-settings`, `modal-ranking`, `notification`
- ゲームエリア `#game-area`: `position:relative; overflow:hidden` のdiv。精霊・障害物は `position:absolute; transform:translate(x,y)` で配置
- SNSシェアカード用: 非表示 `<canvas id="share-canvas" width="400" height="300">`
- 背景: 濃い紺〜紫のグラデーション
- `prefers-reduced-motion` 対応: `animation: none !important`
- ズーム防止: `touchstart` (multi-touch) / `gesturestart` / `dblclick` で `preventDefault()`

### Step 2: `SpiritAscend.Config`

```javascript
STAGES[10] = [
  { id:1, timeLimit:20, obstacleTypes:[1], spawnInterval:1800, baseSpeed:0.12, speedInc:0.000 },
  { id:2, timeLimit:20, obstacleTypes:[1,2], spawnInterval:1600, baseSpeed:0.13, speedInc:0.001 },
  { id:3, timeLimit:25, obstacleTypes:[1,2], spawnInterval:1400, baseSpeed:0.15, speedInc:0.001 },
  { id:4, timeLimit:25, obstacleTypes:[1,2,3], spawnInterval:1200, baseSpeed:0.17, speedInc:0.002 },
  { id:5, timeLimit:30, obstacleTypes:[1,2,3], spawnInterval:1100, baseSpeed:0.19, speedInc:0.002 },
  { id:6, timeLimit:30, obstacleTypes:[1,2,3,4], spawnInterval:900, baseSpeed:0.21, speedInc:0.002 },
  { id:7, timeLimit:35, obstacleTypes:[1,2,3,4], spawnInterval:800, baseSpeed:0.24, speedInc:0.003 },
  { id:8, timeLimit:35, obstacleTypes:[1,2,3,4], spawnInterval:700, baseSpeed:0.27, speedInc:0.003 },
  { id:9, timeLimit:40, obstacleTypes:[1,2,3,4,5], spawnInterval:600, baseSpeed:0.30, speedInc:0.003 },
  { id:10, timeLimit:40, obstacleTypes:[1,2,3,4,5], spawnInterval:500, baseSpeed:0.33, speedInc:0.004 },
]

OBSTACLES[5] = [
  { id:1, name:'流れ星',    radius:8  },
  { id:2, name:'魔法の球',  radius:9  },
  { id:3, name:'三日月',    radius:8  },
  { id:4, name:'炎弾',      radius:9  },
  { id:5, name:'光の渦',    radius:11 },
]

CLEAR_MESSAGES[7]  // セリフ配列（ランダム・連続回避付き）
SCORES = { perSecond:10, nearMiss:50, stageClear:500, perfect:1000 }
```

速度は **px/ms** 単位で統一。

### Step 3: `SpiritAscend.Storage`

neko-punch の `validate()` + `clamp()` パターンを流用。

```javascript
DEFAULTS = {
  version: 1,
  gender: null,           // 'girl'|'boy'|'neutral'|null
  playerName: '冒険者',
  clearedStages: [],      // [1,2,...]
  highScore: 0,
  totalScore: 0,
  playCount: 0,
  perfectCount: 0,
  ranking: [],            // [{name,score,stage,date}] 最大20件
  settings: { soundEnabled:true, saveOnResult:true }
}
```

localStorage 利用不可時はメモリのみで動作し、UI にトースト通知を出す。

### Step 4: `SpiritAscend.Sound`

neko-punch の AudioContext 初期化（初回 `pointerdown` 時 + `suspended` → `resume()`）を流用。

| 音 | 実装 |
|---|---|
| 回避音 | sine 880Hz, 0.08秒 |
| クリア音 | 上昇アルペジオ C5→E5→G5→C6 各0.1秒 |
| 進化音 | triangle 3和音 C4+E4+G4, 0.5秒フェードアウト |
| ゲームオーバー音 | D4→B3下降 + ノイズ, 0.3秒 |
| BGM | sine+triangle の常時ループ。`GainNode.gain` を 0 にランプしてミュート |

### Step 5: `SpiritAscend.Render`

**精霊SVG（5段階×3色・シンプル方針）**

`viewBox="0 0 80 100"` の SVG を `#spirit-container` に配置。進化時は `innerHTML` 書き換え。色は CSS変数で切り替え。

```css
.gender-girl    { --spirit-main:#f472b6; --spirit-sub:#e879f9; --spirit-glow:#fce7f3; }
.gender-boy     { --spirit-main:#60a5fa; --spirit-sub:#3b82f6; --spirit-glow:#eff6ff; }
.gender-neutral { --spirit-main:#a78bfa; --spirit-sub:#7c3aed; --spirit-glow:#f5f3ff; }
```

| Form | ステージ | 主要SVG要素 | アニメーション |
|---|---|---|---|
| 1 | 1-2 | `<circle>` + `feGaussianBlur` ぼかし | `@keyframes pulse`（サイズ+不透明度） |
| 2 | 3-4 | 頭 circle + 体 circle + 手足 `<line>` x4 | 上下ホバー `translateY` |
| 3 | 5-6 | 頭 + 楕円体 + 三角スカート/四角ズボン + 顔の点 | なし |
| 4 | 7-8 | Form3 + 杖 `<line>+<polygon>` + 紋章 `<circle>` | 杖先端 `@keyframes glow` |
| 5 | 9-10 | Form4 + 3重外周 circle + 魔法陣クロス `<line>x2` | 外周 `animateTransform rotate` |

`<defs>` 内の `<filter id="glow">` は全 Form で共有。

**障害物SVG生成 (`Render.createObstacleSVGString(type)`):**

| 種別 | SVG |
|---|---|
| 流れ星 | `<line>` 傾き45° + `<circle>` 先端光点 |
| 魔法の球 | `<circle>` + `feGaussianBlur` グロー |
| 三日月 | `<path>` arc コマンドで月型 |
| 炎弾 | `<path>` しずく型曲線 |
| 光の渦 | 2重 `<circle>` + `animateTransform rotate` |

進化エフェクト: 旧SVGに `scale(1.3)+opacity:0` CSS → 新SVGをフェードイン。

### Step 6: `SpiritAscend.Game`（最重要）

**ゲームループ（`requestAnimationFrame`）:**

```
loop(timestamp) {
  delta = Math.min(timestamp - last, 50)  // 上限50msで巨大delta防止
  last = timestamp

  elapsed += delta
  remainingTime = timeLimit - elapsed/1000
  if (remainingTime <= 0) → stageClear()

  // 精霊移動
  spiritX += input.dx * SPIRIT_SPEED * delta
  spiritX = clamp(spiritX, SPIRIT_R, gameW - SPIRIT_R)

  // スポーン
  spawnTimer += delta
  if (spawnTimer >= spawnInterval) { spawnObstacle(); spawnTimer = 0 }

  // 障害物更新
  for obs: obs.y += obs.vy * delta; 範囲外なら obs.el.remove() + splice

  // 衝突判定
  checkCollisions()

  // スコア
  scoreTimer += delta
  if (scoreTimer >= 1000) { score += SCORES.perSecond; scoreTimer -= 1000 }

  // DOM更新
  applyPositions()

  rafId = requestAnimationFrame(loop)
}
```

**衝突判定:**

```javascript
const dist = Math.hypot(obs.x - spiritX, obs.y - spiritY)
const hit = SPIRIT_R + obs.radius  // 12 + 8〜11
if (dist < hit) → gameOver()
if (!obs.nearMissed && dist < hit * 1.5) { obs.nearMissed = true; score += 50; nearMissEffect() }
```

**スポーン（直上回避付き）:**

```javascript
let x, attempts = 0
do { x = 20 + Math.random() * (gameW - 40); attempts++ }
while (Math.abs(x - spiritX) < 40 && attempts < 10)
```

**入力:**
- キーボード: `keydown/keyup` → `input.left/right` フラグ
- マウス: `pointermove` → 精霊をX座標に緩やかに追従
- スマホ: `pointerdown` で左右判定 + `pointerup` でリセット（`pointerId` 管理）

**精霊フォームの決定:**

```javascript
function getSpiritForm(stage) {
  if (stage <= 2) return 1; if (stage <= 4) return 2;
  if (stage <= 6) return 3; if (stage <= 8) return 4; return 5
}
```

### Step 7: `SpiritAscend.UI`

**画面切り替え:**

```javascript
function showScreen(id) {
  document.querySelectorAll('.screen').forEach(el => el.style.display = 'none')
  document.getElementById(id).style.display = 'flex'
}
```

**カウントダウン:** `setTimeout` 1秒ごとに 3→2→1→GO → `Game.start()`

**ランキング保存フロー:**
1. `settings.saveOnResult === true` なら確認モーダル表示
2. 「保存する」→ 名前が「冒険者」なら変更プロンプト
3. `Storage.saveRanking()` → `ranking` 配列をスコア降順・最大20件で保存

**SNSシェアカード（Canvas API）:**
- 起動時に `typeof CanvasRenderingContext2D` をチェック、非対応なら `#btn-share` を `display:none`
- 精霊SVGを `Blob → objectURL → <img>` 経由で `ctx.drawImage()`
- 背景グラデーション + テキスト + ハッシュタグ `#SpiritAscend` を描画
- `canvas.toDataURL('image/png')` → `<a download>` でPNG保存

**セリフの連続回避:**

```javascript
let lastMsgIdx = -1
function getClearMessage() {
  let idx
  do { idx = Math.random() * CLEAR_MESSAGES.length | 0 } while (idx === lastMsgIdx)
  lastMsgIdx = idx
  return CLEAR_MESSAGES[idx]
}
```

### Step 8: `SpiritAscend.init()`

```javascript
SpiritAscend.init = function() {
  Storage.init()
  UI.init()
  // ズーム防止（猫パンチと同パターン）
  document.addEventListener('touchstart', e => { if (e.touches.length > 1) e.preventDefault() }, { passive:false })
  document.addEventListener('gesturestart', e => e.preventDefault())
  document.addEventListener('dblclick', e => e.preventDefault())
  // 初期画面の決定
  const gender = Storage.get().gender
  UI.showScreen(gender === null ? 'screen-gender' : 'screen-title')
}
SpiritAscend.init()
```

---

## 注意事項

- **速度単位**: すべて `px/ms`（フレームレート非依存）
- **障害物管理**: 配列管理方式（オブジェクトプール不採用）。最大同時数20個程度なので問題なし
- **transform優先**: 位置更新は `transform:translate(x,y)` のみ（`left/top` より再描画コスト低）
- **Canvas SNSカード**: `file://` 起動でも動作するよう、SVG内の `foreignObject` は使わず純粋なCanvas描画のみ
- **BGMミュート**: `AudioContext` は止めない。`GainNode.gain` を 0 にランプするだけ

---

## 検証方法

1. `apps/spirit-ascend/index.html` を Chrome で `file://` 直接開く
2. 性別選択スキップ → タイトル → Stage 1 → カウントダウン → ゲーム開始
3. キーボード ← → で左右移動することを確認
4. 障害物に当たると即ゲームオーバーになることを確認
5. 20秒生き残るとクリア → 精霊セリフが表示され、次回は別のセリフが出ることを確認
6. Stage 3 クリア時に精霊が Form 2 に進化することを確認（エフェクト込み）
7. DevTools でスマホエミュレーション → タップ左右移動の確認
8. localStorage を破損させてリロード → 初期値にリセットされトースト通知が出ることを確認
9. Stage 10 クリア → 全ステージクリア画面 → SNSシェアカードをPNGで保存できることを確認
10. DevTools → Performance でStage 10 中のFPS が 55fps 以上であることを確認
