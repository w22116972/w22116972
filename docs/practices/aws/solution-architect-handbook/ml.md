---

---

# Machine Learning Architecture

> Source: `aws/references/solution-architect-handbook/ml.pdf`

本章說明 ML 基礎、learning types、ML workflow、algorithms、tools、cloud ML、ML architecture、MLOps、deep learning、NLP 與 LLM。核心觀念：ML architecture 不只是訓練 model，而是 data、training、deployment、monitoring、retraining 與 governance 的完整 lifecycle。

## Machine Learning

ML 是 AI 的子集，透過 training data 與 algorithm 建立 predictive model，讓系統從資料中學習 pattern 並做 prediction/classification/recommendation。

ML 常用於：

- Recommendation
- Fraud detection
- Customer service intelligence
- Demand forecasting
- Image recognition
- Natural language processing
- Predictive maintenance

ML 適合有歷史資料、pattern 可從資料中學習、規則難以手寫或需要持續調整的問題。不適合資料量太少、label 不可靠、decision 需要完全 deterministic、或 business process 尚未穩定的場景。架構師要先確認 ML 是否真的比 rule-based system 更合適。

ML 系統通常分為 offline training 與 online/batch inference。Training pipeline 關注 data preparation、feature engineering、training cost、experiment tracking；inference pipeline 關注 latency、throughput、availability、model version、monitoring、rollback。

## Types of Machine Learning

- Supervised learning：有 label，常用於 classification/regression
- Unsupervised learning：無 label，找 hidden pattern，例如 clustering
- Semi-supervised learning：少量 label + 大量 unlabeled data
- Reinforcement learning：agent 透過 reward 學習 sequential decision
- Self-supervised learning：從資料本身產生 label，常見於大型模型 pretraining
- Multi-instance learning：以 bag/collection 為單位判斷 label

選擇 learning type 要看 business problem、data availability、label quality 與 desired output。

Supervised learning 需要標註資料，例如 spam/not spam、fraud/not fraud、房價。Unsupervised learning 用於 segmentation、anomaly detection、topic discovery。Semi-supervised learning 在 label 成本高時有用。Reinforcement learning 適合 sequential decision，例如 game、robotics、dynamic pricing，但 training 與 safety 更複雜。Self-supervised learning 是 modern foundation models 的重要基礎，可從大量未標註資料學 representation。

## Data Science and ML Workflow

典型 ML workflow：

1. Define business problem
2. Collect data
3. Prepare and clean data
4. Feature engineering
5. Split train/test/validation
6. Select algorithm
7. Train model
8. Tune hyperparameters
9. Evaluate model
10. Deploy inference endpoint/batch job
11. Monitor drift/performance
12. Retrain

ML 的核心風險是資料會變：training data、inference data、business environment 都可能 drift。

Workflow 的第一步應是定義 business metric，不是直接選 algorithm。例如 fraud detection 要明確定義 false positive/false negative 成本；recommendation 要看 conversion、click-through、retention；forecasting 要看 error 與 inventory/business impact。資料準備通常是最大工作量，包含 missing value、outlier、duplicate、label leakage、class imbalance、train/test split、feature normalization。

Train/validation/test split 要避免 time leakage。對時間序列或 user behavior，不能隨機把未來資料放進 training。Feature engineering 要可重現，training 與 inference 使用同一份 transformation logic，否則會 training-serving skew。

## Overfitting and Underfitting

- Overfitting：model 記住 training data noise，training 表現好但 generalization 差
- Underfitting：model 太簡單，無法捕捉 pattern，training/test 表現都差

解法包含更多資料、feature selection、regularization、cross-validation、model complexity control、hyperparameter tuning。

Good fit 需要 bias-variance balance。Overfitting 常見跡象是 training metric 很好、validation/test metric 差；underfitting 則是 training/validation 都差。Regularization、dropout、early stopping、data augmentation、cross-validation 可降低 overfitting；增加 feature、提高 model complexity 或改善資料品質可降低 underfitting。

## Popular Algorithms

- Linear regression：連續值預測
- Logistic regression：binary/multiclass classification
- Decision tree：可解釋的 tree-based decision
- Random forest：多棵 decision tree ensemble
- K-Nearest Neighbors (k-NN)：根據相似樣本分類/預測
- Support Vector Machine (SVM)：找 decision boundary
- Neural network：多層 nonlinear representation
- K-means clustering：unsupervised grouping
- XGBoost：高效 gradient boosting，常用於 tabular data

Algorithm selection 要根據 data type、accuracy、explainability、training cost、inference latency。

PDF 對常見 algorithm 的直覺例子：

- Linear regression：用 continuous relationship 預測數值，例如尺寸與價格。
- Logistic regression：輸出 0 到 1 的機率，適合 binary classification。
- Decision tree：像一系列 if/else 問題，容易解釋。
- Random forest：多棵 tree 投票，通常比單棵 tree 穩定。
- k-NN：依距離找最相似樣本，簡單但 inference 可能昂貴。
- SVM：找能分隔 class 的 boundary，對某些高維資料有效。
- Neural network：透過多層 representation 學複雜 pattern。
- K-means：把相似資料分群，例如 customer segmentation。
- XGBoost：tabular data 常用強 baseline，但要調參並注意 overfitting。

Explainability 是架構選型條件。金融、醫療、保險等場景可能需要可解釋模型或 model explanation，而不只是最高 accuracy。

## Tools and Frameworks

Data preparation/exploration：

- Pandas
- NumPy
- Spark
- Jupyter

Visualization：

- Matplotlib
- Seaborn
- Tableau
- QuickSight

Model development/training：

- Scikit-learn
- TensorFlow
- PyTorch
- Keras
- XGBoost

Deployment：

- SageMaker
- Kubernetes
- MLflow
- Docker
- Cloud managed endpoints

工具要對應 lifecycle。Jupyter 適合 exploratory analysis，但不能當 production pipeline。Spark/Glue/EMR 適合大規模 data processing。TensorFlow/PyTorch 適合 deep learning。Scikit-learn/XGBoost 適合 tabular/classical ML。MLflow/SageMaker Experiments 可追蹤 experiment。Model registry 管理 approved model、version、metadata、approval status。

Visualization 工具不只是畫圖，而是協助資料理解與模型診斷，例如 distribution、correlation、feature importance、confusion matrix、ROC/PR curve、residual plot。

## Machine Learning in the Cloud

Cloud ML platforms reduce undifferentiated heavy lifting：

- Data preparation
- Experiment tracking
- Distributed training
- AutoML
- Model registry
- Endpoint deployment
- Batch inference
- Pipeline automation
- Monitoring

Examples：Amazon SageMaker、Google Vertex AI、Azure Machine Learning。

Cloud ML 的價值是提供 managed notebook、distributed training、managed algorithms、hyperparameter tuning、feature store、model registry、deployment endpoint、batch transform、pipeline orchestration、monitoring。它可以讓 data scientist 不必自己維護 GPU cluster、training scheduler、serving infrastructure。

但 managed platform 不會自動解決資料品質與模型治理。架構師仍要設計 data access、encryption、network isolation、IAM、artifact storage、approval workflow、cost guardrail 與 audit trail。

## ML Architecture

以 AWS ML platform 為例，architecture components 包含：

- Prepare and label：data processing、feature engineering、labeling、Feature Store
- Select and build：notebook、algorithm selection、model code
- Train and tune：training jobs、hyperparameter tuning、debugging
- Deploy and manage：real-time endpoint、batch inference、monitoring、model drift detection

Production ML 需要保護 model artifacts，支援 self-service model development，並將 CI/CD 擴展為 ML pipeline。

AWS reference architecture 可用 S3 保存 raw/processed data 與 model artifacts，SageMaker Ground Truth 做 labeling，SageMaker Processing 做 data prep，SageMaker Training/HPO 做 training/tuning，SageMaker Model Registry 管版本，Endpoint 或 Batch Transform 做 inference，CloudWatch/SageMaker Model Monitor 追蹤 latency、errors、data drift。

Prepare and label 階段要定義 label guideline 與 quality audit。Select and build 階段要支援 notebook/IDE、framework、library 與 reproducible environment。Train and tune 階段要記錄 hyperparameters、dataset version、code version、metric。Deploy and manage 階段要支援 canary、shadow deployment、rollback 與 model monitoring。

## Design Principles

- Modularity：data ingestion、feature engineering、training、inference、monitoring 分離
- Scalability：處理 growing data/training/inference workload
- Reproducibility：資料版本、code version、model version、environment version 可追蹤
- Data quality assurance：validate schema、missing values、outliers、label quality
- Security：保護 training data、model artifact、endpoint
- Monitoring：accuracy、latency、drift、bias、cost
- Automation：pipeline、retraining、deployment

ML architecture design principles 還包括：

- Flexibility：可替換 algorithm、feature、serving pattern。
- Robustness/reliability：dependency failure、bad input、model endpoint failure 時能 fallback。
- Privacy/security：training data 與 inference input 可能含 sensitive data，要做 access control、encryption、masking。
- Efficiency：GPU/CPU 資源昂貴，要控制 training duration、endpoint autoscaling、batch size。
- Interpretability：高風險 decision 要能解釋模型輸出。
- Real-time capability：若 use case 需要即時反應，feature retrieval、model latency、API SLA 都要符合。
- Fault tolerance：model endpoint 不可用時要有 degraded path，例如 rule-based fallback、cached result 或 manual review。

## MLOps

MLOps 將 DevOps 思維套用到 ML：

- Version control for code/data/model
- Experiment tracking
- Model registry
- Automated training pipeline
- Approval workflow
- Deployment pipeline
- Monitoring and rollback
- Retraining trigger

ML system 的 production risk 包含 data drift、concept drift、model decay、bias、explainability 與 compliance。

MLOps 的核心是把「模型」當 production artifact 管理。與傳統 DevOps 相比，ML 多了 dataset version、feature pipeline、experiment metadata、model metric、approval gate、drift/retraining trigger。Production deploy 不應只看 unit test，也要看 model evaluation 是否達到 business threshold。

MLOps best practices：

- Version everything：code、data、feature、model、environment。
- Automate pipeline：data validation -> training -> evaluation -> registration -> deployment。
- Separate training and serving environments but keep feature logic consistent。
- Monitor data drift、prediction distribution、accuracy proxy、latency、cost。
- Use model registry and approval workflow for production promotion。
- Keep rollback path to previous model。
- Define human review for high-risk prediction。

## Deep Learning, NLP, LLM

Deep learning 使用 neural networks 學習複雜 representation，廣泛應用於 image recognition、speech、NLP。

NLP 處理 human language，例如 translation、summarization、sentiment analysis、question answering。

LLM 是大規模 transformer-based model，可進行 conversation、generation、reasoning-like tasks，是 GenAI architecture 的基礎。

Deep learning 適合大量非結構化資料，例如 image、audio、text。NLP tasks 包含 translation、summarization、sentiment analysis、named entity recognition、question answering。LLM 帶來更通用的 text/code generation 能力，但也需要 prompt engineering、RAG、guardrails、evaluation 與安全控制。GenAI 章節會進一步處理 FM/RAG/hallucination。

## Summary

ML architecture 的成功取決於資料品質、workflow automation、model lifecycle management 與 production monitoring。架構師需要把 ML 當成持續運作的系統，而不是一次性 notebook 實驗；MLOps 是讓模型能安全、可重現、可擴展地進入 production 的關鍵。
