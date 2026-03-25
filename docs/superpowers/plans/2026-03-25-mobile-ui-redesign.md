# MGscore モバイルUI リデザイン 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `index_iOS.html` をCalmiデザインシステムの問診フロー型UIに全面リニューアルし、QRコード生成機能を追加する。

**Architecture:** 単一HTMLファイル内にCSS変数でCalmiトークンを定義し、JSでフローエンジン（1問ずつ表示→自動遷移）を実装。QR生成はqrcode-generatorライブラリをインライン埋め込み。全てオフライン完結。

**Tech Stack:** HTML5, CSS3 (CSS Custom Properties), Vanilla JS, Canvas API, qrcode-generator (inline)

**Spec:** `docs/superpowers/specs/2026-03-25-mobile-ui-redesign.md`

**Base file:** `index_iOS.html`（現在の内容は全面置換）

---

## Task 1: HTML骨格 + CSS変数 + Calmiスタイルシート

単一HTMLファイルのシェルを作る。データモデルやロジックは後のタスクで追加。
既存のタブUI（`.nav`）は廃止し、線形フロー+プログレスバーに置換する。

**Files:**
- Modify: `index_iOS.html` (全面書き換え)

- [ ] **Step 1: HTML骨格を作成**

`index_iOS.html` を以下の構造で全面置換する:

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>MG臨床スコア</title>
  <style>
    /* === Calmi Design Tokens === */
    :root {
      --mg-primary: #4A9B3F;
      --mg-primary-light: rgba(74, 155, 63, 0.1);
      --mg-primary-border: rgba(74, 155, 63, 0.3);
      --mg-bg: #F5F7F2;
      --mg-surface: #FFFFFF;
      --mg-text: #1c1c1e;
      --mg-text-grey: #63666A;
      --mg-divider: #E5E5EA;
      --mg-error: #DC2626;
      --mg-warning: #D97706;
      --mg-success: #2D8A4E;
      --radius-card: 16px;
      --radius-medium: 12px;
      --radius-small: 8px;
      --shadow-card: 0 2px 8px rgba(0, 0, 0, 0.06);
    }
    /* === Base === */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Helvetica Neue", sans-serif;
      background: var(--mg-bg);
      color: var(--mg-text);
      line-height: 1.5;
      min-height: 100dvh;
      padding: 16px;
      max-width: 480px;
      margin: 0 auto;
    }
    /* === Progress Bar === */
    .progress-bar { display: flex; gap: 4px; margin-bottom: 8px; }
    .progress-segment { flex: 1; height: 4px; border-radius: 2px; background: var(--mg-divider); transition: background 0.3s; }
    .progress-segment.done { background: var(--mg-primary); }
    .progress-counter { text-align: right; font-size: 13px; color: var(--mg-text-grey); margin-bottom: 16px; }
    /* === Question Screen === */
    .screen { display: none; }
    .screen.active { display: block; }
    .question-title { text-align: center; font-size: 26px; font-weight: 700; margin-bottom: 4px; }
    .question-subtitle { text-align: center; font-size: 14px; color: var(--mg-text-grey); margin-bottom: 24px; }
    /* === Option Cards === */
    .option-card {
      display: flex; align-items: center; gap: 14px;
      background: var(--mg-surface); border: 1.5px solid var(--mg-divider);
      border-radius: 14px; padding: 16px 20px; margin-bottom: 10px;
      min-height: 52px; cursor: pointer;
      transition: border-color 0.2s, background 0.2s;
      -webkit-tap-highlight-color: transparent;
    }
    .option-card.selected {
      border-color: var(--mg-primary);
      background: var(--mg-primary-light);
    }
    .option-badge {
      width: 36px; height: 36px; border-radius: 10px;
      background: var(--mg-primary-light);
      display: flex; align-items: center; justify-content: center;
      font-size: 18px; font-weight: 700; color: var(--mg-primary);
      flex-shrink: 0;
    }
    .option-card.selected .option-badge { background: var(--mg-primary); color: #fff; }
    .option-text { font-size: 16px; }
    .option-card.selected .option-text { color: var(--mg-primary); font-weight: 600; }
    /* === Back Link === */
    .back-link {
      display: block; text-align: center; margin-top: 20px;
      font-size: 15px; color: var(--mg-primary); font-weight: 600;
      cursor: pointer; -webkit-tap-highlight-color: transparent;
    }
    /* === Result Screen === */
    .score-circle {
      width: 140px; height: 140px; border-radius: 50%;
      border: 6px solid var(--mg-primary); margin: 20px auto;
      display: flex; flex-direction: column; align-items: center; justify-content: center;
      background: var(--mg-surface);
      box-shadow: 0 2px 12px rgba(74, 155, 63, 0.15);
    }
    .score-circle .score-value { font-size: 48px; font-weight: 700; color: var(--mg-primary); }
    .score-circle .score-max { font-size: 14px; color: var(--mg-text-grey); }
    .score-label { text-align: center; font-size: 20px; font-weight: 700; margin-top: 12px; margin-bottom: 20px; }
    /* === Item Summary List === */
    .item-summary { background: var(--mg-surface); border-radius: var(--radius-card); padding: 14px; margin-bottom: 16px; box-shadow: var(--shadow-card); }
    .item-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #F0F0F0; }
    .item-row:last-child { border-bottom: none; }
    .item-name { font-size: 15px; }
    .item-score { font-size: 15px; font-weight: 600; }
    .item-score.score-0 { color: var(--mg-primary); }
    .item-score.score-low { color: var(--mg-warning); }
    .item-score.score-high { color: var(--mg-error); }
    /* === Buttons === */
    .btn {
      display: block; width: 100%; padding: 16px; border: none;
      border-radius: var(--radius-medium); font-size: 16px; font-weight: 600;
      cursor: pointer; text-align: center;
      -webkit-tap-highlight-color: transparent;
      transition: opacity 0.2s;
    }
    .btn:active { opacity: 0.7; }
    .btn-primary { background: var(--mg-primary); color: #fff; }
    .btn-secondary {
      background: var(--mg-primary-light);
      border: 1.5px solid var(--mg-primary-border);
      color: var(--mg-primary);
    }
    .btn-ghost { background: none; border: none; color: var(--mg-error); font-size: 15px; font-weight: 500; padding: 12px; }
    .btn-row { display: flex; gap: 10px; }
    .btn-row .btn { flex: 1; }
    .btn + .btn, .btn-row + .btn, .btn + .btn-row { margin-top: 10px; }
    /* === QR === */
    .qr-container {
      display: none; background: var(--mg-surface); border-radius: var(--radius-card);
      padding: 24px; text-align: center; margin-top: 16px;
      box-shadow: var(--shadow-card);
    }
    .qr-container.visible { display: block; }
    .qr-container canvas { display: block; margin: 0 auto 12px; }
    .qr-hint { font-size: 13px; color: var(--mg-text-grey); }
    /* === Final: dual score === */
    .dual-scores { display: flex; gap: 16px; justify-content: center; }
    .dual-scores .score-circle { width: 120px; height: 120px; }
    .dual-scores .score-value { font-size: 36px; }
    /* === Slide Animation === */
    .slide-container { position: relative; overflow: hidden; }
    .slide-left-enter { animation: slideLeftIn 0.3s ease-out; }
    .slide-right-enter { animation: slideRightIn 0.3s ease-out; }
    @keyframes slideLeftIn { from { transform: translateX(100%); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
    @keyframes slideRightIn { from { transform: translateX(-100%); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
  </style>
</head>
<body>
  <h1 style="text-align:center; font-size:22px; font-weight:700; margin-bottom:16px;">MG臨床スコア</h1>

  <!-- Progress -->
  <div id="progress-bar" class="progress-bar"></div>
  <div id="progress-counter" class="progress-counter"></div>

  <!-- Slide container: question screens rendered here by JS -->
  <div id="slide-container" class="slide-container"></div>

  <!-- Result screens (hidden until flow completes) -->
  <div id="adl-result" class="screen"></div>
  <div id="mgc-result" class="screen"></div>
  <div id="final-result" class="screen"></div>

  <!-- QR code canvas (shared, moved into active result screen by JS) -->
  <div id="qr-container" class="qr-container">
    <canvas id="qr-canvas" width="200" height="200"></canvas>
    <div class="qr-hint">長押しで画像を保存できます</div>
  </div>

  <script>
    // Task 2以降で実装
  </script>
</body>
</html>
```

- [ ] **Step 2: ブラウザで表示確認**

Run: ブラウザで `index_iOS.html` を開き、タイトル「MG臨床スコア」と空のプログレスバーエリアが表示されることを確認。背景が `#F5F7F2` であること。

- [ ] **Step 3: コミット**

```bash
git add index_iOS.html
git commit -m "feat: scaffold HTML shell with Calmi CSS tokens and component styles"
```

---

## Task 2: データモデル + フローエンジン（ADL 8問）

ADLの8問を1問ずつ表示→選択→自動遷移するフローエンジンを実装。

**Files:**
- Modify: `index_iOS.html` (`<script>` 内)

- [ ] **Step 1: データモデルを定義**

`<script>` タグ内に以下を記述。既存の `index_iOS.html` のデータ定義を移植:

```javascript
const DATA = {
  adl: {
    title: 'MG-ADL',
    maxScore: 24,
    items: [
      { name: '会話', options: ['正常', '間欠的に不明瞭もしくは鼻声', '常に不明瞭もしくは鼻声、しかし聞いて理解可能', '聞いて理解するのが困難'], values: [0,1,2,3] },
      { name: '咀嚼', options: ['正常', '固形物で疲労', '柔らかい食物で疲労', '経管栄養'], values: [0,1,2,3] },
      { name: '嚥下', options: ['正常', 'まれにむせる', '頻回にむせるため、食事の変更が必要', '経管栄養'], values: [0,1,2,3] },
      { name: '呼吸', options: ['正常', '体動時の息切れ', '安静時の息切れ', '人工呼吸を要する'], values: [0,1,2,3] },
      { name: '歯磨き・櫛使用の障害', options: ['なし', '努力を要するが休息を要しない', '休息を要する', 'できない'], values: [0,1,2,3] },
      { name: '椅子からの立ち上がり障害', options: ['なし', '軽度、時々腕を使う', '中等度、常に腕を使う', '高度、介助を要する'], values: [0,1,2,3] },
      { name: '複視', options: ['なし', 'あるが毎日ではない', '毎日起こるが持続的でない', '常にある'], values: [0,1,2,3] },
      { name: '眼瞼下垂', options: ['なし', 'あるが毎日ではない', '毎日起こるが持続的でない', '常にある'], values: [0,1,2,3] }
    ]
  },
  mgc: {
    title: 'MG Composite',
    maxScore: 50,
    items: [
      { name: '上方視時の眼瞼下垂出現までの時間', desc: '（医師の診察）', options: ['>45秒', '11〜45秒', '1〜10秒', '常時'], values: [0,1,2,3] },
      { name: '側方視時の複視出現までの時間', desc: '（医師の診察）', options: ['>45秒', '11〜45秒', '1〜10秒', '常時'], values: [0,1,3,4] },
      { name: '閉眼の筋力', desc: '（医師の診察）', options: ['正常', '軽度低下（閉眼維持可能）', '中等度低下（閉眼維持困難）', '重度低下（閉眼不能）'], values: [0,0,1,2] },
      { name: '会話、発音', desc: '（MGADL）', options: ['正常', '時に不明瞭または鼻声', '常に不明瞭または鼻声だが理解可能', '不明瞭で理解が困難'], values: [0,2,4,6] },
      { name: '咬む動作', desc: '（MGADL）', options: ['正常', '固い食物で疲労', '柔らかい食物でも疲労', '栄養チューブ使用'], values: [0,2,4,6] },
      { name: '飲み込み動作', desc: '（MGADL）', options: ['正常', 'まれにむせる', '頻回のむせのため食事に工夫を要す', '栄養チューブ使用'], values: [0,2,5,6] },
      { name: 'MGによる呼吸状態', desc: '（MGADL）', options: ['正常', '活動時息切れ', '安静時息切れ', '呼吸補助装置使用'], values: [0,2,4,9] },
      { name: '頸の前屈／背屈筋力', desc: '（弱い方を選択、医師の診察）', options: ['正常', '軽度低下', '中等度低下（おおよそ半減）', '重度低下'], values: [0,1,3,4] },
      { name: '上肢の挙上筋力', desc: '（医師の診察）', options: ['正常', '軽度低下', '中等度低下（おおよそ半減）', '重度低下'], values: [0,2,4,5] },
      { name: '下肢の挙上筋力', desc: '（医師の診察）', options: ['正常', '軽度低下', '中等度低下（おおよそ半減）', '重度低下'], values: [0,2,4,5] }
    ]
  },
  adlToMgc: [
    { adl: 0, mgc: 3 }, // 会話 → 会話、発音
    { adl: 1, mgc: 4 }, // 咀嚼 → 咬む動作
    { adl: 2, mgc: 5 }, // 嚥下 → 飲み込み動作
    { adl: 3, mgc: 6 }  // 呼吸 → MGによる呼吸状態
  ]
};
```

- [ ] **Step 2: フローエンジンを実装**

```javascript
const state = {
  phase: 'adl',     // 'adl' | 'mgc' | 'adl-result' | 'mgc-result' | 'final'
  index: 0,         // 現在の項目index
  adlScores: Array(8).fill(-1),  // -1 = 未回答
  mgcScores: Array(10).fill(-1),
};

function currentScale() { return DATA[state.phase]; }
function currentScores() { return state.phase === 'adl' ? state.adlScores : state.mgcScores; }
function totalItems() { return currentScale().items.length; }

// --- Progress Bar ---
function renderProgress() {
  const bar = document.getElementById('progress-bar');
  const counter = document.getElementById('progress-counter');
  const n = totalItems();
  bar.innerHTML = '';
  for (let i = 0; i < n; i++) {
    const seg = document.createElement('div');
    seg.className = 'progress-segment' + (i <= state.index ? ' done' : '');
    bar.appendChild(seg);
  }
  counter.textContent = `${state.index + 1} / ${n}`;
}

// --- Question Screen ---
function renderQuestion() {
  const scale = currentScale();
  const item = scale.items[state.index];
  const scores = currentScores();
  const container = document.getElementById('slide-container');

  let html = `<div class="screen active" id="q-screen">`;
  html += `<div class="question-title">${item.name}</div>`;
  if (item.desc) html += `<div class="question-subtitle">${item.desc}</div>`;
  else html += `<div class="question-subtitle">該当する状態をタップしてください</div>`;

  item.options.forEach((opt, i) => {
    const selected = scores[state.index] === i ? ' selected' : '';
    html += `<div class="option-card${selected}" data-idx="${i}" onclick="selectOption(${i})">`;
    html += `<div class="option-badge">${item.values[i]}</div>`;
    html += `<div class="option-text">${opt}</div>`;
    html += `</div>`;
  });

  if (state.index > 0) {
    html += `<div class="back-link" onclick="goBack()">← 前の項目に戻る</div>`;
  }
  html += `</div>`;

  container.innerHTML = html;
  renderProgress();
  hideAllResults();
}

function hideAllResults() {
  ['adl-result', 'mgc-result', 'final-result'].forEach(id => {
    document.getElementById(id).className = 'screen';
  });
}

// --- Selection + Auto-advance ---
function selectOption(optIdx) {
  const scores = currentScores();
  scores[state.index] = optIdx;

  // Visual feedback
  document.querySelectorAll('#q-screen .option-card').forEach((el, i) => {
    el.classList.toggle('selected', i === optIdx);
  });

  // Auto-advance after 300ms
  setTimeout(() => {
    if (state.index < totalItems() - 1) {
      state.index++;
      const container = document.getElementById('slide-container');
      container.querySelector('.screen').className = 'screen active slide-left-enter';
      renderQuestion();
    } else {
      showResult();
    }
  }, 300);
}

// --- Go Back ---
function goBack() {
  if (state.index > 0) {
    state.index--;
    const container = document.getElementById('slide-container');
    container.querySelector('.screen').className = 'screen active slide-right-enter';
    renderQuestion();
  }
}

// --- Stub for Task 3 ---
function showResult() { console.log('Flow complete — implemented in Task 3'); }

// --- Scroll to top on screen change ---
function scrollTop() { window.scrollTo({ top: 0, behavior: 'smooth' }); }

// --- Init ---
document.addEventListener('DOMContentLoaded', () => {
  renderQuestion();
});
```

- [ ] **Step 3: ブラウザで動作確認**

ブラウザで開き、ADL 1問目「会話」が表示されること。選択肢をタップすると次の項目に自動遷移すること。「← 前の項目に戻る」で戻れること。プログレスバーが正しく更新されること。

- [ ] **Step 4: コミット**

```bash
git add index_iOS.html
git commit -m "feat: data model + flow engine for ADL 8-question stepper"
```

---

## Task 3: 結果画面（ADL + MGC + 最終）

スコア円、項目一覧、ボタンを含む結果画面を実装。

**Files:**
- Modify: `index_iOS.html` (`<script>` 内に `showResult()` 等を追加)

- [ ] **Step 1: showResult() を実装**

```javascript
function showResult() {
  document.getElementById('slide-container').innerHTML = '';
  document.getElementById('progress-bar').innerHTML = '';
  document.getElementById('progress-counter').textContent = '';
  document.getElementById('qr-container').classList.remove('visible');
  scrollTop();

  if (state.phase === 'adl') {
    renderResultScreen('adl');
  } else if (state.phase === 'mgc') {
    renderResultScreen('mgc');
  }
}

function calcTotal(phase) {
  const scale = DATA[phase];
  const scores = phase === 'adl' ? state.adlScores : state.mgcScores;
  return scores.reduce((sum, sel, i) => sum + (sel >= 0 ? scale.items[i].values[sel] : 0), 0);
}

function scoreColorClass(phase, value) {
  if (value === 0) return 'score-0';
  if (phase === 'adl') return value <= 2 ? 'score-low' : 'score-high';
  return value <= 3 ? 'score-low' : 'score-high'; // MGC thresholds
}

function renderResultScreen(phase) {
  const scale = DATA[phase];
  const scores = phase === 'adl' ? state.adlScores : state.mgcScores;
  const total = calcTotal(phase);
  const el = document.getElementById(phase + '-result');

  let html = '';
  // Score circle
  html += `<div class="score-circle"><div class="score-value">${total}</div><div class="score-max">/ ${scale.maxScore}点</div></div>`;
  html += `<div class="score-label">${scale.title}</div>`;

  // Item summary
  html += '<div class="item-summary">';
  scale.items.forEach((item, i) => {
    const sel = scores[i];
    const val = sel >= 0 ? item.values[sel] : 0;
    const cls = scoreColorClass(phase, val);
    html += `<div class="item-row"><span class="item-name">${item.name}</span><span class="item-score ${cls}">${val}点</span></div>`;
  });
  html += '</div>';

  // Buttons
  if (phase === 'adl') {
    html += `<button class="btn btn-primary" onclick="startMgc()">MGCに進む →</button>`;
    html += `<div class="btn-row"><button class="btn btn-secondary" onclick="copyResult('adl')">コピー</button><button class="btn btn-secondary" onclick="toggleQR('adl')">QRコード</button></div>`;
    html += `<button class="btn btn-ghost" onclick="resetAdl()">やり直す</button>`;
  } else {
    html += `<button class="btn btn-primary" onclick="showFinal()">最終結果を見る</button>`;
    html += `<div class="btn-row"><button class="btn btn-secondary" onclick="copyResult('mgc')">コピー</button><button class="btn btn-secondary" onclick="toggleQR('mgc')">QRコード</button></div>`;
    html += `<button class="btn btn-ghost" onclick="resetMgc()">やり直す</button>`;
  }

  el.innerHTML = html;
  el.className = 'screen active';
}

function showFinal() {
  hideAllResults();
  document.getElementById('qr-container').classList.remove('visible');
  scrollTop();
  const adlTotal = calcTotal('adl');
  const mgcTotal = calcTotal('mgc');
  const el = document.getElementById('final-result');

  let html = '<div class="dual-scores">';
  html += `<div><div class="score-circle"><div class="score-value">${adlTotal}</div><div class="score-max">/ ${DATA.adl.maxScore}点</div></div><div class="score-label">MG-ADL</div></div>`;
  html += `<div><div class="score-circle"><div class="score-value">${mgcTotal}</div><div class="score-max">/ ${DATA.mgc.maxScore}点</div></div><div class="score-label">MGC</div></div>`;
  html += '</div>';

  // 統合結果テキスト表示
  html += '<div class="item-summary">';
  html += '<div style="font-weight:700;margin-bottom:8px;">MG-ADL</div>';
  DATA.adl.items.forEach((item, i) => {
    const sel = state.adlScores[i];
    const val = sel >= 0 ? item.values[sel] : 0;
    html += `<div class="item-row"><span class="item-name">${item.name}</span><span class="item-score ${scoreColorClass('adl', val)}">${val}点</span></div>`;
  });
  html += '<div style="font-weight:700;margin:12px 0 8px;padding-top:8px;border-top:1px solid var(--mg-divider);">MG Composite</div>';
  DATA.mgc.items.forEach((item, i) => {
    const sel = state.mgcScores[i];
    const val = sel >= 0 ? item.values[sel] : 0;
    html += `<div class="item-row"><span class="item-name">${item.name}</span><span class="item-score ${scoreColorClass('mgc', val)}">${val}点</span></div>`;
  });
  html += '</div>';

  html += `<div class="btn-row"><button class="btn btn-secondary" onclick="copyResult('final')">コピー</button><button class="btn btn-secondary" onclick="toggleQR('final')">QRコード</button></div>`;
  html += `<button class="btn btn-ghost" onclick="resetAll()">最初からやり直す</button>`;

  el.innerHTML = html;
  el.className = 'screen active';
}
```

- [ ] **Step 2: MGCフロー開始 + ADLマッピングを実装**

```javascript
function applyAdlToMgcMapping() {
  DATA.adlToMgc.forEach(m => {
    state.mgcScores[m.mgc] = state.adlScores[m.adl];
  });
}

function startMgc() {
  hideAllResults();
  state.phase = 'mgc';
  state.index = 0;
  applyAdlToMgcMapping();
  scrollTop();
  renderQuestion();
}

function resetAdl() {
  state.phase = 'adl';
  state.index = 0;
  state.adlScores.fill(-1);
  hideAllResults();
  document.getElementById('qr-container').classList.remove('visible');
  scrollTop();
  renderQuestion();
}

function resetMgc() {
  // MGCリセット後もADLマッピングを再適用
  state.phase = 'mgc';
  state.index = 0;
  state.mgcScores.fill(-1);
  applyAdlToMgcMapping();
  hideAllResults();
  document.getElementById('qr-container').classList.remove('visible');
  scrollTop();
  renderQuestion();
}

function resetAll() {
  state.phase = 'adl';
  state.index = 0;
  state.adlScores.fill(-1);
  state.mgcScores.fill(-1);
  hideAllResults();
  document.getElementById('qr-container').classList.remove('visible');
  scrollTop();
  renderQuestion();
}
```

- [ ] **Step 3: ブラウザで全フロー確認**

ADL 8問を回答 → ADL結果画面（スコア円+項目一覧） → 「MGCに進む」 → MGC 10問（重複4項目はプリセレクト済み） → MGC結果 → 「最終結果」 → 両スコア表示。「やり直す」で各フェーズをリセットできること。

- [ ] **Step 4: コミット**

```bash
git add index_iOS.html
git commit -m "feat: result screens (ADL/MGC/final) with score circle + item summary"
```

---

## Task 4: 結果テキスト生成 + コピー機能

既存フォーマットを維持した結果テキスト生成とクリップボードコピー。

**Files:**
- Modify: `index_iOS.html` (`<script>` 内)

- [ ] **Step 1: テキスト生成関数**

```javascript
function generateResultText(phase) {
  const date = new Date().toLocaleDateString('ja-JP');
  if (phase === 'final') {
    let text = 'MG評価スケール 総合結果\n';
    text += '日付: ' + date + '\n';
    text += '========================\n\n';
    text += generateScaleText('adl', false);
    text += '\n\n';
    text += generateScaleText('mgc', false);
    return text;
  }
  return generateScaleText(phase, true);
}

function generateScaleText(phase, includeHeader) {
  const scale = DATA[phase];
  const scores = phase === 'adl' ? state.adlScores : state.mgcScores;
  const total = calcTotal(phase);
  let text = '';
  if (includeHeader !== false) {
    const title = phase === 'adl' ? 'MG-ADLスケール結果' : 'MG Composite Scale結果';
    text += title + '\n';
    text += '日付: ' + new Date().toLocaleDateString('ja-JP') + '\n';
    text += '------------------------\n';
  } else {
    text += (phase === 'adl' ? '【MG-ADL スケール】' : '【MG Composite Scale】') + '\n';
    text += '合計点: ' + total + '/' + scale.maxScore + '点\n\n';
  }
  scale.items.forEach((item, i) => {
    const sel = scores[i];
    const val = sel >= 0 ? item.values[sel] : 0;
    const optText = sel >= 0 ? item.options[sel] : '未回答';
    if (phase === 'mgc') {
      text += item.name + '\n';
      text += '  ' + val + '点: ' + optText + '\n';
    } else {
      text += item.name + ': ' + val + '点 (' + optText + ')\n';
    }
  });
  if (includeHeader !== false) {
    text += '------------------------\n';
    text += '合計点: ' + total + '/' + scale.maxScore + '点';
  }
  return text;
}

async function copyResult(phase) {
  const text = generateResultText(phase);
  try {
    await navigator.clipboard.writeText(text);
    showToast('コピーしました');
  } catch {
    // Fallback for older iOS
    const ta = document.createElement('textarea');
    ta.value = text;
    document.body.appendChild(ta);
    ta.select();
    document.execCommand('copy');
    document.body.removeChild(ta);
    showToast('コピーしました');
  }
}

function showToast(msg) {
  let toast = document.getElementById('toast');
  if (!toast) {
    toast = document.createElement('div');
    toast.id = 'toast';
    toast.style.cssText = 'position:fixed;bottom:80px;left:50%;transform:translateX(-50%);background:rgba(0,0,0,0.8);color:#fff;padding:10px 20px;border-radius:20px;font-size:14px;z-index:999;transition:opacity 0.3s;';
    document.body.appendChild(toast);
  }
  toast.textContent = msg;
  toast.style.opacity = '1';
  setTimeout(() => { toast.style.opacity = '0'; }, 2000);
}
```

- [ ] **Step 2: ブラウザでコピー動作確認**

ADL結果画面で「コピー」をタップ → クリップボードに既存フォーマットのテキストが入ること。トースト「コピーしました」が表示されること。

- [ ] **Step 3: コミット**

```bash
git add index_iOS.html
git commit -m "feat: result text generation + clipboard copy with toast"
```

---

## Task 5: QRコード生成（qrcode-generator インライン）

QRコード生成ライブラリをインライン埋め込みし、結果テキストのQRを表示。

**Files:**
- Modify: `index_iOS.html` (`<script>` 内)

- [ ] **Step 1: qrcode-generator のminified版を取得・インライン化**

```bash
cd /tmp && npm pack qrcode-generator@1.4.4 && tar -xzf qrcode-generator-1.4.4.tgz
cat package/qrcode.js | npx terser --compress --mangle > /tmp/qrcode.min.js
wc -c /tmp/qrcode.min.js  # ~8KB expected
```

生成された `/tmp/qrcode.min.js` の内容を `index_iOS.html` の `<script>` タグ冒頭（DATAモデル定義の前）にペーストする。`qrcode()` グローバル関数が使えるようになる。

- [ ] **Step 2: toggleQR() を実装**

```javascript
function toggleQR(phase) {
  const qrEl = document.getElementById('qr-container');
  // 既に表示中なら非表示に
  if (qrEl.classList.contains('visible') && qrEl.dataset.phase === phase) {
    qrEl.classList.remove('visible');
    return;
  }
  // QR生成
  const text = generateResultText(phase);
  const qr = qrcode(0, 'L');
  qr.addData(text);
  qr.make();

  const canvas = document.getElementById('qr-canvas');
  const ctx = canvas.getContext('2d');
  const size = 200;
  canvas.width = size;
  canvas.height = size;
  ctx.fillStyle = '#FFFFFF';
  ctx.fillRect(0, 0, size, size);

  const modules = qr.getModuleCount();
  const cellSize = size / modules;
  ctx.fillStyle = '#000000';
  for (let r = 0; r < modules; r++) {
    for (let c = 0; c < modules; c++) {
      if (qr.isDark(r, c)) {
        ctx.fillRect(c * cellSize, r * cellSize, cellSize, cellSize);
      }
    }
  }

  qrEl.dataset.phase = phase;
  // QRコンテナを現在のアクティブな結果画面の末尾に移動
  const activeScreen = document.querySelector('.screen.active');
  if (activeScreen) activeScreen.appendChild(qrEl);
  qrEl.classList.add('visible');
}
```

- [ ] **Step 3: ブラウザでQR動作確認**

ADL結果画面で「QRコード」タップ → QRコード画像が表示されること。再度タップで非表示。MGC結果・最終結果でも同様。QRをスマホカメラで読み取ると結果テキストが表示されること。

- [ ] **Step 4: コミット**

```bash
git add index_iOS.html
git commit -m "feat: QR code generation from result text (qrcode-generator inline)"
```

---

## Task 6: 最終調整 + 動作検証

全フローの統合テスト、エッジケース対応、コミット。

**Files:**
- Modify: `index_iOS.html`

- [ ] **Step 1: MGCプリセレクト表示の確認**

MGCフロー中、ADL重複4項目（index 3,4,5,6）に遷移した時、ADLで選択した値が `.selected` クラス付きで表示されることを確認。ユーザーが別の値をタップすると上書きされ、ADLスコアには影響しないことを確認。

- [ ] **Step 2: エッジケース対応**

以下を確認・修正:
- ADL全問0点で回答 → 結果画面で `0/24` が表示されること
- MGC全問最高点で回答 → `50/50` が表示されること
- QR生成後にやり直し → QRコンテナが非表示に戻ること
- 最終結果からの「最初からやり直す」で全状態がリセットされること

```javascript
// resetAll 内に追加
document.getElementById('qr-container').classList.remove('visible');
```

- [ ] **Step 3: iOS Safari実機確認**

iPhone実機またはシミュレータで以下を確認:
- タップレスポンス（300ms delay なし — `touch-action: manipulation` を必要に応じて追加）
- スクロールの滑らかさ
- QR長押しでの画像保存
- 文字サイズの適切さ（16px以上）

必要に応じてCSSに追加:
```css
.option-card { touch-action: manipulation; }
```

- [ ] **Step 4: 最終コミット**

```bash
git add index_iOS.html
git commit -m "feat: complete mobile UI redesign — Calmi flow + QR code generation"
```

---

## Summary

| Task | 内容 | 見積もり |
|------|------|---------|
| 1 | HTML骨格 + CSS | 基盤 |
| 2 | データモデル + フローエンジン | コア |
| 3 | 結果画面（ADL/MGC/最終） | コア |
| 4 | テキスト生成 + コピー | 機能 |
| 5 | QRコード生成 | 機能 |
| 6 | 最終調整 + 検証 | 仕上げ |
