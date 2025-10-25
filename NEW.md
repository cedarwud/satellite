 ---
  🚀 新環境部署步驟

  在新環境執行以下命令：

  # 1. 克隆專案
  git clone <your-orbit-engine-repo>
  cd orbit-engine

  # 2. 設置環境
  cp .env.example .env
  # 編輯 .env 設置 TLE 數據路徑（如需要）

  # 3. 創建虛擬環境
  python3 -m venv venv
  source venv/bin/activate  # Linux/Mac
  # 或 venv\Scripts\activate  # Windows

  # 4. 安裝依賴
  pip install -r requirements.txt

  # 5. 驗證安裝
  python -c "import skyfield, astropy, itur; print('✅ Core modules OK')"

  # 6. 執行測試
  ./run.sh --stage 1  # 測試單階段

  # 7. 完整執行
  ./run.sh  # 六階段處理

  Docker 執行：

  # 方式 1: 使用腳本
  ./run-docker.sh

  # 方式 2: 使用 Make
  make docker

  ---
  ✅ 驗證清單

  在新環境執行後，確認：

  - git clone 成功
  - venv 創建成功
  - pip install -r requirements.txt 無錯誤
  - 核心模組導入成功
  - ./run.sh --stage 1 執行成功
  - data/outputs/stage1/ 產生 JSON 檔案
  - Docker 容器可正常運行（如使用 Docker）

  ---
  📊 requirements.txt 對比

  清理前（舊版）

  - 包含 RL 訓練套件（torch, gymnasium, stable-baselines3）
  - 包含 GPU 套件（cupy-cuda11x, cupy-cuda12x）
  - 總計 ~40+ 個套件

  清理後（當前）

  - ✅ 僅包含數據生成必要套件
  - ✅ 移除所有 RL 相關套件
  - ✅ 移除 GPU 加速套件
  - ✅ 保留核心計算、軌道計算、測試套件
  - 總計 ~33 個核心套件

  ---
  🎯 職責劃分（清理後）

  orbit-engine ✅

  ✅ Stage 1-6: TLE → 軌道計算 → 信號分析 → 換手事件
  ✅ HDF5 訓練數據生成（77-dim temporal features）
  ✅ 工具: ml_training_data_generator
  ❌ 不包含 RL 訓練代碼

  handover-rl ✅

  ✅ DQN/A3C/PPO/SAC 訓練
  ✅ 使用 orbit-engine 生成的 HDF5 數據
  ✅ 包含 torch, gymnasium, stable-baselines3

  ---
  📝 重要提醒

  1. TLE 數據目錄:
    - 確保 tle_data/ 與 orbit-engine/ 在同一父目錄
    - .env 中設置: SATELLITE_TLE_DATA_DIR=../tle_data
  2. Python 版本:
    - 需要 Python 3.12+
  3. 系統需求:
    - 4GB+ RAM
    - 2GB+ 磁碟空間
  4. 執行產生的數據:
    - data/outputs/ - 六階段 JSON 輸出（可重新生成）
    - data/ml_training/*.h5 - HDF5 訓練數據（可重新生成）
    - 這些檔案已被 .gitignore，不會提交到 Git

  ---
  環境遷移準備狀態: ✅ 完全就緒

  您現在可以在新環境使用 git clone 下載專案，按照 README.md 或 ENVIRONMENT_MIGRATION_CHECKLIST.md
  的指引安裝，即可正常執行生成前端渲染數據和強化學習數據！