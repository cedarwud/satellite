# 快速参考手册

本文档提供项目开发过程中常用的代码片段、命令和配置，方便快速查阅。

---

## 📋 目录

- [常用命令](#常用命令)
- [配置文件模板](#配置文件模板)
- [代码片段](#代码片段)
- [调试技巧](#调试技巧)
- [常见错误](#常见错误)

---

## 常用命令

### 项目管理

```bash
# 初始化项目
npm init -y

# 安装所有依赖
npm install

# 清理并重新安装
rm -rf node_modules package-lock.json && npm install

# 更新依赖
npm update

# 检查过时的依赖
npm outdated
```

### 开发流程

```bash
# 启动开发服务器（热更新）
npm run dev

# 构建生产版本
npm run build

# 预览生产构建
npm run preview

# 类型检查
tsc --noEmit

# 代码检查
npm run lint

# 自动修复 lint 问题
npm run lint -- --fix

# 代码格式化
npm run format

# 格式化并检查
npm run format && npm run lint
```

### 资源管理

```bash
# 复制 3D 模型
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/uav.glb public/models/
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/models/sat.glb public/models/
cp /home/sat/satellite/ntn-stack/simworld/backend/app/static/scenes/NTPU/NTPU.glb public/scenes/

# 检查文件大小
ls -lh public/models/
ls -lh public/scenes/

# 下载 TLE 数据
curl -o public/data/starlink.txt https://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=tle
```

### Git 操作

```bash
# 初始化 Git
git init
git add .
git commit -m "Initial commit"

# 创建 .gitignore
cat > .gitignore << EOF
node_modules/
dist/
*.log
.DS_Store
.env
EOF

# 查看状态
git status

# 提交更改
git add .
git commit -m "feat: implement satellite rendering"

# 创建分支
git checkout -b feature/handover-algorithm
```

---

## 配置文件模板

### package.json（最小配置）

```json
{
  "name": "ntpu-sat-handover-viz",
  "version": "1.0.0",
  "type": "module",
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
    "prettier": "^3.3.0"
  }
}
```

### tsconfig.json

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
  "exclude": ["node_modules", "dist"]
}
```

### vite.config.ts

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

### .eslintrc.json

```json
{
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 2020,
    "sourceType": "module",
    "ecmaFeatures": {
      "jsx": true
    }
  },
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended"
  ],
  "rules": {
    "react/react-in-jsx-scope": "off",
    "@typescript-eslint/no-unused-vars": "warn",
    "@typescript-eslint/no-explicit-any": "warn"
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
```

### .prettierrc

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "arrowParens": "always"
}
```

---

## 代码片段

### 基础 React + Three.js 组件

```typescript
import React from 'react';
import { Canvas } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';

export function BasicScene() {
  return (
    <Canvas camera={{ position: [0, 100, 200], fov: 60 }}>
      <OrbitControls />
      <ambientLight intensity={0.5} />
      <directionalLight position={[10, 10, 5]} />

      <mesh>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="orange" />
      </mesh>
    </Canvas>
  );
}
```

### 加载 GLTF 模型

```typescript
import { useGLTF } from '@react-three/drei';

export function Model({ path }: { path: string }) {
  const { scene } = useGLTF(path);

  return <primitive object={scene} />;
}

// 预加载
useGLTF.preload('/models/sat.glb');
```

### SGP4 卫星计算

```typescript
import * as satellite from 'satellite.js';
import { Vector3 } from 'three';

function calculateSatellitePosition(
  tle1: string,
  tle2: string,
  date: Date
): Vector3 | null {
  try {
    const satrec = satellite.twoline2satrec(tle1, tle2);
    const positionAndVelocity = satellite.propagate(satrec, date);

    if (satellite.error(satrec)) return null;

    const positionEci = positionAndVelocity.position as satellite.EciVec3<number>;

    // ECI to Three.js coordinates
    return new Vector3(positionEci.x, positionEci.z, -positionEci.y);
  } catch (error) {
    console.error('SGP4 calculation failed:', error);
    return null;
  }
}
```

### 可见性计算

```typescript
import * as satellite from 'satellite.js';

function isVisible(
  satelliteEci: Vector3,
  observerLat: number,
  observerLon: number,
  observerAlt: number,
  date: Date,
  minElevation: number = 10
): boolean {
  const observerGeodetic = {
    latitude: satellite.degreesToRadians(observerLat),
    longitude: satellite.degreesToRadians(observerLon),
    height: observerAlt / 1000, // km
  };

  const positionEci: satellite.EciVec3<number> = {
    x: satelliteEci.x,
    y: -satelliteEci.z,
    z: satelliteEci.y,
  };

  const gmst = satellite.gstime(date);
  const positionEcf = satellite.eciToEcf(positionEci, gmst);
  const lookAngles = satellite.ecfToLookAngles(observerGeodetic, positionEcf);

  const elevation = satellite.radiansToDegrees(lookAngles.elevation);

  return elevation >= minElevation;
}
```

### 实例化网格（性能优化）

```typescript
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import { InstancedMesh, Matrix4, Color } from 'three';

export function InstancedSatellites({ satellites }: { satellites: Vector3[] }) {
  const meshRef = useRef<InstancedMesh>(null);
  const tempMatrix = useMemo(() => new Matrix4(), []);

  useFrame(() => {
    if (!meshRef.current) return;

    satellites.forEach((position, i) => {
      tempMatrix.setPosition(position);
      meshRef.current!.setMatrixAt(i, tempMatrix);
    });

    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={meshRef} args={[undefined, undefined, satellites.length]}>
      <sphereGeometry args={[5, 16, 16]} />
      <meshStandardMaterial color="#ffffff" />
    </instancedMesh>
  );
}
```

### 连接线

```typescript
import { Line } from '@react-three/drei';
import { Vector3 } from 'three';

export function ConnectionLine({ from, to }: { from: Vector3; to: Vector3 }) {
  return (
    <Line
      points={[from.toArray(), to.toArray()]}
      color="#00ff00"
      lineWidth={2}
      transparent
      opacity={0.6}
    />
  );
}
```

### React Hooks 示例

```typescript
// 时间更新 Hook
function useSimulationTime(speed: number = 1) {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    const interval = setInterval(() => {
      setTime((prev) => new Date(prev.getTime() + 1000 * speed));
    }, 1000);

    return () => clearInterval(interval);
  }, [speed]);

  return time;
}

// 卫星位置 Hook
function useSatellitePositions(
  calculator: SatelliteOrbitCalculator | null,
  time: Date
) {
  return useMemo(() => {
    if (!calculator) return new Map();

    const ids = calculator.getAllSatelliteIds();
    return calculator.calculateBatch(ids, time);
  }, [calculator, time]);
}
```

---

## 调试技巧

### 浏览器控制台调试

```typescript
// 添加到组件中进行调试
console.log('Component rendered');
console.log('Satellites:', satellites);
console.table(Array.from(satellites.entries()));

// 性能监控
console.time('calculation');
const result = calculator.calculateBatch(ids, date);
console.timeEnd('calculation');

// 帧率监控
let frames = 0;
let lastTime = performance.now();

useFrame(() => {
  frames++;
  const now = performance.now();
  if (now - lastTime >= 1000) {
    console.log(`FPS: ${frames}`);
    frames = 0;
    lastTime = now;
  }
});
```

### React DevTools

```bash
# 安装 React DevTools 浏览器扩展
# Chrome: https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi
# Firefox: https://addons.mozilla.org/en-US/firefox/addon/react-devtools/
```

### Three.js Inspector

```typescript
// 添加到场景中
import { useHelper } from '@react-three/drei';
import { BoxHelper } from 'three';

function DebugMesh({ meshRef }) {
  useHelper(meshRef, BoxHelper, 'cyan');
  return null;
}
```

### Performance 分析

```typescript
// 使用 React Profiler
import { Profiler } from 'react';

function onRenderCallback(
  id: string,
  phase: string,
  actualDuration: number
) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}

<Profiler id="MainScene" onRender={onRenderCallback}>
  <MainScene />
</Profiler>
```

### Network 监控

```typescript
// 监控模型加载
const { scene, error } = useGLTF(modelPath, true, true, (loader) => {
  loader.manager.onProgress = (url, loaded, total) => {
    console.log(`Loading ${url}: ${(loaded / total) * 100}%`);
  };
});
```

---

## 常见错误

### 错误 1：Cannot read property 'position' of undefined

**原因**：尝试访问尚未加载的对象

**解决**：
```typescript
// ❌ 错误
const position = satellites.get(id).position;

// ✅ 正确
const satellite = satellites.get(id);
if (satellite) {
  const position = satellite.position;
}

// ✅ 更好
const position = satellites.get(id)?.position;
```

### 错误 2：SGP4 error: mean eccentricity out of range

**原因**：TLE 数据过期或损坏

**解决**：
```typescript
try {
  const satrec = satellite.twoline2satrec(tle1, tle2);
  if (satellite.error(satrec)) {
    console.error('Invalid TLE data');
    return null;
  }
} catch (error) {
  console.error('Failed to parse TLE:', error);
  return null;
}
```

### 错误 3：WebGL context lost

**原因**：GPU 内存不足或浏览器标签页被挂起

**解决**：
```typescript
// 添加上下文恢复处理
const canvas = document.querySelector('canvas');
canvas?.addEventListener('webglcontextlost', (event) => {
  event.preventDefault();
  console.warn('WebGL context lost, attempting to restore...');
});

canvas?.addEventListener('webglcontextrestored', () => {
  console.log('WebGL context restored');
  // 重新初始化资源
});
```

### 错误 4：Module not found: Can't resolve '@/...'

**原因**：路径别名未配置

**解决**：
```typescript
// vite.config.ts
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});

// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### 错误 5：InstancedMesh count mismatch

**原因**：实例化网格的数量与实际对象数量不匹配

**解决**：
```typescript
// ❌ 错误：固定数量
<instancedMesh args={[geometry, material, 100]} />

// ✅ 正确：动态数量
const [satellites, setSatellites] = useState([]);

<instancedMesh args={[geometry, material, satellites.length]} />

// 注意：当数量变化时需要重新创建 mesh
const [meshKey, setMeshKey] = useState(0);

useEffect(() => {
  setMeshKey(prev => prev + 1);
}, [satellites.length]);

<instancedMesh key={meshKey} args={[geometry, material, satellites.length]} />
```

---

## 性能优化清单

### ✅ 渲染优化

```typescript
// 1. 使用 useMemo 缓存昂贵计算
const visibleSatellites = useMemo(() => {
  return filterVisibleSatellites(allSatellites, minElevation);
}, [allSatellites, minElevation]);

// 2. 使用 useCallback 避免函数重建
const handleClick = useCallback(() => {
  setSelected(id);
}, [id]);

// 3. 节流频繁更新
const updatePositions = useMemo(
  () => throttle(() => {
    // 更新逻辑
  }, 100),
  []
);

// 4. 条件渲染
{satellites.size > 0 && <SatelliteRenderer satellites={satellites} />}

// 5. 延迟加载
<Suspense fallback={<Loader />}>
  <HeavyComponent />
</Suspense>
```

### ✅ 资源优化

```typescript
// 1. 预加载模型
useGLTF.preload('/models/sat.glb');

// 2. 清理资源
useEffect(() => {
  return () => {
    geometry.dispose();
    material.dispose();
  };
}, []);

// 3. 限制渲染距离
<mesh renderOrder={0} frustumCulled>
  {/* ... */}
</mesh>

// 4. LOD (Level of Detail)
import { Lod } from '@react-three/drei';

<Lod distances={[0, 50, 100]}>
  <HighDetailMesh />
  <MediumDetailMesh />
  <LowDetailMesh />
</Lod>
```

---

## 快速测试

### 单元测试模板

```typescript
// tests/utils/SatelliteOrbitCalculator.test.ts
import { describe, it, expect } from 'vitest';
import { SatelliteOrbitCalculator } from '@/utils/satellite/SatelliteOrbitCalculator';

describe('SatelliteOrbitCalculator', () => {
  it('should calculate position correctly', () => {
    const calculator = new SatelliteOrbitCalculator([mockTLE]);
    const position = calculator.calculatePosition('test-sat', new Date());

    expect(position).toBeDefined();
    expect(position?.position.length()).toBeGreaterThan(0);
  });
});
```

### E2E 测试检查清单

- [ ] 页面加载成功
- [ ] NTPU 场景渲染
- [ ] 卫星显示在天空中
- [ ] UAV 沿路径飞行
- [ ] 连接线正确显示
- [ ] 控制面板功能正常
- [ ] 帧率 ≥ 60 FPS

---

## 有用的 VSCode 扩展

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "pmndrs.react-three-fiber-snippets",
    "stkb.rewrap"
  ]
}
```

---

## 快速链接

- [Three.js Docs](https://threejs.org/docs/)
- [React Three Fiber](https://docs.pmnd.rs/react-three-fiber)
- [Drei Helpers](https://github.com/pmndrs/drei)
- [Satellite.js](https://github.com/shashwatak/satellite-js)
- [TLE Data Source](https://celestrak.org/)

---

**最后更新**：2025-10-28
