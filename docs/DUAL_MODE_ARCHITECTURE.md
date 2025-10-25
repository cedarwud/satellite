# orbit-engine 雙模式架構設計

**創建日期**: 2025-10-20
**最後更新**: 2025-10-24
**狀態**: ✅ 已實現（實際採用環境變數方式）
**目的**: 支援前端渲染和 RL 訓練兩種模式並存，互不干擾

---

## 問題分析

### 當前限制
orbit-engine 目前只有**單一配置模式**：
- 配置文件: `config/stage*.yaml`
- 輸出目錄: `data/outputs/stageN/`
- 執行方式: `./run.sh --stages 1-6`

如果直接調整配置參數（如 `coverage_cycles: 106.1`），會導致：
- ❌ 前端渲染輸出被覆蓋（Elite pool → Candidate pool）
- ❌ 無法同時維護兩種輸出
- ❌ 開發/測試/生產環境混亂
- ❌ 訓練數據和演示數據互相覆蓋

---

## 設計目標

### 需求
1. ✅ **前端渲染模式**（原始）：123 衛星（Elite pool），94 分鐘，供 Blender 3D 可視化
2. ✅ **RL 訓練模式**（新增）：123 衛星（Elite pool），7 天，供強化學習訓練
3. ✅ **並存運行**：兩種模式可以獨立執行，輸出分開存放
4. ✅ **零干擾**：修改 RL 配置不影響前端渲染
5. ✅ **Pool 分離**：Elite (123) vs Candidate (3,262) pool 互不覆蓋

### 非需求
- ❌ 不需要同時執行兩種模式（資源消耗過大）
- ❌ 不需要合併輸出（數據結構完全不同）

---

## 解決方案：獨立配置 + 專用腳本

### 架構概覽

```
orbit-engine/
├── config/                              # 前端渲染配置（原始，保持不變）
│   ├── stage1_orbital_initialization_config.yaml
│   ├── stage2_orbital_computing_config.yaml
│   ├── stage3_coordinate_transformation_config.yaml
│   ├── stage4_link_feasibility_config.yaml
│   ├── stage5_signal_analysis_config.yaml
│   └── stage6_research_optimization_config.yaml
│
├── config/rl_training/                  # RL 訓練配置（新增）
│   ├── stage1_rl_config.yaml
│   ├── stage2_rl_config.yaml
│   ├── stage3_rl_config.yaml
│   ├── stage4_rl_config.yaml
│   ├── stage5_rl_config.yaml
│   └── stage6_rl_config.yaml
│
├── data/outputs/                        # 前端渲染輸出（原始）
│   ├── stage1/
│   ├── stage2/
│   ├── ...
│   └── stage6/
│
├── data/outputs/rl_training/            # RL 訓練輸出（新增）
│   ├── stage1/
│   ├── stage2/
│   ├── ...
│   └── stage6/
│
├── run.sh                               # 前端渲染執行（原始）
└── scripts/generate_rl_training_data.sh # RL 訓練執行（新增）
```

---

## 實施細節

### 1. 創建 RL 訓練配置目錄

```bash
mkdir -p /home/sat/satellite/orbit-engine/config/rl_training
```

### 2. RL 訓練配置差異

**只需調整關鍵參數**（其餘保持與原始配置一致）：

#### Stage 1: `stage1_rl_config.yaml`
```yaml
# 基於原始配置，調整衛星篩選邏輯
satellite_selection:
  max_satellites: null          # 移除上限（原始: 1000）
  selection_mode: "coverage_optimized"  # 覆蓋優化模式
  target_satellite_count: 800   # 目標 800 衛星
```

#### Stage 2: `stage2_rl_config.yaml`
```yaml
# 關鍵差異：時間範圍擴展至 7 天
time_series:
  coverage_cycles: 106.1        # 7 天（原始: 1.0 = 94分鐘）
  interval_seconds: 30          # 保持 30 秒
```

#### Stage 4: `stage4_rl_config.yaml`
```yaml
# 關鍵差異：池優化目標調整
pool_optimization_targets:
  starlink:
    expected_visible_satellites: [4, 6]   # 原始: [10, 15]
    min_pool_size: 800                    # 原始: 10
    max_pool_size: 1000                   # 原始: 15
    target_coverage_rate: 0.95            # 保持 95%
```

#### Stage 5/6: 保持一致
```yaml
# Stage 5 和 Stage 6 配置與原始相同
# 只是處理更多衛星和更長時間範圍
```

### 3. 創建 RL 訓練執行腳本

**文件**: `scripts/generate_rl_training_data.sh`

```bash
#!/bin/bash
# RL 訓練數據生成專用腳本
# 使用獨立配置，輸出到獨立目錄

SCRIPT_DIR="$(dirname "$0")"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
CONFIG_DIR="$PROJECT_ROOT/config/rl_training"
OUTPUT_DIR="$PROJECT_ROOT/data/outputs/rl_training"

echo "🎯 RL 訓練數據生成模式"
echo "   配置目錄: $CONFIG_DIR"
echo "   輸出目錄: $OUTPUT_DIR"
echo ""

# 創建輸出目錄
mkdir -p "$OUTPUT_DIR"/{stage1,stage2,stage3,stage4,stage5,stage6}

# 設置環境變數，指定配置目錄
export ORBIT_ENGINE_CONFIG_DIR="$CONFIG_DIR"
export ORBIT_ENGINE_OUTPUT_DIR="$OUTPUT_DIR"

# 執行六階段處理
cd "$PROJECT_ROOT"
source venv/bin/activate
python scripts/run_six_stages_with_validation.py "$@"
```

### 4. 修改配置加載邏輯（最小侵入性）

**文件**: `src/shared/configs/config_manager.py`

```python
def _get_default_config_path(self) -> Path:
    """獲取預設配置文件路徑（支援 RL 訓練模式）"""
    stage_num = self.get_stage_number()

    # 檢查是否使用 RL 訓練配置
    config_dir = os.getenv('ORBIT_ENGINE_CONFIG_DIR')
    if config_dir:
        # RL 訓練模式：使用自定義配置目錄
        config_path = Path(config_dir) / f"stage{stage_num}_rl_config.yaml"
    else:
        # 前端渲染模式（原始）：使用標準配置
        from .executor_utils import project_root
        config_path = project_root / "config" / f"stage{stage_num}_*_config.yaml"

    return config_path
```

### 5. 修改輸出路徑邏輯

**文件**: `scripts/stage_executors/executor_utils.py`

```python
def get_output_dir(stage_number: int) -> Path:
    """獲取輸出目錄（支援 RL 訓練模式）"""
    output_base = os.getenv('ORBIT_ENGINE_OUTPUT_DIR')
    if output_base:
        # RL 訓練模式
        return Path(output_base) / f"stage{stage_number}"
    else:
        # 前端渲染模式（原始）
        return project_root / "data" / "outputs" / f"stage{stage_number}"
```

---

## 使用方式

### 前端渲染模式（原始，保持不變）

```bash
cd /home/sat/satellite/orbit-engine

# 執行全部六階段（101 衛星，94 分鐘）
./run.sh --stages 1-6

# 輸出位置
ls data/outputs/stage6/
# → stage6_research_optimization_*.json (101 衛星，220 時間點)
```

### RL 訓練模式（新增）

```bash
cd /home/sat/satellite/orbit-engine

# 執行 RL 訓練數據生成（800 衛星，7 天）
./scripts/generate_rl_training_data.sh --stages 1-6

# 輸出位置
ls data/outputs/rl_training/stage6/
# → stage6_research_optimization_*.json (800 衛星，20,160 時間點)
```

### 兩種模式並存驗證

```bash
# 1. 執行前端渲染
./run.sh --stages 1-6

# 2. 執行 RL 訓練
./scripts/generate_rl_training_data.sh --stages 1-6

# 3. 驗證兩種輸出都存在
ls data/outputs/stage6/*.json
# → 101 衛星版本

ls data/outputs/rl_training/stage6/*.json
# → 800 衛星版本
```

---

## 參數對比表

| 參數 | 前端渲染模式 | RL 訓練模式 | 配置位置 |
|------|-------------|------------|---------|
| 衛星數量 | 101 顆 | 800 顆 | Stage 1 `max_satellites` |
| 時間範圍 | 94 分鐘（1 週期） | 7 天（106 週期） | Stage 2 `coverage_cycles` |
| 時間解析度 | 30 秒 | 30 秒 | Stage 2 `interval_seconds` |
| 目標可見數 | 10-15 顆 | 4-6 顆 | Stage 4 `expected_visible_satellites` |
| 池大小 | 10-15 顆 | 800-1000 顆 | Stage 4 `min/max_pool_size` |
| 覆蓋率目標 | 95% | 95% | Stage 4 `target_coverage_rate` |
| 輸出時間點 | ~220 點 | ~20,160 點 | 自動計算 |
| 輸出目錄 | `data/outputs/` | `data/outputs/rl_training/` | 環境變數 |
| 配置目錄 | `config/` | `config/rl_training/` | 環境變數 |

---

## 預期輸出大小

### 前端渲染模式
- Stage 1: ~500 KB (9,040 衛星元數據)
- Stage 2: ~50 MB (101 衛星 × 220 時間點)
- Stage 4: ~20 MB (101 衛星可見性)
- Stage 6: ~4 MB (101 衛星事件數據)
- **總計**: ~75 MB

### RL 訓練模式
- Stage 1: ~500 KB (9,040 衛星元數據，篩選至 800)
- Stage 2: ~4 GB (800 衛星 × 20,160 時間點)
- Stage 4: ~1.5 GB (800 衛星可見性)
- Stage 6: ~300 MB (800 衛星事件數據)
- **總計**: ~6 GB

### 磁碟空間需求
- 前端渲染: ~100 MB
- RL 訓練: ~7 GB
- **建議保留**: 10 GB 空閒空間

---

## 優勢

### ✅ 零干擾
- 前端渲染配置完全不變
- RL 訓練配置獨立維護
- 輸出目錄分離，無覆蓋風險

### ✅ 可維護性
- 配置文件清晰對應模式
- Git 版本控制友好
- 容易回滾/對比

### ✅ 擴展性
- 未來可添加更多模式（如 `config/testing/`）
- 環境變數機制靈活

### ✅ 學術合規
- 所有配置都有明確的 SOURCE 標註
- 參數差異有清晰的學術依據
- 可重現性強

---

## 實施步驟

### Phase 1: 創建 RL 配置（1-2 小時）
```bash
# 1. 複製原始配置
cp -r config config/rl_training

# 2. 調整 RL 配置參數（見上述差異表）
vim config/rl_training/stage2_rl_config.yaml  # coverage_cycles: 106.1
vim config/rl_training/stage4_rl_config.yaml  # pool_size: 800-1000

# 3. 驗證配置語法
python -c "import yaml; yaml.safe_load(open('config/rl_training/stage2_rl_config.yaml'))"
```

### Phase 2: 創建執行腳本（30 分鐘）
```bash
# 1. 創建 RL 訓練腳本
cat > scripts/generate_rl_training_data.sh << 'EOF'
[見上述腳本內容]
EOF

# 2. 賦予執行權限
chmod +x scripts/generate_rl_training_data.sh
```

### Phase 3: 修改配置加載邏輯（1 小時）
```bash
# 修改 config_manager.py 和 executor_utils.py
# （見上述代碼修改）
```

### Phase 4: 小規模測試（1 小時）
```bash
# 測試 RL 模式（50 衛星）
export ORBIT_ENGINE_STAGE2_PERFORMANCE___TESTING_MODE___ENABLED=true
export ORBIT_ENGINE_STAGE2_PERFORMANCE___TESTING_MODE___SATELLITE_SAMPLE_SIZE=50
./scripts/generate_rl_training_data.sh --stages 2-4

# 驗證輸出
ls data/outputs/rl_training/stage4/
jq '.satellites | length' data/outputs/rl_training/stage4/*.json
# 預期: 50 衛星
```

### Phase 5: 全規模執行（2-3 小時）
```bash
# 執行完整 RL 訓練數據生成
./scripts/generate_rl_training_data.sh --stages 1-6

# 驗證與前端渲染輸出並存
ls data/outputs/stage6/*.json              # 101 衛星
ls data/outputs/rl_training/stage6/*.json  # 800 衛星
```

---

## 風險評估

### 低風險
- ✅ 配置分離，原始配置零修改
- ✅ 環境變數機制成熟
- ✅ 輸出目錄分離，無覆蓋風險

### 中風險
- ⚠️ 代碼修改涉及配置加載邏輯（需仔細測試）
- ⚠️ RL 模式處理時間長（800 衛星 × 7 天）

### 緩解措施
- 先在測試分支進行修改
- 使用 50 衛星小規模測試驗證邏輯
- 監控資源使用（記憶體、磁碟空間）

---

## handover-rl 整合

### 數據加載
```python
# handover-rl/src/data/rl_data_loader.py
import json
from pathlib import Path

def load_rl_training_data():
    """從 orbit-engine RL 訓練輸出加載數據"""
    orbit_engine_path = Path('/home/sat/satellite/orbit-engine')
    stage6_output = orbit_engine_path / 'data/outputs/rl_training/stage6'

    # 找到最新的輸出文件
    latest_file = max(stage6_output.glob('stage6_*.json'), key=lambda p: p.stat().st_mtime)

    with open(latest_file) as f:
        data = json.load(f)

    # 提取 A4/D2 事件
    a4_events = data['gpp_events']['a4_events']
    d2_events = data['gpp_events']['d2_events']

    # 提取 ML 訓練數據
    ml_data = data['ml_training_data']

    return {
        'a4_events': a4_events,
        'd2_events': d2_events,
        'ml_data': ml_data,
        'satellites': data['pool_verification']['satellites'],
        'time_points': len(ml_data['timestamps'])
    }
```

---

## 🆕 實際實現（2025-10-24）

### 實現方式差異

**原提案**：獨立配置文件 + 專用輸出目錄
**實際採用**：環境變數控制 + 檔名後綴區分

#### 為何採用環境變數方式？

| 面向 | 配置文件方式 | 環境變數方式（實際） |
|------|-------------|---------------------|
| **複雜度** | 需複製 6 個配置文件 | 只需設定 1 個環境變數 ✅ |
| **維護成本** | 兩組配置需同步更新 | 單一配置源，零重複 ✅ |
| **執行方式** | 需專用腳本 | 原有 run.sh + export ✅ |
| **靈活性** | 靜態配置 | 動態切換 ✅ |
| **適用場景** | 參數差異大 | 只有 pool 類型差異 ✅ |

**結論**: 當前需求下，環境變數方式更簡潔高效。

---

### 實際實現細節

#### 環境變數控制

```bash
# Elite pool（RL 訓練用，123 顆衛星）
export ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL=false
./run.sh --stages 5-6

# Candidate pool（前端渲染用，3,262 顆衛星）
export ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL=true
./run.sh --stages 5-6
```

#### 檔名自動區分

**Stage 5 輸出**:
```
data/outputs/stage5/
├── stage5_signal_analysis_elite_pool_20251024_095213.json
└── stage5_signal_analysis_candidate_pool_20251024_100156.json
```

**Stage 6 輸出**:
```
data/outputs/stage6/
├── stage6_research_optimization_elite_pool_20251024_095318.json
└── stage6_research_optimization_candidate_pool_20251024_100245.json
```

#### 選擇性清理機制

```python
# orbit-engine/scripts/stage_executors/executor_utils.py:clean_stage_outputs()

def clean_stage_outputs(stage_number: int):
    if stage_number in [5, 6]:
        # 根據當前環境變數決定清理哪種 pool
        use_candidate = os.getenv('ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL', 'false') == 'true'
        pool_suffix = "_candidate_pool" if use_candidate else "_elite_pool"

        # 只刪除匹配的檔案
        for file in output_dir.iterdir():
            if pool_suffix in file.name:
                file.unlink()  # 安全：不會誤刪其他 pool 的檔案
```

---

### Pool 分離詳情

| Pool 類型 | 衛星數量 | 來源階段 | 用途 | 狀態維度 |
|-----------|----------|----------|------|----------|
| **Elite Pool** | 123 顆 | Stage 4.2 pool optimization | RL 訓練 | 77 維（含時間特徵） |
| **Candidate Pool** | 3,262 顆 | Stage 4.1 visibility filtering | 前端渲染/研究 | 77 維（含時間特徵） |

**差異說明**:
- **Elite Pool**: 經過 Set Cover 貪心優化，確保 95% 時間覆蓋率，避免 diversity explosion
- **Candidate Pool**: 完整可見性篩選結果，包含所有曾經可見的衛星

---

### 時間特徵整合（2025-10-24 新增）

**影響範圍**: 兩種 pool 均受影響

| 特徵 | Elite Pool | Candidate Pool |
|------|------------|----------------|
| 狀態維度 | 77 維 | 77 維 |
| Velocity | ✅ dRSRP/dt, dDistance/dt | ✅ dRSRP/dt, dDistance/dt |
| Prediction | ✅ 30s/60s RSRP | ✅ 30s/60s RSRP |
| 用途 | RL 訓練特徵 | 前端渲染額外資訊 |

**HDF5 數據集**:
- Elite pool: `rl_training_dataset_temporal.h5` (2,590 transitions, 77-dim)
- Candidate pool: 未生成 HDF5（前端使用 JSON）

---

### D2 Integration 影響（2025-10-24 完成）

**影響範圍**: Stage 6 輸出

| Pool 類型 | A4 events | D2 events | 總換手機會 |
|-----------|-----------|-----------|-----------|
| Elite Pool (123) | 21,224 | 6,074 | 27,298 |
| Candidate Pool (3,262) | 未生成 | 未生成 | - |

**說明**: D2 Integration 目前僅用於 RL 訓練（Elite pool），前端渲染暫不使用。

---

## 總結

**這個架構確保**:
1. ✅ 前端渲染和 RL 訓練**完全獨立**（透過環境變數 + 檔名後綴）
2. ✅ 修改 RL 配置**不影響**前端渲染（選擇性清理機制）
3. ✅ 兩種輸出可以**並存**（Elite vs Candidate pool）
4. ✅ 配置清晰，易於維護（單一配置源，零重複）
5. ✅ 符合學術標準（77-dim temporal features, D2 integration）
6. ✅ **實際採用更簡潔的環境變數方式**（vs 原提案的配置文件方式）

**實施狀態**: ✅ **已完成**（2025-10-24）

**文檔版本**: 1.1
**創建日期**: 2025-10-20
**最後更新**: 2025-10-24
