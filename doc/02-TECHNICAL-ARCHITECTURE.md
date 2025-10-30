# 技术架构文档

## 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     用户浏览器 (Browser)                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │           React Application (SPA)                     │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────┐    │ │
│  │  │    UI Layer (React Components)              │    │ │
│  │  │  - MainScene.tsx (主场景容器)               │    │ │
│  │  │  - ControlPanel.tsx (控制面板)              │    │ │
│  │  │  - HandoverStatus.tsx (切换状态显示)       │    │ │
│  │  └─────────────────────────────────────────────┘    │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────┐    │ │
│  │  │    3D Rendering Layer                       │    │ │
│  │  │  (React Three Fiber + Three.js)            │    │ │
│  │  │                                             │    │ │
│  │  │  - NTPUScene.tsx (NTPU 场景)               │    │ │
│  │  │  - SatelliteRenderer.tsx (卫星渲染器)     │    │ │
│  │  │  - UAVRenderer.tsx (UAV 渲染器)           │    │ │
│  │  │  - ConnectionLine.tsx (连接线)            │    │ │
│  │  │  - OrbitControls (轨道相机控制)           │    │ │
│  │  └─────────────────────────────────────────────┘    │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────┐    │ │
│  │  │    Logic Layer (Business Logic)             │    │ │
│  │  │                                             │    │ │
│  │  │  - SatelliteOrbitCalculator.ts             │    │ │
│  │  │    (使用 satellite.js 进行 SGP4 计算)      │    │ │
│  │  │                                             │    │ │
│  │  │  - HandoverAlgorithm.ts                    │    │ │
│  │  │    (切换决策逻辑)                          │    │ │
│  │  │                                             │    │ │
│  │  │  - VisibilityCalculator.ts                 │    │ │
│  │  │    (可见性计算：仰角、方位角)             │    │ │
│  │  │                                             │    │ │
│  │  │  - SignalStrengthEstimator.ts              │    │ │
│  │  │    (信号强度估算)                          │    │ │
│  │  └─────────────────────────────────────────────┘    │ │
│  │                                                       │ │
│  │  ┌─────────────────────────────────────────────┐    │ │
│  │  │    Data Layer                               │    │ │
│  │  │                                             │    │ │
│  │  │  - TLE Data (静态 JSON)                    │    │ │
│  │  │  - NTPU Scene Config                       │    │ │
│  │  │  - UAV Flight Path Config                  │    │ │
│  │  │  - Satellite Pool Cache                    │    │ │
│  │  └─────────────────────────────────────────────┘    │ │
│  │                                                       │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │           Static Assets (3D Models)                   │ │
│  │  - uav.glb                                           │ │
│  │  - sat.glb                                           │ │
│  │  - NTPU.glb                                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心技术栈

### 前端框架
```json
{
  "react": "^19.0.0",
  "react-dom": "^19.0.0",
  "typescript": "~5.7.2"
}
```

### 3D 渲染
```json
{
  "@react-three/fiber": "^9.1.2",      // React renderer for Three.js
  "@react-three/drei": "^10.0.7",      // Three.js helpers
  "three": "latest",                    // Three.js core
  "@types/three": "^0.176.0"           // TypeScript types
}
```

### 卫星轨道计算
```json
{
  "satellite.js": "^5.0.0"              // SGP4 orbit propagation
}
```

### 工具库
```json
{
  "date-fns": "^3.6.0",                // 日期时间处理
  "lodash": "^4.17.21"                  // 工具函数
}
```

### 构建工具
```json
{
  "vite": "^6.3.1",                    // 快速构建工具
  "sass": "^1.89.0"                     // CSS 预处理器
}
```

## 数据流架构

```
用户交互
   ↓
State Management (React Hooks)
   ↓
   ├→ 时间更新 → SatelliteOrbitCalculator
   │              ↓
   │           计算所有卫星位置 (SGP4)
   │              ↓
   │           VisibilityCalculator
   │              ↓
   │           筛选可见卫星
   │              ↓
   │           SatelliteRenderer (更新渲染)
   │
   ├→ UAV 位置更新 → UAVRenderer
   │                    ↓
   │                 HandoverAlgorithm
   │                    ↓
   │                 决策当前最佳卫星
   │                    ↓
   │                 ConnectionLine (绘制连接)
   │                    ↓
   │                 HandoverStatus (更新 UI)
   │
   └→ 相机控制 → OrbitControls → 场景视角更新
```

## 核心模块设计

### 1. SatelliteOrbitCalculator（卫星轨道计算器）

**职责**：
- 解析 TLE 数据
- 使用 SGP4 算法计算卫星位置
- 坐标系转换（ECI → ECEF → 场景坐标）

**接口设计**：
```typescript
class SatelliteOrbitCalculator {
  // 初始化
  constructor(tleData: TLEData[]);

  // 计算特定时间的卫星位置
  calculatePosition(satelliteId: string, date: Date): Vector3;

  // 批量计算多颗卫星位置
  calculateBatch(satelliteIds: string[], date: Date): Map<string, Vector3>;

  // 计算轨道预测（未来 N 分钟）
  predictOrbit(satelliteId: string, startDate: Date, durationMinutes: number): Vector3[];
}
```

### 2. VisibilityCalculator（可见性计算器）

**职责**：
- 计算卫星相对于观测点的仰角、方位角
- 判断卫星是否在地平线以上
- 应用仰角阈值（默认 10°）

**接口设计**：
```typescript
interface VisibilityInfo {
  isVisible: boolean;
  elevation: number;    // 仰角（度）
  azimuth: number;      // 方位角（度）
  range: number;        // 距离（km）
}

class VisibilityCalculator {
  constructor(
    observerLat: number,
    observerLon: number,
    observerAlt: number,
    minElevation: number = 10
  );

  calculateVisibility(
    satellitePosition: Vector3,
    date: Date
  ): VisibilityInfo;

  filterVisibleSatellites(
    satellites: Map<string, Vector3>,
    date: Date
  ): Map<string, VisibilityInfo>;
}
```

### 3. HandoverAlgorithm（切换算法）

**职责**：
- 评估所有可见卫星的信号质量
- 选择最佳卫星
- 触发切换事件

**接口设计**：
```typescript
interface HandoverDecision {
  currentSatellite: string | null;
  targetSatellite: string | null;
  shouldHandover: boolean;
  reason: string;
}

class HandoverAlgorithm {
  constructor(
    minElevation: number = 10,
    handoverHysteresis: number = 2  // 滞后阈值
  );

  evaluateBestSatellite(
    visibleSatellites: Map<string, VisibilityInfo>,
    currentSatellite: string | null
  ): HandoverDecision;

  calculateSignalStrength(visibility: VisibilityInfo): number;
}
```

### 4. SignalStrengthEstimator（信号强度估算器）

**职责**：
- 基于距离和仰角估算信号强度
- 使用 Friis 传输方程
- 计算 RSRP（参考信号接收功率）

**接口设计**：
```typescript
interface SignalMetrics {
  rsrp: number;      // dBm
  rsrq: number;      // dB
  sinr: number;      // dB
  pathLoss: number;  // dB
}

class SignalStrengthEstimator {
  constructor(
    transmitPower: number = 20,     // dBm
    frequency: number = 2e9,        // Hz (S-band)
    antennaGain: number = 15        // dBi
  );

  estimateSignal(
    range: number,
    elevation: number
  ): SignalMetrics;
}
```

## 3D 渲染架构

### 场景层次结构

```
<Canvas>
  <PerspectiveCamera position={[0, 100, 200]} />
  <OrbitControls />

  <ambientLight intensity={0.5} />
  <directionalLight position={[100, 100, 50]} />

  <Suspense fallback={<Loader />}>
    {/* NTPU 场景模型 */}
    <NTPUScene position={[0, 0, 0]} />

    {/* 卫星群 */}
    <SatelliteRenderer
      satellites={visibleSatellites}
      currentSatellite={currentSatId}
    />

    {/* UAV */}
    <UAVRenderer
      position={uavPosition}
      rotation={uavRotation}
    />

    {/* 连接线 */}
    {currentSatId && (
      <ConnectionLine
        from={uavPosition}
        to={satellitePositions.get(currentSatId)}
        color="#00ff00"
      />
    )}

    {/* 预测路径 */}
    {showPrediction && (
      <PredictionPath
        satellites={predictedOrbits}
        handoverPoints={handoverPoints}
      />
    )}
  </Suspense>
</Canvas>
```

### 坐标系定义

**NTPU 观测点**：
- 纬度：24.9441667°N
- 经度：121.3713889°E
- 海拔：50m

**场景坐标系**：
- 原点：NTPU 观测点
- X 轴：东
- Y 轴：上（天顶）
- Z 轴：北
- 单位：米

**卫星坐标转换**：
```
TLE → SGP4 → ECI (惯性坐标系)
              ↓
            ECEF (地心地固坐标系)
              ↓
            ENU (东北天坐标系，以 NTPU 为原点)
              ↓
            Three.js 场景坐标
```

## 性能优化策略

### 1. 卫星池管理
```typescript
// 只渲染可见范围内的卫星
const MAX_VISIBLE_SATELLITES = 50;
const MIN_ELEVATION = 10; // 度

// 每 5 秒更新一次可见性筛选
const VISIBILITY_UPDATE_INTERVAL = 5000; // ms
```

### 2. 模型实例化
```typescript
// 使用 InstancedMesh 渲染大量卫星
<InstancedMesh
  ref={satellitesRef}
  args={[geometry, material, visibleSatellites.length]}
/>
```

### 3. 轨道预计算
```typescript
// 启动时预计算 120 分钟的轨道数据
const orbitCache = new Map<string, Vector3[]>();

function precomputeOrbits(satellites: string[], durationMin: number) {
  satellites.forEach(satId => {
    orbitCache.set(satId, calculator.predictOrbit(satId, new Date(), durationMin));
  });
}
```

### 4. 节流与防抖
```typescript
// 相机移动事件节流
const handleCameraMove = throttle(() => {
  updateVisibleArea();
}, 100);

// 窗口调整大小防抖
const handleResize = debounce(() => {
  renderer.setSize(window.innerWidth, window.innerHeight);
}, 200);
```

## 状态管理

使用 React Hooks 进行状态管理：

```typescript
// 全局时间状态
const [currentTime, setCurrentTime] = useState(new Date());
const [timeSpeed, setTimeSpeed] = useState(1); // 1x 实时

// 卫星状态
const [visibleSatellites, setVisibleSatellites] = useState<Map<string, SatelliteInfo>>(new Map());
const [currentSatellite, setCurrentSatellite] = useState<string | null>(null);

// UAV 状态
const [uavPosition, setUAVPosition] = useState<Vector3>(new Vector3(0, 50, 0));
const [uavPath, setUAVPath] = useState<Vector3[]>([]);

// UI 状态
const [showPrediction, setShowPrediction] = useState(true);
const [showLabels, setShowLabels] = useState(true);

// Handover 状态
const [handoverStatus, setHandoverStatus] = useState<HandoverDecision | null>(null);
```

## 配置文件结构

```
/src/config/
  ├── ntpu.config.ts           # NTPU 场景配置
  ├── satellite.config.ts      # 卫星配置
  ├── uav.config.ts           # UAV 配置
  └── handover.config.ts      # 切换算法配置
```

**示例配置**：
```typescript
// ntpu.config.ts
export const NTPU_CONFIG = {
  observer: {
    latitude: 24.9441667,
    longitude: 121.3713889,
    altitude: 50,
  },
  scene: {
    modelPath: '/models/NTPU.glb',
    scale: 1,
    rotation: [0, 0, 0],
  },
};

// satellite.config.ts
export const SATELLITE_CONFIG = {
  minElevation: 10,          // 最小仰角（度）
  maxVisibleCount: 50,       // 最大可见卫星数
  updateInterval: 1000,      // 位置更新间隔（ms）
  modelPath: '/models/sat.glb',
  scale: 5,                  // 模型缩放（视觉增强）
};

// handover.config.ts
export const HANDOVER_CONFIG = {
  minElevation: 10,
  hysteresis: 2,             // 滞后阈值（度）
  connectionTimeout: 5000,   // 连接超时（ms）
  predictionWindow: 120,     // 预测窗口（分钟）
};
```

## 错误处理

```typescript
// 3D 模型加载错误
try {
  const model = await loadGLB('/models/sat.glb');
} catch (error) {
  console.error('Failed to load satellite model:', error);
  // 使用简单几何体替代
  const fallbackGeometry = new SphereGeometry(5);
}

// SGP4 计算错误
try {
  const position = calculator.calculatePosition(satId, date);
} catch (error) {
  console.error(`SGP4 calculation failed for ${satId}:`, error);
  // 从缓存中获取最近一次的有效位置
  const cachedPosition = positionCache.get(satId);
}

// WebGL 不支持
if (!WebGL.isWebGLAvailable()) {
  const warning = WebGL.getWebGLErrorMessage();
  document.body.appendChild(warning);
}
```

---

**文档版本**：v1.0
**创建日期**：2025-10-28
**最后更新**：2025-10-28
