# 算法共享架構分析

## 📊 當前架構狀態

### 當前實現：orbit-engine 作為算法庫

```python
# handover-rl/src/adapters/orbit_engine_adapter.py

# 添加 orbit-engine 到 Python 路徑
ORBIT_ENGINE_ROOT = Path(__file__).parent.parent.parent.parent / "orbit-engine"
sys.path.insert(0, str(ORBIT_ENGINE_ROOT))

# 直接導入 orbit-engine 的算法模組
from src.stages.stage2_orbital_computing.sgp4_calculator import SGP4Calculator
from src.stages.stage5_signal_analysis.itur_physics_calculator import create_itur_physics_calculator
from src.stages.stage5_signal_analysis.gpp_ts38214_signal_calculator import create_3gpp_signal_calculator
from src.stages.stage5_signal_analysis.itur_official_atmospheric_model import create_itur_official_model
```

**共享的算法模組**：
1. ✅ **SGP4Calculator**: 軌道傳播（orbit-engine/stage2）
2. ✅ **ITURPhysicsCalculator**: 自由空間路徑損耗（orbit-engine/stage5）
3. ✅ **GPPTS38214SignalCalculator**: RSRP/RSRQ 計算（orbit-engine/stage5）
4. ✅ **ITUROfficialAtmosphericModel**: 大氣衰減（orbit-engine/stage5）
5. ✅ **TemporalFeatureCalculator**: 時間特徵計算（orbit-engine/ml_training_data_generator）⭐ 新增

## 🏗️ 架構模式分析

### 當前模式：直接依賴（Direct Dependency）

```
┌─────────────────────────────────────────────────────────┐
│ handover-rl (Git repo 2)                                 │
├─────────────────────────────────────────────────────────┤
│ src/adapters/                                            │
│ └─ orbit_engine_adapter.py                              │
│    ├─ sys.path.insert(orbit-engine/)                    │
│    └─ from orbit-engine.src.stages... import            │
│                                                          │
│         ↓ (Runtime dependency)                          │
│                                                          │
│ orbit-engine (Git repo 1)                               │
│ ├─ src/stages/stage2_orbital_computing/                 │
│ │  └─ sgp4_calculator.py          ← 被 handover-rl 使用 │
│ ├─ src/stages/stage5_signal_analysis/                   │
│ │  ├─ itur_physics_calculator.py  ← 被 handover-rl 使用 │
│ │  ├─ gpp_ts38214_signal_calculator.py ← 被使用         │
│ │  └─ itur_official_atmospheric_model.py ← 被使用       │
│ └─ run.sh (六階段處理) ← handover-rl 不使用              │
└─────────────────────────────────────────────────────────┘
```

### 優點 ✅

1. **避免代碼重複（DRY 原則）**
   - SGP4、ITU-R、3GPP 算法只維護一份
   - 修復 bug 一次，兩邊都受益
   - 確保兩個專案使用完全相同的算法（一致性）

2. **學術合規性保證**
   - orbit-engine 已經通過學術標準驗證
   - handover-rl 直接使用，繼承合規性
   - 不會出現「簡化版」算法

3. **維護成本低**
   - 只需維護 orbit-engine 的算法實現
   - handover-rl 的 adapter 只是薄層封裝

4. **版本一致性**
   - orbit-engine 更新算法，handover-rl 自動受益
   - 不會出現兩個倉庫算法版本不一致

### 缺點 ❌

1. **運行時依賴（Runtime Dependency）**
   ```bash
   # handover-rl 需要 orbit-engine 存在才能運行
   parent_directory/
   ├─ orbit-engine/      ← 必須存在！
   └─ handover-rl/       ← 依賴 orbit-engine
   ```
   - 不能單獨部署 handover-rl
   - Docker 部署需要複製兩個倉庫
   - CI/CD 需要 checkout 兩個倉庫

2. **Path Hacking (`sys.path.insert`）**
   ```python
   # 不是標準的 Python package 導入方式
   sys.path.insert(0, str(ORBIT_ENGINE_ROOT))
   ```
   - 容易受路徑變化影響
   - IDE 可能無法正確解析（影響 autocomplete）
   - 不符合 Python packaging 最佳實踐

3. **orbit-engine 內部路徑問題**
   ```python
   # 需要 chdir 到 orbit-engine 才能導入
   os.chdir(ORBIT_ENGINE_ROOT)
   from src.stages.stage2_orbital_computing.sgp4_calculator import SGP4Calculator
   os.chdir(_ORIGINAL_CWD)
   ```
   - orbit-engine 的內部導入使用相對路徑
   - 需要臨時改變工作目錄（不優雅）

4. **Git 倉庫耦合**
   - 雖然是兩個獨立倉庫，但邏輯上緊耦合
   - 無法單獨測試 handover-rl（需要 orbit-engine）

## 🎯 改進方案

### 方案 A：保持現狀（推薦短期）

**適用場景**：
- 團隊規模小（1-3人）
- 兩個專案都在活躍開發中
- 快速迭代階段

**改進措施**：
1. 文檔化依賴關係
2. 添加設置腳本確保兩個倉庫都存在
3. Docker 部署腳本處理多倉庫

**實施**：
```bash
# handover-rl/scripts/setup/check_dependencies.sh
#!/bin/bash

HANDOVER_RL_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
ORBIT_ENGINE_ROOT="$(dirname "$HANDOVER_RL_ROOT")/orbit-engine"

if [ ! -d "$ORBIT_ENGINE_ROOT" ]; then
    echo "❌ orbit-engine not found!"
    echo "   Expected location: $ORBIT_ENGINE_ROOT"
    echo ""
    echo "   Please ensure orbit-engine is cloned as a sibling directory:"
    echo "   parent_directory/"
    echo "   ├─ orbit-engine/      ← Required dependency"
    echo "   └─ handover-rl/       ← Current directory"
    echo ""
    echo "   Run: git clone https://github.com/your-org/orbit-engine.git $ORBIT_ENGINE_ROOT"
    exit 1
else
    echo "✅ orbit-engine found at: $ORBIT_ENGINE_ROOT"
fi

# 驗證關鍵模組存在
REQUIRED_MODULES=(
    "src/stages/stage2_orbital_computing/sgp4_calculator.py"
    "src/stages/stage5_signal_analysis/itur_physics_calculator.py"
    "src/stages/stage5_signal_analysis/gpp_ts38214_signal_calculator.py"
)

for module in "${REQUIRED_MODULES[@]}"; do
    if [ ! -f "$ORBIT_ENGINE_ROOT/$module" ]; then
        echo "❌ Required module not found: $module"
        exit 1
    fi
done

echo "✅ All required orbit-engine modules found"
```

---

### 方案 B：提取共享算法庫（推薦長期）

**適用場景**：
- 專案成熟期
- 團隊規模擴大
- 需要獨立部署

**架構設計**：
```
parent_directory/
├─ orbit-engine/              (Git repo 1 - 前端渲染)
│  ├─ src/stages/             ← 六階段處理流程
│  └─ run.sh
│
├─ handover-rl/               (Git repo 2 - RL 訓練)
│  ├─ src/agents/
│  └─ src/environments/
│
├─ tle_data/                  (Git repo 3 - 共享數據)
│  └─ starlink/tle/
│
└─ satellite-physics/         (Git repo 4 - 共享算法庫) ← 新建
   ├─ setup.py                ← Python package
   ├─ pyproject.toml
   └─ satellite_physics/
      ├─ __init__.py
      ├─ orbital/             ← SGP4 等軌道算法
      │  └─ sgp4_calculator.py
      ├─ signal/              ← 3GPP 信號計算
      │  ├─ gpp_signal_calculator.py
      │  └─ rsrp_calculator.py
      └─ propagation/         ← ITU-R 傳播模型
         ├─ itur_physics.py
         └─ atmospheric_model.py
```

**優點**：
- ✅ orbit-engine 和 handover-rl 都依賴 satellite-physics
- ✅ 標準 Python package（可以 `pip install -e ../satellite-physics`）
- ✅ 獨立測試、版本控制
- ✅ 可以發布到 PyPI（如果開源）

**缺點**：
- ❌ 初期工作量大（需要重構 orbit-engine）
- ❌ 增加一個倉庫（維護複雜度）
- ❌ 需要協調三個倉庫的版本

---

### 方案 C：Git Submodules（不推薦）

```
orbit-engine/
└─ lib/satellite-physics/  ← submodule

handover-rl/
└─ lib/satellite-physics/  ← same submodule
```

**為什麼不推薦**：
- Submodules 複雜，容易出錯
- 團隊協作困難（版本不一致）
- 本項目已經有 3 個獨立倉庫，再加 submodule 更複雜

---

### 方案 D：Python Package（pip installable）

將 orbit-engine 的算法打包成可安裝的 Python package：

```bash
# 在 orbit-engine 添加 setup.py
cd orbit-engine
pip install -e .  # 開發模式安裝

# 在 handover-rl 中安裝
cd handover-rl
pip install -e ../orbit-engine  # 從本地安裝

# 或從 PyPI 安裝（如果發布）
pip install satellite-orbit-engine
```

**優點**：
- ✅ 標準 Python 做法
- ✅ 不需要 `sys.path.insert` hacking
- ✅ IDE 自動識別

**缺點**：
- ❌ orbit-engine 需要重構為標準 Python package
- ❌ 前端渲染專案（orbit-engine）可能不適合作為 library

---

## 📋 推薦方案總結

### 當前階段（開發/驗證期）：**方案 A - 保持現狀**

**理由**：
1. ✅ 已經能工作，不要破壞
2. ✅ 團隊規模小，耦合可接受
3. ✅ 快速迭代，頻繁修改算法
4. ✅ 確保算法一致性

**行動項**：
- [x] 創建 TLE 數據架構文檔（已完成）
- [ ] 創建 `handover-rl/scripts/setup/check_dependencies.sh`
- [ ] 更新 `handover-rl/README.md` 說明依賴關係
- [ ] Docker/CI 腳本處理多倉庫

### 未來階段（生產/發布期）：**方案 B - 提取共享算法庫**

**時機**：
- orbit-engine 和 handover-rl 都穩定
- 算法不再頻繁修改
- 需要獨立部署或開源

**行動項**：
1. 分析 orbit-engine 哪些模組是純算法（無副作用）
2. 創建 satellite-physics 倉庫
3. 重構 orbit-engine 和 handover-rl 依賴新庫
4. 添加 CI/CD 自動測試

---

## 🔍 當前共享模組詳細分析

### 1. SGP4Calculator (軌道傳播)

**位置**: `orbit-engine/src/stages/stage2_orbital_computing/sgp4_calculator.py`

**功能**:
- 使用 Skyfield（NASA JPL）計算衛星位置
- 從 TLE 傳播到任意時間點
- 輸出 TEME 坐標

**是否適合共享**: ✅ **是**
- 純計算模組，無副作用
- orbit-engine 和 handover-rl 都需要
- 算法標準化（SGP4 是 NORAD 標準）

---

### 2. GPPTS38214SignalCalculator (RSRP/RSRQ)

**位置**: `orbit-engine/src/stages/stage5_signal_analysis/gpp_ts38214_signal_calculator.py`

**功能**:
- 計算 RSRP（參考信號接收功率）
- 計算 RSRQ（參考信號接收質量）
- 符合 3GPP TS 38.214/38.215 標準

**是否適合共享**: ✅ **是**
- RL 訓練核心依賴（reward 計算）
- 標準算法，不會頻繁修改
- 學術合規性關鍵組件

---

### 3. ITURPhysicsCalculator (路徑損耗)

**位置**: `orbit-engine/src/stages/stage5_signal_analysis/itur_physics_calculator.py`

**功能**:
- 自由空間路徑損耗（ITU-R P.525）
- 可能包含其他 ITU-R 模型

**是否適合共享**: ✅ **是**
- 物理公式，標準化
- RL 需要準確的信號衰減計算

---

### 4. ITUROfficialAtmosphericModel (大氣衰減)

**位置**: `orbit-engine/src/stages/stage5_signal_analysis/itur_official_atmospheric_model.py`

**功能**:
- ITU-R P.676-13 大氣衰減（44+35 條譜線）
- 考慮溫度、壓力、水汽密度

**是否適合共享**: ✅ **是**
- 學術標準算法
- 不會頻繁修改

---

### 5. TemporalFeatureCalculator (時間特徵) ⭐ 新增 2025-10-24

**位置**: `orbit-engine/tools/ml_training_data_generator/core/temporal_feature_calculator.py`

**功能**:
- Velocity 計算（dRSRP/dt, dDistance/dt）
- 未來 RSRP 預測（30s, 60s ahead）
- 基於 Finite difference method + Free-space path loss

**是否適合共享**: ✅ **是**
- RL 訓練核心依賴（狀態空間從 53 → 77 維）
- 標準數值方法（Finite difference）
- 物理公式（Friis transmission equation）

**學術依據**:
- **Velocity**: ∂RSRP/∂t ≈ (RSRP_t - RSRP_{t-Δt}) / Δt
- **Prediction**: RSRP_future = RSRP_current - 20 log₁₀(D_future / D_current)
- **SOURCE**: Finite difference method + ITU-R P.525 (Free-space path loss)

**handover-rl 使用**:
```python
# handover-rl/src/adapters/orbit_engine_adapter.py (未來整合)
from orbit_engine.tools.ml_training_data_generator.core.temporal_feature_calculator import TemporalFeatureCalculator

temporal_calc = TemporalFeatureCalculator()
rsrp_velocity, distance_velocity = temporal_calc.calculate_velocity_features(...)
predicted_rsrp = temporal_calc.predict_future_rsrp(...)
```

**狀態**: 2025-10-24 已實現，僅用於 orbit-engine ML data generator，handover-rl 尚未整合

---

## ⚠️ orbit-engine 不共享的部份

以下模組**不被 handover-rl 使用**：

1. **Stage 1-6 的資料處理流程**
   - 六階段的 processor、validator、executor
   - 這些是 orbit-engine 特定的業務邏輯
   - handover-rl 不需要（只需要實時計算）

2. **資料輸出管理**
   - results_manager、file_io
   - orbit-engine 需要保存大量中間結果
   - handover-rl 在 episode 中實時計算，不保存

3. **批次處理邏輯**
   - 平行處理、任務隊列
   - orbit-engine 處理 8000+ 衛星
   - handover-rl 只處理選定的 800 顆

---

## 🎓 最終建議

### 短期（現在）：保持當前架構 ✅

**原因**：
- 已經能工作，不要破壞
- 驗證階段，算法可能調整
- 團隊可以接受耦合

**改進措施**：
1. 創建依賴檢查腳本
2. 文檔化架構設計
3. 更新 README 說明 sibling repo 需求

### 長期（未來）：評估是否提取共享庫 ⚠️

**時機**：
- 兩個專案都穩定（不再頻繁修改算法）
- 團隊擴大（多人協作需要更清晰的邊界）
- 需要獨立部署（例如 handover-rl 部署到雲端）

**決策標準**：
```
如果滿足以下條件，考慮提取共享庫：
├─ 算法穩定（3 個月沒有大改動）
├─ 需要獨立部署 handover-rl
├─ 有資源維護第 4 個倉庫
└─ orbit-engine 願意重構為 library
```

---

## 📝 當前架構總結

```
✅ 優點：
├─ 算法一致性保證（兩邊用同一份代碼）
├─ 避免代碼重複（DRY 原則）
├─ 學術合規性繼承（orbit-engine 已驗證）
└─ 維護成本低（一處修改，兩邊受益）

⚠️ 缺點：
├─ 運行時依賴（handover-rl 需要 orbit-engine）
├─ Path hacking（sys.path.insert 不優雅）
├─ 部署複雜（需要兩個倉庫）
└─ IDE 支持弱（autocomplete 可能失效）

🎯 結論：
├─ 當前階段：保持現狀（成本效益最優）
├─ 未來階段：評估提取共享庫（視專案成熟度）
└─ 核心原則：不要過早優化
```

---

**文檔版本**: 1.1
**創建日期**: 2025-10-20
**最後更新**: 2025-10-24
**建議審查**: 專案穩定後 6 個月
