# 分阶段实施计划

本文档详细描述项目的分阶段实施策略，每个阶段都有明确的目标、验证标准和交付物。

---

## 📋 实施概览

| 阶段 | 名称 | 预计时间 | 核心目标 |
|------|------|---------|---------|
| Phase 0 | 项目初始化 | 0.5 天 | 搭建开发环境 |
| Phase 1 | 基础 3D 场景 | 2-3 天 | NTPU 场景渲染 |
| Phase 2 | 卫星系统 | 2-3 天 | 卫星轨道计算与渲染 |
| Phase 3 | UAV 与连接 | 1-2 天 | UAV 模型与连接线 |
| Phase 4 | Handover 逻辑 | 2-3 天 | 切换算法实现 |
| Phase 5 | 优化与完善 | 1-2 天 | 性能优化与 UI 完善 |

**总预计时间**：8-13 天

---

## Phase 0: 项目初始化 (0.5 天)

### 🎯 目标
搭建完整的开发环境，确保所有依赖正确安装。

### 📝 任务清单

#### 0.1 创建项目结构
```bash
mkdir -p ntpu-sat-handover-viz
cd ntpu-sat-handover-viz

# 创建目录结构
mkdir -p src/{components,utils,config,types,styles,assets}
mkdir -p src/components/{scene,satellite,uav,ui}
mkdir -p public/{models,scenes,data}
```

#### 0.2 初始化 package.json
```bash
npm init -y
```

编辑 `package.json`，添加必要的 scripts 和依赖（参考 03-DEPENDENCIES.md）。

#### 0.3 安装依赖
```bash
# 核心依赖
npm install react react-dom
npm install three @react-three/fiber @react-three/drei
npm install satellite.js date-fns lodash lucide-react

# 开发依赖
npm install -D typescript @types/react @types/react-dom @types/three @types/lodash
npm install -D vite @vitejs/plugin-react sass
npm install -D eslint prettier
```

#### 0.4 配置文件

**tsconfig.json**：
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "allowSyntheticDefaultImports": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

**vite.config.ts**：
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    open: true,
  },
});
```

#### 0.5 创建基础文件

**src/main.tsx**：
```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles/main.scss';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**src/App.tsx**：
```typescript
import React from 'react';

function App() {
  return (
    <div className="app">
      <h1>NTPU Satellite-UAV Handover Visualization</h1>
      <p>Phase 0: Project Initialized ✅</p>
    </div>
  );
}

export default App;
```

**index.html**：
```html
<!DOCTYPE html>
<html lang="zh-TW">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>NTPU Sat-UAV Handover Viz</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### ✅ 验证标准
- [ ] `npm run dev` 能成功启动开发服务器
- [ ] 浏览器能打开 `http://localhost:3000` 并显示基础页面
- [ ] 无 TypeScript 编译错误
- [ ] 无 ESLint 警告

### 📦 交付物
- 完整的项目结构
- 所有依赖已安装
- 配置文件就绪
- 开发服务器可运行

---

## Phase 1: 基础 3D 场景 (2-3 天)

### 🎯 目标
成功渲染 NTPU 3D 场景，实现基础相机控制和灯光系统。

### 📝 任务清单

#### 1.1 复制 3D 资源
```bash
# 从原项目复制 NTPU 场景
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/scenes/NTPU/NTPU.glb \
   ./public/scenes/

# 验证文件大小
ls -lh public/scenes/NTPU.glb  # 应该约 8.1 MB
```

#### 1.2 创建 NTPU 配置

**src/config/ntpu.config.ts**：
```typescript
import { Vector3 } from 'three';

export const NTPU_CONFIG = {
  observer: {
    name: 'National Taipei University',
    latitude: 24.9441667,    // 度
    longitude: 121.3713889,  // 度
    altitude: 50,            // 米
  },
  scene: {
    modelPath: '/scenes/NTPU.glb',
    position: new Vector3(0, 0, 0),
    scale: 1,
    rotation: [0, 0, 0] as [number, number, number],
  },
  camera: {
    initialPosition: new Vector3(0, 100, 200),
    fov: 60,
    near: 0.1,
    far: 10000,
  },
};
```

#### 1.3 实现 NTPU 场景组件

**src/components/scene/NTPUScene.tsx**：
```typescript
import React, { useRef } from 'react';
import { useGLTF } from '@react-three/drei';
import { NTPU_CONFIG } from '@/config/ntpu.config';

export function NTPUScene() {
  const { scene } = useGLTF(NTPU_CONFIG.scene.modelPath);
  const groupRef = useRef();

  return (
    <group ref={groupRef} position={NTPU_CONFIG.scene.position}>
      <primitive object={scene} scale={NTPU_CONFIG.scene.scale} />
    </group>
  );
}

// 预加载模型
useGLTF.preload(NTPU_CONFIG.scene.modelPath);
```

#### 1.4 创建主 3D 容器

**src/components/scene/MainScene.tsx**：
```typescript
import React, { Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { OrbitControls, PerspectiveCamera, Environment } from '@react-three/drei';
import { NTPUScene } from './NTPUScene';
import { NTPU_CONFIG } from '@/config/ntpu.config';

export function MainScene() {
  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <Canvas>
        {/* 相机 */}
        <PerspectiveCamera
          makeDefault
          position={NTPU_CONFIG.camera.initialPosition.toArray()}
          fov={NTPU_CONFIG.camera.fov}
          near={NTPU_CONFIG.camera.near}
          far={NTPU_CONFIG.camera.far}
        />

        {/* 轨道控制 */}
        <OrbitControls
          enableDamping
          dampingFactor={0.05}
          minDistance={50}
          maxDistance={500}
          maxPolarAngle={Math.PI / 2}
        />

        {/* 灯光 */}
        <ambientLight intensity={0.5} />
        <directionalLight position={[100, 100, 50]} intensity={1} castShadow />
        <hemisphereLight args={['#87ceeb', '#545454', 0.6]} />

        {/* 环境 */}
        <Environment preset="sunset" />

        {/* 场景模型 */}
        <Suspense fallback={null}>
          <NTPUScene />
        </Suspense>

        {/* 网格辅助线（调试用） */}
        <gridHelper args={[1000, 50, '#888888', '#444444']} />
        <axesHelper args={[100]} />
      </Canvas>

      {/* 加载提示 */}
      <div className="loading-overlay">
        <p>Loading NTPU Scene...</p>
      </div>
    </div>
  );
}
```

#### 1.5 添加基础样式

**src/styles/main.scss**：
```scss
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  overflow: hidden;
}

#root {
  width: 100vw;
  height: 100vh;
}

.loading-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-size: 20px;
  pointer-events: none;
  z-index: 100;
}
```

#### 1.6 更新 App.tsx

**src/App.tsx**：
```typescript
import React from 'react';
import { MainScene } from './components/scene/MainScene';

function App() {
  return <MainScene />;
}

export default App;
```

### ✅ 验证标准
- [ ] NTPU 3D 场景成功加载并显示
- [ ] 可以用鼠标旋转、缩放、平移场景
- [ ] 灯光效果正确（场景清晰可见）
- [ ] 网格和坐标轴辅助线显示正确
- [ ] 无控制台错误
- [ ] 帧率 ≥ 60 FPS

### 🐛 常见问题与解决方案

**问题 1：模型不显示**
```typescript
// 检查模型路径
console.log('Model path:', NTPU_CONFIG.scene.modelPath);

// 检查模型加载状态
const { scene, error } = useGLTF(NTPU_CONFIG.scene.modelPath);
if (error) console.error('Model loading error:', error);
```

**问题 2：相机视角不对**
```typescript
// 调整相机初始位置
initialPosition: new Vector3(0, 200, 300),  // 调高或调远

// 调整 OrbitControls target
<OrbitControls target={[0, 0, 0]} />
```

**问题 3：场景太暗**
```typescript
// 增加灯光强度
<ambientLight intensity={0.8} />
<directionalLight position={[100, 100, 50]} intensity={1.5} />
```

### 📦 交付物
- NTPU 3D 场景渲染
- 可交互的相机控制
- 基础灯光系统
- 加载状态显示

---

## Phase 2: 卫星系统 (2-3 天)

### 🎯 目标
实现卫星轨道计算、位置更新和 3D 渲染。

### 📝 任务清单

#### 2.1 准备 TLE 数据

**public/data/tle-data.json**：
```json
{
  "satellites": [
    {
      "id": "starlink-30270",
      "name": "STARLINK-30270",
      "constellation": "Starlink",
      "tle1": "1 56171U 23049AV  25300.50000000  .00002840  00000+0  20279-3 0  9997",
      "tle2": "2 56171  43.0019 123.4567 0001234  12.3456 347.6543 15.02345678123456"
    }
  ]
}
```

💡 **获取真实 TLE 数据**：
```bash
# 下载 Starlink TLE
curl -o starlink.txt https://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=tle

# 转换为 JSON 格式（需要编写脚本）
```

#### 2.2 复制卫星模型
```bash
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/sat.glb \
   ./public/models/

# 验证
ls -lh public/models/sat.glb  # 约 2.6 MB
```

#### 2.3 实现卫星轨道计算器

**src/utils/satellite/SatelliteOrbitCalculator.ts**：
```typescript
import * as satellite from 'satellite.js';
import { Vector3 } from 'three';

export interface TLEData {
  id: string;
  name: string;
  tle1: string;
  tle2: string;
}

export interface SatellitePosition {
  position: Vector3;
  velocity: Vector3;
  timestamp: Date;
}

export class SatelliteOrbitCalculator {
  private satRecords: Map<string, satellite.SatRec> = new Map();

  constructor(tleData: TLEData[]) {
    this.loadTLEData(tleData);
  }

  private loadTLEData(tleData: TLEData[]) {
    tleData.forEach((tle) => {
      try {
        const satrec = satellite.twoline2satrec(tle.tle1, tle.tle2);
        this.satRecords.set(tle.id, satrec);
      } catch (error) {
        console.error(`Failed to parse TLE for ${tle.id}:`, error);
      }
    });
  }

  /**
   * 计算单颗卫星在指定时间的位置
   */
  calculatePosition(satelliteId: string, date: Date): SatellitePosition | null {
    const satrec = this.satRecords.get(satelliteId);
    if (!satrec) return null;

    try {
      const positionAndVelocity = satellite.propagate(satrec, date);

      if (satellite.error(satrec)) {
        console.error(`SGP4 error for ${satelliteId}`);
        return null;
      }

      const positionEci = positionAndVelocity.position as satellite.EciVec3<number>;
      const velocityEci = positionAndVelocity.velocity as satellite.EciVec3<number>;

      // ECI to Three.js coordinates (scale to km)
      return {
        position: new Vector3(positionEci.x, positionEci.z, -positionEci.y),
        velocity: new Vector3(velocityEci.x, velocityEci.z, -velocityEci.y),
        timestamp: date,
      };
    } catch (error) {
      console.error(`Failed to calculate position for ${satelliteId}:`, error);
      return null;
    }
  }

  /**
   * 批量计算多颗卫星位置
   */
  calculateBatch(satelliteIds: string[], date: Date): Map<string, SatellitePosition> {
    const results = new Map<string, SatellitePosition>();

    satelliteIds.forEach((id) => {
      const pos = this.calculatePosition(id, date);
      if (pos) {
        results.set(id, pos);
      }
    });

    return results;
  }

  /**
   * 预测轨道（未来 N 分钟）
   */
  predictOrbit(
    satelliteId: string,
    startDate: Date,
    durationMinutes: number,
    stepSeconds: number = 10
  ): Vector3[] {
    const positions: Vector3[] = [];
    const steps = (durationMinutes * 60) / stepSeconds;

    for (let i = 0; i <= steps; i++) {
      const time = new Date(startDate.getTime() + i * stepSeconds * 1000);
      const pos = this.calculatePosition(satelliteId, time);
      if (pos) {
        positions.push(pos.position);
      }
    }

    return positions;
  }

  getAllSatelliteIds(): string[] {
    return Array.from(this.satRecords.keys());
  }
}
```

#### 2.4 实现可见性计算器

**src/utils/satellite/VisibilityCalculator.ts**：
```typescript
import { Vector3 } from 'three';
import * as satellite from 'satellite.js';

export interface VisibilityInfo {
  isVisible: boolean;
  elevation: number; // 仰角（度）
  azimuth: number; // 方位角（度）
  range: number; // 距离（km）
}

export class VisibilityCalculator {
  private observerGeodetic: satellite.GeodeticLocation;
  private minElevation: number;

  constructor(
    latitude: number,
    longitude: number,
    altitude: number,
    minElevation: number = 10
  ) {
    this.observerGeodetic = {
      latitude: satellite.degreesToRadians(latitude),
      longitude: satellite.degreesToRadians(longitude),
      height: altitude / 1000, // 转换为 km
    };
    this.minElevation = minElevation;
  }

  calculateVisibility(satelliteEci: Vector3, date: Date): VisibilityInfo {
    // 将 Three.js 坐标转回 ECI
    const positionEci: satellite.EciVec3<number> = {
      x: satelliteEci.x,
      y: -satelliteEci.z,
      z: satelliteEci.y,
    };

    // ECI → ECEF
    const gmst = satellite.gstime(date);
    const positionEcf = satellite.eciToEcf(positionEci, gmst);

    // ECEF → Look Angles
    const lookAngles = satellite.ecfToLookAngles(this.observerGeodetic, positionEcf);

    const elevation = satellite.radiansToDegrees(lookAngles.elevation);
    const azimuth = satellite.radiansToDegrees(lookAngles.azimuth);
    const range = lookAngles.rangeSat;

    return {
      isVisible: elevation >= this.minElevation,
      elevation,
      azimuth,
      range,
    };
  }

  filterVisibleSatellites(
    satellites: Map<string, Vector3>,
    date: Date
  ): Map<string, VisibilityInfo> {
    const visible = new Map<string, VisibilityInfo>();

    satellites.forEach((position, id) => {
      const visibility = this.calculateVisibility(position, date);
      if (visibility.isVisible) {
        visible.set(id, visibility);
      }
    });

    return visible;
  }
}
```

#### 2.5 实现卫星渲染器

**src/components/satellite/SatelliteRenderer.tsx**：
```typescript
import React, { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { InstancedMesh, Matrix4, Color, Object3D } from 'three';

interface SatelliteRendererProps {
  satellites: Map<string, { position: Vector3; isActive: boolean }>;
  modelPath: string;
  scale: number;
}

export function SatelliteRenderer({ satellites, modelPath, scale }: SatelliteRendererProps) {
  const { scene } = useGLTF(modelPath);
  const instancedMeshRef = useRef<InstancedMesh>(null);
  const tempObject = useMemo(() => new Object3D(), []);
  const tempMatrix = useMemo(() => new Matrix4(), []);

  // 更新实例化网格
  useFrame(() => {
    if (!instancedMeshRef.current) return;

    let index = 0;
    satellites.forEach((sat) => {
      tempObject.position.copy(sat.position);
      tempObject.scale.set(scale, scale, scale);
      tempObject.updateMatrix();

      instancedMeshRef.current!.setMatrixAt(index, tempObject.matrix);
      instancedMeshRef.current!.setColorAt(
        index,
        new Color(sat.isActive ? '#00ff00' : '#ffffff')
      );

      index++;
    });

    instancedMeshRef.current.instanceMatrix.needsUpdate = true;
    if (instancedMeshRef.current.instanceColor) {
      instancedMeshRef.current.instanceColor.needsUpdate = true;
    }
  });

  // 提取几何体和材质
  const geometry = useMemo(() => {
    const mesh = scene.children[0] as any;
    return mesh.geometry;
  }, [scene]);

  const material = useMemo(() => {
    const mesh = scene.children[0] as any;
    return mesh.material.clone();
  }, [scene]);

  return (
    <instancedMesh
      ref={instancedMeshRef}
      args={[geometry, material, satellites.size]}
      castShadow
    />
  );
}

useGLTF.preload('/models/sat.glb');
```

#### 2.6 集成卫星系统到主场景

**src/components/scene/MainScene.tsx**（更新）：
```typescript
import React, { useState, useEffect } from 'react';
import { SatelliteOrbitCalculator } from '@/utils/satellite/SatelliteOrbitCalculator';
import { VisibilityCalculator } from '@/utils/satellite/VisibilityCalculator';
import { SatelliteRenderer } from '@/components/satellite/SatelliteRenderer';
import { NTPU_CONFIG } from '@/config/ntpu.config';

export function MainScene() {
  const [currentTime, setCurrentTime] = useState(new Date());
  const [visibleSatellites, setVisibleSatellites] = useState(new Map());
  const [calculator, setCalculator] = useState<SatelliteOrbitCalculator | null>(null);
  const [visibilityCalc, setVisibilityCalc] = useState<VisibilityCalculator | null>(null);

  // 初始化
  useEffect(() => {
    fetch('/data/tle-data.json')
      .then((res) => res.json())
      .then((data) => {
        const calc = new SatelliteOrbitCalculator(data.satellites);
        setCalculator(calc);

        const visCal = new VisibilityCalculator(
          NTPU_CONFIG.observer.latitude,
          NTPU_CONFIG.observer.longitude,
          NTPU_CONFIG.observer.altitude,
          10 // 最小仰角
        );
        setVisibilityCalc(visCal);
      });
  }, []);

  // 时间更新
  useEffect(() => {
    const interval = setInterval(() => {
      setCurrentTime((prev) => new Date(prev.getTime() + 1000));
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  // 更新卫星位置
  useEffect(() => {
    if (!calculator || !visibilityCalc) return;

    const allSatIds = calculator.getAllSatelliteIds();
    const positions = calculator.calculateBatch(allSatIds, currentTime);

    // 转换为 Map<id, Vector3>
    const positionMap = new Map();
    positions.forEach((pos, id) => {
      positionMap.set(id, pos.position);
    });

    // 筛选可见卫星
    const visible = visibilityCalc.filterVisibleSatellites(positionMap, currentTime);

    // 转换为渲染格式
    const renderData = new Map();
    visible.forEach((vis, id) => {
      renderData.set(id, {
        position: positionMap.get(id),
        isActive: false,
      });
    });

    setVisibleSatellites(renderData);
  }, [currentTime, calculator, visibilityCalc]);

  return (
    <Canvas>
      {/* ... 相机、灯光、NTPU 场景 ... */}

      {/* 卫星渲染 */}
      {visibleSatellites.size > 0 && (
        <SatelliteRenderer
          satellites={visibleSatellites}
          modelPath="/models/sat.glb"
          scale={5}
        />
      )}
    </Canvas>
  );
}
```

### ✅ 验证标准
- [ ] 卫星成功渲染在天空中
- [ ] 卫星位置随时间更新
- [ ] 只显示仰角 > 10° 的可见卫星
- [ ] 控制台输出卫星数量统计
- [ ] 帧率保持 ≥ 60 FPS
- [ ] 卫星轨迹符合物理规律

### 🧪 调试工具

**src/components/ui/DebugPanel.tsx**：
```typescript
import React from 'react';

interface DebugPanelProps {
  currentTime: Date;
  visibleCount: number;
  totalCount: number;
}

export function DebugPanel({ currentTime, visibleCount, totalCount }: DebugPanelProps) {
  return (
    <div className="debug-panel">
      <h3>Debug Info</h3>
      <p>Time: {currentTime.toLocaleTimeString()}</p>
      <p>Visible Satellites: {visibleCount} / {totalCount}</p>
    </div>
  );
}
```

### 📦 交付物
- 卫星轨道计算系统
- 可见性筛选算法
- 卫星 3D 渲染
- 实时位置更新
- 调试面板

---

## Phase 3: UAV 与连接 (1-2 天)

### 🎯 目标
添加 UAV 模型、飞行路径和卫星连接线可视化。

### 📝 任务清单

#### 3.1 复制 UAV 模型
```bash
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/uav.glb \
   ./public/models/
```

#### 3.2 UAV 配置

**src/config/uav.config.ts**：
```typescript
import { Vector3 } from 'three';

export const UAV_CONFIG = {
  model: {
    path: '/models/uav.glb',
    scale: 3,
  },
  flight: {
    initialPosition: new Vector3(0, 50, 0),
    altitude: 50, // 米
    speed: 10, // m/s
    // 简单的圆形飞行路径
    pathType: 'circle' as const,
    radius: 100,
  },
};
```

#### 3.3 UAV 路径生成器

**src/utils/uav/UAVPathGenerator.ts**：
```typescript
import { Vector3 } from 'three';

export class UAVPathGenerator {
  /**
   * 生成圆形飞行路径
   */
  static generateCirclePath(
    center: Vector3,
    radius: number,
    altitude: number,
    segments: number = 360
  ): Vector3[] {
    const path: Vector3[] = [];

    for (let i = 0; i <= segments; i++) {
      const angle = (i / segments) * Math.PI * 2;
      const x = center.x + Math.cos(angle) * radius;
      const z = center.z + Math.sin(angle) * radius;
      path.push(new Vector3(x, altitude, z));
    }

    return path;
  }

  /**
   * 根据时间获取路径上的位置
   */
  static getPositionAtTime(
    path: Vector3[],
    elapsedTime: number,
    speed: number
  ): { position: Vector3; index: number } {
    const totalDistance = this.calculatePathLength(path);
    const distance = (elapsedTime * speed) % totalDistance;

    let accumulatedDistance = 0;
    for (let i = 0; i < path.length - 1; i++) {
      const segmentLength = path[i].distanceTo(path[i + 1]);
      if (accumulatedDistance + segmentLength >= distance) {
        const t = (distance - accumulatedDistance) / segmentLength;
        const position = new Vector3().lerpVectors(path[i], path[i + 1], t);
        return { position, index: i };
      }
      accumulatedDistance += segmentLength;
    }

    return { position: path[0].clone(), index: 0 };
  }

  private static calculatePathLength(path: Vector3[]): number {
    let length = 0;
    for (let i = 0; i < path.length - 1; i++) {
      length += path[i].distanceTo(path[i + 1]);
    }
    return length;
  }
}
```

#### 3.4 UAV 渲染器

**src/components/uav/UAVRenderer.tsx**：
```typescript
import React, { useRef, useEffect } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { Group, Vector3, Euler } from 'three';

interface UAVRendererProps {
  position: Vector3;
  rotation?: Euler;
  scale?: number;
  modelPath: string;
}

export function UAVRenderer({ position, rotation, scale = 3, modelPath }: UAVRendererProps) {
  const { scene } = useGLTF(modelPath);
  const groupRef = useRef<Group>(null);

  useFrame(() => {
    if (groupRef.current) {
      groupRef.current.position.copy(position);
      if (rotation) {
        groupRef.current.rotation.copy(rotation);
      }
    }
  });

  return (
    <group ref={groupRef}>
      <primitive object={scene.clone()} scale={scale} />
    </group>
  );
}

useGLTF.preload('/models/uav.glb');
```

#### 3.5 连接线组件

**src/components/satellite/ConnectionLine.tsx**：
```typescript
import React, { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { Line } from '@react-three/drei';
import { Vector3, Color } from 'three';

interface ConnectionLineProps {
  from: Vector3;
  to: Vector3;
  color?: string;
  opacity?: number;
  lineWidth?: number;
  animated?: boolean;
}

export function ConnectionLine({
  from,
  to,
  color = '#00ff00',
  opacity = 0.6,
  lineWidth = 2,
  animated = true,
}: ConnectionLineProps) {
  const lineRef = useRef();
  const points = useMemo(() => [from.toArray(), to.toArray()], [from, to]);

  // 动画效果（可选）
  useFrame(({ clock }) => {
    if (animated && lineRef.current) {
      const pulse = Math.sin(clock.getElapsedTime() * 2) * 0.2 + 0.8;
      lineRef.current.material.opacity = opacity * pulse;
    }
  });

  return (
    <Line
      ref={lineRef}
      points={points}
      color={color}
      lineWidth={lineWidth}
      transparent
      opacity={opacity}
    />
  );
}
```

#### 3.6 集成到主场景

**src/components/scene/MainScene.tsx**（再次更新）：
```typescript
import { UAVRenderer } from '@/components/uav/UAVRenderer';
import { ConnectionLine } from '@/components/satellite/ConnectionLine';
import { UAVPathGenerator } from '@/utils/uav/UAVPathGenerator';
import { UAV_CONFIG } from '@/config/uav.config';

export function MainScene() {
  const [uavPosition, setUAVPosition] = useState(UAV_CONFIG.flight.initialPosition);
  const [currentSatellite, setCurrentSatellite] = useState<string | null>(null);
  const [flightPath] = useState(() =>
    UAVPathGenerator.generateCirclePath(
      new Vector3(0, 0, 0),
      UAV_CONFIG.flight.radius,
      UAV_CONFIG.flight.altitude
    )
  );

  // UAV 位置更新
  useEffect(() => {
    const startTime = Date.now();
    const interval = setInterval(() => {
      const elapsed = (Date.now() - startTime) / 1000; // 秒
      const { position } = UAVPathGenerator.getPositionAtTime(
        flightPath,
        elapsed,
        UAV_CONFIG.flight.speed
      );
      setUAVPosition(position);
    }, 1000 / 60); // 60 FPS

    return () => clearInterval(interval);
  }, [flightPath]);

  // 选择最近的可见卫星
  useEffect(() => {
    if (visibleSatellites.size === 0) {
      setCurrentSatellite(null);
      return;
    }

    let closestId: string | null = null;
    let minDistance = Infinity;

    visibleSatellites.forEach((sat, id) => {
      const distance = uavPosition.distanceTo(sat.position);
      if (distance < minDistance) {
        minDistance = distance;
        closestId = id;
      }
    });

    setCurrentSatellite(closestId);
  }, [visibleSatellites, uavPosition]);

  return (
    <Canvas>
      {/* ... 其他组件 ... */}

      {/* UAV */}
      <UAVRenderer position={uavPosition} modelPath={UAV_CONFIG.model.path} scale={UAV_CONFIG.model.scale} />

      {/* 飞行路径可视化（调试用） */}
      <Line points={flightPath.map(p => p.toArray())} color="#ffff00" lineWidth={1} />

      {/* 卫星连接线 */}
      {currentSatellite && visibleSatellites.get(currentSatellite) && (
        <ConnectionLine
          from={uavPosition}
          to={visibleSatellites.get(currentSatellite).position}
          color="#00ff00"
          lineWidth={3}
          animated
        />
      )}
    </Canvas>
  );
}
```

### ✅ 验证标准
- [ ] UAV 模型成功渲染
- [ ] UAV 沿圆形路径飞行
- [ ] 连接线连接 UAV 和最近的可见卫星
- [ ] 连接线有动画效果（脉冲）
- [ ] 飞行路径平滑无卡顿

### 📦 交付物
- UAV 模型渲染
- 飞行路径生成器
- 卫星-UAV 连接线
- 动态连接可视化

---

## Phase 4: Handover 逻辑 (2-3 天)

### 🎯 目标
实现完整的切换决策算法和可视化反馈。

### 📝 任务清单

#### 4.1 信号强度估算器

**src/utils/handover/SignalStrengthEstimator.ts**：
```typescript
export interface SignalMetrics {
  rsrp: number; // dBm
  rsrq: number; // dB
  sinr: number; // dB
  pathLoss: number; // dB
}

export class SignalStrengthEstimator {
  private transmitPower: number; // dBm
  private frequency: number; // Hz
  private antennaGain: number; // dBi

  constructor(transmitPower = 20, frequency = 2e9, antennaGain = 15) {
    this.transmitPower = transmitPower;
    this.frequency = frequency;
    this.antennaGain = antennaGain;
  }

  /**
   * 估算信号强度（基于 Friis 传输方程）
   */
  estimateSignal(range: number, elevation: number): SignalMetrics {
    // 自由空间路径损耗（FSPL）
    const wavelength = 3e8 / this.frequency; // 光速 / 频率
    const pathLoss = 20 * Math.log10(range * 1000) + 20 * Math.log10(this.frequency) - 147.55;

    // 仰角衰减（简化模型）
    const elevationFactor = Math.max(0, Math.sin((elevation * Math.PI) / 180));
    const elevationLoss = -10 * Math.log10(elevationFactor + 0.1);

    // RSRP（参考信号接收功率）
    const rsrp = this.transmitPower + this.antennaGain - pathLoss - elevationLoss;

    // RSRQ（参考信号接收质量）
    const rsrq = rsrp - pathLoss / 2;

    // SINR（信噪比，简化）
    const sinr = rsrp + 10;

    return {
      rsrp,
      rsrq,
      sinr,
      pathLoss: pathLoss + elevationLoss,
    };
  }
}
```

#### 4.2 Handover 算法

**src/utils/handover/HandoverAlgorithm.ts**：
```typescript
import { VisibilityInfo } from '../satellite/VisibilityCalculator';
import { SignalStrengthEstimator, SignalMetrics } from './SignalStrengthEstimator';

export interface HandoverDecision {
  currentSatellite: string | null;
  targetSatellite: string | null;
  shouldHandover: boolean;
  reason: string;
  metrics: Map<string, SignalMetrics>;
}

export class HandoverAlgorithm {
  private minElevation: number;
  private hysteresis: number; // 滞后阈值
  private signalEstimator: SignalStrengthEstimator;

  constructor(minElevation = 10, hysteresis = 2) {
    this.minElevation = minElevation;
    this.hysteresis = hysteresis;
    this.signalEstimator = new SignalStrengthEstimator();
  }

  /**
   * 评估最佳卫星并决定是否切换
   */
  evaluateBestSatellite(
    visibleSatellites: Map<string, VisibilityInfo>,
    currentSatellite: string | null
  ): HandoverDecision {
    const metrics = new Map<string, SignalMetrics>();

    // 计算所有卫星的信号强度
    visibleSatellites.forEach((vis, id) => {
      const signal = this.signalEstimator.estimateSignal(vis.range, vis.elevation);
      metrics.set(id, signal);
    });

    // 找出信号最强的卫星
    let bestSatellite: string | null = null;
    let bestRsrp = -Infinity;

    metrics.forEach((signal, id) => {
      if (signal.rsrp > bestRsrp) {
        bestRsrp = signal.rsrp;
        bestSatellite = id;
      }
    });

    // 如果没有当前卫星，直接连接最佳卫星
    if (!currentSatellite) {
      return {
        currentSatellite: null,
        targetSatellite: bestSatellite,
        shouldHandover: true,
        reason: 'Initial connection',
        metrics,
      };
    }

    // 检查当前卫星是否仍然可见
    const currentMetrics = metrics.get(currentSatellite);
    if (!currentMetrics) {
      return {
        currentSatellite,
        targetSatellite: bestSatellite,
        shouldHandover: true,
        reason: 'Current satellite lost',
        metrics,
      };
    }

    // 应用滞后阈值：只有当最佳卫星的信号强度显著高于当前卫星时才切换
    const rsrpDifference = bestRsrp - currentMetrics.rsrp;
    if (rsrpDifference > this.hysteresis && bestSatellite !== currentSatellite) {
      return {
        currentSatellite,
        targetSatellite: bestSatellite,
        shouldHandover: true,
        reason: `Better signal (+${rsrpDifference.toFixed(1)} dB)`,
        metrics,
      };
    }

    // 保持当前连接
    return {
      currentSatellite,
      targetSatellite: currentSatellite,
      shouldHandover: false,
      reason: 'Stable connection',
      metrics,
    };
  }
}
```

#### 4.3 Handover 状态面板

**src/components/ui/HandoverStatusPanel.tsx**：
```typescript
import React from 'react';
import { Satellite, ArrowRight } from 'lucide-react';
import { HandoverDecision } from '@/utils/handover/HandoverAlgorithm';

interface HandoverStatusPanelProps {
  decision: HandoverDecision | null;
}

export function HandoverStatusPanel({ decision }: HandoverStatusPanelProps) {
  if (!decision) return null;

  const currentMetrics = decision.currentSatellite
    ? decision.metrics.get(decision.currentSatellite)
    : null;

  const targetMetrics = decision.targetSatellite
    ? decision.metrics.get(decision.targetSatellite)
    : null;

  return (
    <div className="handover-status-panel">
      <h3>
        <Satellite size={20} />
        Handover Status
      </h3>

      <div className="connection-status">
        <div className="satellite-info">
          <span className="label">Current:</span>
          <span className="value">{decision.currentSatellite || 'None'}</span>
          {currentMetrics && (
            <div className="metrics">
              <span>RSRP: {currentMetrics.rsrp.toFixed(1)} dBm</span>
              <span>SINR: {currentMetrics.sinr.toFixed(1)} dB</span>
            </div>
          )}
        </div>

        {decision.shouldHandover && (
          <>
            <ArrowRight className="handover-arrow" size={24} />
            <div className="satellite-info target">
              <span className="label">Target:</span>
              <span className="value">{decision.targetSatellite}</span>
              {targetMetrics && (
                <div className="metrics">
                  <span>RSRP: {targetMetrics.rsrp.toFixed(1)} dBm</span>
                  <span>SINR: {targetMetrics.sinr.toFixed(1)} dB</span>
                </div>
              )}
            </div>
          </>
        )}
      </div>

      <div className="status-reason">
        <span className={decision.shouldHandover ? 'handover-active' : 'stable'}>
          {decision.reason}
        </span>
      </div>
    </div>
  );
}
```

**src/styles/_handover.scss**：
```scss
.handover-status-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 20px;
  border-radius: 8px;
  min-width: 400px;
  font-family: monospace;
  z-index: 10;

  h3 {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 15px;
    font-size: 18px;
  }

  .connection-status {
    display: flex;
    align-items: center;
    gap: 15px;
    margin-bottom: 15px;
  }

  .satellite-info {
    flex: 1;
    padding: 10px;
    background: rgba(255, 255, 255, 0.1);
    border-radius: 4px;

    .label {
      display: block;
      font-size: 12px;
      color: #aaa;
      margin-bottom: 5px;
    }

    .value {
      display: block;
      font-size: 16px;
      font-weight: bold;
      color: #00ff00;
      margin-bottom: 10px;
    }

    .metrics {
      display: flex;
      flex-direction: column;
      gap: 5px;
      font-size: 12px;
      color: #ccc;
    }

    &.target {
      .value {
        color: #0080ff;
      }
    }
  }

  .handover-arrow {
    color: #ffaa00;
    animation: pulse 1s infinite;
  }

  .status-reason {
    text-align: center;
    padding: 8px;
    border-radius: 4px;

    .handover-active {
      color: #ffaa00;
      font-weight: bold;
    }

    .stable {
      color: #00ff00;
    }
  }
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}
```

#### 4.4 集成 Handover 系统

**src/components/scene/MainScene.tsx**（最终版本）：
```typescript
import { HandoverAlgorithm } from '@/utils/handover/HandoverAlgorithm';
import { HandoverStatusPanel } from '@/components/ui/HandoverStatusPanel';

export function MainScene() {
  const [handoverDecision, setHandoverDecision] = useState<HandoverDecision | null>(null);
  const [handoverAlgorithm] = useState(() => new HandoverAlgorithm(10, 2));

  // Handover 决策
  useEffect(() => {
    if (!visibilityCalc || visibleSatellites.size === 0) return;

    // 构建可见性信息 Map
    const visibilityMap = new Map();
    visibleSatellites.forEach((sat, id) => {
      const vis = visibilityCalc.calculateVisibility(sat.position, currentTime);
      visibilityMap.set(id, vis);
    });

    // 评估 handover
    const decision = handoverAlgorithm.evaluateBestSatellite(
      visibilityMap,
      currentSatellite
    );

    setHandoverDecision(decision);

    // 执行 handover
    if (decision.shouldHandover && decision.targetSatellite) {
      setCurrentSatellite(decision.targetSatellite);
    }
  }, [visibleSatellites, currentSatellite, currentTime, visibilityCalc, handoverAlgorithm]);

  // 更新卫星渲染状态
  const satellitesWithStatus = useMemo(() => {
    const updated = new Map(visibleSatellites);
    updated.forEach((sat, id) => {
      sat.isActive = id === currentSatellite;
    });
    return updated;
  }, [visibleSatellites, currentSatellite]);

  return (
    <>
      <Canvas>
        {/* ... */}
        <SatelliteRenderer satellites={satellitesWithStatus} modelPath="/models/sat.glb" scale={5} />

        {/* 连接线颜色根据状态变化 */}
        {currentSatellite && satellitesWithStatus.get(currentSatellite) && (
          <ConnectionLine
            from={uavPosition}
            to={satellitesWithStatus.get(currentSatellite).position}
            color={handoverDecision?.shouldHandover ? '#ffaa00' : '#00ff00'}
            lineWidth={3}
            animated
          />
        )}
      </Canvas>

      {/* UI 覆盖层 */}
      <HandoverStatusPanel decision={handoverDecision} />
    </>
  );
}
```

### ✅ 验证标准
- [ ] Handover 决策算法正常工作
- [ ] UI 面板显示当前和目标卫星信息
- [ ] 信号强度估算合理（基于仰角和距离）
- [ ] 卫星切换有视觉反馈（颜色变化、动画）
- [ ] 滞后阈值防止频繁切换

### 📦 交付物
- Handover 决策算法
- 信号强度估算器
- 状态显示面板
- 完整的切换可视化

---

## Phase 5: 优化与完善 (1-2 天)

### 🎯 目标
性能优化、UI 完善、添加控制面板。

### 📝 任务清单

#### 5.1 性能优化

**优化 1：轨道预计算**
```typescript
// 启动时预计算 120 分钟轨道
const [orbitCache, setOrbitCache] = useState(new Map<string, Vector3[]>());

useEffect(() => {
  if (!calculator) return;

  const cache = new Map();
  calculator.getAllSatelliteIds().forEach((id) => {
    const orbit = calculator.predictOrbit(id, new Date(), 120, 10);
    cache.set(id, orbit);
  });

  setOrbitCache(cache);
}, [calculator]);
```

**优化 2：节流更新**
```typescript
import { throttle } from 'lodash';

const updateSatellites = useMemo(
  () =>
    throttle(() => {
      // 卫星位置更新逻辑
    }, 1000), // 每秒更新一次
  []
);
```

**优化 3：使用 Web Worker（高级）**
```typescript
// src/workers/satellite.worker.ts
self.onmessage = (e) => {
  const { type, data } = e.data;

  if (type === 'calculatePositions') {
    const positions = calculateSatellitePositions(data);
    self.postMessage({ type: 'positions', data: positions });
  }
};
```

#### 5.2 控制面板

**src/components/ui/ControlPanel.tsx**：
```typescript
import React from 'react';
import { Play, Pause, RotateCcw, Settings } from 'lucide-react';

interface ControlPanelProps {
  isPlaying: boolean;
  timeSpeed: number;
  onPlayPause: () => void;
  onReset: () => void;
  onSpeedChange: (speed: number) => void;
  showLabels: boolean;
  onToggleLabels: () => void;
}

export function ControlPanel({
  isPlaying,
  timeSpeed,
  onPlayPause,
  onReset,
  onSpeedChange,
  showLabels,
  onToggleLabels,
}: ControlPanelProps) {
  return (
    <div className="control-panel">
      <div className="control-group">
        <button onClick={onPlayPause} className="control-button">
          {isPlaying ? <Pause size={20} /> : <Play size={20} />}
        </button>
        <button onClick={onReset} className="control-button">
          <RotateCcw size={20} />
        </button>
      </div>

      <div className="control-group">
        <label>Speed: {timeSpeed}x</label>
        <input
          type="range"
          min="1"
          max="100"
          value={timeSpeed}
          onChange={(e) => onSpeedChange(Number(e.target.value))}
        />
      </div>

      <div className="control-group">
        <label>
          <input type="checkbox" checked={showLabels} onChange={onToggleLabels} />
          Show Labels
        </label>
      </div>
    </div>
  );
}
```

#### 5.3 卫星标签

**src/components/satellite/SatelliteLabel.tsx**：
```typescript
import React from 'react';
import { Html } from '@react-three/drei';
import { Vector3 } from 'three';

interface SatelliteLabelProps {
  position: Vector3;
  name: string;
  isActive: boolean;
}

export function SatelliteLabel({ position, name, isActive }: SatelliteLabelProps) {
  return (
    <Html position={position.toArray()} center>
      <div className={`satellite-label ${isActive ? 'active' : ''}`}>
        {name}
      </div>
    </Html>
  );
}
```

#### 5.4 统计信息面板

**src/components/ui/StatsPanel.tsx**：
```typescript
import React from 'react';
import { Activity } from 'lucide-react';

interface StatsPanelProps {
  fps: number;
  visibleSatellites: number;
  totalSatellites: number;
  currentTime: Date;
}

export function StatsPanel({
  fps,
  visibleSatellites,
  totalSatellites,
  currentTime,
}: StatsPanelProps) {
  return (
    <div className="stats-panel">
      <h4>
        <Activity size={16} />
        Statistics
      </h4>
      <div className="stat-item">
        <span>FPS:</span>
        <span className="value">{fps.toFixed(0)}</span>
      </div>
      <div className="stat-item">
        <span>Satellites:</span>
        <span className="value">
          {visibleSatellites} / {totalSatellites}
        </span>
      </div>
      <div className="stat-item">
        <span>Time:</span>
        <span className="value">{currentTime.toLocaleTimeString()}</span>
      </div>
    </div>
  );
}
```

#### 5.5 样式完善

**src/styles/main.scss**（完整版）：
```scss
@import 'variables';
@import 'handover';

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  overflow: hidden;
  background: #000;
}

#root {
  width: 100vw;
  height: 100vh;
}

// 控制面板
.control-panel {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px 20px;
  border-radius: 8px;
  display: flex;
  gap: 30px;
  align-items: center;
  z-index: 10;

  .control-group {
    display: flex;
    align-items: center;
    gap: 10px;

    label {
      font-size: 14px;
    }

    input[type='range'] {
      width: 150px;
    }
  }

  .control-button {
    background: rgba(255, 255, 255, 0.1);
    border: none;
    color: white;
    padding: 10px;
    border-radius: 4px;
    cursor: pointer;
    display: flex;
    align-items: center;

    &:hover {
      background: rgba(255, 255, 255, 0.2);
    }
  }
}

// 统计面板
.stats-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px;
  border-radius: 8px;
  min-width: 200px;
  font-family: monospace;
  z-index: 10;

  h4 {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 10px;
  }

  .stat-item {
    display: flex;
    justify-content: space-between;
    padding: 5px 0;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);

    &:last-child {
      border-bottom: none;
    }

    .value {
      color: #00ff00;
      font-weight: bold;
    }
  }
}

// 卫星标签
.satellite-label {
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  white-space: nowrap;
  pointer-events: none;

  &.active {
    background: rgba(0, 255, 0, 0.7);
    color: black;
    font-weight: bold;
  }
}
```

### ✅ 验证标准
- [ ] 帧率稳定在 60 FPS
- [ ] 所有 UI 控件正常工作
- [ ] 时间加速功能正确
- [ ] 标签显示清晰
- [ ] 响应式布局适配不同屏幕

### 📦 交付物
- 性能优化代码
- 完整的控制面板
- 统计信息显示
- 卫星标签系统
- 完善的样式

---

## 🎉 完成标准

当完成所有阶段后，项目应满足：

1. ✅ 可独立运行，无需后端
2. ✅ 流畅的 3D 渲染（≥60 FPS）
3. ✅ 真实的卫星轨道计算
4. ✅ 清晰的 handover 可视化
5. ✅ 完整的交互控制
6. ✅ 代码结构清晰，易于维护
7. ✅ 完善的文档

---

## 📚 附录：快速命令参考

```bash
# 启动开发服务器
npm run dev

# 构建生产版本
npm run build

# 预览生产版本
npm run preview

# 代码检查
npm run lint

# 代码格式化
npm run format

# 安装依赖
npm install

# 清理重装
rm -rf node_modules package-lock.json && npm install
```

---

**文档版本**：v1.0
**创建日期**：2025-10-28
**最后更新**：2025-10-28
