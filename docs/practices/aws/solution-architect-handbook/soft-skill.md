---

---

# Learning Soft Skills to Become a Better Solutions Architect

> Source: `aws/references/solution-architect-handbook/soft-skill.pdf`

本章整理 Solutions Architect 所需的 soft skills。核心觀念：架構師不只做 technical design，也要能溝通 business value、建立共識、承擔 ownership、推動 strategy execution，並持續學習與影響組織。

## Importance of Soft Skills

Solutions Architect 的工作跨 business、engineering、operations、security、procurement、executives。技術能力不足以讓 architecture 成功；還需要：

- Communication
- Stakeholder management
- Negotiation
- Presentation
- Ownership
- Strategic thinking
- Adaptability
- Design thinking
- Mentoring
- Technology evangelism

Solutions Architect 的主要輸出不是 diagram，而是讓組織做出可執行且可防守的技術決策。這需要把 complex technical trade-off 翻成 business impact，並讓不同角色在同一個方向上行動。Soft skills 會直接影響 architecture 是否被採納、是否被正確實作、是否能長期維運。

常見溝通對象包含 C-level executive、product owner、engineering manager、developer、operations、security、finance、procurement、vendor、customer。每個對象關心的資訊不同，架構師要調整深度與語言。

## Pre-Sales Skills

Pre-sales 是 complex technology procurement 的重要階段，常涉及 RFP/RFI/RFQ、customer discovery、solution proposal、demo、POC。

Key skills：

- Technical depth
- Business understanding
- Requirement discovery
- Proposal writing
- Presentation
- Objection handling
- Competitive positioning
- Cost/value articulation

這些能力也適用於日常 architecture work，因為架構師常需要說服 stakeholder 採納設計。

Pre-sales 的重點不是硬推產品，而是快速理解 customer pain、business driver、constraints、decision process 與 success criteria。Discovery 問題要能找出 hidden requirement，例如 compliance、integration、migration timeline、budget owner、procurement process、existing vendor commitment。

Proposal 要把 technical solution 連到 measurable value，例如降低 outage risk、縮短 release lead time、支援 peak traffic、降低 TCO、改善 compliance posture。Demo/POC 要聚焦 customer 最不確定的風險，不是展示所有功能。

## Presenting to C-Level Executives

Executive time limited，不應把所有 technical details 都放進簡報。應聚焦：

- Business outcome
- Risk
- Cost
- Timeline
- Strategic alignment
- Customer impact
- Decision needed

需準備回答：

- Why now?
- What is the business value?
- What is the risk of doing/not doing?
- What investment is required?
- How will success be measured?
- What trade-offs exist?

C-level presentation 應先講 outcome 與 decision request，再補關鍵風險與選項。不要用服務清單淹沒 executive。有效結構通常是：business problem -> proposed direction -> expected value -> investment/cost -> risk/mitigation -> decision needed -> timeline。

對 executive 說明技術時，要用他們熟悉的概念，例如 revenue impact、customer churn、regulatory exposure、operating cost、time-to-market、risk reduction。細節要準備好，但只在被問到時展開。

## Ownership and Accountability

架構師需要 end-to-end ownership，從 customer/sponsor 角度思考結果。Ownership 包含：

- 主動找出 gap
- 明確決策與 trade-off
- 對設計結果負責
- 追蹤 execution
- 面對問題時推動修正而非只指出問題

Ownership 表現在 ambiguity 下仍能推進。架構師不一定擁有所有 team，但要能指出 decision gap、找對 stakeholder、整理 options、讓風險透明、推動下一步。Accountability 也代表承認設計假設錯誤並修正，而不是把問題推給 implementation team。

## OKRs

OKR (Objectives and Key Results) 用於 strategy execution。

- Objective：清楚、有方向的目標
- Key Results：可量化、可驗證的成果

例如 architecture OKR 可包含 system reliability、performance、cost reduction、migration progress、security posture improvement。

OKR 的價值是讓不同層級 stakeholder 對 outcome 有共同理解。

OKR 的 superpower 在於 focus、alignment、tracking、stretch goal。Architecture OKR 應避免寫成 task list，例如「導入 Kubernetes」不是好 objective；更好的 objective 是「提升 checkout platform reliability」，KR 可是「p95 checkout latency 降到 300 ms 以下」「payment failure caused by timeout 下降 50%」「完成 multi-AZ failover drill 並達成 RTO 30 分鐘」。

## Thinking Big

Thinking big 是預測組織未來需要，提前規劃 scalable、extendable、future-proof architecture。不是盲目 over-engineering，而是辨識趨勢與長期 constraints。

Thinking big 要同時保留 execution path。架構師可以提出 long-term target architecture，但要拆成 MVP、migration waves、incremental milestones。否則願景太大會變成無法落地的投影片。好的 thinking big 是讓今天的決策不阻礙明天的擴展。

## Flexibility and Adaptability

架構師需要適應不同 assignment、technology、team maturity、business priority。

表現包含：

- 接受不同 team 使用不同技術，但要求 secure API/interface
- 能在 constraints 下調整方案
- 能處理 ambiguity
- 能在 business priority 改變時更新 roadmap

Flexibility 不是沒有標準。架構師可以接受不同 team 有不同技術選擇，但要堅持共同 boundary：secure API、logging、monitoring、data ownership、deployment standard、compliance baseline。Adaptability 也包含在 budget、timeline、team skill 不足時調整方案，而不是只提出理想架構。

## Design Thinking

Design thinking 強調 human-centric problem solving。

原則：

- Empathize：理解 user/stakeholder
- Define：重新定義問題
- Ideate：提出多種解法
- Prototype：快速驗證
- Test：根據 feedback iteration

架構設計應從 user/business problem 出發，而不是從技術產品出發。

Design thinking 的 iterative nature 很適合 architecture discovery。Empathize 階段理解 user/stakeholder pain；Define 階段把模糊需求轉成明確問題；Ideate 階段提出多個架構選項；Prototype 階段用 POC、diagram、thin slice 驗證；Test 階段收集 feedback 調整。

這能避免「solution looking for a problem」。架構師應先確認問題是否正確，再決定用 serverless、microservices、data lake、GenAI 或其他技術。

## Being a Builder

架構師應保有 hands-on coding/prototyping 能力。Prototype 可幫助：

- 驗證 technical feasibility
- 理解 implementation complexity
- 建立 team confidence
- 發現 hidden constraints
- 支援 pre-sales 或 architecture decision

不代表架構師要寫所有 production code，但不能完全脫離實作現實。

Hands-on builder 能力讓架構師更能判斷方案可行性。Prototype 可以暴露 IAM/network/library/runtime 限制，避免 architecture 在白板上成立但實作不可行。架構師至少應能閱讀 code、操作 cloud console/CLI、理解 CI/CD pipeline、建立小型 POC。

## Continuous Learning

Technology 變化快，架構師必須持續學習：

- Reading documentation/books
- Hands-on labs
- Certifications
- Conferences
- POCs
- Community participation
- Reviewing postmortems
- Learning from adjacent domains

Continuous learning 讓架構師能判斷新技術是否有真實 value，而不是追逐 hype。

學習方式應包含 breadth 與 depth。Breadth 讓架構師知道哪些技術存在；depth 讓架構師能做可防守的決策。有效方法包括閱讀 official documentation、做 hands-on lab、參與 architecture review、研究 outage postmortem、比較 reference architecture、與 domain expert 討論。

持續學習也包含淘汰舊知識。Cloud service、security threat、compliance、AI tooling 變化很快，過期的 best practice 可能變成風險。

## Mentoring

Mentoring 是幫助他人成長。好的 mentor 需要：

- 建立 informal communication
- 分享經驗與失敗教訓
- 提供清楚 feedback
- 幫助 mentee 建立 decision framework
- 鼓勵 ownership

Mentoring 不是直接給答案，而是幫 mentee 建立思考框架。可以透過 design review、pairing、postmortem discussion、career goal review、presentation coaching 協助成長。好的 mentor 會分享自己犯過的錯，讓 mentee 避免重複踩坑。

Mentoring 也能擴大 architecture influence。當更多 engineer 理解 trade-off、security、operability、cost，架構品質會分散到 team，而不是只靠架構師審查。

## Technology Evangelist and Thought Leader

Technology evangelism 包含 speaking、blog、whitepaper、internal sharing、reference architecture。目的是擴散實務經驗與推動 organization adoption。

Thought leadership 需要可信度。內容應來自實際經驗、POC、production lessons、customer problem，而不是只轉述 vendor marketing。對內 evangelism 可以是 brown bag、architecture guild、reference implementation、coding standard；對外可以是 conference、blog、community contribution。

Evangelist 的責任是幫組織採用合適技術，也要指出不適合的情境。推廣技術時要同時說明 trade-off、migration path、operation requirement。

## Summary

Solutions Architect 的 soft skills 直接影響 architecture 是否能被理解、採納、實作與維運。Pre-sales、executive presentation、ownership、OKR、thinking big、adaptability、design thinking、hands-on prototyping、continuous learning、mentoring、evangelism 都是架構師從 technical designer 成為 technical leader 的關鍵能力。
