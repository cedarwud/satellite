# 依赖项说明文档

## 前端依赖清单

### package.json

```json
{
  "name": "ntpu-sat-handover-viz",
  "version": "1.0.0",
  "type": "module",
  "description": "NTPU 卫星-UAV 切换可视化系统",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,scss}\""
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",

    "@react-three/fiber": "^9.1.2",
    "@react-three/drei": "^10.0.7",
    "three": "^0.170.0",

    "satellite.js": "^5.0.0",

    "date-fns": "^3.6.0",
    "lodash": "^4.17.21",
    "lucide-react": "^0.525.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@types/three": "^0.176.0",
    "@types/lodash": "^4.17.0",

    "typescript": "~5.7.2",

    "vite": "^6.3.1",
    "@vitejs/plugin-react": "^4.3.4",

    "sass": "^1.89.0",

    "eslint": "^9.0.0",
    "@typescript-eslint/parser": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "eslint-plugin-react": "^7.37.0",
    "eslint-plugin-react-hooks": "^5.0.0",

    "prettier": "^3.3.0"
  }
}
```

## 核心依赖说明

### 1. React 生态系统

#### react & react-dom (^19.0.0)
**用途**：前端 UI 框架
- 组件化开发
- 状态管理（Hooks）
- 虚拟 DOM 渲染

**关键 Hooks 使用**：
- `useState` - 状态管理
- `useEffect` - 副作用处理（时间更新、数据加载）
- `useRef` - 引用 Three.js 对象
- `useMemo` - 性能优化（卫星位置缓存）
- `useCallback` - 回调函数优化

---

### 2. 3D 渲染核心

#### @react-three/fiber (^9.1.2)
**用途**：React 的 Three.js 渲染器
- 将 Three.js 对象转换为 React 组件
- 声明式 3D 场景构建
- 自动处理渲染循环

**核心组件**：
```tsx
<Canvas>        // WebGL 画布容器
<mesh>          // 3D 网格
<primitive>     // 原生 Three.js 对象
<useFrame>      // 每帧更新 Hook
```

**优势**：
- 无需手动管理 Three.js 生命周期
- 与 React 状态无缝集成
- 性能优化（自动垃圾回收）

#### @react-three/drei (^10.0.7)
**用途**：Three.js 常用功能的 React 包装
- 提供开箱即用的辅助组件

**本项目使用的组件**：
```tsx
<OrbitControls />          // 轨道相机控制
<PerspectiveCamera />      // 透视相机
<Environment />            // 环境光照
<ContactShadows />         // 接触阴影
<Text />                   // 3D 文本
<Line />                   // 线段渲染
<Html />                   // 在 3D 场景中嵌入 HTML
<useGLTF />                // GLTF 模型加载 Hook
```

#### three (^0.170.0)
**用途**：核心 3D 图形库
- WebGL 渲染引擎
- 几何体、材质、灯光系统
- 数学工具（Vector3, Matrix4, Quaternion）

**本项目使用的类**：
```typescript
import {
  Vector3,           // 3D 向量
  Euler,             // 欧拉角
  Matrix4,           // 4x4 矩阵
  Spherical,         // 球坐标
  Color,             // 颜色
  Line,              // 线段
  BufferGeometry,    // 缓冲几何体
  InstancedMesh,     // 实例化网格
} from 'three';
```

---

### 3. 卫星轨道计算

#### satellite.js (^5.0.0)
**用途**：浏览器端 SGP4 卫星轨道计算库
- 解析 TLE（两行根数）数据
- 实现 SGP4/SDP4 模型
- 坐标系转换（ECI, ECEF, LLA）

**核心 API**：
```typescript
import * as satellite from 'satellite.js';

// 解析 TLE
const satrec = satellite.twoline2satrec(tleLine1, tleLine2);

// 计算位置和速度
const positionAndVelocity = satellite.propagate(satrec, date);

// 坐标转换
const positionEci = positionAndVelocity.position;
const positionEcef = satellite.eciToEcf(positionEci, date);

// 计算观测角度
const lookAngles = satellite.ecfToLookAngles(
  observerGeodetic,
  positionEcef
);
```

**数据结构**：
```typescript
interface TLE {
  line1: string;  // TLE 第一行
  line2: string;  // TLE 第二行
  name: string;   // 卫星名称
}

interface LookAngles {
  azimuth: number;    // 方位角（弧度）
  elevation: number;  // 仰角（弧度）
  rangeSat: number;   // 距离（km）
}
```

**性能考量**：
- SGP4 计算密集，需要缓存结果
- 建议使用 Web Worker 进行后台计算（阶段性优化）

---

### 4. 工具库

#### date-fns (^3.6.0)
**用途**：日期时间处理
- 时间格式化
- 时间加减运算
- 时区处理

**常用函数**：
```typescript
import {
  format,           // 格式化日期
  addSeconds,       // 添加秒数
  differenceInSeconds, // 计算时间差
  parseISO,         // 解析 ISO 字符串
} from 'date-fns';
```

#### lodash (^4.17.21)
**用途**：JavaScript 工具函数库

**常用函数**：
```typescript
import {
  throttle,         // 节流
  debounce,         // 防抖
  clamp,            // 限制数值范围
  sortBy,           // 排序
  groupBy,          // 分组
} from 'lodash';
```

#### lucide-react (^0.525.0)
**用途**：React 图标库
- 提供清晰的 SVG 图标
- 完全可定制

**本项目使用的图标**：
```tsx
import {
  Satellite,        // 卫星图标
  Plane,            // UAV 图标
  Play,             // 播放
  Pause,            // 暂停
  Settings,         // 设置
  Info,             // 信息
  ZoomIn,           // 放大
  ZoomOut,          // 缩小
} from 'lucide-react';
```

---

### 5. TypeScript

#### typescript (~5.7.2)
**用途**：类型安全的 JavaScript 超集
- 静态类型检查
- 更好的 IDE 支持
- 减少运行时错误

**tsconfig.json 配置**：
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
    "forceConsistentCasingInFileNames": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

**类型定义文件**：
```typescript
// src/types/satellite.ts
export interface SatelliteInfo {
  id: string;
  name: string;
  position: Vector3;
  velocity: Vector3;
  elevation: number;
  azimuth: number;
  range: number;
}

// src/types/handover.ts
export interface HandoverEvent {
  timestamp: Date;
  fromSatellite: string | null;
  toSatellite: string;
  reason: 'elevation' | 'signal_strength' | 'prediction';
}
```

---

### 6. 构建工具

#### vite (^6.3.1)
**用途**：快速的前端构建工具
- 极速热更新（HMR）
- 原生 ESM 支持
- 高效的生产构建（基于 Rollup）

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
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'three-vendor': ['three', '@react-three/fiber', '@react-three/drei'],
          'satellite-vendor': ['satellite.js'],
        },
      },
    },
  },
});
```

**优势**：
- 开发环境启动速度 < 1 秒
- 热更新响应时间 < 100ms
- 代码分割优化加载性能

#### @vitejs/plugin-react (^4.3.4)
**用途**：Vite 的 React 插件
- 支持 React Fast Refresh
- JSX 转换
- React 优化

---

### 7. CSS 预处理器

#### sass (^1.89.0)
**用途**：CSS 预处理器
- 变量、嵌套、混入
- 模块化样式管理

**目录结构**：
```
/src/styles/
  ├── _variables.scss     # 全局变量
  ├── _mixins.scss        # 混入
  ├── _base.scss          # 基础样式
  ├── _layout.scss        # 布局
  └── main.scss           # 主入口
```

**示例变量**：
```scss
// _variables.scss
$color-primary: #00ff00;
$color-satellite-current: #00ff00;
$color-satellite-target: #0080ff;
$color-satellite-transition: #ffaa00;

$z-index-canvas: 1;
$z-index-ui: 10;
$z-index-modal: 100;
```

---

### 8. 代码质量工具

#### ESLint (^9.0.0)
**用途**：JavaScript/TypeScript 代码检查

**.eslintrc.json**：
```json
{
  "parser": "@typescript-eslint/parser",
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended"
  ],
  "rules": {
    "react/react-in-jsx-scope": "off",
    "@typescript-eslint/no-unused-vars": "warn"
  }
}
```

#### Prettier (^3.3.0)
**用途**：代码格式化

**.prettierrc**：
```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

---

## 静态资源

### 3D 模型文件

需要从原项目复制的文件：

```bash
# 从 ntn-stack 复制到新项目
源路径：/ntn-stack/simworld/backend/app/static/models/
目标路径：/ntpu-sat-handover-viz/public/models/

文件：
- uav.glb (10.3 MB)
- sat.glb (2.6 MB)

源路径：/ntn-stack/simworld/backend/app/static/scenes/NTPU/
目标路径：/ntpu-sat-handover-viz/public/scenes/

文件：
- NTPU.glb (8.1 MB)
```

### TLE 数据

需要准备的数据文件：

```
/public/data/
  └── tle-data.json
```

**示例格式**：
```json
{
  "satellites": [
    {
      "id": "starlink-1",
      "name": "STARLINK-1",
      "tle1": "1 44713U 19074A   23300.50000000  .00001234  00000-0  12345-4 0  9999",
      "tle2": "2 44713  53.0000 123.4567 0001234  12.3456 347.6543 15.12345678123456"
    }
  ]
}
```

**数据来源**：
- https://celestrak.org/NORAD/elements/starlink.txt
- https://celestrak.org/NORAD/elements/oneweb.txt

---

## 安装命令

### 初始化项目
```bash
# 创建项目目录
mkdir ntpu-sat-handover-viz
cd ntpu-sat-handover-viz

# 初始化 package.json
npm init -y

# 安装所有依赖
npm install
```

### 手动安装核心依赖
```bash
# React
npm install react react-dom

# Three.js 生态
npm install three @react-three/fiber @react-three/drei

# 卫星计算
npm install satellite.js

# 工具库
npm install date-fns lodash lucide-react

# TypeScript 和类型定义
npm install -D typescript @types/react @types/react-dom @types/three @types/lodash

# Vite
npm install -D vite @vitejs/plugin-react

# 样式
npm install -D sass

# 代码质量
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D eslint-plugin-react eslint-plugin-react-hooks
npm install -D prettier
```

### 验证安装
```bash
# 检查版本
npm list react three satellite.js

# 启动开发服务器
npm run dev
```

---

## 浏览器兼容性

### 最低要求
- Chrome 90+
- Firefox 88+
- Safari 15+
- Edge 90+

### WebGL 支持
- WebGL 2.0（必需）
- 检测方法：
```typescript
function checkWebGLSupport(): boolean {
  try {
    const canvas = document.createElement('canvas');
    return !!(
      canvas.getContext('webgl2') ||
      canvas.getContext('webgl') ||
      canvas.getContext('experimental-webgl')
    );
  } catch (e) {
    return false;
  }
}
```

---

## 依赖更新策略

### 定期更新（每月）
```bash
# 检查过时的依赖
npm outdated

# 更新补丁版本
npm update

# 更新主版本（谨慎）
npm install react@latest
```

### 重要版本锁定
```json
{
  "dependencies": {
    "react": "19.0.0",           // 锁定主版本
    "three": "^0.170.0",         // 允许小版本更新
    "satellite.js": "~5.0.0"     // 只允许补丁更新
  }
}
```

---

**文档版本**：v1.0
**创建日期**：2025-10-28
**最后更新**：2025-10-28
