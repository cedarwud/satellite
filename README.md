# Satellite Handover Optimization - 項目總覽

**LEO 衛星換手決策優化研究項目**

本項目使用深度強化學習 (DQN) 優化 LEO 衛星換手決策，基於真實 TLE 數據和完整物理模型。

---

## 📁 項目結構

```
satellite/
├── README.md              # 本文件
├── docs/                  # 項目級別文檔
│   ├── ALGORITHM_SHARING_ANALYSIS.md
│   ├── DUAL_MODE_ARCHITECTURE.md
│   └── RL_VS_ORBIT_ENGINE_COMPARISON.md
├── handover-rl/          # RL 訓練子項目 ⭐
│   ├── train.py         # DQN 主訓練入口
│   ├── TODO.md          # 當前狀態與待辦事項
│   ├── checkpoints/     # 訓練模型
│   │   └── bc_policy_v4_best_20251021_020013.pth (88.81%)
│   ├── src/             # 源代碼
│   ├── scripts/         # 訓練腳本
│   └── ...
├── orbit-engine/         # 軌道計算與數據生成 ⭐
│   ├── data/            # 輸出數據
│   │   └── outputs/rl_training/
│   ├── src/             # 源代碼
│   ├── docs/            # orbit-engine 文檔
│   └── ...
├── tle_data/            # TLE 數據源
│   ├── starlink/
│   ├── oneweb/
│   └── migrate_tle_data.sh
└── rl-paper/            # 論文相關
```

---

## 🎯 子項目說明

### 1. handover-rl/ - RL 訓練

**用途**: 深度強化學習模型訓練與評估

**核心功能**:
- DQN 訓練 (`train.py`)
- BC (Behavior Cloning) 預訓練
- 模型評估與測試
- 訓練結果追蹤

**當前狀態**:
- ✅ BC V4 訓練完成 (88.81% 準確率)
- ✅ Temporal features 整合完成 (77-dim state)
- ✅ D2 Integration Phase 1 完成 (A4+D2 加權組合)
- ✅ Elite/Candidate pool 分離機制完成
- 📍 準備開始 DQN 訓練
- 詳見: `handover-rl/TODO.md`

**關鍵文件**:
```
train.py                              # DQN 訓練入口
train_offline_bc_v4_candidate_pool.py # BC 訓練
TODO.md                               # 項目狀態
checkpoints/bc_policy_v4_best_*.pth   # BC 模型
```

### 2. orbit-engine/ - 軌道計算引擎

**用途**: 衛星軌道計算、信號分析、換手事件生成

**核心功能**:
- 6 階段數據處理流程
- TLE 數據載入與軌道傳播 (SGP4/Skyfield)
- 座標系統轉換 (TEME → WGS84 → ECEF)
- 可見性分析與鏈路預算
- 信號品質分析 (RSRP, SINR, 路徑損耗)
- 3GPP 換手事件生成 (A3, A4, A5, D2)

**輸出數據**:
```
data/outputs/rl_training/
├── stage5/stage5_signal_analysis_*.json    # 信號分析
└── stage6/stage6_research_optimization_*.json # 換手事件
```

**文檔位置**: `orbit-engine/docs/`

### 3. tle_data/ - TLE 數據

**用途**: 存儲來自 Space-Track.org 的 TLE 數據

**結構**:
```
tle_data/
├── starlink/  # Starlink 衛星 TLE
├── oneweb/    # OneWeb 衛星 TLE
└── migrate_tle_data.sh
```

**數據來源**: Space-Track.org (官方數據源)

### 4. docs/ - 項目級別文檔

**包含**:
- 算法共享分析
- 雙模式架構設計
- RL vs Orbit-Engine 比較

### 5. rl-paper/ - 論文相關

**用途**: 論文撰寫與相關資料

---

## 🚀 快速開始

### 1. 查看當前狀態

```bash
cd handover-rl
cat TODO.md
```

### 2. 開始 DQN 訓練

```bash
cd handover-rl
python train.py --algorithm dqn --level 1 --output-dir output/dqn_level1
```

### 3. 生成 RL 訓練數據

```bash
cd orbit-engine
./scripts/generate_rl_training_data.sh
```

---

## 📊 當前進度

### ✅ 已完成

1. **數據洩漏問題解決** (2025-10-21)
   - 診斷並修復 100% 準確率問題
   - 實現數據驅動閾值設計
   - BC V4 訓練成功 (88.81%)

2. **RL 訓練數據集**
   - Elite Pool: 123 satellites (避免 diversity explosion)
   - Candidate Pool: 3,262 satellites (完整可見性篩選)
   - Stage 5: 信號分析 (Elite pool: 123 satellites)
   - Stage 6: 換手事件 (Elite pool: 21,224 A4, 6,074 D2)
   - HDF5 數據集: 77-dim state, 2,590 transitions

3. **Temporal Features 系統** (2025-10-24)
   - Velocity: dRSRP/dt, dDistance/dt
   - Prediction: 30s/60s RSRP ahead
   - 狀態空間擴展: 53 → 77 維

4. **D2 Integration Phase 1** (2025-10-24)
   - A4+D2 加權組合策略
   - D2 補充 34.4% 換手機會
   - 顯著提升換手決策多樣性

5. **項目結構整理**
   - 根目錄清理完成
   - 文檔分類歸檔
   - 文檔深度清理: 86 → 34 active docs (-60.5%)

### 📍 當前任務

- **DQN 訓練準備** (handover-rl/)
  - 檢查 BC-DQN 整合機制
  - 開始 DQN Level 1 訓練

詳見: `handover-rl/TODO.md`

---

## 📚 核心文檔索引

| 文檔 | 位置 | 說明 |
|------|------|------|
| 項目狀態與待辦 | `handover-rl/TODO.md` | ⭐ 當前狀態 |
| BC 訓練報告 | `handover-rl/TRAINING_REPORT_V4_FINAL.md` | V4 成果 |
| 數據洩漏診斷 | `handover-rl/DIAGNOSIS_100_ACCURACY.md` | 問題分析 |
| 解決方案總結 | `handover-rl/FINAL_SOLUTION_SUMMARY.md` | 完整方案 |
| 閾值建議 | `handover-rl/FINAL_THRESHOLD_RECOMMENDATIONS.md` | 數據驅動設計 |
| 架構比較 | `docs/RL_VS_ORBIT_ENGINE_COMPARISON.md` | 架構分析 |
| 雙模式設計 | `docs/DUAL_MODE_ARCHITECTURE.md` | 系統設計 |

---

## 🔗 子項目 README

- **handover-rl**: `handover-rl/README.md`
- **orbit-engine**: `orbit-engine/README.md`

---

## 🎓 學術合規

- ✅ 真實 TLE 數據 (Space-Track.org)
- ✅ 完整物理模型 (ITU-R, 3GPP)
- ✅ 無簡化算法
- ✅ 無模擬數據
- ✅ 可重現性 (seed-controlled)

---

## 📞 快速參考

| 想要... | 命令/文件 |
|---------|----------|
| 了解當前狀態 | `cat handover-rl/TODO.md` |
| 開始 DQN 訓練 | `cd handover-rl && python train.py --help` |
| 查看訓練成果 | `cat handover-rl/TRAINING_REPORT_V4_FINAL.md` |
| 生成訓練數據 | `cd orbit-engine && ./scripts/generate_rl_training_data.sh` |
| 查看項目架構 | `cat docs/DUAL_MODE_ARCHITECTURE.md` |

---

**項目狀態**: Orbit-engine 100% 完成（含時間特徵、D2 Integration），準備 DQN 訓練
**最後更新**: 2025-10-24
**維護者**: Claude Code

---

## 🆕 最近重大更新 (2025-10-24)

### Temporal Features 系統
- ✅ 新增 4 個時間特徵（velocity, prediction）
- ✅ 狀態空間擴展至 77 維
- ✅ HDF5 數據集生成完成 (2,590 transitions)
- 位置: `orbit-engine/tools/ml_training_data_generator/`

### D2 Integration Phase 1
- ✅ A4 (RSRP) + D2 (Distance) 加權組合
- ✅ 實測效果: 34.4% D2 使用率
- ✅ 換手機會增加 7,304 個
- 詳見: `orbit-engine/docs/development_plans/d2_integration/`

### Pool 分離機制
- ✅ Elite Pool (123 satellites) for RL training
- ✅ Candidate Pool (3,262 satellites) for frontend rendering
- ✅ 環境變數控制: `ORBIT_ENGINE_STAGE5_USE_CANDIDATE_POOL`
- ✅ 選擇性清理機制避免數據互相覆蓋

### 文檔深度清理
- ✅ 三輪優化: 86 → 34 active docs (-60.5%)
- ✅ Proposals 全部歸檔 (3 個提案已完成)
- ✅ 架構分析歸檔 (150KB, 8 files)
- ✅ 文檔準確性 100% 驗證
- 詳見: `orbit-engine/DOCUMENTATION_CLEANUP_FINAL_REPORT.md`
