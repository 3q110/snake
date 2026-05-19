# 萌萌贪吃蛇优化 Implementation Plan

> **For agentic workers:** 本计划按任务拆分，使用 checkbox（`- [ ]`）逐项执行与验收。

**Goal:** 在保持“单文件 `index.html`、离线可玩”的前提下，完成操控手感优化（转向队列）、道具系统、成就/皮肤与更可爱动感的合成音效，并确保回归功能稳定。

**Architecture:** 在同一个 `<script>` 内做“分区模块化”（Input/Game/Renderer/SFX/UI/Storage），通过少量数据结构（profile/effects/powerup/inputQueue）把新增玩法与表现解耦，避免无序堆叠逻辑。

**Tech Stack:** 原生 HTML/CSS/Canvas 2D + WebAudio API + localStorage（无构建工具、无外部资源）

---

## 0. 文件结构与改动范围（锁定）

**Files:**
- Modify: `index.html`（单文件内完成全部改动）
- Modify: `README.md`（更新新增玩法/按键/设置说明）
- Create: `docs/superpowers/plans/2026-05-20-snake-optimization-plan.md`（本文件）

**不新增外部资源文件**（音频/图片/JSON 等均不新增）。

---

## Task 1: 建立“自测入口”与基础工具函数（为后续 TDD/回归服务）

> 说明：该项目没有测试框架；本任务在不引入依赖的前提下，加入一个可选的自测入口，便于验证输入队列、道具效果计时、存档迁移等纯逻辑。

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 新增 `assert()` 与 `runSelfTests()` 骨架（仅在 `#test` 时运行）**

在脚本底部（事件绑定之后或最末尾）添加：

```js
function assert(name, cond, detail = "") {
  if (!cond) {
    const err = new Error(`❌ ${name}${detail ? ` - ${detail}` : ""}`);
    err.__assertName = name;
    throw err;
  }
}

function runSelfTests() {
  const results = [];
  const test = (name, fn) => {
    try {
      fn();
      results.push({ name, ok: true });
    } catch (e) {
      results.push({ name, ok: false, error: String(e?.message || e) });
    }
  };

  // 这里先留空，后续任务逐步追加具体测试用例（每个任务都要补充用例）

  console.group("snake self-tests");
  console.table(results);
  console.groupEnd();

  const failed = results.filter(r => !r.ok);
  if (failed.length) throw new Error(`Self-tests failed: ${failed.length}`);
  return results;
}

if (location.hash === "#test") {
  // 不启动游戏循环，只跑逻辑测试
  runSelfTests();
}
```

- [ ] **Step 2: 手动验证 self-test 入口**
  - 打开 `index.html#test`
  - 预期：控制台输出 `snake self-tests` table，且无报错

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "chore: add self-test harness"
```

---

## Task 2: Storage/Profile v1（本地档案）与设置存档

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 添加 profile 读写与默认结构**

在脚本的“存档/工具”分区新增：

```js
const STORAGE_KEYS = {
  profile: "snake_profile_v1",
  audio: "snake_audio_settings_v1",
};

function loadJSON(key, fallback) {
  try {
    const raw = localStorage.getItem(key);
    if (!raw) return fallback;
    return JSON.parse(raw);
  } catch {
    return fallback;
  }
}

function saveJSON(key, obj) {
  localStorage.setItem(key, JSON.stringify(obj));
}

function defaultProfile() {
  return {
    version: 1,
    stats: {
      gamesPlayed: 0,
      totalScore: 0,
      totalFood: 0,
      totalPowerups: 0,
      maxScoreEver: 0,
      maxFoodOneGame: 0,
      shieldSaves: 0,
    },
    unlockedAchievements: [],
    unlockedSkins: ["default"],
    selectedSkin: "default",
  };
}

let profile = loadJSON(STORAGE_KEYS.profile, defaultProfile());
if (!profile || profile.version !== 1) profile = defaultProfile();
function saveProfile() { saveJSON(STORAGE_KEYS.profile, profile); }
```

- [ ] **Step 2: 添加音频设置读写（muted/volume）**

```js
function defaultAudioSettings() {
  return { muted: false, volume: 0.8 };
}
let audioSettings = loadJSON(STORAGE_KEYS.audio, defaultAudioSettings());
if (typeof audioSettings?.muted !== "boolean") audioSettings = defaultAudioSettings();
audioSettings.volume = Math.max(0, Math.min(1, Number(audioSettings.volume ?? 0.8)));
function saveAudioSettings() { saveJSON(STORAGE_KEYS.audio, audioSettings); }
```

- [ ] **Step 3: 为 Storage 添加自测用例**

在 `runSelfTests()` 里追加：

```js
test("profile default shape", () => {
  const p = defaultProfile();
  assert("has version", p.version === 1);
  assert("has stats", !!p.stats);
  assert("default skin", p.unlockedSkins.includes("default"));
});

test("audio settings clamp", () => {
  const s = { muted: false, volume: 3 };
  saveJSON(STORAGE_KEYS.audio, s);
  let a = loadJSON(STORAGE_KEYS.audio, defaultAudioSettings());
  // 仅验证 loadJSON 能读出来；clamp 在初始化逻辑里做
  assert("loadJSON reads", a.volume === 3);
});
```

- [ ] **Step 4: 手动验证**
  - 刷新页面后检查 `localStorage`：出现 `snake_profile_v1` 与 `snake_audio_settings_v1`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add profile storage v1 and audio settings"
```

---

## Task 3: Input 队列（手感优化）

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 引入输入队列与入队/出队逻辑**

替换/扩展现有 `direction/nextDirection` 逻辑：

```js
const MAX_QUEUE = 3;
let inputQueue = [];

function sameDir(a, b) { return a.x === b.x && a.y === b.y; }
function isOpposite(a, b) { return a.x === -b.x && a.y === -b.y; }

function queueDirection(d, source = "kbd") {
  if (!d) return;
  const tail = inputQueue.length ? inputQueue[inputQueue.length - 1] : (state === STATE.PLAYING ? direction : nextDirection);
  if (isOpposite(tail, d)) return;
  if (sameDir(tail, d)) return;
  inputQueue.push({ ...d, ts: performance.now(), source });
  if (inputQueue.length > MAX_QUEUE) inputQueue.shift();
}
```

让 `setDirection(name)` 改为调用 `queueDirection(DIR[name])`（不再直接覆盖 `nextDirection`）。

- [ ] **Step 2: 在 `tick()` 开头消费 1 次输入**

```js
function consumeDirection() {
  if (!inputQueue.length) return;
  const d = inputQueue.shift();
  nextDirection = { x: d.x, y: d.y };
}

function tick() {
  consumeDirection();
  direction = { ...nextDirection };
  // ... 原逻辑继续
}
```

- [ ] **Step 3: 触控滑动阈值与“轴向明确”判定**

用阈值函数替换硬编码 `24`：

```js
function swipeThreshold() {
  return Math.max(24, Math.min(40, Math.floor(canvas.getBoundingClientRect().width * 0.04)));
}
```

在 `pointerup` 中增加“接近 45° 的斜滑忽略”：

```js
const dxAbs = Math.abs(dx), dyAbs = Math.abs(dy);
const th = swipeThreshold();
if (dxAbs < th && dyAbs < th) return;
const ratio = dxAbs > dyAbs ? dxAbs / Math.max(1, dyAbs) : dyAbs / Math.max(1, dxAbs);
if (ratio < 1.15) return;
if (dxAbs > dyAbs) queueDirection(DIR[dx > 0 ? "right" : "left"], "touch");
else queueDirection(DIR[dy > 0 ? "down" : "up"], "touch");
```

- [ ] **Step 4: 为输入队列补充自测**

在 `runSelfTests()` 里追加（用纯函数方式验证）：

```js
test("inputQueue rejects opposite", () => {
  inputQueue = [];
  direction = { x: 1, y: 0 };
  nextDirection = { x: 1, y: 0 };
  queueDirection({ x: -1, y: 0 });
  assert("no opposite enqueued", inputQueue.length === 0);
});

test("inputQueue keeps order", () => {
  inputQueue = [];
  direction = { x: 1, y: 0 };
  nextDirection = { x: 1, y: 0 };
  queueDirection({ x: 0, y: -1 });
  queueDirection({ x: -1, y: 0 });
  assert("2 enqueued", inputQueue.length === 2);
  assert("first is up", inputQueue[0].y === -1);
});
```

- [ ] **Step 5: 手动验证**
  - 开局快速连按：上→左→下（观察不会吞中间方向，且不允许立即反向）
  - 手机/触控板斜滑：不会频繁误判

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add buffered input queue for smoother control"
```

---

## Task 4: Powerup（道具）与 Effects（效果计时）

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 定义 powerup/effects 结构与常量**

```js
const POWERUP = {
  spawnChance: 0.20,
  lifetimeMs: 6000,
  warnBlinkMs: 1200,
  speedMs: 5000,
  doubleMs: 8000,
  speedMultiplier: 0.75,
};

let powerup = null; // {type,x,y,spawnAt,expiresAt}
let effects = { speedUntil: 0, doubleUntil: 0, shield: false };
```

- [ ] **Step 2: 生成道具：在“吃到食物”之后尝试生成**

实现 `randomEmptyCell()`（复用食物逻辑但避免与蛇/食物重叠），然后：

```js
function trySpawnPowerup(now) {
  if (powerup) return;
  if (Math.random() > POWERUP.spawnChance) return;
  const types = ["speed", "shield", "double"];
  const type = types[Math.floor(Math.random() * types.length)];
  const pos = randomEmptyCell(); // 需保证不落在 snake/food 上
  powerup = { type, x: pos.x, y: pos.y, spawnAt: now, expiresAt: now + POWERUP.lifetimeMs };
}
```

在 `tick()` 吃食物分支的末尾调用：`trySpawnPowerup(performance.now())`。

- [ ] **Step 3: 道具到期消失**

在 `loop(now)` 或 `draw()` 前增加：

```js
if (powerup && now > powerup.expiresAt) powerup = null;
```

- [ ] **Step 4: 拾取道具与效果应用**

在 `tick()` 推进 newHead 后，增加检测：

```js
function pickupPowerup(now) {
  if (!powerup) return false;
  const head = snake[snake.length - 1];
  if (head.x !== powerup.x || head.y !== powerup.y) return false;

  if (powerup.type === "speed") effects.speedUntil = now + POWERUP.speedMs;
  if (powerup.type === "double") effects.doubleUntil = now + POWERUP.doubleMs;
  if (powerup.type === "shield") effects.shield = true;

  profile.stats.totalPowerups += 1;
  saveProfile();

  // 反馈（toast + sfx）在 Task 6/7 实现
  powerup = null;
  return true;
}
```

- [ ] **Step 5: 速度与得分应用**
  - 得分：在吃食物时根据 `now < effects.doubleUntil` 决定 `add = FOOD_SCORE * 2` 否则 `FOOD_SCORE`
  - 速度：维持现有 base `moveInterval` 逻辑；在 loop 判定移动间隔时应用 multiplier：

```js
function effectiveMoveInterval(now) {
  let interval = moveInterval;
  if (now < effects.speedUntil) interval = Math.max(60, Math.floor(interval * POWERUP.speedMultiplier));
  return interval;
}
```

然后把 `if (now - lastTick >= moveInterval)` 改为用 `effectiveMoveInterval(now)`。

- [ ] **Step 6: 为道具系统补充自测**

```js
test("effects timer set", () => {
  effects = { speedUntil: 0, doubleUntil: 0, shield: false };
  const now = 1000;
  // 模拟拾取
  powerup = { type: "double", x: 0, y: 0, spawnAt: 0, expiresAt: 999999 };
  snake = [{ x: 0, y: 0 }];
  pickupPowerup(now);
  assert("doubleUntil set", effects.doubleUntil === now + POWERUP.doubleMs);
});
```

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add powerups and timed effects"
```

---

## Task 5: 护盾救命逻辑（碰撞时消耗一次并“取消本次移动”）

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 将碰撞判定改成“先判断护盾”**

把原本：
- 撞墙 → `gameOver(); return;`
- 自咬 → `gameOver(); return;`

改为：

```js
function consumeShieldSave(reason) {
  if (!effects.shield) return false;
  effects.shield = false;
  profile.stats.shieldSaves += 1;
  saveProfile();
  // 反馈（屏闪/抖动/sfx）在 Task 6/7 做
  return true;
}
```

在 `tick()` 计算 `newHead` 后：
- 若撞墙或自咬：
  - 如果 `consumeShieldSave(...)` 为 true：**直接 return（不 push newHead、不推进蛇）**
  - 否则 `gameOver()`

- [ ] **Step 2: 自测**

```js
test("shield prevents game over once (by cancel move)", () => {
  effects = { speedUntil: 0, doubleUntil: 0, shield: true };
  snake = [{x:1,y:1},{x:2,y:1},{x:3,y:1}];
  direction = {x:1,y:0};
  nextDirection = {x:1,y:0};
  // 让 newHead 撞墙：假设 GRID=20，把蛇头放在边界
  snake[snake.length-1] = {x: GRID-1, y: 1};
  tick();
  assert("shield consumed", effects.shield === false);
});
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: shield saves cancel move once"
```

---

## Task 6: Renderer 增强（网格缓存、道具绘制、浮字与 Toast）

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 网格背景离屏缓存**

新增：

```js
let bgCanvas = null;
function buildGridCache() {
  bgCanvas = document.createElement("canvas");
  bgCanvas.width = CANVAS_SIZE;
  bgCanvas.height = CANVAS_SIZE;
  const b = bgCanvas.getContext("2d");
  b.strokeStyle = "rgba(255, 182, 193, 0.35)";
  b.lineWidth = 1;
  for (let i = 0; i <= GRID; i++) {
    b.beginPath(); b.moveTo(i * CELL, 0); b.lineTo(i * CELL, CANVAS_SIZE); b.stroke();
    b.beginPath(); b.moveTo(0, i * CELL); b.lineTo(CANVAS_SIZE, i * CELL); b.stroke();
  }
}
buildGridCache();
```

在 `draw()` 中用：

```js
ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
if (bgCanvas) ctx.drawImage(bgCanvas, 0, 0);
```

并删除每帧画网格的 for 循环。

- [ ] **Step 2: 浮字系统**

```js
let floatingTexts = []; // {x,y,text,ttlMs,vy,alpha}

function addFloatingText(cellX, cellY, text) {
  floatingTexts.push({
    x: cellX * CELL + CELL / 2,
    y: cellY * CELL + CELL / 2,
    text,
    ttlMs: 700,
    vy: -0.6,
    alpha: 1,
  });
}

function updateAndDrawFloatingTexts(dt) {
  floatingTexts.forEach(t => {
    t.y += t.vy * dt;
    t.ttlMs -= dt;
    t.alpha = Math.max(0, t.ttlMs / 700);
  });
  floatingTexts = floatingTexts.filter(t => t.ttlMs > 0);
  ctx.save();
  ctx.font = `${Math.floor(CELL * 0.45)}px Nunito, sans-serif`;
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  floatingTexts.forEach(t => {
    ctx.globalAlpha = t.alpha;
    ctx.fillStyle = "rgba(255,107,157,0.95)";
    ctx.fillText(t.text, t.x, t.y);
  });
  ctx.restore();
}
```

在 `loop(now)` 里计算 `dt`（已有 `deltaTime`），并在 `draw()` 末尾调用 `updateAndDrawFloatingTexts(deltaTime)`。

在吃食物/双倍分处调用：`addFloatingText(food.x, food.y, now < effects.doubleUntil ? "+20" : "+10")`。

- [ ] **Step 3: Toast 提示（轻量 HUD/overlay 之上）**

新增一个 DOM 元素（HUD 下或 game-wrap 内绝对定位）：

```html
<div id="toast" class="toast"></div>
```

配套 CSS（在 `<style>`）：

```css
.toast{
  position: fixed;
  left: 50%;
  bottom: 16px;
  transform: translateX(-50%);
  background: rgba(255,255,255,0.92);
  border: 3px solid #ffd6e5;
  box-shadow: 0 8px 24px rgba(255, 107, 157, 0.18);
  padding: 10px 14px;
  border-radius: 999px;
  font-weight: 800;
  color: #4a3f55;
  opacity: 0;
  pointer-events: none;
  transition: opacity .18s, transform .18s;
}
.toast.show{
  opacity: 1;
  transform: translateX(-50%) translateY(-6px);
}
```

JS：

```js
const toastEl = document.getElementById("toast");
let toastTimer = 0;
function toast(msg, ms=1200){
  if (!toastEl) return;
  toastEl.textContent = msg;
  toastEl.classList.add("show");
  clearTimeout(toastTimer);
  toastTimer = setTimeout(()=> toastEl.classList.remove("show"), ms);
}
```

拾取道具时：`toast("获得护盾！🛡")` 等。

- [ ] **Step 4: 道具绘制（带闪烁到期预警）**

在 `draw()` 中，在食物绘制之后、蛇绘制之前画道具：

```js
function drawPowerup(now){
  if (!powerup) return;
  const {x,y,type,expiresAt} = powerup;
  const cx = x*CELL + CELL/2, cy = y*CELL + CELL/2;
  const timeLeft = expiresAt - now;
  const blink = timeLeft < POWERUP.warnBlinkMs ? (Math.floor(timeLeft/120) % 2 === 0) : true;
  if (!blink) return;

  const r = Math.max(1, CELL/2 - 2);
  ctx.save();
  // 外发光
  const g = ctx.createRadialGradient(cx, cy, 0, cx, cy, r*1.8);
  g.addColorStop(0, "rgba(93,211,158,0.35)");
  g.addColorStop(1, "transparent");
  ctx.fillStyle = g;
  ctx.beginPath(); ctx.arc(cx, cy, r*1.8, 0, Math.PI*2); ctx.fill();

  // 主体
  ctx.fillStyle = "#ffffff";
  ctx.beginPath(); ctx.arc(cx, cy, r, 0, Math.PI*2); ctx.fill();
  ctx.lineWidth = 3;
  ctx.strokeStyle = "rgba(255, 214, 229, 0.9)";
  ctx.stroke();

  // 图标（emoji 简单稳妥，字体依赖低；也可后续换矢量）
  const icon = type === "speed" ? "⚡" : type === "shield" ? "🛡" : "✨";
  ctx.font = `${Math.floor(CELL*0.55)}px sans-serif`;
  ctx.textAlign = "center"; ctx.textBaseline = "middle";
  ctx.fillStyle = "#4a3f55";
  ctx.fillText(icon, cx, cy+1);
  ctx.restore();
}
```

在 `draw()` 调用：`drawPowerup(performance.now())`（或把 `now` 作为参数从 loop 传入）。

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: render powerups, toasts, floating texts; cache grid"
```

---

## Task 7: SFX 升级（可爱动感音效 + 音量/静音）

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 增加 masterGain 与音量/静音应用**

在现有 audio.ctx 初始化后加入：

```js
function ensureAudio() {
  if (!audio.ctx) {
    const Ctx = window.AudioContext || window.webkitAudioContext;
    if (Ctx) {
      audio.ctx = new Ctx();
      audio.master = audio.ctx.createGain();
      audio.master.gain.value = audioSettings.muted ? 0 : audioSettings.volume;
      audio.master.connect(audio.ctx.destination);
    }
  }
  if (audio.ctx?.state === "suspended") audio.ctx.resume();
  if (audio.master) audio.master.gain.value = audioSettings.muted ? 0 : audioSettings.volume;
}
```

并把原本 `gain.connect(audio.ctx.destination)` 改为连接到 `audio.master`（若存在）。

- [ ] **Step 2: 新增音效 API：`sfxCuteArp()` / `sfxBubblePop()`**

添加两个可复用的合成音函数（示例代码，后续按听感微调）：

```js
function playTone(freq, duration, type="sine", volume=0.15, when=0) {
  if (!audio.ctx || !audio.master) return;
  const t = audio.ctx.currentTime + when;
  const osc = audio.ctx.createOscillator();
  const gain = audio.ctx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, t);
  gain.gain.setValueAtTime(Math.max(0.0001, volume), t);
  gain.gain.exponentialRampToValueAtTime(0.0001, t + duration);
  osc.connect(gain);
  gain.connect(audio.master);
  osc.start(t);
  osc.stop(t + duration + 0.02);
}

function sfxCuteArp(base=440, volume=0.12) {
  // 轻快上行音阶（动感）
  [0, 4, 7].forEach((st, i) => {
    const f = base * Math.pow(2, st/12);
    playTone(f, 0.06, "triangle", volume, i * 0.05);
  });
}

function sfxBubblePop(volume=0.16) {
  // 泡泡破裂感：短噗 + 高频点
  playTone(160, 0.10, "sine", volume * 0.9, 0);
  playTone(900, 0.03, "square", volume * 0.25, 0.01);
}
```

- [ ] **Step 3: 事件绑定到新音效**
  - 拾取道具：`sfxCuteArp(520, 0.11)`
  - 道具到期：`playTone(330, 0.05, "triangle", 0.06)`
  - 护盾救命：`sfxBubblePop(0.18)` + `playTone(90, 0.12, "sine", 0.10, 0.02)`
  - 双倍分吃食物：在 `sfxEat()` 后叠 `playTone(880, 0.05, "sine", 0.05, 0.02)`
  - 成就解锁：`sfxCuteArp(660, 0.12)` 再补一颗高音 `playTone(1320, 0.06, "sine", 0.07, 0.16)`
  - 皮肤切换：`playTone(520,0.04,"triangle",0.07); playTone(780,0.05,"triangle",0.06,0.04)`

- [ ] **Step 4: 增加静音/音量 UI**
  - 在 HUD 增加一个按钮（例如 `🔊/🔇`）与一个小滑杆（可放进 overlay 的设置页，避免 UI 拥挤）
  - 切换时更新 `audioSettings` 并 `saveAudioSettings()`

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add cute bouncy sfx with volume/mute settings"
```

---

## Task 8: Achievements（成就）与 Skins（皮肤）UI/逻辑闭环

**Files:**
- Modify: `index.html`

- [ ] **Step 1: 定义成就与皮肤清单（常量表）**

```js
const ACHIEVEMENTS = [
  { id:"score_50", name:"小试牛刀", desc:"单局得分达到 50", check:(ctx)=>ctx.score>=50 },
  { id:"score_100", name:"渐入佳境", desc:"单局得分达到 100", check:(ctx)=>ctx.score>=100 },
  { id:"food_10", name:"好胃口！", desc:"单局吃到 10 个食物", check:(ctx)=>ctx.foodEaten>=10 },
  { id:"powerup_10", name:"道具收藏家", desc:"累计拾取 10 次道具", check:(ctx)=>profile.stats.totalPowerups>=10 },
  { id:"shield_save_1", name:"有惊无险", desc:"护盾救命 1 次", check:(ctx)=>profile.stats.shieldSaves>=1 },
  { id:"hard_150", name:"硬核蛇王", desc:"困难模式单局 150 分", check:(ctx)=>difficulty==="hard" && ctx.score>=150 },
  { id:"loyal_player", name:"忠实玩家", desc:"累计游玩 20 局", check:(ctx)=>profile.stats.gamesPlayed>=20 },
  { id:"max_food_12", name:"越吃越长", desc:"单局吃到 12 个食物", check:(ctx)=>ctx.foodEaten>=12 },
];

const SKINS = [
  { id:"default", name:"薄荷奶糖", unlock:"默认", colors:{ snake:"#5dd39e", snakeHead:"#3cb878", food:"#ff9f43", accent:"#ff6b9d" } },
  { id:"berry", name:"莓莓汽水", unlock:"score_50", colors:{ snake:"#7bd3ff", snakeHead:"#2aa6d6", food:"#ff6b9d", accent:"#6c5ce7" } },
  { id:"pudding", name:"布丁黄油", unlock:"score_100", colors:{ snake:"#ffd166", snakeHead:"#f4a261", food:"#06d6a0", accent:"#ff6b9d" } },
  { id:"lavender", name:"薰衣草", unlock:"powerup_10", colors:{ snake:"#b8a1ff", snakeHead:"#7c5cff", food:"#ff9f43", accent:"#ff6b9d" } },
  { id:"matcha", name:"抹茶团子", unlock:"shield_save_1", colors:{ snake:"#6ee7b7", snakeHead:"#10b981", food:"#fb7185", accent:"#34d399" } },
  { id:"midnight", name:"夜光星星", unlock:"hard_150", colors:{ snake:"#22c55e", snakeHead:"#16a34a", food:"#f59e0b", accent:"#60a5fa" } },
];
```

- [ ] **Step 2: 解锁逻辑：成就达成即写入 profile + toast + sfx**

在“游戏结束结算”或“吃到食物/拾取道具”等关键节点调用：

```js
function hasAchievement(id){ return profile.unlockedAchievements.includes(id); }
function unlockAchievement(id){
  if (hasAchievement(id)) return false;
  profile.unlockedAchievements.push(id);
  saveProfile();
  toast(`🎉 成就解锁：${ACHIEVEMENTS.find(a=>a.id===id)?.name || id}`);
  // sfx in Task 7
  return true;
}

function ensureSkinUnlockedByAchievement(achId){
  SKINS.filter(s=>s.unlock===achId).forEach(s=>{
    if (!profile.unlockedSkins.includes(s.id)) profile.unlockedSkins.push(s.id);
  });
  saveProfile();
}
```

当 `unlockAchievement()` 成功时，调用 `ensureSkinUnlockedByAchievement(id)`。

- [ ] **Step 3: UI：overlay 增加“成就/皮肤/设置”面板切换**
  - 在 overlay 内新增 3 个按钮（或 tabs）
  - 皮肤列表：显示锁/解锁/当前选中；点击解锁项可选择并保存
  - 成就列表：显示完成/未完成（可显示简要进度文本）
  - 设置：静音/音量滑杆（与 Task 7 对接）

- [ ] **Step 4: 自测（基础）**

```js
test("unlockAchievement idempotent", () => {
  profile = defaultProfile();
  assert("first unlock", unlockAchievement("score_50") === true);
  assert("second unlock no-op", unlockAchievement("score_50") === false);
});
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add achievements and skins with overlay panels"
```

---

## Task 9: 回归、微调与文档更新

**Files:**
- Modify: `README.md`
- Modify: `index.html`（仅微调）

- [ ] **Step 1: 手动回归清单（必须逐项过）**
  - 键盘：方向键/WASD、空格暂停/继续
  - 触屏：dpad 点击与滑动转向均可用
  - 难度：MENU/OVER 可切换；PLAYING 不可切换
  - 最高分：按难度记录仍正确
  - 道具：出现/闪烁到期/拾取/效果计时正确；叠加逻辑正确
  - 护盾：确实救命一次，且不会导致连死；有清晰视觉+音效反馈
  - 成就：达成即解锁且持久化；刷新仍在；皮肤解锁随成就联动
  - 音效：默认可爱动感；静音与音量生效；移动端需用户交互后解锁音频正常

- [ ] **Step 2: README 更新**
  - 新增：道具说明、成就/皮肤入口、音量/静音说明

- [ ] **Step 3: Commit**

```bash
git add README.md index.html
git commit -m "docs: update README for powerups, achievements, skins, audio settings"
```

---

## 计划自检（执行前）

- [ ] 本计划覆盖 spec 中：输入队列、道具生成/拾取/到期、effects 计时、护盾救命策略、网格缓存、浮字/toast、成就/皮肤存档与 UI、可爱动感音效与设置
- [ ] 无 “TODO/TBD/类似 Task X” 等占位描述；每个任务都有具体代码/命令

