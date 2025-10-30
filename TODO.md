請 ultrathink 仔細閱讀 @TODO.md 之後開始進行開發，請一併閱讀 @NEW.md 跟 @README.md ，應該會需要先執行 @orbit-engine/ 來生成 @handover-rl 所需要的數據

  ---
  🎯 開始 D2 Integration Phase 1b - DQN 訓練

  專案背景

  我正在開發 LEO 衛星換手優化系統，包含兩個專案：
  - orbit-engine: 數據生成管道（✅ 已完成）
  - handover-rl: 強化學習訓練（🔄 代碼完成，待訓練）

  當前狀態

  ✅ Orbit-Engine（已完成）

  - 6 階段數據處理管道：完成
  - 時間特徵（velocity, predicted RSRP）：✅ 已整合（77 維狀態）
  - D2 Integration Phase 1（加權組合）：✅ 已整合（D2 使用率 34.4%）
  - 訓練數據集已生成：
    - 位置：/home/sat/satellite/orbit-engine/data/ml_training/rl_training_dataset_temporal.h5
    - State 維度：77
    - 數據量：2,590 transitions (Train: 1,812 | Val: 388 | Test: 390)
    - 生成時間：2025-10-24 09:52

  ✅ Handover-RL（代碼完成）

  - DQN Agent：✅ 已實現（src/agents/dqn/dqn_agent.py）
  - 訓練腳本：✅ 已實現（train.py）
  - 配置檔：✅ 已更新（state_dim: 77, action_dim: 6）
  - 待執行：DQN 訓練

  訓練目標（D2 Integration Phase 1b）

  根據 @orbit-engine/docs/development_plans/d2_integration/README.md：

  Phase 1b 目標：
  - 使用 weighted combination 數據集訓練 DQN
  - 驗證 D2 使用率提升效果（目標：34.4% 已達成）
  - 對比 DQN vs RSRP Baseline
  - 總獎勵改進目標：> +20% vs baseline

  請執行以下任務

  1. 準備數據集

  # 複製最新數據集到 handover-rl
  cp /home/sat/satellite/orbit-engine/data/ml_training/rl_training_dataset_temporal.h5 \
     /home/sat/satellite/handover-rl/data/

  2. 檢查訓練環境

  - 確認 handover-rl 項目狀態
  - 檢查 train.py 配置是否正確（state_dim: 77）
  - 確認 Python 環境和依賴

  3. 執行 DQN 訓練

  使用 Offline RL 方式（從 HDF5 數據集訓練）：
  cd /home/sat/satellite/handover-rl
  python train.py \
    --algorithm dqn \
    --dataset data/rl_training_dataset_temporal.h5 \
    --config config/training_config.yaml

  4. 監控訓練進度

  - 檢查 TensorBoard logs
  - 監控訓練 loss 和 reward
  - 確認模型 checkpoint 儲存

  5. 評估訓練結果

  訓練完成後：
  - 對比 DQN vs RSRP Baseline
  - 計算總獎勵改進百分比
  - 驗證是否達到 Phase 1b 目標（> +20% 獎勵改進）

  相關文檔參考

  必讀：
  - @orbit-engine/docs/development_plans/d2_integration/README.md - Phase 1b 詳細計劃
  - @orbit-engine/docs/development_plans/d2_integration/PHASE1_COMPLETION_SUMMARY.md - Phase 1 完成報告

  配置檔：
  - @handover-rl/config/training_config.yaml - 訓練超參數
  - @handover-rl/train.py - 訓練主程式

  專案狀態：
  - @orbit-engine/PROJECT_STATUS_COMPREHENSIVE_ANALYSIS.md - 完整狀態分析
  - @orbit-engine/DOCUMENTATION_CLEANUP_FINAL_REPORT.md - 文檔整理報告

  預期結果

  訓練時間：30-60 分鐘
  成功標準：
  - ✅ 訓練收斂（loss 下降）
  - ✅ 獎勵持續提升
  - ✅ 總獎勵改進 > +20% vs RSRP Baseline
  - ✅ D2 使用率體現在換手決策中

  問題排查

  如遇到問題，檢查：
  1. 數據集路徑是否正確
  2. State 維度是否為 77
  3. Python 環境是否正確（需要 PyTorch, h5py）
  4. 配置檔參數是否合理

  ---
  開始訓練，並回報訓練進度和結果！ 🚀