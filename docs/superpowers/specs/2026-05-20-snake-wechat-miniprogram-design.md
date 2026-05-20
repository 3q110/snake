---
title: 萌萌贪吃蛇微信小程序迁移设计（单页即玩）
date: 2026-05-20
project: snake
target: wechat-miniprogram
version: snake.1.2.1
appId: wxca36be1a98b67bba
---

# 1. 目标与范围

## 1.1 目标

把当前浏览器 Canvas 贪吃蛇（单文件 `index.html`）迁移为 **微信小程序**版本，满足：

- **单页即玩**：打开小程序进入游戏页，直接开始（不做首页/导航）。
- **网格升级**：从 `20×20` 改为 **`30×30`**，其余规则不变（撞墙死亡、吃食物变长、难度/加速逻辑、道具、成就、皮肤等沿用）。
- **版本号**：小程序内显示/写入版本为 **`snake.1.2.1`**。
- **存档**：继续使用本地存档（profile、最高分、设置）。
- **音效**：使用**内置短音频文件**（软萌泡泡糖风格）在小程序中播放（不依赖 WebAudio 合成）。

## 1.2 非目标

- 不做微信“小游戏”形态与发行流程。
- 不接入在线排行/登录/云存储。
- 不引入构建系统（如 webpack/vite）；保持小程序原生项目结构。

## 1.3 兼容性基线

- 使用 **`<canvas type="2d">`**（2D Canvas 节点）以最大化复用现有 Canvas 2D 渲染逻辑。
- 基础库建议：`>= 2.9.0`（实际以 2D Canvas 节点可用为准；后续实现时在 `project.config.json` 标注）。

# 2. 方案选择

采用方案 **A：微信小程序 + `canvas type="2d"` + 复用 2D API**。

理由：
- 可复用现有 `draw()` / `drawSnake()` / 粒子 / 渐变等 2D 绘制代码；
- 输入、存档、音频替换为小程序能力即可；
- 维护成本最低。

# 3. 项目结构（小程序）

目标是在仓库中新增一个小程序目录（建议放在 `miniprogram/`），避免与网页版本混杂。

```
snake/
  miniprogram/
    app.js
    app.json
    app.wxss
    project.config.json
    sitemap.json
    pages/
      game/
        game.wxml
        game.wxss
        game.js
        game.json
    assets/
      audio/
        chomp.mp3
        start.mp3
        pause.mp3
        die.mp3
        powerup_pick_speed.mp3
        powerup_pick_shield.mp3
        powerup_pick_double.mp3
        powerup_expire.mp3
        shield_save.mp3
        achievement.mp3
        skin_switch.mp3
```

备注：
- 音频文件均为短音效（建议 0.06s~0.35s），体积可控。
- 若要进一步减少体积，可合并部分音效（例如 powerup_pick 复用同一音效），但初版先保留区分以增强反馈。

# 4. 页面与 UI 设计（单页即玩）

## 4.1 游戏页布局

`pages/game/game` 页面包含：

- 顶部标题行（可选）：`萌萌贪吃蛇` + 版本 `snake.1.2.1`（小字）。
- HUD 行：
  - 得分、最高分
  - 难度按钮（容易/中等/困难；仅在 MENU/OVER 可切换）
  - 🔊/🔇 静音按钮 + 音量滑杆（与现有一致）
- Canvas 游戏区域：正方形自适应屏幕宽度
- 触屏 dpad（仅在触控设备显示；或始终显示以便新手）
- Overlay（菜单/暂停/结算/皮肤/成就）：
  - 初版沿用网页 overlay 的概念：在 canvas 上方叠层呈现
  - MENU/OVER 可进入“皮肤/成就”面板

## 4.2 交互规则

- 进入页面后处于 MENU，并展示“开始游戏”按钮（符合“进入即玩”的直觉：无需跳页）。
- PLAYING/PAUSED 不允许切换难度与皮肤（保持现状）；可暂停/继续。

# 5. 关键技术设计

## 5.1 Canvas 初始化与 DPR 适配

### 目标
小程序中画面不模糊、坐标体系与逻辑格点一致。

### 设计

- 通过 `wx.getSystemInfoSync()` 获取：
  - `windowWidth`
  - `pixelRatio`（dpr）
- canvas 显示尺寸：`cssSize = windowWidth - 32`（两侧留白 16px，可根据 UI 调整）
- canvas 实际像素尺寸：
  - `canvas.width  = cssSize * dpr`
  - `canvas.height = cssSize * dpr`
- 将 ctx 缩放到 CSS 坐标系：
  - `ctx.scale(dpr, dpr)`

> 这样后续绘制以 CSS 像素为单位，与网页版接近。

## 5.2 网格与单元尺寸

变更：
- `GRID = 30`

单元尺寸从“固定 CELL=40”改为动态：
- `CELL = Math.floor(cssSize / GRID)`
- `CANVAS_SIZE = CELL * GRID`

并把 canvas 的绘制区域限定为 `CANVAS_SIZE×CANVAS_SIZE`（居中绘制，避免除不尽导致的边缘抖动）：
- `offsetX = Math.floor((cssSize - CANVAS_SIZE) / 2)`
- `offsetY = offsetX`
- 所有绘制都加上 offset（或 `ctx.translate(offsetX, offsetY)` 后再绘制，推荐 translate 方式）

## 5.3 游戏循环

沿用现有模式：
- `requestAnimationFrame(loop)` 驱动渲染
- `if (now - lastTick >= effectiveMoveInterval(now)) tick(now)`

在小程序 `canvas type="2d"` 下优先使用 `canvas.requestAnimationFrame`（若可用），否则退化到 `setTimeout` 近似 60fps。

## 5.4 输入

### 方向缓冲队列
保持现有：
- `MAX_QUEUE = 3`
- 入队：反向/重复丢弃；超长丢弃最早输入
- 每 tick 消费 1 个方向

### 触控输入
保留现有 swipe 判定：
- 阈值：`threshold = max(24, min(40, width*0.04))`
- 斜滑忽略：`ratio < 1.15` 忽略

小程序事件来源：
- dpad：`bindtap`/`bindtouchstart` → `setDirection(dir, "touch")`
- canvas swipe：`bindtouchstart`/`bindtouchend` 计算 dx/dy → `setDirection`

## 5.5 存档与数据结构

把网页的 localStorage 替换为小程序 storage：

- `localStorage.getItem/setItem` → `wx.getStorageSync/wx.setStorageSync`

存档 key 保持一致：
- `snake_profile_v1`
- `snake_audio_settings_v1`
- `snake_difficulty`
- `snake_high_<difficulty>`

Profile 结构保持网页一致（最小化迁移风险）：
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

## 5.6 音效（软萌泡泡糖：内置音频文件）

### 播放方案
使用 `wx.createInnerAudioContext()`：
- 短音效建议复用多个 AudioContext 实例或使用“对象池”，避免频繁创建导致延迟。
- 音量与静音：
  - 静音：全局 `muted` 时直接不播放
  - 音量：为每个音效设置 `audio.volume = settings.volume`

### 音效文件清单与触发点（固定、无占位）

| 文件 | 时长建议 | 触发时机 |
|---|---:|---|
| `start.mp3` | 0.12s | 开始游戏 |
| `pause.mp3` | 0.08s | 暂停/继续切换（同音） |
| `chomp.mp3` | 0.12s | 吃到食物 |
| `die.mp3` | 0.28s | GameOver（爆炸/失败） |
| `powerup_pick_speed.mp3` | 0.16s | 拾取加速 ⚡ |
| `powerup_pick_shield.mp3` | 0.18s | 拾取护盾 🛡 |
| `powerup_pick_double.mp3` | 0.16s | 拾取双倍 ✨ |
| `powerup_expire.mp3` | 0.10s | 道具到期消失（一次） |
| `shield_save.mp3` | 0.20s | 护盾救命触发（取消本次移动） |
| `achievement.mp3` | 0.22s | 成就解锁 |
| `skin_switch.mp3` | 0.12s | 切换皮肤 |

说明：
- “软萌泡泡糖”总体特征：短促、圆润、无尖锐高频；优先使用温和的上扬音阶与轻微“啵”的瞬态。
- 实现阶段可用一套统一音色，按 pitch/节奏做差异，降低资源制作成本。

# 6. 网页代码到小程序代码的映射

| 网页（index.html）模块 | 小程序落点 |
|---|---|
| DOM（HUD/按钮/overlay） | `game.wxml` + `game.wxss` + `game.js` 数据绑定 |
| Canvas 绘制（ctx 2d） | `game.js` 内 `this.ctx`（2d 节点 ctx） |
| requestAnimationFrame loop | 优先 `canvas.requestAnimationFrame`，fallback `setTimeout` |
| localStorage | `wx.getStorageSync/wx.setStorageSync` |
| WebAudio 合成 | 内置 mp3 + `InnerAudioContext` |

# 7. 验收标准（Definition of Done）

功能：
- 进入小程序即进入游戏页，点击开始即可玩
- 30×30 网格生效，蛇移动/碰撞/吃食物逻辑正确
- 方向缓冲队列有效：快速转向不吞，且不允许即时反向
- 道具：生成/闪烁到期/拾取/效果计时正确；护盾救命为“取消本次移动”
- 成就/皮肤：解锁与持久化正常；MENU/OVER 可查看与切换皮肤
- 音效：各事件触发正确；静音与音量控制立即生效

性能与体验：
- 画面清晰（无明显模糊），在主流机型无明显掉帧
- 触控操作可靠（滑动阈值与斜滑忽略有效）

# 8. 风险与应对

1) **2D Canvas 节点兼容性**：不同基础库/机型差异  
→ 实现时做能力检测与 fallback；必要时降低某些特效（例如复杂渐变/阴影）。

2) **音频播放延迟**：首次播放可能卡顿  
→ 进入页面后预创建音频实例并 `seek(0)` 预热；使用对象池复用。

3) **CELL 动态计算导致边缘抖动**  
→ 使用 `CANVAS_SIZE = CELL*GRID` 并居中 offset；所有绘制基于整数像素。

