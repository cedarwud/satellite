# NTPU 卫星-UAV Handover 可视化系统

> 从 ntn-stack 大型项目中提取并简化的卫星切换可视化系统

<div align="center">

**专注 · 轻量 · 高性能**

[项目概述](#-项目概述) • [快速开始](#-快速开始) • [文档导航](#-文档导航) • [实施路线](#-实施路线)

</div>

---

## 📖 项目概述

### 什么是这个项目？

这是一个**专注于 NTPU 场景的卫星-UAV 切换可视化系统**，从复杂的 ntn-stack 项目中提取核心"立体图"功能，创建一个：

- ✅ **简化版本**：只使用 NTPU 场景
- ✅ **轻量化**：仅需 uav.glb 和 sat.glb 两个模型
- ✅ **专注**：聚焦于卫星-UAV handover 可视化
- ✅ **独立运行**：纯前端实现，无需后端服务

### 核心功能

```
┌─────────────────────────────────────────────┐
│                                             │
│    🌏 NTPU 3D 场景                          │
│    🛰️  实时卫星轨道计算与渲染               │
│    🚁 UAV 飞行路径模拟                       │
│    🔗 卫星-UAV 连接线可视化                  │
│    🔄 智能 Handover 决策算法                 │
│    📊 实时状态监控面板                       │
│                                             │
└─────────────────────────────────────────────┘
```

### 技术亮点

| 特性 | 说明 |
|------|------|
| **实时 3D 渲染** | 基于 Three.js + React Three Fiber，60 FPS 流畅体验 |
| **真实轨道计算** | 使用 SGP4 算法，基于真实 TLE 数据 |
| **智能切换算法** | 基于信号强度（RSRP）的 handover 决策 |
| **纯前端实现** | 无需后端，使用浏览器端 satellite.js |
| **TypeScript** | 类型安全，易于维护 |

---

## 📚 文档导航

本项目包含完整的规划文档，建议按顺序阅读：

### 1️⃣ [项目概述](./01-PROJECT-OVERVIEW.md)
**阅读时间**：10 分钟

- 项目背景与目标
- 功能范围界定
- 技术简化策略
- 预期成果与时程

**适合人群**：所有人，首次了解项目

---

### 2️⃣ [技术架构](./02-TECHNICAL-ARCHITECTURE.md)
**阅读时间**：20 分钟

- 系统架构图
- 核心技术栈
- 数据流设计
- 模块接口定义
- 坐标系转换
- 性能优化策略

**适合人群**：开发者，需要理解系统设计

**关键内容**：
```
- SatelliteOrbitCalculator（卫星轨道计算器）
- VisibilityCalculator（可见性计算器）
- HandoverAlgorithm（切换算法）
- 3D 渲染层次结构
```

---

### 3️⃣ [依赖项说明](./03-DEPENDENCIES.md)
**阅读时间**：15 分钟

- 完整的 package.json
- 每个依赖的详细说明
- 配置文件模板
- 安装步骤
- 浏览器兼容性

**适合人群**：开发者，准备开发环境

**关键依赖**：
```json
{
  "react": "^19.0.0",
  "@react-three/fiber": "^9.1.2",
  "@react-three/drei": "^10.0.7",
  "satellite.js": "^5.0.0",
  "three": "^0.170.0"
}
```

---

### 4️⃣ [分阶段实施计划](./04-IMPLEMENTATION-PHASES.md) ⭐ 最重要
**阅读时间**：30 分钟

**完整的逐步实施指南**，包含：

- **Phase 0**：项目初始化（0.5 天）
- **Phase 1**：基础 3D 场景（2-3 天）
- **Phase 2**：卫星系统（2-3 天）
- **Phase 3**：UAV 与连接（1-2 天）
- **Phase 4**：Handover 逻辑（2-3 天）
- **Phase 5**：优化与完善（1-2 天）

每个阶段包含：
- ✅ 明确的验证标准
- 📝 详细的任务清单
- 💻 完整的代码示例
- 🐛 常见问题解决方案
- 📦 交付物清单

**适合人群**：开发者，执行项目实施

---

## 🚀 快速开始

### 前置要求

- Node.js ≥ 18.0
- npm ≥ 9.0
- 现代浏览器（支持 WebGL 2.0）

### 1. 创建项目

```bash
# 创建项目目录
mkdir ntpu-sat-handover-viz
cd ntpu-sat-handover-viz

# 初始化 npm
npm init -y
```

### 2. 安装依赖

```bash
# 核心依赖
npm install react react-dom three @react-three/fiber @react-three/drei satellite.js date-fns lodash lucide-react

# 开发依赖
npm install -D typescript @types/react @types/react-dom @types/three @types/lodash vite @vitejs/plugin-react sass eslint prettier
```

### 3. 复制 3D 资源

```bash
# 创建目录
mkdir -p public/{models,scenes,data}

# 从原项目复制模型
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/uav.glb public/models/
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/sat.glb public/models/
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/scenes/NTPU/NTPU.glb public/scenes/
```

### 4. 创建基础文件结构

```bash
mkdir -p src/{components/{scene,satellite,uav,ui},utils/{satellite,uav,handover},config,types,styles}
```

### 5. 启动开发

详细步骤请参考 [04-IMPLEMENTATION-PHASES.md](./04-IMPLEMENTATION-PHASES.md)

---

## 🗺️ 实施路线

```
Phase 0 (0.5 天)
    │
    ├─ 项目结构搭建
    ├─ 依赖安装
    └─ 配置文件
    │
    ↓
Phase 1 (2-3 天)
    │
    ├─ NTPU 场景渲染
    ├─ 相机控制
    └─ 基础灯光
    │
    ↓
Phase 2 (2-3 天)
    │
    ├─ TLE 数据准备
    ├─ SGP4 轨道计算
    ├─ 可见性筛选
    └─ 卫星渲染
    │
    ↓
Phase 3 (1-2 天)
    │
    ├─ UAV 模型加载
    ├─ 飞行路径生成
    └─ 连接线渲染
    │
    ↓
Phase 4 (2-3 天)
    │
    ├─ 信号强度估算
    ├─ Handover 算法
    └─ 状态面板
    │
    ↓
Phase 5 (1-2 天)
    │
    ├─ 性能优化
    ├─ UI 完善
    └─ 测试验收
    │
    ↓
🎉 完成！
```

**总预计时间**：8-13 天

---

## 📂 项目结构

```
ntpu-sat-handover-viz/
├── public/
│   ├── models/
│   │   ├── uav.glb         (10.3 MB)
│   │   └── sat.glb         (2.6 MB)
│   ├── scenes/
│   │   └── NTPU.glb        (8.1 MB)
│   └── data/
│       └── tle-data.json   (卫星 TLE 数据)
│
├── src/
│   ├── components/
│   │   ├── scene/
│   │   │   ├── MainScene.tsx        (主场景容器)
│   │   │   └── NTPUScene.tsx        (NTPU 场景)
│   │   ├── satellite/
│   │   │   ├── SatelliteRenderer.tsx
│   │   │   ├── ConnectionLine.tsx
│   │   │   └── SatelliteLabel.tsx
│   │   ├── uav/
│   │   │   └── UAVRenderer.tsx
│   │   └── ui/
│   │       ├── ControlPanel.tsx
│   │       ├── HandoverStatusPanel.tsx
│   │       └── StatsPanel.tsx
│   │
│   ├── utils/
│   │   ├── satellite/
│   │   │   ├── SatelliteOrbitCalculator.ts
│   │   │   └── VisibilityCalculator.ts
│   │   ├── uav/
│   │   │   └── UAVPathGenerator.ts
│   │   └── handover/
│   │       ├── HandoverAlgorithm.ts
│   │       └── SignalStrengthEstimator.ts
│   │
│   ├── config/
│   │   ├── ntpu.config.ts
│   │   ├── satellite.config.ts
│   │   ├── uav.config.ts
│   │   └── handover.config.ts
│   │
│   ├── types/
│   │   ├── satellite.ts
│   │   └── handover.ts
│   │
│   ├── styles/
│   │   ├── main.scss
│   │   ├── _variables.scss
│   │   └── _handover.scss
│   │
│   ├── App.tsx
│   └── main.tsx
│
├── package.json
├── tsconfig.json
├── vite.config.ts
├── .eslintrc.json
├── .prettierrc
│
└── 📚 文档/
    ├── README.md                      (本文档)
    ├── 01-PROJECT-OVERVIEW.md         (项目概述)
    ├── 02-TECHNICAL-ARCHITECTURE.md   (技术架构)
    ├── 03-DEPENDENCIES.md             (依赖说明)
    └── 04-IMPLEMENTATION-PHASES.md    (实施计划)
```

---

## 🎯 验证里程碑

### Phase 1 完成标准
- [ ] NTPU 3D 场景成功显示
- [ ] 相机可交互控制（旋转、缩放、平移）
- [ ] 帧率 ≥ 60 FPS

### Phase 2 完成标准
- [ ] 卫星成功渲染在天空中
- [ ] 卫星位置实时更新
- [ ] 可见性筛选正确（仰角 > 10°）

### Phase 3 完成标准
- [ ] UAV 沿路径飞行
- [ ] 连接线正确连接 UAV 和最近卫星
- [ ] 连接线有动画效果

### Phase 4 完成标准
- [ ] Handover 算法正常工作
- [ ] 状态面板显示正确信息
- [ ] 卫星切换有视觉反馈

### Phase 5 完成标准
- [ ] 所有功能正常工作
- [ ] 性能优化完成
- [ ] UI 完善且易用

---

## 🛠️ 开发命令

```bash
# 开发模式（热更新）
npm run dev

# 构建生产版本
npm run build

# 预览生产版本
npm run preview

# 代码检查
npm run lint

# 代码格式化
npm run format

# 类型检查
tsc --noEmit
```

---

## 🌐 浏览器兼容性

| 浏览器 | 最低版本 | 推荐版本 |
|--------|---------|---------|
| Chrome | 90+ | 最新 |
| Firefox | 88+ | 最新 |
| Safari | 15+ | 最新 |
| Edge | 90+ | 最新 |

**必需特性**：
- WebGL 2.0
- ES2020
- WebAssembly（可选，用于性能优化）

---

## 📊 性能指标

| 指标 | 目标值 | 实际值 |
|------|-------|--------|
| 渲染帧率 | ≥ 60 FPS | - |
| 卫星数量 | 20-50 颗 | - |
| 加载时间 | < 5 秒 | - |
| 内存占用 | < 500 MB | - |
| 包体积 | < 2 MB (gzip) | - |

---

## 🔧 故障排除

### 问题 1：模型不显示
**症状**：场景空白，看不到 3D 模型

**解决方案**：
1. 检查模型文件路径是否正确
2. 打开浏览器控制台，查看是否有加载错误
3. 验证模型文件是否存在且未损坏

```bash
# 验证模型文件
ls -lh public/models/
ls -lh public/scenes/
```

### 问题 2：卫星位置不正确
**症状**：卫星位置异常或不移动

**解决方案**：
1. 检查 TLE 数据是否有效
2. 验证时间是否正确更新
3. 查看控制台是否有 SGP4 计算错误

```typescript
// 添加调试日志
console.log('Current time:', currentTime);
console.log('Satellite positions:', satellitePositions);
```

### 问题 3：性能低下
**症状**：帧率低于 30 FPS

**解决方案**：
1. 减少可见卫星数量（调整 maxVisibleCount）
2. 降低模型复杂度
3. 使用 Chrome DevTools Performance 分析瓶颈

```typescript
// 限制卫星数量
const MAX_VISIBLE_SATELLITES = 30;
```

---

## 📈 后续扩展方向

完成基础版本后，可以考虑以下扩展：

1. **多场景支持**：添加 NYCU、Nanliao 等场景
2. **历史回放**：保存和回放 handover 事件
3. **数据导出**：导出 handover 统计数据
4. **高级算法**：实现更复杂的 handover 算法（考虑多普勒频移、QoS 等）
5. **多 UAV 支持**：同时渲染多架 UAV
6. **Web Worker 优化**：将 SGP4 计算移到 Worker 线程
7. **VR 支持**：添加 WebXR 支持

---

## 📝 开发日志

### 2025-10-28
- ✅ 创建项目规划文档
- ✅ 完成技术架构设计
- ✅ 编写详细的实施计划
- 📋 准备开始 Phase 0 实施

---

## 👥 贡献者

- **项目发起**：sat
- **技术咨询**：Claude (Anthropic)
- **原项目参考**：ntn-stack

---

## 📄 许可证

本项目基于原 ntn-stack 项目，仅用于学习和研究目的。

---

## 🆘 获取帮助

遇到问题？

1. 📖 首先查看 [04-IMPLEMENTATION-PHASES.md](./04-IMPLEMENTATION-PHASES.md) 的"常见问题"部分
2. 🔍 检查浏览器控制台的错误信息
3. 📝 查看相关配置文件是否正确

---

## 🎓 学习资源

### Three.js 学习
- [Three.js 官方文档](https://threejs.org/docs/)
- [React Three Fiber 文档](https://docs.pmnd.rs/react-three-fiber)
- [Drei 组件库](https://github.com/pmndrs/drei)

### 卫星轨道计算
- [Satellite.js 文档](https://github.com/shashwatak/satellite-js)
- [SGP4 算法介绍](https://en.wikipedia.org/wiki/Simplified_perturbations_models)
- [TLE 数据格式](https://en.wikipedia.org/wiki/Two-line_element_set)

### WebGL 和 3D 图形
- [WebGL 基础教程](https://webglfundamentals.org/)
- [3D 数学基础](https://www.euclideanspace.com/)

---

<div align="center">

**准备好开始了吗？**

👉 前往 [04-IMPLEMENTATION-PHASES.md](./04-IMPLEMENTATION-PHASES.md) 开始 Phase 0！

---

Made with ❤️ by sat

**最后更新**：2025-10-28

</div>
