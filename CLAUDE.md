# 项目：城市轨交车站数字孪生（视觉模拟优先）

Three.js 纯前端的地铁车站可视化仿真。**先读文档再动手：**

- `docs/PRD.md` — 需求与验收标准（做什么，做到什么程度）
- `docs/SPEC.md` — 技术设计、参数表、决策记录（怎么做，为什么这样做）
- `PLAN.md` — 里程碑 M1–M4（现在做到哪了）

## 硬约束

- **零构建**：纯 ES Modules，依赖全部 vendor 进仓库（`vendor/`），不引入 npm 构建链、不引第三方图表库。
- **仿真/渲染解耦**：`src/sim/` 禁止 import three.js；用户操作只走 `engine.dispatch(command)`。
- **确定性**：sim 代码禁用 `Math.random()`，一律用 `src/sim/rng.js` 的种子随机（渲染层纯视觉抖动除外）。
- **性能红线**：1080p 全效果 60fps；draw call ≤ 150；`MAX_AGENTS = 2000`。
- 可调数值一律进 `src/config/params.js`，不散落在代码里。
- 界面文案与文档用中文；代码标识符用英文。

## 运行与验证

```bash
python3 -m http.server 8080   # 或任意静态服务器，打开 http://localhost:8080
```

- 调试参数：`?seed=<int>` 固定随机种子，`?perf=1` 显示 FPS 探针。
- 验证：Playwright + 系统 Chromium 截图走查（截图存 `docs/milestones/`），断言无 console error。

## 工程流程

- 分支：`claude/digital-twin-simulation-qy9pm4`，完成即 commit + push。
- 每个里程碑结束：回写 SPEC（实际参数/偏差）、更新 PLAN 进度、存验收截图。
- 技术取舍变更记入 SPEC §10 决策记录，不要静默改变已定决策。
- 每期验收两条硬标准：十秒吸引力测试 + 60fps 无报错。
