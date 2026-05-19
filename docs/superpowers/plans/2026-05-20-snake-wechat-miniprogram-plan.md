# 萌萌贪吃蛇微信小程序（单页即玩）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: 使用 `superpowers:subagent-driven-development`（推荐）或 `superpowers:executing-plans` 按任务执行。本计划使用 checkbox（`- [ ]`）追踪。

**Goal:** 将现有 Canvas 贪吃蛇迁移为微信小程序（单页即玩），网格升级为 30×30，版本号 1.2.1，使用 wx storage 持久化数据，使用内置“软萌泡泡糖”短音效替代 WebAudio 合成音。

**Architecture:** 采用 `canvas type="2d"` 获取 2D ctx，最大化复用现有渲染与游戏循环逻辑；在小程序页面内重建 UI（HUD/overlay/dpad），并将 storage/audio/事件系统替换为 wx 能力。把核心逻辑封装为可独立测试的纯 JS 模块（不依赖 wx），页面层只负责输入、渲染上下文与平台适配。

**Tech Stack:** 微信小程序原生（WXML/WXSS/JS） + Canvas 2D（node） + `wx.getStorageSync`/`wx.setStorageSync` + `InnerAudioContext`

---

## 0. 文件结构与职责（锁定）

**Files (Create):**
- `miniprogram/app.js`
- `miniprogram/app.json`
- `miniprogram/app.wxss`
- `miniprogram/project.config.json`（AppID: `wxca36be1a98b67bba`）
- `miniprogram/pages/game/game.wxml`
- `miniprogram/pages/game/game.wxss`
- `miniprogram/pages/game/game.json`
- `miniprogram/pages/game/game.js`
- `miniprogram/utils/core.js`（平台无关核心：状态机/规则/绘制调用接口）
- `miniprogram/utils/storage.js`（wx storage 封装）
- `miniprogram/utils/audio.js`（音效池）
- `miniprogram/assets/audio/*.mp3`（软萌泡泡糖短音效，见 Task 5）

**Files (Modify):**
- `README.md`（补充小程序运行方式与目录说明；版本标注 1.2.1）

**Notes:**
- 网页版 `index.html` 保留不删；小程序为并行目标产物。

---

## Task 1: 初始化小程序工程骨架（单页即玩）

**Files:**
- Create: `miniprogram/app.js`
- Create: `miniprogram/app.json`
- Create: `miniprogram/app.wxss`
- Create: `miniprogram/project.config.json`
- Create: `miniprogram/pages/game/game.{wxml,wxss,js,json}`

- [ ] **Step 1: 创建 app 基础文件**

`miniprogram/app.js`
```js
App({});
```

`miniprogram/app.json`
```json
{
  "pages": ["pages/game/game"],
  "window": {
    "navigationBarTitleText": "萌萌贪吃蛇"
  }
}
```

`miniprogram/app.wxss`
```css
page { background: #fff5f8; }
```

- [ ] **Step 2: 创建 project.config.json（含 AppID）**

`miniprogram/project.config.json`
```json
{
  "miniprogramRoot": "miniprogram/",
  "projectname": "snake-miniprogram",
  "appid": "wxca36be1a98b67bba",
  "setting": {
    "urlCheck": false
  }
}
```

- [ ] **Step 3: 创建单页 game 页面骨架（先能跑起来）**

`miniprogram/pages/game/game.json`
```json
{
  "navigationBarTitleText": "萌萌贪吃蛇"
}
```

`miniprogram/pages/game/game.wxml`
```xml
<view class="page">
  <view class="title">
    <text class="h1">萌萌贪吃蛇</text>
    <text class="ver">snake.1.2.1</text>
  </view>

  <view class="hud">
    <view class="pill">得分 <text class="accent">{{score}}</text></view>
    <view class="pill">最高 <text class="accent">{{highScore}}</text></view>
    <view class="sound">
      <button class="sound-btn" bindtap="onToggleMute" aria-label="静音切换">{{muted ? "🔇" : "🔊"}}</button>
      <slider class="vol" min="0" max="1" step="0.01" value="{{volume}}" bindchanging="onVolumeChanging" />
    </view>
  </view>

  <view class="wrap">
    <canvas id="game" type="2d" class="canvas"></canvas>

    <view class="overlay" wx:if="{{overlayVisible}}">
      <view class="overlay-card">
        <text class="overlay-title">{{overlayTitle}}</text>
        <text class="overlay-msg">{{overlayMsg}}</text>
        <button class="btn" bindtap="onPrimaryAction">{{overlayBtnText}}</button>
      </view>
    </view>
  </view>
</view>
```

`miniprogram/pages/game/game.wxss`（最小样式，后续再精修）
```css
.page{ padding: 16px; }
.title{ display:flex; align-items:baseline; gap:8px; justify-content:center; margin:8px 0 12px; }
.h1{ font-weight: 800; color:#ff6b9d; font-size: 22px; }
.ver{ color:#9b8aa8; font-size: 12px; }
.hud{ display:flex; gap:10px; flex-wrap:wrap; justify-content:center; margin-bottom:12px; }
.pill{ background:#fff; border:3px solid #ffd6e5; border-radius:999px; padding:6px 14px; font-weight:800; }
.accent{ color:#ff6b9d; }
.wrap{ position:relative; background:#fff; border:4px solid #ffd6e5; border-radius:20px; padding:10px; }
.canvas{ width: 100%; aspect-ratio: 1; background:#fffafc; border-radius:12px; }
.overlay{ position:absolute; inset:10px; display:flex; align-items:center; justify-content:center; background: rgba(255,250,252,0.88); border-radius:12px; }
.overlay-card{ display:flex; flex-direction:column; gap:12px; align-items:center; padding:16px; text-align:center; }
.overlay-title{ font-size:18px; font-weight:800; color:#ff6b9d; }
.overlay-msg{ color:#9b8aa8; font-weight:600; white-space:pre-line; }
.btn{ background: linear-gradient(180deg, #ff8fb8 0%, #ff6b9d 100%); color:#fff; font-weight:800; border-radius:999px; padding:10px 20px; }
.sound{ display:flex; align-items:center; gap:8px; }
.sound-btn{ width:44px; height:36px; padding:0; border-radius:12px; border:3px solid #ffd6e5; background:#fff; }
.vol{ width:120px; }
```

`miniprogram/pages/game/game.js`（先能显示 overlay）
```js
Page({
  data: {
    score: 0,
    highScore: 0,
    muted: false,
    volume: 0.8,
    overlayVisible: true,
    overlayTitle: "准备好了吗？",
    overlayMsg: "点按钮开始游戏～",
    overlayBtnText: "开始游戏"
  },
  onPrimaryAction() {
    // Task 3 接入 core 后替换
    this.setData({ overlayVisible: false });
  },
  onToggleMute() {},
  onVolumeChanging(e) {}
});
```

- [ ] **Step 4: 手动验证**
  - 用微信开发者工具打开 `snake/miniprogram`，可看到单页 UI，按钮点击可隐藏 overlay。

- [ ] **Step 5: Commit**
```bash
git add miniprogram README.md
git commit -m "feat(miniprogram): scaffold single-page game"
```

---

## Task 2: 实现 wx storage 封装（保持 key 与网页版一致）

**Files:**
- Create: `miniprogram/utils/storage.js`
- Modify: `miniprogram/pages/game/game.js`

- [ ] **Step 1: 实现 storage API**

`miniprogram/utils/storage.js`
```js
export const STORAGE_KEYS = {
  profile: "snake_profile_v1",
  audio: "snake_audio_settings_v1",
  difficulty: "snake_difficulty",
  high: (diff) => `snake_high_${diff}`,
};

export function loadJSON(key, fallback) {
  try {
    const v = wx.getStorageSync(key);
    if (!v) return fallback;
    return typeof v === "string" ? JSON.parse(v) : v;
  } catch {
    return fallback;
  }
}

export function saveJSON(key, obj) {
  try {
    wx.setStorageSync(key, JSON.stringify(obj));
  } catch {}
}
```

- [ ] **Step 2: 在 game 页初始化 audioSettings（muted/volume）**

在 `game.js`：
```js
import { STORAGE_KEYS, loadJSON, saveJSON } from "../../utils/storage";

function defaultAudioSettings() { return { muted: false, volume: 0.8 }; }
function normalizeAudioSettings(s) {
  if (typeof s?.muted !== "boolean") return defaultAudioSettings();
  const v = Number(s.volume ?? 0.8);
  return { muted: s.muted, volume: Math.max(0, Math.min(1, v)) };
}
```

onLoad：
```js
onLoad() {
  const audio = normalizeAudioSettings(loadJSON(STORAGE_KEYS.audio, defaultAudioSettings()));
  this.setData({ muted: audio.muted, volume: audio.volume });
}
```

- [ ] **Step 3: 响应 UI 修改并持久化**

```js
onToggleMute() {
  const muted = !this.data.muted;
  this.setData({ muted });
  saveJSON(STORAGE_KEYS.audio, { muted, volume: this.data.volume });
},
onVolumeChanging(e) {
  const volume = Math.max(0, Math.min(1, Number(e.detail.value)));
  this.setData({ volume });
  saveJSON(STORAGE_KEYS.audio, { muted: this.data.muted, volume });
}
```

- [ ] **Step 4: 手动验证**
  - 改音量/静音后重启小程序，设置仍保留。

- [ ] **Step 5: Commit**
```bash
git add miniprogram/utils/storage.js miniprogram/pages/game/game.js
git commit -m "feat(miniprogram): add wx storage for audio settings"
```

---

## Task 3: 提取平台无关 core（30×30 网格、tick/loop、输入队列）

**Files:**
- Create: `miniprogram/utils/core.js`
- Modify: `miniprogram/pages/game/game.js`

目标：把“规则/状态/渲染调用”封装成可在小程序页面内驱动的 core，避免 page.js 变巨无霸。

- [ ] **Step 1: 定义 core 的输入/输出接口（无 wx 依赖）**

`miniprogram/utils/core.js`
```js
export function createGameCore(opts) {
  const GRID = 30;
  const FOOD_SCORE = 10;
  const MAX_QUEUE = 3;

  const DIFFICULTY = {
    easy:   { label: "容易", initialSpeed: 380, minSpeed: 200, speedStep: 2 },
    medium: { label: "中等", initialSpeed: 300, minSpeed: 160, speedStep: 4 },
    hard:   { label: "困难", initialSpeed: 220, minSpeed: 120, speedStep: 6 },
  };

  const STATE = { MENU:"menu", PLAYING:"playing", PAUSED:"paused", OVER:"over", EXPLODING:"exploding" };

  let state = STATE.MENU;
  let difficulty = opts?.difficulty || "medium";
  let snake = [];
  let direction = { x: 1, y: 0 };
  let inputQueue = [];
  let food = { x: 0, y: 0 };
  let score = 0;
  let moveInterval = DIFFICULTY[difficulty].initialSpeed;

  // 道具/effects/成就/皮肤：从网页版迁移同名结构（后续 step 补齐）

  function isOpposite(a,b){ return a.x===-b.x && a.y===-b.y; }
  function isSameDir(a,b){ return a.x===b.x && a.y===b.y; }

  function queueDirection(dir) {
    const base = inputQueue.length ? inputQueue[inputQueue.length-1] : direction;
    if (isOpposite(base, dir)) return;
    if (isSameDir(base, dir)) return;
    inputQueue.push({ x: dir.x, y: dir.y });
    if (inputQueue.length > MAX_QUEUE) inputQueue.shift();
  }

  function reset() {
    const mid = Math.floor(GRID / 2);
    snake = [{x:mid-2,y:mid},{x:mid-1,y:mid},{x:mid,y:mid}];
    direction = { x: 1, y: 0 };
    inputQueue = [];
    score = 0;
    moveInterval = DIFFICULTY[difficulty].initialSpeed;
    // randomFood() 在下一步实现
  }

  function setDifficulty(d) { if (DIFFICULTY[d]) difficulty = d; }
  function getState() { return { state, difficulty, score, snake, food }; }

  return {
    GRID,
    STATE,
    DIFFICULTY,
    reset,
    queueDirection,
    setDifficulty,
    getState,
    // tick/loop/draw 将在后续 step 填充完整实现
  };
}
```

- [ ] **Step 2: 在 game 页创建 core 实例并保留旧 overlay 流程**

`game.js`：
```js
import { createGameCore } from "../../utils/core";
let core = null;
Page({
  onLoad() {
    core = createGameCore({ difficulty: "medium" });
  }
});
```

- [ ] **Step 3: Commit**
```bash
git add miniprogram/utils/core.js miniprogram/pages/game/game.js
git commit -m "feat(miniprogram): add game core skeleton (GRID=30)"
```

---

## Task 4: Canvas 2D 初始化 + DPR 适配 + 渲染管线接入 core

**Files:**
- Modify: `miniprogram/pages/game/game.js`
- Modify: `miniprogram/pages/game/game.wxml`
- Modify: `miniprogram/utils/core.js`

- [ ] **Step 1: 在 game 页 onReady 获取 canvas node 与 2d ctx**

`game.js`：
```js
onReady() {
  const sys = wx.getSystemInfoSync();
  const dpr = sys.pixelRatio || 2;
  this._dpr = dpr;

  const query = wx.createSelectorQuery().in(this);
  query.select("#game").node().exec((res) => {
    const canvas = res[0].node;
    const ctx = canvas.getContext("2d");

    const cssSize = sys.windowWidth - 32;
    const cell = Math.floor(cssSize / 30);
    const boardSize = cell * 30;
    const offset = Math.floor((cssSize - boardSize) / 2);

    canvas.width = cssSize * dpr;
    canvas.height = cssSize * dpr;
    ctx.scale(dpr, dpr);
    ctx.translate(offset, offset);

    this._canvas = canvas;
    this._ctx = ctx;
    this._cssSize = cssSize;
    this._cell = cell;
    this._boardSize = boardSize;

    // core 需要 cell/boardSize 才能画：通过 opts 传入或 setRenderConfig
    // 下一步会把渲染接入 loop
  });
}
```

- [ ] **Step 2: 在 core 中加入 draw(ctx, renderCfg, now, dt)**

要求：沿用网页版 draw 流程（网格缓存、食物、道具、蛇、粒子、飘字）。

（此步会复制并改造 `index.html` 中相关绘制函数到 `core.js`，确保不引用 DOM。）

- [ ] **Step 3: 在 page 中启动 loop（仅在 PLAYING 时 tick）**
  - `this._raf = canvas.requestAnimationFrame(loop)`（若不可用 fallback `setTimeout`）
  - 每帧 `core.draw(this._ctx, cfg, now, dt)`
  - 计时达到 `effectiveMoveInterval` 时 `core.tick(now)`

- [ ] **Step 4: 手动验证**
  - 能看到 30×30 网格（或背景），且画面不模糊

- [ ] **Step 5: Commit**
```bash
git add miniprogram/pages/game/game.js miniprogram/utils/core.js
git commit -m "feat(miniprogram): init canvas 2d with dpr and render loop"
```

---

## Task 5: 内置音效系统（软萌泡泡糖）与事件绑定

**Files:**
- Create: `miniprogram/utils/audio.js`
- Create: `miniprogram/assets/audio/*.mp3`
- Modify: `miniprogram/pages/game/game.js`
- Modify: `miniprogram/utils/core.js`（触发事件回调）

- [ ] **Step 1: 实现音效池（避免频繁创建导致延迟）**

`miniprogram/utils/audio.js`
```js
export function createSfx(audioSettingsGetter) {
  const pool = new Map(); // name -> InnerAudioContext[]
  const FILES = {
    start: "assets/audio/start.mp3",
    pause: "assets/audio/pause.mp3",
    chomp: "assets/audio/chomp.mp3",
    die: "assets/audio/die.mp3",
    powerup_speed: "assets/audio/powerup_pick_speed.mp3",
    powerup_shield: "assets/audio/powerup_pick_shield.mp3",
    powerup_double: "assets/audio/powerup_pick_double.mp3",
    powerup_expire: "assets/audio/powerup_expire.mp3",
    shield_save: "assets/audio/shield_save.mp3",
    achievement: "assets/audio/achievement.mp3",
    skin_switch: "assets/audio/skin_switch.mp3",
  };

  function getCtx(name) {
    const list = pool.get(name) || [];
    for (const a of list) {
      if (!a._busy) return a;
    }
    const a = wx.createInnerAudioContext();
    a.src = FILES[name];
    a.obeyMuteSwitch = false;
    a._busy = false;
    a.onEnded(() => { a._busy = false; });
    a.onStop(() => { a._busy = false; });
    list.push(a);
    pool.set(name, list);
    return a;
  }

  function play(name) {
    const s = audioSettingsGetter();
    if (s?.muted) return;
    const a = getCtx(name);
    a._busy = true;
    a.volume = Math.max(0, Math.min(1, Number(s?.volume ?? 0.8)));
    try { a.seek(0); } catch {}
    a.play();
  }

  return { play };
}
```

- [ ] **Step 2: 在 game 页初始化 sfx，并把 core 的事件映射到 sfx.play**
  - `onStart` → `start`
  - `onPauseToggle` → `pause`
  - `onEatFood` → `chomp`
  - `onDie` → `die`
  - `onPowerupPickup(type)` → `powerup_speed|shield|double`
  - `onPowerupExpire` → `powerup_expire`
  - `onShieldSave` → `shield_save`
  - `onAchievementUnlocked` → `achievement`
  - `onSkinChanged` → `skin_switch`

- [ ] **Step 3: 放入音频文件（软萌泡泡糖）**
  - 将文件放到 `miniprogram/assets/audio/`，保持文件名与 spec 一致
  - 手动验证：真机/模拟器均能播放，无明显延迟（必要时提前 warm-up）

- [ ] **Step 4: Commit**
```bash
git add miniprogram/utils/audio.js miniprogram/assets/audio
git commit -m "feat(miniprogram): add bubblegum sfx pack with audio pool"
```

---

## Task 6: UI 功能补齐（难度按钮、dpad、overlay 面板：皮肤/成就）

**Files:**
- Modify: `miniprogram/pages/game/game.wxml`
- Modify: `miniprogram/pages/game/game.wxss`
- Modify: `miniprogram/pages/game/game.js`
- Modify: `miniprogram/utils/core.js`

- [ ] **Step 1: 难度按钮组（MENU/OVER 可用，PLAYING/PAUSED 禁用）**
  - UI：3 个按钮
  - 行为：调用 `core.setDifficulty()` 并刷新 highScore 展示

- [ ] **Step 2: dpad（四向按钮）**
  - `bindtouchstart` → `core.queueDirection`
  - 与 swipe 并存

- [ ] **Step 3: overlay 扩展为多面板**
  - `panel = main|skins|achievements`
  - MENU/OVER 可进入 skins/achievements
  - 皮肤：展示 6 套，锁定/解锁/当前选中
  - 成就：展示 8 项，已解锁高亮

- [ ] **Step 4: Commit**
```bash
git add miniprogram/pages/game miniprogram/utils/core.js
git commit -m "feat(miniprogram): add difficulty, dpad, overlay panels (skins/achievements)"
```

---

## Task 7: 存档与统计对齐网页版（profile/highScore/settings）+ 回归验证

**Files:**
- Modify: `miniprogram/utils/core.js`
- Modify: `miniprogram/pages/game/game.js`
- Modify: `README.md`

- [ ] **Step 1: profile 结构对齐并迁移缺失字段**
  - defaultProfile/version=1
  - ensure default skin unlocked

- [ ] **Step 2: highScore 按难度持久化（snake_high_<difficulty>）**
  - gameOver 结算时写入

- [ ] **Step 3: 回归清单（手动）**
  - 进入即玩
  - 30×30 网格与不模糊
  - 输入队列可靠、触控斜滑忽略生效
  - 道具生成/到期闪烁/拾取/效果计时/护盾救命
  - 成就/皮肤解锁与持久化
  - 音效与静音/音量

- [ ] **Step 4: README 更新（小程序运行指南）**
  - 微信开发者工具打开路径
  - 目录说明

- [ ] **Step 5: Commit**
```bash
git add README.md miniprogram
git commit -m "docs(miniprogram): add run guide and finalize storage alignment"
```

---

## 计划自检（执行前）
- [ ] 覆盖 spec：单页即玩、30×30、1.2.1、wx storage、2D canvas DPR、内置音效清单与触发点、皮肤/成就面板
- [ ] 无 TBD/TODO/“类似上一步”；每步包含具体代码与命令

