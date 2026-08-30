---

---

# Generative AI Architecture

> Source: `aws/references/solution-architect-handbook/genai.pdf`

本章整理 Generative AI、Foundation Models (FMs)、generative models、FM selection、hallucination mitigation、RAG reference architecture 與 implementation challenges。重點是 GenAI application 需要同時設計 model、data、prompt、guardrails、security、evaluation 與 operation。

## Generative AI

Generative AI 可產生 text、image、audio、video、code、conversation 等內容。現代 GenAI 多建立在大型 Foundation Models 上，例如 GPT、Gemini、Claude、Llama、Stable Diffusion。

FMs 的特點：

- 大規模資料訓練
- General-purpose capability
- 可透過 prompt、fine-tuning、RAG 適配特定任務
- 可透過 API 或 managed cloud platform 使用

Foundation Models 成功的原因包含 large-scale training data、massive compute、transformer architecture 與 transfer learning。它們可以在未針對單一任務重新訓練的情況下，用 prompt 完成 summarization、classification、question answering、code generation 等任務。Enterprise 應把 FM 視為可組合能力，而不是完整 application。

使用 FM 的方式通常有三種：prompting 直接使用 base/instruction model；RAG 以企業知識補充 context；fine-tuning/customization 讓模型更符合特定格式、語氣或 domain。選擇取決於資料敏感性、品質要求、成本與維護能力。

## Use Cases

### Customer Experience

- Chatbot / virtual assistant
- Personalized recommendation
- Document explanation
- Customer support automation
- Marketing content generation

### Employee Productivity

- Coding assistant
- Knowledge search
- Document summarization
- Meeting/email drafting
- Internal IT/HR support

### Business Operations

- Process automation
- Contract/document review
- Report generation
- Data insight explanation
- Content localization

Enterprise use case 需注意 legal、copyright、privacy、compliance 與 generated content review。

Customer experience use case 強調 response quality、safety、latency、handoff to human。Employee productivity use case 重點是 knowledge access、workflow integration、permission-aware retrieval。Business operations use case 常涉及文件處理、流程自動化與 decision support，因此需要 auditability 與 human approval。

GenAI 導入時應先選低風險、高回饋、資料權限清楚的 use case。對外 customer-facing assistant 必須有更嚴格 guardrails、monitoring、fallback 與 legal review。

## Basic Architecture

GenAI system 通常包含：

- User interface
- Prompt orchestration
- Foundation Model API
- Knowledge base / vector store
- Retrieval component
- Guardrails / policy filters
- Logging and monitoring
- Feedback and evaluation
- Security controls

若需要 domain-specific knowledge，常使用 RAG (Retrieval-Augmented Generation)，讓 model 根據企業文件與檢索結果回答。

Basic GenAI architecture 的 request flow 通常是：user query -> authentication/authorization -> prompt orchestration -> query rewriting/classification -> retrieval from knowledge base/vector store/search -> context ranking/filtering -> FM generation -> safety/PII/policy check -> response -> logging/feedback/evaluation。

Security design 必須確保 retrieval 是 permission-aware。使用者不能透過 assistant 看到自己原本無權存取的文件。Prompt、retrieved context、model output、feedback logs 也可能含 sensitive data，需要 encryption、retention policy、redaction 與 access control。

## Types of Generative Models

### GANs

GAN (Generative Adversarial Network) 包含 generator 與 discriminator。Generator 產生假資料，discriminator 判斷真假，兩者互相競爭以提升生成品質。常見於 image generation，但 training stability 具挑戰。

GAN 的優點是可產生高品質 image/synthetic data；缺點是 training 容易不穩、mode collapse、評估困難。Generator 與 discriminator 的平衡很重要，任一方過強都可能導致學習失敗。

### VAEs

VAE (Variational Autoencoder) 將高維資料壓縮到 latent space，再從 latent representation 生成新資料。適合 representation learning、variation generation。

VAE 的 latent space 讓相似資料在空間中接近，因此可用於資料變形、補全、生成 variation。相較 GAN，VAE training 通常較穩定，但生成結果可能較模糊。

### Transformer-Based Models

Transformer 使用 attention mechanism 處理 sequence data，是 LLM 的核心架構。Encoder/decoder 或 decoder-only 架構可用於 translation、summarization、conversation、code generation。

Attention mechanism 讓模型在產生下一個 token 時參考 context 中相關 token。Decoder-only transformer 常用於 autoregressive text generation。Context window 限制會影響可放入 prompt 的資料量，因此長文件問答常需要 retrieval、chunking、ranking 與 summarization。

### Other Models

其他 generative models 包含 diffusion models、autoregressive models、flow-based models 等，各自適合不同資料型態與生成任務。

Diffusion models 常用於 image generation，透過逐步去噪產生內容。Autoregressive models 逐 token/pixel 生成，適合 sequence generation。Flow-based models 可做可逆轉換與 density estimation。架構師通常不需要自行訓練這些模型，但要理解其 latency、cost、quality、control capability 差異。

## Hyperparameter Tuning and Regularization

Hyperparameters 是控制 model learning 的「旋鈕」，例如 learning rate、batch size、temperature、top-p、max tokens 等。

Regularization 用於降低 overfitting、提升 generalization。GenAI 中也需要限制不合理輸出與增加一致性。

在 inference 階段，temperature、top-p、top-k、max tokens、stop sequence 會影響輸出創造性與穩定性。Temperature 越高通常越有創意但越不穩；低 temperature 適合 factual/structured tasks。Hyperparameter 要針對 use case 測試，不應用同一組設定套所有任務。

Regularization 在 training/fine-tuning 階段可降低模型記憶 training data noise。Enterprise customization 還要避免 overfitting 到少量 domain examples，導致泛化能力下降。

## Popular FMs and Cloud Platforms

主要 provider 包含：

- OpenAI
- Google
- Amazon
- Anthropic
- Meta
- Nvidia
- Cohere

Cloud 平台透過 API/managed service 提供 FM，例如 Amazon Bedrock、SageMaker JumpStart、Vertex AI、Azure OpenAI 等。選擇平台時需看 model capability、latency、cost、region、data privacy、fine-tuning、RAG integration、compliance。

End users 可直接使用 ChatGPT、Claude、Gemini、Copilot 類工具提升個人生產力。Builders 則通常透過 API、SDK、managed platform 或 self-hosted open model 建 application。Public cloud provider 提供 model catalog、managed endpoint、security integration、logging、private networking、knowledge base、fine-tuning 或 evaluation tooling。

Amazon Bedrock 的價值是透過單一 managed service 使用多家 FMs，搭配 IAM、VPC/private access、guardrails、knowledge bases、agents 等 AWS integration。SageMaker JumpStart 則更偏向模型訓練、部署、fine-tuning、自管 lifecycle。

## Choosing the Right FM

選 FM 時應評估：

- Task type：chat、summarization、classification、code、image
- Required accuracy
- Context window
- Latency
- Cost per token/request
- Supported language
- Tool/function calling
- Fine-tuning support
- RAG compatibility
- Safety and compliance
- Deployment model：API、private endpoint、self-hosted

不要只用 benchmark 排名決策；應以 own dataset 與 target user journey 做 evaluation。

FM selection 應建立 evaluation set，包含代表性 user questions、edge cases、禁止回答內容、格式要求、多語言資料、權限場景。比較時要看 answer correctness、groundedness、latency、cost、token usage、refusal behavior、safety、tool-use reliability、JSON/structured output adherence。

小模型若能完成任務，通常比大型模型更便宜、更快。大型模型適合 complex reasoning、多步驟 tool use、長 context、跨語言或高品質生成。架構可以採 routing：簡單問題用小模型，高風險/複雜問題升級到大模型。

## Preventing Hallucinations

Hallucination 是 model 產生看似合理但不正確的內容。

降低方法：

- 使用 RAG 提供 grounding context
- 限制回答只能基於 retrieved sources
- Prompt 中要求引用來源或說明未知
- 使用 guardrails 與 policy checks
- Human-in-the-loop review
- Evaluation dataset
- Monitoring user feedback
- 對高風險任務加入 deterministic validation

Enterprise 通常先部署 internal use case，降低外部風險並累積 feedback。

Hallucination mitigation 要分層：

- Grounding：RAG、tool/API lookup、structured database query。
- Prompt constraint：要求 only answer from context，不知道就說不知道。
- Retrieval quality：chunk size、metadata filter、reranker、freshness、deduplication。
- Verification：用 deterministic rule、schema validation、secondary model/judge 或 source citation check。
- Human review：法律、醫療、金融、HR 等高風險內容需人工審核。
- Monitoring：收集 user feedback、hallucination report、low-confidence query。

RAG 不能自動消除 hallucination。若 retrieval 找錯文件、context 太長、權限過濾錯誤、prompt 沒限制，模型仍會生成錯誤答案。

## Reference Architecture: Mortgage Assistant

Mortgage assistant 可使用：

- Amazon Lex 作為 virtual assistant interface
- Amazon Kendra 或 search service 檢索 mortgage documents
- Foundation Model 生成簡化說明
- RAG 將 domain-specific knowledge 注入回答
- Access control 確保使用者只能看授權文件
- Monitoring 與 feedback loop 改善回答品質

此架構能將複雜 mortgage 文件轉成易懂內容，改善 homebuying experience。

Mortgage assistant 的資料流程可設計為：loan policy、FAQ、mortgage document 進入 document repository；Indexer 解析文件、切 chunk、抽 metadata、建立 search/vector index；使用者透過 Lex/chat UI 提問；系統根據使用者身份與授權範圍檢索 Kendra/vector store；FM 根據 retrieved context 生成回答；回答附來源、簡化術語，必要時轉給 loan officer。

此類架構要特別注意 regulated content。回答不能承諾利率、法律/財務建議或超出授權文件內容。應記錄 prompt、retrieval source、model version、response、feedback，以支援 audit 與品質改善。

## Implementation Challenges

- Training stability issues
- Mode collapse
- Latent space interpolation challenges
- Hallucination
- Ethical misuse，例如 deepfake、fraud、misinformation
- Data privacy
- Copyright and legal risk
- Prompt injection
- Cost control
- Evaluation difficulty

Training stability issues 包含 gradient instability、資料品質不足、hyperparameter 不合適。Mode collapse 是 generator 只產生少數類型結果，缺乏多樣性。Latent space interpolation challenge 是在 latent representation 之間移動時生成結果不連續或不符合語義。

Ethical misuse 包含 deepfake、phishing、fraud、misinformation、copyright infringement、bias amplification。Enterprise 必須建立 acceptable use policy、content filtering、abuse monitoring、data governance、human oversight。

Prompt injection 是 GenAI application 的特殊風險。攻擊者可能在 user prompt 或 retrieved document 中放入指令，試圖讓 model 忽略系統規則、洩漏資料或執行不當 tool call。Mitigation 包含 instruction hierarchy、tool permission checks、retrieval sanitization、output validation、least privilege tools。

## Summary

Generative AI architecture 不是單純接一個 FM API。可靠 GenAI system 需要選擇合適 FM、設計 RAG/knowledge base、處理 hallucination、建立 guardrails、管理 security/compliance、監控成本與品質，並以 evaluation 驗證 business outcome。
