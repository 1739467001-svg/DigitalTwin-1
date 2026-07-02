# SPEC · 技术设计文档

> 文档角色：回答"怎么做"。需求见 [PRD.md](./PRD.md)，排期见 [../PLAN.md](../PLAN.md)。
> 本文档是**活文档**：每个里程碑结束后回写实际实现与参数调整；重大技术取舍记入 §10 决策记录。

## 1. 架构总览

三层结构，单向数据流，仿真与渲染严格解耦：

```
┌────────────────────────────────────────────────┐
│  SimEngine（纯逻辑，无 three.js 依赖）           │
│  固定步长 tick → StateSnapshot + SimEvent[]     │
└──────────────┬─────────────────────────────────┘
               │ snapshot / events（只读）
       ┌───────┴────────┐
┌──────▼──────┐  ┌──────▼──────┐
│ SceneRenderer│  │  HUD (DOM)  │
│ three.js     │  │  vanilla JS │
│ 插值渲染      │  │  订阅事件    │
└─────────────┘  └─────────────┘
```

- **SimEngine** 不 import three.js，可脱离渲染在纯 Node 里跑（为 A/B 影子仿真和测试铺路）；
- **SceneRenderer** 只读快照，在两个 tick 之间做插值，不回写仿真状态；
- **HUD** 订阅事件总线 + 每秒拉一次 KPI 快照；
- 用户操作（故障注入、限流、变速）统一走 `engine.dispatch(command)`，是唯一入口。

## 2. 目录结构

```
index.html              # 入口，importmap 指向 vendor/
vendor/                 # three.module.js r160, OrbitControls.js（已就位）
src/
  main.js               # 装配：engine + renderer + hud + 主循环
  config/params.js      # ★ 全部可调参数集中于此（见 §8）
  sim/
    engine.js           # SimEngine：时钟、tick 调度、命令分发
    rng.js              # mulberry32 种子随机
    arrivals.js         # λ(t) 到达曲线 + 泊松生成
    agent.js            # 乘客状态机
    navgraph.js         # 导航图 + 队列槽位
    train.js            # 列车状态机 + 上下车模型
    devices.js          # 闸机/扶梯/屏蔽门设备模型（健康度、故障）
    kpi.js              # 滚动窗口 KPI 计算
    events.js           # 事件总线 + SimEvent 定义
  scene/
    renderer.js         # WebGLRenderer + composer + bloom + 主相机
    station.js          # 程序化站体建模（三层剖切）
    trainMesh.js        # 列车网格 + 进站演出
    crowd.js            # InstancedMesh 客流渲染
    heatmap.js          # 密度热力图（CanvasTexture）
    cameraRig.js        # 预设机位 / tween / 导演模式
    fx.js               # 情绪化光效、警示灯、光束
  ui/
    hud.js              # 顶栏 + 面板装配
    panels.js           # 指标面板、设备列表、事件日志
    charts.js           # 迷你趋势线（SVG sparkline）
    theme.css           # 视觉令牌（见 §7）
docs/                   # PRD / SPEC / 每期验收截图
```

## 3. 仿真设计

### 3.1 时钟与主循环

- 仿真固定步长 `SIM_DT = 0.1s`（仿真时间）；变速只改"每真实帧消耗多少仿真时间"，逻辑步长不变；
- 60× 快进时一帧执行多个 tick；时间轴拖拽 = 从最近的整点检查点重放（确定性保证可行）；
- 渲染帧率与 tick 解耦：renderer 在 `snapshot[t-1] → snapshot[t]` 间线性插值。

### 3.2 随机性与确定性（NFR-3）

- 全局唯一 RNG：mulberry32(seed)，seed 默认 `20260702`，URL 参数可覆盖；
- 禁止在 sim 代码中使用 `Math.random()`（渲染层的纯视觉抖动除外）；
- 用户命令带仿真时间戳入事件日志，构成"操作序列"，重放 = seed + 命令序列。

### 3.3 乘客状态机

```
SPAWN → WALK_TO_SECURITY → QUEUE_SECURITY → WALK_TO_GATE → QUEUE_GATE
  → WALK_TO_ESCALATOR → RIDE_ESCALATOR → WALK_TO_PLATFORM_SLOT
  → WAIT_ON_PLATFORM → BOARDING → ONBOARD（离场）
出站流：ALIGHT → WALK_TO_ESCALATOR_UP → RIDE → WALK_TO_EXIT_GATE → QUEUE → EXIT
```

- 寻路：手工导航图（节点=功能点，边=走廊段），A* 不必要，静态最短路 + 设备停用时重算；
- 局部运动：目标点引导 + 邻近分离（网格空间哈希，半径 0.4m），无社会力模型；
- 站台候车：每扇屏蔽门前预生成候车槽位（纵队 2 列 × 6 深），按满载率溢出到相邻门。

### 3.4 列车模型

- 状态机：`APPROACH（-350m, 匀减速 0.9 m/s²）→ DWELL（30–45s，开关门各 3s）→ DEPART（加速 1.0 m/s²）`；
- 6 编组，每节 4 门，定员 1400/列；上下车速率 1.2 人/秒/门，容量满即停止上客（产生留乘）；
- 双向独立时刻表，间隔见 §8 参数表。

### 3.5 KPI（滚动窗口 60s 仿真时间）

在站人数、进/出站速率（人/分）、站台密度（站台有效面积 900 m²）、满载率、平均候车时间（上车时结算）、留乘率（未上到站后首班车人数占比）、设备可用率。

## 4. 渲染设计

### 4.1 基线（M1 即全量到位）

- `WebGLRenderer{antialias:true}`，`ACESFilmicToneMapping`，`toneMappingExposure≈1.1`；
- `EffectComposer`：RenderPass → UnrealBloomPass（**半分辨率**，strength 0.55 / radius 0.4 / threshold 0.85）→ OutputPass；
- `FogExp2`，密度 0.012，颜色与背景一致（#0a0c10 一族）；
- 光照：低强度环境光 + 站厅/站台条形自发光灯带（发光靠 emissive+bloom，真实光源数 ≤ 6 盏）。

### 4.2 反光地面

站台层与站厅层地面用 three.js `Reflector`（分辨率 512，仅两面），其余表面高粗糙度标准材质。若 FPS 探针 < 60 则降级：Reflector → 高金属度 + envMap 假反射（决策记录 D-3）。

### 4.3 客流渲染

- 一个 `InstancedMesh`（胶囊近似：拉伸八面体低模，≤ 60 三角形），上限 `MAX_AGENTS = 2000`；
- 每帧写 `instanceMatrix` + `instanceColor`（进站 #3987e5 / 出站 #199e70 / 候车 #c98500）；
- 行走起伏：顶点着色器按 instance id 相位做 y 向正弦微移，避免 CPU 逐个体动画。

### 4.4 热力图

站厅/站台各一张 `CanvasTexture`（网格 1m×1m）：计数 → 盒模糊 → 单色蓝渐变（低密度近透明 → 高密度 #3987e5 → 预警叠加红），以 4 Hz 更新，叠加为地面第二层材质。

### 4.5 相机系统与导演模式

- OrbitControls + 4 预设机位（全景/站厅/站台/跟车），切换用 600ms easeInOut tween；
- 导演模式：空闲 30s 进入；镜头脚本 = 状态机（环绕 20s → 等下一班车切进站机位 → 跟拍一名随机乘客 15s → 循环），任何输入立即退出；
- 进站镜头微震：相机位置叠加 0.5s 衰减噪声，振幅 0.05m。

## 5. HUD 设计

- 纯 DOM/SVG 覆盖层，不进 three.js 场景；`pointer-events` 仅面板区域拦截；
- 布局：顶栏（时钟/核心数字/变速）、左栏（指标+趋势线）、右栏（设备+事件日志）、底部（24h 时间轴）；
- 趋势线：手写 SVG sparkline（120 点环形缓冲），不引第三方图表库。

## 6. 事件总线（sim → 渲染/HUD 的唯一通知通道）

`TRAIN_APPROACHING / TRAIN_ARRIVED / TRAIN_DEPARTED / DOORS_OPEN / DOORS_CLOSE / ALERT_LEVEL_CHANGED(level) / DEVICE_FAILED(id) / DEVICE_RESTORED(id) / FLOW_CONTROL_ON|OFF / AGENT_LEFT_BEHIND(count)`

渲染层据此触发演出（头灯光束、警示灯、色调偏移）；HUD 据此写事件日志。

## 7. 视觉令牌（theme.css）

深色工业风，与作品集一致；图表遵循暗面数据可视化规范：

| 令牌 | 值 | 用途 |
|---|---|---|
| `--surface` | `#1a1a19` | 面板底 |
| `--page` | `#0d0d0d` | 页面/场景背景族 |
| `--ink-1 / --ink-2 / --muted` | `#ffffff / #c3c2b7 / #898781` | 文字三级 |
| `--series-1 / -2 / -3` | `#3987e5 / #199e70 / #c98500` | 进站/出站/候车 + 图表系列 |
| `--status-good / -warn / -serious / -critical` | `#0ca30c / #fab219 / #ec835a / #d03b3b` | 密度分级、设备状态（带图标+文字，永不只靠颜色） |
| 网格线/描边 | `#2c2c2a / rgba(255,255,255,.10)` | 面板 hairline |

规则：文字永远用 ink 令牌，不穿系列色；状态色不挪作系列色；数字列用 `tabular-nums`。

## 8. 参数表（config/params.js，全部集中可调）

| 参数 | 默认值 | 依据 |
|---|---|---|
| 行车间隔 高峰/平峰 | 150s / 360s | 国内大城市地铁常见值 |
| 停站时长 | 35s（±5s） | 常规站 |
| 列车定员 | 1400 人（6B 编组） | AW2 定员近似 |
| 步行速度 | 1.3 m/s（±0.2） | 行人工程常用值 |
| 闸机通行 | 25 人/分/台，进站 6 台出站 6 台 | 厂商标称 30 的保守值 |
| 扶梯运力 | 100 人/分/台（上下各 2 台 + 楼梯） | 1m/s 梯速标称折减 |
| 安检通道 | 3 条 × 20 人/分 | 高峰瓶颈来源 |
| 到达峰值 | 早高峰 3000 人/时（进站），晚高峰 2600 人/时 | 中等客流站 |
| 密度阈值 | 1.0 / 2.0 / 2.5 人/m² | 舒适/繁忙/预警 |
| `MAX_AGENTS` | 2000 | 性能预算 |
| seed | 20260702 | URL `?seed=` 可覆盖 |

## 9. 性能预算与验证

- Draw call ≤ 150；三角形 ≤ 40 万；真实光源 ≤ 6；bloom 半分辨率；
- 内置 FPS 探针（左下角，均值/1% low），`?perf=1` 显示；
- 每里程碑用 Playwright 无头跑 60s：截图存 `docs/milestones/`、断言无 console error、FPS 均值 ≥ 55（无头环境放宽）。

## 10. 决策记录（ADR-lite）

| # | 决策 | 理由 | 状态 |
|---|---|---|---|
| D-1 | 仿真/渲染解耦，SimEngine 零 three.js 依赖 | A/B 影子仿真要双引擎并跑；测试可脱离 WebGL | 已定 |
| D-2 | 客流用 InstancedMesh 低模 + 双层表达（个体/热力），不做骨骼动画 | 千人 60fps 的唯一可靠路径；沙盘尺度下低模更协调 | 已定 |
| D-3 | 反光地面首选 Reflector(512)，FPS 不达标降级 envMap | 效果差距大，先试真实反射 | 已定，M1 验证 |
| D-4 | 时间轴拖拽用"检查点+重放"而非状态插值 | 确定性优先，实现简单 | 已定 |
| D-5 | 图表手写 SVG，不引第三方库 | 零构建原则；需求只有 sparkline 量级 | 已定 |
