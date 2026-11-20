# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 运行

无构建、无 npm 依赖、无测试框架。必须用本地服务器打开（ES 模块在 `file://` 下会被浏览器拦截，页面会提示）：

```
python3 -m http.server     # 然后打开 http://localhost:8000
```

three.js 生态放在 `vendor/`（从 jsDelivr 原样下载，不依赖外部 CDN），经 `index.html` 里的 importmap 引用，版本钉死：three@0.186.1（`three.module.js` + `three.core.js`）、three.quarks@0.17.1（依赖 quarks.core@0.17.1）、postprocessing@6.39.5。postprocessing 6.39.5 要求 three < 0.187，升级 three 前先查兼容。唯一外部请求是 Google Fonts（非阻塞，加载不到用系统字体）。

两种脚本混用：`skills.js → characters.js → fx.js → engine.js` 是普通 `<script>`，靠全局变量串联；`render.js`、`main.js` 是 ES 模块（模块里能直接读写这些全局 `let`/`const`）。`main.js` 用 `window.keys` 把键盘状态暴露给 engine.js。

## 项目定位

本地双人同键盘对战游戏的最简框架，目标用户是初中生（口述点子、AI 改配置）。改动要保持这个定位：**优先加配置，不加抽象**；文档和注释精简、机制为主、不自创 flavor。

## 架构：配置 ↔ 引擎 ↔ 渲染

**配置文件（孩子/AI 改的）**
- `characters.js` — `CHARACTERS`：名字/颜色/键位/技能列表/出生点。标题、图例、主题色由 main.js 从这里生成。
- `skills.js` — `SKILLS`：每个技能 = 一个配置对象，`spawn`（生成什么）× `entity`（`behavior` 怎么动 + `hit` 打击参数）× `look`（`draw` 造型名 + 颜色）。

**框架层**
- `engine.js` — 状态 + 规则，不碰 DOM、不知道 three.js。`cast()` 是所有技能的唯一施放路径；`BEHAVIORS` 按 `entity.behavior` 挑行为。命中统一走 `hitCheck()`/`damage()`：伤害、击退、命中定格 `hitstop`（`hit.stop` 可覆盖）、定向屏震、漂浮字、状态效果（`hit.slow` 减速 / `hit.freeze` 减速中再中则冰冻）都在这里，任何技能配上就生效。
- `fx.js` — 特效**事件队列**：引擎调 `burst/boom/ring/puff/smoke/scorch/confetti` 只是往 `fxQueue` 排事件，不画任何东西。漂浮字 `floats`、屏震 `shake/shakeDir`、闪白 `flashA` 是状态。`resetFx()` 递增 `fxEpoch`，渲染层据此清场。
- `render.js` — three.js 场景，只读状态。三个按名挑选的注册表：
  - `LOOKS[look.draw](en)` 建实体模型，返回 `{ obj, update(en, t) }`；
  - `FX[event.type](e)` 把 fx 事件变成 three.quarks 粒子（原型在 `FXP`，每次 `clone()` 一份，`autoDestroy` 自动回收）；
  - 角色模型在 `makePlayerView/updatePlayerView`。
  HUD、漂浮字、倒计时/KO 大字、闪白画在上层 2D 画布 `#hud`。
- `main.js` — 装配：标题/图例、键盘输入、KO 弹层、requestAnimationFrame 主循环（`update(dt)` → `draw(t)`）。

**扩展路径**：新技能 = 在 SKILLS 加对象；现有 behavior/造型不够时，才在 `BEHAVIORS` 或 `LOOKS` 加一个函数，配置里按名引用。新特效 = `FXP` 加原型 + `FX` 加事件处理，引擎侧在 fx.js 加一个排事件的函数。

## 需要知道的约定

- 坐标：游戏 `(x, y)` 平面 = three 的 XZ 地面，1 像素 = 1 单位，Y 朝上。换算用 render.js 的 `wx()/wz()`。场地常量 `W/H/WALL/TOP` 在 engine.js，镜头 `fitCamera()` 自动把整片场地放进画面并给顶部 HUD 留空。
- 发光靠 bloom（阈值 1.3）：想让东西发光就用 `hdr()/hdrC()` 把颜色乘到 1 以上；普通材质保持 ≤1，否则地面也会泛光。
- 点光源是固定 4 个的池（`takeLight/freeLight/pulseLight`），不要临时 `new PointLight`——光源数量变化会让所有材质重编译卡顿。
- 定格/慢动作也作用于粒子：`BATCH.update` 用 `hitstop`/`slowmo` 缩放时间。
- 观赏兜底原则：点子不拒绝，量级悄悄截断——`ENTITY_CAP`（engine.js）、`FX_QUEUE_CAP`（fx.js）、`FX_SYSTEM_CAP`（render.js）。
- `prefers-reduced-motion`（fx.js 的 `REDUCED`）：屏震/闪白/慢动作/KO 推镜头关闭，粒子数量 ×0.4。
- 键位两套写法：`CHARACTERS.keys` 写键码（`KeyW`/`ArrowUp`），`SKILLS.key` 写标注（`'Q'`/`'.'`），换算在 engine.js 顶部。
- 素材占位：`CHARACTERS[i].image` / `look.image` 可选，写了就用图片面片代替程序模型，未写或未加载完回退程序模型。

## 给改游戏的 agent：自动提交

这份是某个孩子自己的游戏副本，改动会自动收回给大人。

- 每完成一个能玩的改动，立刻提交并推送：`git add -A && git commit -m "<一句话说改了什么>" && git push`
- 推送地址和密钥已经配好，不要改 git 配置（`remote`、`core.sshCommand`），不要删 `.git`。
- push 失败就告诉孩子，不要 `--force`。
