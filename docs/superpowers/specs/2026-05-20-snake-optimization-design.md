---
title: 萌萌贪吃蛇优化设计（手感 + 道具 + 成就/皮肤 + 音效）
date: 2026-05-20
project: snake
version: draft-1
---

# 1. 背景与目标

现有项目为单文件 HTML 浏览器贪吃蛇（`snake.1.1.1`），包含难度、触屏/键盘输入、音效、最高分记录与爆炸粒子等效果。

本次优化目标（按优先级）：

1) **操控/手感优化**：减少误触、提高快速连按/连滑时的转向可靠性。  
2) **玩法扩展：道具系统**：引入可控、可扩展、不会破坏节奏的临时增益。  
3) **成长：成就/皮肤**：用轻量的本地存档驱动解锁与展示。  
4) **可爱动感音效**：在不引入外部资源文件的前提下，用 WebAudio 合成更“萌、动感”的反馈，并提供音量/静音控制。

硬性约束：
- **继续保持单文件 `index.html`**，本地离线打开可玩
- 不引入构建工具、不依赖额外资源文件（图片/音频文件等）

非目标（本轮不做）：
- 地图/障碍物、穿墙模式（除非后续另开迭代）
- 在线排行榜/账号体系

# 2. 总体方案（推荐：小幅结构化，仍在单文件内）

在同一 `<script>` 内做逻辑分区（不需要模块系统），把代码按职责拆成若干“模块对象/函数区”：

- `Input`：键盘/触控输入采集 + **转向缓冲队列**
- `Game`：状态机、蛇/食物/道具生成、碰撞与得分、效果计时
- `Renderer`：背景缓存、蛇/食物/道具/粒子/浮字绘制
- `SFX`：WebAudio 合成音效（含音量、静音与节流）
- `UI`：overlay、皮肤/成就面板、toast 提示、按钮与可访问性
- `Storage`：本地存档读写与版本迁移

# 3. 关键机制设计

## 3.1 手感：转向缓冲队列（Input Queue）

### 问题
当前仅有 `nextDirection`，当用户短时间内输入多个转向时，后输入会覆盖先输入，导致：
- “想急转”时中间转向被吞
- 触屏滑动在某些角度下易误判

### 设计
新增：
- `inputQueue: Array<{x:number,y:number,ts:number,source:'kbd'|'touch'}>`，长度上限 `MAX_QUEUE=3`

入队规则：
- 新方向 `d` 与**队尾方向**（若队列为空则与当前生效方向）相反时丢弃（避免立即反向）
- 与队尾方向相同也丢弃（避免无意义重复）
- 入队成功则写入 `ts`（用于调试/统计与潜在节流）

消费规则：
- 每个 `tick()` **最多消费 1 个方向**，保证“每走一格最多转一次”
- `direction`（当前生效方向）在 `tick()` 开始时设为消费后的方向

触控滑动优化：
- 将 `pointerup` 的阈值从固定值调整为：`threshold = max(24px, min(40px, canvas宽 * 0.04))`
- 引入“主轴判定”：`abs(dx) > abs(dy)` 走左右，否则走上下；并在 `abs(dx)` 与 `abs(dy)` 接近时（例如比例 < 1.15）直接忽略，减少斜滑误判

验收标准：
- 快速连按（上→左→下）不会吞中间转向（只要每步之间至少经过一次 tick）
- 触屏斜滑不应频繁误判成另一轴向

## 3.2 道具系统（Powerup）

目标：道具带来“可爱刺激”的短期变化，同时简单易懂、易扩展。

### 数据结构
- `powerup: null | { type: PowerupType, x:number, y:number, spawnAt:number, expiresAt:number }`
- `PowerupType = 'speed' | 'shield' | 'double'`
- `effects: { speedUntil:number, doubleUntil:number, shield:boolean }`

### 生成规则（建议默认）
- 每次吃到食物后以 `POWERUP_SPAWN_CHANCE = 0.20` 概率生成道具
- 场上同一时间最多 1 个道具（已有则不生成）
- 道具默认存在 `POWERUP_LIFETIME = 6000ms`，到期消失（并闪烁 1.2s 作为预警）

### 生效规则
拾取道具时：
- `speed`：`effects.speedUntil = now + 5000`
- `double`：`effects.doubleUntil = now + 8000`
- `shield`：`effects.shield = true`（可叠加规则：再次拾取刷新/仍为 true）

得分结算：
- 基础 `FOOD_SCORE = 10`
- 若 `now < effects.doubleUntil` 则本次得分 *2（显示浮字 `+20`）

速度结算：
- 现有“难度随分数加速”的 `moveInterval` 保留作为 base
- 若 `now < effects.speedUntil`，移动间隔乘以 `0.75`（或减去固定值，需实测手感；优先乘法更平滑）

护盾结算：
- 若发生撞墙/咬到自己：
  - 若 `effects.shield === true`：消耗护盾并触发“救命反馈”，本次不 Game Over；蛇位置回退策略见下
  - 否则 Game Over

护盾救命的回退策略（推荐简单稳妥）：
- 发生碰撞时先检测护盾：
  - **墙碰撞**：将 `newHead` clamp 回边界内（或直接取消本次 tick，不推进蛇身），并把方向保持不变/或者改为与墙相切方向（可选，需避免让玩家困死）
  - **自咬**：取消本次 tick（不 push newHead），等效“擦肩而过”
注：为了避免复杂物理，优先实现为“取消本次移动并给强反馈”，手感更直观。

验收标准：
- 道具出现率体感明显但不干扰（平均 3~7 个食物出现 1 次）
- 道具到期消失有明确提示
- 护盾能明确救命一次，且不会导致瞬间连死

## 3.3 成就与皮肤（Achievements & Skins）

### 存档（localStorage）
新增统一档案键：
- `snake_profile_v1`

结构：
```json
{
  "version": 1,
  "stats": {
    "gamesPlayed": 0,
    "totalScore": 0,
    "totalFood": 0,
    "totalPowerups": 0,
    "maxScoreEver": 0,
    "maxFoodOneGame": 0,
    "shieldSaves": 0
  },
  "unlockedAchievements": [],
  "unlockedSkins": ["default"],
  "selectedSkin": "default"
}
```

兼容：
- 原最高分按难度存 `snake_high_<difficulty>` 保留（不破坏旧记录）
- `profile.stats.maxScoreEver` 作为全局统计，不替代按难度最高分

### 成就（初版 ~8 个）
建议列表（ID → 描述 → 解锁条件）：
- `score_50`：单局≥50
- `score_100`：单局≥100
- `food_10`：单局吃到≥10 个食物
- `powerup_10`：累计拾取≥10 次道具
- `shield_save_1`：触发护盾救命≥1 次
- `hard_150`：困难模式单局≥150
- `combo_5`：在 15 秒内连续吃到 5 个食物（可选，稍增实现复杂度）
- `loyal_player`：累计游玩≥20 局

解锁反馈：
- toast：`🎉 成就解锁：xxx`
- 轻快音效（见 3.4）
- 皮肤奖励可绑定到成就（例如解锁 2~3 个主题色）

### 皮肤（初版 6 套配色主题）
皮肤以“主题 token”实现，不引入图片：
- 皮肤定义：`{ id, name, colors: { snake, snakeHead, food, accent, bg? } }`
- Renderer 读取当前 skin 的颜色 token 绘制蛇/食物/光效

UI：
- MENU/OVER 的 overlay 增加「皮肤」「成就」入口
- 同一 overlay 内切换面板（不新增页面）

## 3.4 可爱动感音效（Cute & Bouncy SFX）

目标：用 WebAudio 合成更丰富的动感反馈，并提供基础音频设置。

### 音频设置
- `snake_audio_settings_v1`：
  - `muted: boolean`
  - `volume: number`（0~1，默认 0.8）
- UI：增加一个小“音量/静音”按钮（HUD 右侧或 overlay 内）

### 音效原则
- 所有音效走统一出口 `masterGain` 控制总体音量
- 对高频触发音效（如移动）做节流，避免疲劳与性能问题

### 音效清单（建议新增/增强）
1) **拾取道具（powerup_pick）**：上扬的“跳跳糖”音阶（2~3 个音，短延迟串联）
2) **道具到期（powerup_expire）**：轻微下滑提示音（很短，音量低）
3) **护盾触发救命（shield_pop）**：柔和“泡泡破裂” + 低频“噗”一下
4) **双倍分期间吃食物（eat_double）**：在现有 `sfxEat` 上叠加一个更亮的短音
5) **成就解锁（achievement）**：短小胜利和弦（比 `sfxRecord` 更萌、更短）
6) **皮肤切换（skin_switch）**：短促“啪嗒”+ 上扬单音

实现建议：
- `SFX.play(name, opts)` 内部组合 `oscillator + gain envelope + optional noise + filter`
- 使用 `when` 参数做“动感连击”（微延迟的音阶）
- 全部为合成音，不引入音频文件

验收标准：
- 音效有明显“可爱动感”，且不刺耳（默认音量适中）
- 静音后不再触发音频上下文开销

# 4. 渲染与性能

## 4.1 背景网格缓存
当前每帧绘制网格线（`for i in 0..GRID`），可改为：
- 预渲染到 `bgCanvas`（离屏 canvas）一次
- 每帧 `ctx.drawImage(bgCanvas, 0, 0)` 即可

## 4.2 轻量浮字/Toast
- 浮字：`floatingTexts[] = {x,y,text,ttl,vy,alpha}`
- toast：overlay 顶部/底部轻量条，自动消失

# 5. 状态机与 UI

沿用现有状态机：
- `MENU / PLAYING / PAUSED / OVER / EXPLODING`

新增 UI 面板态（仅 UI 层，不进入 Game 状态）：
- `overlayPanel: 'main' | 'skins' | 'achievements' | 'settings'`

交互规则：
- PLAYING 时不允许切换难度（保持现状）
- MENU/OVER 可切换难度、皮肤、查看成就

# 6. 测试与验收清单

功能验收：
- 输入队列：快速连按/连滑不吞转向、不允许立即反向
- 道具：生成/拾取/到期/效果计时正确；双倍分与加速可叠加；护盾救命可用且只消耗一次
- 成就：满足条件即时解锁并持久化；重复不会重复弹出
- 皮肤：解锁/选择/持久化正确；渲染颜色切换全面
- 音效：静音/音量有效；各事件对应音效触发正确；不会因频繁触发造成明显卡顿

回归点：
- 原本的最高分（按难度）仍正常
- 暂停/继续、爆炸动画、触屏 dpad 仍可用

# 7. 风险与取舍

- 单文件内模块化会增加代码量：通过明确分区注释与命名保持可读性
- 护盾救命“回退策略”若设计过复杂会引入 bug：优先采用“取消本次移动并强反馈”的简化策略
- WebAudio 在 iOS/移动端需用户交互后解锁：沿用现有 `ensureAudio()`，并在设置里提示

