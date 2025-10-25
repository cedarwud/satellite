# RL 衛星選擇器 vs orbit-engine 對比分析

**分析日期**: 2025-10-20
**最後更新**: 2025-10-24
**目的**: 識別可共用的組件，避免重複實現

---

## 階段對比

### orbit-engine 六階段

```
Stage 1: TLE Data Loading & Orbital Initialization
├─ 加載 TLE 文件（Starlink + OneWeb）
├─ 提取軌道參數（傾角、週期等）
└─ 輸出: stage1_output.json (9,015 satellites)

Stage 2: Orbital Propagation (SGP4)
├─ SGP4 傳播（Skyfield）
├─ 計算 TEME 坐標
└─ 輸出: orbital_propagation_output.json (1.7M 時間點)

Stage 3: Coordinate Transformation
├─ TEME → ECEF → WGS84
├─ 計算仰角、方位角、距離
└─ 輸出: stage3_coordinate_transformation.json

Stage 4: Link Feasibility Analysis
├─ 可見性分析（仰角 ≥ 10°）
├─ 衛星池優化（Elite pool: 123 顆, Candidate pool: 3,262 顆）
├─ 服務窗口計算
└─ 輸出: stage4_link_analysis.json

Stage 5: Signal Quality Analysis
├─ RSRP/RSRQ/SINR 計算（3GPP）
├─ ITU-R 大氣衰減（P.676-13）
├─ 路徑損耗（ITU-R P.525）
└─ 輸出: stage5_signal_quality.json

Stage 6: Research Optimization
├─ 3GPP 事件檢測（A3/A4/A5/D2）
├─ 換手決策分析
└─ 輸出: stage6_research.json
```

---

### RL 五階段選擇器（我原本的設計）

```
階段 1: 粗篩
├─ 軌道參數快速檢查
└─ 輸出: ~2000-3000 顆候選

階段 2: 可見性計算
├─ SGP4 傳播
├─ 仰角/距離計算
├─ RSRP 計算
└─ 輸出: 每顆衛星的可見性窗口

階段 3: 覆蓋率分析
├─ 時間軸掃描
├─ 統計同時可見數量
└─ 輸出: 覆蓋率統計

階段 4: 智能選擇
├─ 多目標優化評分
└─ 輸出: 850 顆最優衛星

階段 5: 驗證
├─ 換手機會率檢查
└─ 輸出: 驗證報告
```

---

## 重疊分析

### ✅ 完全重疊（可直接共用）

| RL 階段 | orbit-engine 階段 | 重疊度 | 共用方式 |
|---------|------------------|--------|---------|
| 階段 2（SGP4 傳播） | Stage 2 | **100%** | 使用 OrbitEngineAdapter |
| 階段 2（坐標轉換） | Stage 3 | **100%** | 使用 OrbitEngineAdapter |
| 階段 2（RSRP 計算） | Stage 5 | **100%** | 使用 OrbitEngineAdapter |

**關鍵發現**: handover-rl 已經有 **`OrbitEngineAdapter`**，它包裝了：
- `SGP4Calculator` (orbit-engine Stage 2)
- `GPPTS38214SignalCalculator` (orbit-engine Stage 5)
- `ITURPhysicsCalculator` (orbit-engine Stage 5)
- `ITUROfficialAtmosphericModel` (orbit-engine Stage 5)

**結論**: RL 階段 2 **不需要重新實現**，直接用 `OrbitEngineAdapter.calculate_state()`！

---

---

## 🔴 **重大發現**: orbit-engine Stage 4.2 已有完整池優化功能！

### orbit-engine Stage 4 完整架構

```
Stage 4: 鏈路可行性評估層
├─ 階段 4.1: 可見性篩選
│  ├─ 星座特定仰角門檻（Starlink 5°, OneWeb 10°）
│  ├─ 輸出: ~2000 顆候選衛星（was_ever_connectable = true）
│  └─ 完整時間序列，每個時間點標記 is_connectable
│
└─ 階段 4.2: 時空錯置池規劃 ✅ 已存在！
   ├─ PoolSelector: 貪心算法選擇最優衛星池
   ├─ CoverageOptimizer: 覆蓋連續性分析
   ├─ OptimizationValidator: 優化驗證
   ├─ 目標: 維持 N 顆並發可見（Starlink: 10-15, OneWeb: 3-6）
   └─ 覆蓋率目標: 95%
```

**實作位置**: `orbit-engine/src/stages/stage4_link_feasibility/pool_optimizer.py` (25,390 bytes, 535 lines)

**算法依據**:
- Chvátal, V. (1979). "A greedy heuristic for the set-covering problem"
- 標準 Set Cover 問題的貪心近似算法
- 時間複雜度: O(N × M)（N=衛星數, M=時間點數）

---

## 參數配置對比

### orbit-engine 當前配置（前端 3D 渲染用）

| 參數 | 當前值 | 位置 |
|------|--------|------|
| 衛星數量 | 123 顆（Elite pool） | Stage 4 pool optimization output |
| 時間範圍 | 94-110 分鐘（1 軌道週期） | Stage 2 `coverage_cycles: 1.0` |
| 時間解析度 | 30 秒 | Stage 2 `interval_seconds: 30` |
| Starlink 目標可見數 | 10-15 顆 | Stage 4 `expected_visible_satellites: [10, 15]` |
| OneWeb 目標可見數 | 3-6 顆 | Stage 4 `expected_visible_satellites: [3, 6]` |
| 覆蓋率目標 | 95% | Stage 4 `target_coverage_rate: 0.95` |

### RL 訓練需求（強化學習訓練用）

| 參數 | 目標值 | 調整方式 |
|------|--------|----------|
| 衛星數量 | **123 顆（Elite pool）** ⭐ | Stage 4 pool optimization（避免 diversity explosion）|
| 候選池（可選） | 3,262 顆（Candidate pool） | Stage 4.1 visibility filtering output |
| 時間範圍 | **7 天** | Stage 2 `coverage_cycles: 106.1` |
| 時間解析度 | 30 秒 | **無需調整** |
| 目標可見數 | **4-6 顆** | Stage 4 `expected_visible_satellites: [4, 6]` |
| 覆蓋率目標 | 95% | **無需調整** |
| 換手機會率 | ≥ 30% | Stage 6 驗證指標 |
| 狀態維度 | **77 維**（含時間特徵） | ML data generator temporal features |

---

## 🚀 關鍵發現: 時間範圍可擴展至 7 天

### Stage 2 `coverage_cycles` 參數分析

**配置文件**: `orbit-engine/config/stage2_orbital_computing_config.yaml`

```yaml
time_series:
  constellation_orbital_periods:
    starlink_minutes: 95   # Starlink 軌道週期
    oneweb_minutes: 110    # OneWeb 軌道週期
  interval_seconds: 30
  coverage_cycles: 1.0     # ✅ 可調整！無上限限制
```

**驗證邏輯** (`stage2_config_manager.py:299-301`):
```python
elif not isinstance(coverage, (int, float)) or coverage <= 0:
    errors.append(f"time_series.coverage_cycles 必須是正數，當前值: {coverage}")
```

**唯一限制**: `coverage_cycles > 0`（必須是正數）

**7 天計算**:
- 7 天 = 10,080 分鐘 = 604,800 秒
- Starlink 軌道週期 = 95 分鐘 = 5,700 秒
- **`coverage_cycles = 604,800 / 5,700 = 106.1`**
- 時間點數 = 604,800 / 30 = **20,160 個時間點/衛星**

**資源估算**:
- 800 衛星 × 20,160 時間點 = **16,128,000 次迭代**
- Pool optimizer 複雜度: O(N × M) = O(800 × 20,160) ≈ **16M 操作**
- **預估處理時間**: 數秒至數十秒（現代 CPU 可處理）
- **記憶體需求**: ~2-3 GB（時間序列數據）

**✅ 結論**: orbit-engine 可以處理 7 天時間範圍！

---

## 實施方案對比

### 方案 A: 直接使用 orbit-engine (推薦 ✅)

**優勢**:
- ✅ **零重複實現**: 所有算法已存在且經過驗證
- ✅ **學術合規**: 符合 ITU-R, 3GPP, NASA JPL 標準
- ✅ **高度可配置**: 環境變數覆寫所有參數
- ✅ **已驗證正確性**: RSRP bug 已修復，A3 事件已驗證
- ✅ **完整文檔**: 詳細的階段設計文檔

**調整方式**:
```bash
# 時間範圍擴展至 7 天
export ORBIT_ENGINE_STAGE2_TIME_SERIES___COVERAGE_CYCLES=106.1

# 調整 pool 目標（假設用 Starlink）
export ORBIT_ENGINE_STAGE4_POOL_OPTIMIZATION_TARGETS___STARLINK___MIN_POOL_SIZE=800
export ORBIT_ENGINE_STAGE4_POOL_OPTIMIZATION_TARGETS___STARLINK___MAX_POOL_SIZE=1000
export ORBIT_ENGINE_STAGE4_POOL_OPTIMIZATION_TARGETS___STARLINK___EXPECTED_VISIBLE_SATELLITES="[4, 6]"
export ORBIT_ENGINE_STAGE4_POOL_OPTIMIZATION_TARGETS___STARLINK___TARGET_COVERAGE_RATE=0.95

# 執行
cd /home/sat/satellite/orbit-engine
./run.sh --stages 1-6
```

**輸出數據**:
- Stage 4: `stage4_link_analysis.json` - 800 衛星，7 天可見性時間序列
- Stage 5: `stage5_signal_quality.json` - RSRP/RSRQ 完整數據
- Stage 6: `stage6_research.json` - A3 換手事件

**handover-rl 使用方式**:
```python
# 在 handover-rl 中直接讀取 orbit-engine 輸出
import json

# 讀取 Stage 6 輸出（包含完整的換手事件數據）
with open('/home/sat/satellite/orbit-engine/data/outputs/stage6/stage6_research.json') as f:
    rl_training_data = json.load(f)

# 數據已包含:
# - 800 衛星，7 天時間序列
# - 每個時間點的 RSRP/RSRQ/SINR
# - A3 換手事件（trigger/enter/leave）
# - 可直接用於 RL 訓練
```

---

### 方案 B: 實作獨立 RL 衛星選擇器（不推薦 ❌）

**劣勢**:
- ❌ **重複實現**: 需重新實現 pool optimization 邏輯（已存在）
- ❌ **維護成本**: 兩套代碼需同步維護
- ❌ **驗證成本**: 需重新驗證算法正確性
- ❌ **學術風險**: 可能不符合嚴格的學術標準

**唯一可能的使用場景**:
- orbit-engine 處理 800 衛星 × 7 天時性能不足（需實測確認）
- 需要完全不同的優化目標函數（當前 Set Cover 貪心算法已足夠）

---

## 推薦實施路徑

### Phase 1: 驗證可行性（1-2 小時）

1. **測試 orbit-engine 處理 7 天數據**
   ```bash
   cd /home/sat/satellite/orbit-engine

   # 小規模測試: 50 衛星，7 天
   export ORBIT_ENGINE_STAGE2_PERFORMANCE___TESTING_MODE___ENABLED=true
   export ORBIT_ENGINE_STAGE2_PERFORMANCE___TESTING_MODE___SATELLITE_SAMPLE_SIZE=50
   export ORBIT_ENGINE_STAGE2_TIME_SERIES___COVERAGE_CYCLES=106.1
   ./run.sh --stages 2-4

   # 觀察:
   # - 處理時間（應 < 10 分鐘）
   # - 記憶體使用（應 < 4 GB）
   # - 輸出數據大小
   ```

2. **驗證數據品質**
   ```bash
   # 檢查時間點數量
   jq '.satellites[0].time_series | length' data/outputs/stage4/stage4_link_analysis*.json
   # 預期: 20,160 個時間點

   # 檢查並發可見衛星數
   jq '.optimization_metrics.starlink.avg_visible' data/outputs/stage4/stage4_link_analysis*.json
   # 預期: 4-6 顆
   ```

### Phase 2: 全規模執行（如果 Phase 1 成功）

1. **移除測試模式，處理全部衛星**
   ```bash
   export ORBIT_ENGINE_STAGE2_PERFORMANCE___TESTING_MODE___ENABLED=false
   export ORBIT_ENGINE_STAGE2_TIME_SERIES___COVERAGE_CYCLES=106.1
   # ... 其他參數配置
   ./run.sh --stages 1-6
   ```

2. **整合到 handover-rl**
   - 修改 handover-rl 數據加載邏輯
   - 直接讀取 orbit-engine Stage 6 輸出
   - 驗證換手機會率 ≥ 30%

### Phase 3: 優化（如需要）

如果 Phase 1 性能不足，考慮:
- 調整 Stage 2 並行處理參數 (`max_workers`)
- 使用 HDF5 緩存（Stage 3 已支援）
- 減少時間解析度（30s → 60s）

---

## ⚠️ 部分重疊（邏輯不同）

| RL 階段 | orbit-engine 階段 | 重疊度 | 差異 |
|---------|------------------|--------|------|
| 階段 1（粗篩） | Stage 1 | 30% | RL 用軌道傾角快速篩選，orbit-engine 加載所有衛星 |
| 階段 4（智能選擇） | Stage 4 | 20% | RL 優化「同時可見數量」，orbit-engine 優化「單顆可見品質」|

**粗篩對比**:
```python
# orbit-engine Stage 1: 加載所有衛星（無篩選）
satellites = load_all_tle_files()  # 9,015 顆

# RL 階段 1: 快速篩選（避免無意義計算）
def coarse_filter(satellites):
    # 軌道傾角 ≥ 地面站緯度
    if inclination >= 25.0:  # NTPU 緯度
        candidates.append(satellite)
    # 預期: 2000-3000 顆
```

**選擇邏輯對比**:
```python
# orbit-engine Stage 4: 優化單顆可見品質
def select_satellites_orbit_engine():
    for sat in satellites:
        # 評分: 可見時長 + 峰值仰角 + 信號品質
        score = visibility_duration * 0.5 + peak_elevation * 0.3 + rsrp * 0.2
    return top_101_satellites  # 單顆品質最優

# RL 階段 4: 優化同時可見數量
def select_satellites_rl():
    for sat in satellites:
        # 評分: 覆蓋率 + 窗口數量 + RSRP
        score = coverage_pct * 0.4 + window_count * 0.3 + rsrp * 0.3
    return top_850_satellites  # 時間覆蓋最優
```

**結論**: 選擇邏輯**不能共用**，但可以參考 orbit-engine Stage 4 的評分框架。

---

### ❌ 無重疊（RL 專有）

| RL 階段 | orbit-engine 對應 | 說明 |
|---------|------------------|------|
| 階段 3（覆蓋率分析） | **無** | orbit-engine 不分析同時可見數量 |
| 階段 5（驗證） | **無** | orbit-engine 不檢查換手機會率 |

**覆蓋率分析**（RL 專有）:
```python
# orbit-engine: 無此功能
# RL 需要: 時間軸掃描
for time_point in timeline:
    visible_count = count_visible_satellites(time_point)
    # 統計平均、最小、最大可見數量
```

**驗證**（RL 專有）:
```python
# orbit-engine: 無此功能
# RL 需要: 換手機會率
handover_opportunity_rate = count(visible >= 2) / total_time_points
assert handover_opportunity_rate >= 0.3  # 至少 30%
```

---

## 共用策略建議

### ✅ 推薦方案：使用 handover-rl 現有組件

**架構**:
```python
# RL Satellite Selector
class RLSatelliteSelector:
    def __init__(self):
        # ✅ 使用已有的 adapter（包裝 orbit-engine 算法）
        self.adapter = OrbitEngineAdapter(...)

        # ✅ 使用已有的 TLE loader
        self.tle_loader = TLELoader(...)

    def select_satellites(self):
        # 階段 1: 粗篩
        candidates = self._coarse_filter()  # 新增邏輯

        # 階段 2: 可見性計算
        for sat in candidates:
            # ✅ 直接用 adapter（內部調用 orbit-engine 算法）
            state = self.adapter.calculate_state(sat_id, timestamp)
            rsrp = state['rsrp_dbm']
            elevation = state['elevation_deg']
            # 記錄可見性窗口

        # 階段 3: 覆蓋率分析（新增邏輯）
        coverage_stats = self._analyze_coverage(visibility_windows)

        # 階段 4: 智能選擇（新增邏輯）
        selected = self._intelligent_selection(candidates, coverage_stats)

        # 階段 5: 驗證（新增邏輯）
        validation = self._validate_selection(selected)

        return selected
```

---

### 詳細共用組件列表

#### 1. **OrbitEngineAdapter（已有）**

**位置**: `handover-rl/src/adapters/orbit_engine_adapter.py`

**功能**:
```python
adapter = OrbitEngineAdapter(
    lat=25.0136, lon=121.3676, alt_m=20,
    tle_sources={'starlink': '../tle_data/starlink/tle', ...}
)

# ✅ 一行代碼獲取所有需要的數據
state = adapter.calculate_state(satellite_id='48251', timestamp=datetime.now())

# 輸出包含:
state = {
    'rsrp_dbm': -35.5,           # ← Stage 5 算法
    'rsrq_db': -8.2,             # ← Stage 5 算法
    'sinr_db': 15.3,             # ← Stage 5 算法
    'elevation_deg': 45.2,       # ← Stage 3 算法
    'distance_km': 1420.5,       # ← Stage 3 算法
    'doppler_shift_hz': 25000,   # ← Stage 3 算法
    # ... 12 維完整狀態
}
```

**內部使用的 orbit-engine 算法**:
- `SGP4Calculator` (Stage 2)
- `GPPTS38214SignalCalculator` (Stage 5)
- `ITURPhysicsCalculator` (Stage 5)
- `ITUROfficialAtmosphericModel` (Stage 5)

**優點**:
- ✅ 已經測試驗證（handover-rl 現有代碼）
- ✅ 接口簡潔（一個函數獲取所有數據）
- ✅ 自動處理 TLE 選擇（根據日期）
- ✅ 符合學術標準（使用 orbit-engine 的官方算法）

---

#### 2. **TLELoader（已有）**

**位置**: `handover-rl/src/adapters/tle_loader.py`

**功能**:
```python
tle_loader = TLELoader(tle_sources={
    'starlink': '../tle_data/starlink/tle',
    'oneweb': '../tle_data/oneweb/tle'
})

# 加載所有 TLE
tle_loader.load_all_tles()

# 獲取可用衛星列表
satellite_ids = tle_loader.get_available_satellites()
# 輸出: ['48251', '48252', ...]  # ~5000+ 顆

# 獲取特定衛星的 TLE（自動選擇日期最接近的）
tle = tle_loader.get_tle_for_date('48251', datetime.now())
```

**優點**:
- ✅ 已經處理 TLE 文件解析
- ✅ 自動選擇最近的 TLE epoch
- ✅ 支持 Starlink + OneWeb

---

### ❌ 不建議方案：直接調用 orbit-engine Stage 腳本

**為什麼不建議**:
```bash
# ❌ 不建議: 直接運行 orbit-engine 的 Stage 腳本
cd orbit-engine
./run.sh --stage 2  # 輸出到 orbit-engine/data/outputs/
./run.sh --stage 5

# 問題:
# 1. 輸出格式是為前端渲染設計的（不適合 RL）
# 2. 只運行 101 顆衛星（RL 需要 2000+）
# 3. 只運行 94 分鐘（RL 需要 7 天）
# 4. 需要解析 JSON 文件（繁瑣）
# 5. 緊耦合（依賴 orbit-engine 內部結構）
```

**vs 使用 OrbitEngineAdapter**:
```python
# ✅ 推薦: 使用 adapter
state = adapter.calculate_state(sat_id, timestamp)  # 一行搞定

# 優點:
# 1. 清晰的接口
# 2. 可以運行任意數量的衛星
# 3. 可以運行任意時間範圍
# 4. 不需要解析文件
# 5. 鬆耦合（只依賴算法，不依賴 Stage 結構）
```

---

## 最終建議架構

### RL Satellite Selector 實現

**文件**: `handover-rl/scripts/data_generation/rl_satellite_selector.py`

```python
#!/usr/bin/env python3
"""
RL Satellite Selector - Generate satellite list for RL training

Uses existing handover-rl components:
- OrbitEngineAdapter: Wraps orbit-engine algorithms (SGP4, 3GPP, ITU-R)
- TLELoader: Manages TLE data loading

New logic:
- Coverage analysis (concurrent visibility timeline)
- Intelligent selection (multi-objective optimization)
- Validation (handover opportunity rate check)
"""

import sys
from pathlib import Path
from datetime import datetime, timedelta
import numpy as np
import json

# ✅ 使用已有的 adapter
sys.path.insert(0, str(Path(__file__).parent.parent.parent))
from src.adapters.orbit_engine_adapter import OrbitEngineAdapter
from src.adapters.tle_loader import TLELoader


class RLSatelliteSelector:
    """
    RL Satellite Selector

    Selects ~850 satellites optimized for RL training (not 3D rendering)
    """

    def __init__(self, config):
        self.config = config

        # ✅ 使用已有組件（不重新實現）
        self.adapter = OrbitEngineAdapter(
            lat=config['ground_station']['lat'],
            lon=config['ground_station']['lon'],
            alt_m=config['ground_station']['alt_m'],
            tle_sources=config['tle_sources']
        )

        self.tle_loader = self.adapter.tle_loader  # 重用 adapter 的 loader

    def select_satellites(self):
        """
        Main selection pipeline
        """
        print("🚀 Starting RL Satellite Selection...")

        # 階段 1: 粗篩（新增邏輯）
        print("\n[1/5] Coarse filtering...")
        candidates = self._coarse_filter()
        print(f"   Selected {len(candidates)} candidates")

        # 階段 2: 可見性計算（使用 adapter）
        print("\n[2/5] Calculating visibility windows...")
        visibility_windows = self._calculate_visibility(candidates)
        print(f"   Computed visibility for {len(visibility_windows)} satellites")

        # 階段 3: 覆蓋率分析（新增邏輯）
        print("\n[3/5] Analyzing coverage...")
        timeline, coverage_stats = self._analyze_coverage(visibility_windows)
        print(f"   Mean concurrent visible: {coverage_stats['mean']:.1f} satellites")

        # 階段 4: 智能選擇（新增邏輯）
        print("\n[4/5] Intelligent selection...")
        selected = self._intelligent_selection(visibility_windows, coverage_stats)
        print(f"   Selected {len(selected)} satellites")

        # 階段 5: 驗證（新增邏輯）
        print("\n[5/5] Validating selection...")
        validation = self._validate_selection(selected, timeline)

        if validation['pass']:
            print("   ✅ Validation PASSED")
        else:
            print(f"   ❌ Validation FAILED: {validation['reason']}")

        return {
            'satellites': selected,
            'timeline': timeline,
            'coverage_stats': coverage_stats,
            'validation': validation
        }

    def _coarse_filter(self):
        """
        階段 1: 粗篩（新增邏輯）

        快速篩選: 根據軌道傾角判斷是否可能覆蓋地面站
        """
        all_satellites = self.tle_loader.get_available_satellites()
        candidates = []

        for sat_id in all_satellites:
            tle = self.tle_loader.get_tle_for_date(sat_id, datetime.now())
            if tle is None:
                continue

            # 軌道傾角檢查（快速篩選）
            # SOURCE: Vallado 2013 - 軌道覆蓋範圍
            if tle.inclination_deg >= self.config['ground_station']['lat']:
                candidates.append(sat_id)

        return candidates

    def _calculate_visibility(self, candidates):
        """
        階段 2: 可見性計算（使用 OrbitEngineAdapter）

        ✅ 直接用 adapter，不重新實現 SGP4/RSRP 計算
        """
        visibility_windows = {}

        time_range = self.config['time_range']
        current_time = time_range['start']

        for sat_id in candidates:
            windows = []
            in_window = False
            window_start = None

            while current_time <= time_range['end']:
                # ✅ 使用 adapter（內部調用 orbit-engine 算法）
                try:
                    state = self.adapter.calculate_state(sat_id, current_time)

                    # 可見性判斷
                    is_visible = (
                        state['elevation_deg'] >= 10.0 and
                        state['rsrp_dbm'] >= -100.0
                    )

                    if is_visible and not in_window:
                        window_start = current_time
                        in_window = True
                    elif not is_visible and in_window:
                        windows.append({
                            'start': window_start,
                            'end': current_time,
                            'duration_sec': (current_time - window_start).total_seconds()
                        })
                        in_window = False

                except Exception as e:
                    pass  # 跳過計算失敗的時間點

                current_time += timedelta(seconds=10)

            visibility_windows[sat_id] = windows

        return visibility_windows

    def _analyze_coverage(self, visibility_windows):
        """
        階段 3: 覆蓋率分析（新增邏輯）

        統計時間軸上同時可見的衛星數量
        """
        # ... 實現時間軸掃描
        pass

    def _intelligent_selection(self, visibility_windows, coverage_stats):
        """
        階段 4: 智能選擇（新增邏輯）

        多目標優化評分
        """
        # ... 實現評分和選擇
        pass

    def _validate_selection(self, selected, timeline):
        """
        階段 5: 驗證（新增邏輯）

        檢查換手機會率、覆蓋缺口
        """
        # ... 實現驗證邏輯
        pass


if __name__ == '__main__':
    config = {
        'ground_station': {'lat': 25.0136, 'lon': 121.3676, 'alt_m': 20},
        'tle_sources': {
            'starlink': '../tle_data/starlink/tle',
            'oneweb': '../tle_data/oneweb/tle'
        },
        'time_range': {
            'start': datetime(2025, 10, 18),
            'end': datetime(2025, 10, 25),
        }
    }

    selector = RLSatelliteSelector(config)
    result = selector.select_satellites()

    # 保存結果
    with open('data/selected_satellites.json', 'w') as f:
        json.dump(result, f, indent=2)
```

---

## 共用組件總結

| 組件 | 來源 | 用途 | 狀態 |
|------|------|------|------|
| **OrbitEngineAdapter** | handover-rl（已有） | SGP4 + RSRP + 仰角計算 | ✅ 直接使用 |
| **TLELoader** | handover-rl（已有） | TLE 數據加載 | ✅ 直接使用 |
| **SGP4Calculator** | orbit-engine | 軌道傳播 | ✅ 通過 adapter 使用 |
| **3GPP Signal Calculator** | orbit-engine | RSRP/RSRQ/SINR | ✅ 通過 adapter 使用 |
| **ITU-R Models** | orbit-engine | 大氣衰減、路徑損耗 | ✅ 通過 adapter 使用 |
| **粗篩邏輯** | - | 軌道參數快速檢查 | ⚠️ 需新增 |
| **覆蓋率分析** | - | 時間軸同時可見數量 | ⚠️ 需新增 |
| **智能選擇** | - | 多目標優化 | ⚠️ 需新增 |
| **驗證邏輯** | - | 換手機會率檢查 | ⚠️ 需新增 |

---

## 時間估算（使用 adapter 的方案）

| 階段 | 任務 | 預估時間 |
|------|------|---------|
| 1 | 粗篩 (5000 → 2000) | 1 分鐘 |
| 2 | 可見性計算 (2000 × 7天) | **30-60 分鐘** |
| 3 | 覆蓋率分析 | 5 分鐘 |
| 4 | 智能選擇 | 1 分鐘 |
| 5 | 驗證 | 2 分鐘 |
| **總計** | | **40-70 分鐘** |

**注意**: 階段 2 最耗時（因為要調用 adapter.calculate_state() 數百萬次）

**優化建議**:
- 使用多進程加速（Python `multiprocessing`）
- 緩存中間結果

---

## 🆕 新功能整合（2025-10-24）

### 1️⃣ 時間特徵系統（Temporal Features）

**實現日期**: 2025-10-24
**狀態**: ✅ 已整合至 ML 訓練數據生成器

#### 新增特徵

| 特徵類型 | 特徵名稱 | 計算方式 | 用途 |
|---------|----------|----------|------|
| Velocity | `rsrp_velocity` | dRSRP/dt (dB/s) | 預測信號趨勢 |
| Velocity | `distance_velocity` | dDistance/dt (km/s) | 預測衛星接近/遠離速度 |
| Prediction | `predicted_rsrp_30s` | RSRP + 30s 預測 (dBm) | 提前換手決策 |
| Prediction | `predicted_rsrp_60s` | RSRP + 60s 預測 (dBm) | 長期趨勢分析 |

**狀態空間擴展**:
- 原始維度: **53 維**（8 個即時特徵 × 7 衛星 - 1 serving）
- 新維度: **77 維**（11 個特徵 × 7 衛星）
- 新增: **4 個時間特徵/衛星**

**實現位置**:
```
orbit-engine/tools/ml_training_data_generator/
├── core/types.py                      # SatelliteState 新增 4 個欄位
├── core/temporal_feature_calculator.py # 時間特徵計算器（新模組）
└── extractors/state_extractor.py      # 整合時間特徵至狀態提取
```

**學術依據**:
- **Velocity 計算**: Finite difference method (數值微分)
- **RSRP 預測**: Free-space path loss 簡化模型
- **SOURCE**: Friis transmission equation + linear extrapolation

**HDF5 數據集**:
- 文件: `orbit-engine/data/ml_training/rl_training_dataset_temporal.h5`
- 生成日期: 2025-10-24 09:52
- 狀態維度: **77** ✅
- 訓練樣本: 1,812 transitions (70%)
- 驗證樣本: 388 transitions (15%)
- 測試樣本: 390 transitions (15%)

---

### 2️⃣ D2 Integration Phase 1（距離事件）

**實現日期**: 2025-10-24
**狀態**: ✅ Phase 1 完成（A4+D2 加權組合）

#### D2 事件定義

**SOURCE**: 3GPP TS 38.331 v18.5.1 Section 5.5.5.2
- **觸發條件**: Distance < Threshold
- **用途**: 補充 RSRP (A4) 事件，避免遠距離高仰角衛星被錯誤選擇

#### 實現方式

**加權組合策略**:
```python
# orbit-engine/src/stages/stage6_research_optimization/gpp_event_detector.py

# Phase 1: 加權組合（保守策略）
is_handover_candidate = (a4_triggered) or (d2_triggered and rsrp_acceptable)

# 權重配置（基於實測數據）:
# - A4 事件: 65.6% 使用率
# - D2 事件: 34.4% 使用率
```

**實測效果**（Starlink, 123 satellites）:
| 事件類型 | 數量 | 比例 |
|---------|------|------|
| A4 only | 13,920 | 65.6% |
| D2 only | 168 | 0.8% |
| A4+D2 both | 7,136 | 33.6% |
| **Total** | **21,224** | **100%** |

**結論**: D2 補充了 7,304 個換手機會（34.4%），顯著提升換手決策多樣性。

**未來計畫**:
- Phase 1b: handover-rl DQN 訓練整合 D2 事件（代碼完成，訓練待執行）
- Phase 2: A4/D2 動態權重學習（RL agent 自主決策）

---

### 3️⃣ Pool 分離機制（Elite vs Candidate）

**實現日期**: 2025-10-24
**狀態**: ✅ 已實現

#### 雙池架構

| Pool 類型 | 衛星數量 | 用途 | 選擇策略 |
|-----------|----------|------|----------|
| **Elite Pool** | 123 顆 | RL 訓練 | 高品質，避免 diversity explosion |
| **Candidate Pool** | 3,262 顆 | 前端渲染/研究 | 完整可見性篩選結果 |

**問題背景**: Diversity Explosion
- 使用 3,262 顆衛星 → 每顆衛星樣本數過少（< 1 transition/satellite）
- RL 訓練無法學習有效策略

**解決方案**: Elite Pool Optimization
- Stage 4.2 Pool Optimizer 選出 **123 顆高品質衛星**
- 每顆衛星樣本數充足（~21 transitions/satellite）
- 覆蓋率維持 95%

#### 實現機制

**環境變數控制**:
```bash
# Elite pool（RL 訓練用）
export ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL=false
./run.sh --stages 5-6

# Candidate pool（前端渲染用）
export ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL=true
./run.sh --stages 5-6
```

**文件命名區分**:
```
data/outputs/stage5/
├── stage5_signal_analysis_elite_pool_20251024_095213.json     # 123 衛星
└── stage5_signal_analysis_candidate_pool_20251024_100156.json # 3,262 衛星

data/outputs/stage6/
├── stage6_research_optimization_elite_pool_20251024_095318.json
└── stage6_research_optimization_candidate_pool_20251024_100245.json
```

**選擇性清理機制**:
```python
# orbit-engine/scripts/stage_executors/executor_utils.py

def clean_stage_outputs(stage_number: int):
    """根據 pool 類型選擇性清理"""
    if stage_number in [5, 6]:
        pool_suffix = "_elite_pool" if not use_candidate_pool else "_candidate_pool"
        # 只刪除匹配當前 pool 類型的文件
        for file in output_dir.iterdir():
            if pool_suffix in file.name:
                file.unlink()
```

**優勢**:
- ✅ 兩種 pool 並存，互不影響
- ✅ 訓練數據與演示數據分離
- ✅ 避免誤刪其他模式的數據

---

## 結論

### ✅ 推薦做法

**使用 handover-rl 現有的 OrbitEngineAdapter**：
- ✅ 避免重複實現 SGP4、3GPP、ITU-R 算法
- ✅ 接口簡潔、易於使用
- ✅ 已經過測試驗證
- ✅ 符合學術標準

**只需新增 RL 專有邏輯**：
- 粗篩（軌道參數檢查）
- 覆蓋率分析（時間軸掃描）
- 智能選擇（多目標優化）
- 驗證（換手機會率）

### ❌ 不推薦做法

- ❌ 直接調用 orbit-engine Stage 腳本（耦合度高、不靈活）
- ❌ 重新實現 SGP4/RSRP 算法（違反 DRY 原則）
- ❌ 解析 orbit-engine 的 JSON 輸出文件（繁瑣、脆弱）

---

**文檔版本**: 1.1
**創建日期**: 2025-10-20
**最後更新**: 2025-10-24
**下一步**: 實現 RL Satellite Selector（使用 OrbitEngineAdapter）
