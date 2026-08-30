---

---

# Security Considerations

> Source: `aws/references/solution-architect-handbook/security.pdf`

本章整理 architectural security 的原則與技術選型，涵蓋 authentication、authorization、IAM、web security、network boundary、data protection、API security、compliance、shared responsibility 與 threat modeling。核心觀念：security 必須套用在每一層，並以 automation 與 monitoring 持續驗證。

## Design Principles

### Authentication and Authorization

- Authentication：確認使用者是誰
- Authorization：確認使用者能做什麼

應使用 centralized identity、SSO、MFA、least privilege、role-based 或 attribute-based access control。

Authentication 要處理使用者、service account、machine identity 與 third-party integration。Authorization 則要集中管理 policy，避免每個 application 各自硬編規則。實務上應區分 employee、vendor、customer、application/service-to-service identity，並使用不同 trust boundary 與 access pattern。

MFA、password rotation、credential lifecycle、session timeout、token expiration、privileged access management 都要納入設計。對高風險操作應使用 step-up authentication 或 break-glass process。

### Security Everywhere

Security 不只在 edge firewall。應涵蓋：

- Web layer
- Application layer
- Infrastructure layer
- Database layer
- Network layer
- Data lifecycle
- CI/CD pipeline

Security everywhere 的意思是 defense in depth。Web layer 用 WAF/rate limit；application layer 做 input validation 與 authorization check；infrastructure layer 做 hardening/patching；network layer 做 segmentation；database layer 做 encryption/access policy；pipeline 做 dependency/container/IaC scanning；operation layer 做 monitoring/audit/incident response。

### Reducing Blast Radius

透過 segmentation、least privilege、network isolation、per-service role、small trust boundary，讓 compromise 影響範圍最小化。

Blast radius reduction 在 AWS 上常透過 multi-account strategy、separate VPC/subnet、security group、IAM role per workload、KMS key policy、resource policy、SCP 與 environment isolation 達成。不要讓所有 application 共用同一組 admin credential、同一個 database user 或同一個 broad IAM role。

### Monitoring and Auditing

所有重要 activity 都應 logging/auditing，並建立 alert。Audit trail 對 incident response 與 compliance 都必要。

Audit logs 需要防竄改。CloudTrail、VPC Flow Logs、WAF logs、application audit log 應送到集中帳號或 log archive，限制 production team 直接刪除。Monitoring 要關注 privilege escalation、abnormal API calls、public exposure、failed login spikes、unexpected data access、configuration drift。

### Automating Security

Security controls 應 as code 管理，包含 policy、network rule、compliance check、vulnerability scan、incident response。

Security automation 可用在 preventive、detective、responsive controls。Preventive 包含 IaC policy gate、SCP、permission boundary；detective 包含 Config rule、GuardDuty、Security Hub、Inspector；responsive 包含隔離 instance、revoke key、quarantine bucket、disable user、trigger incident workflow。

### Protecting Data

資料要依 sensitivity classification 處理，例如 public、private、confidential。依分類套用 encryption、tokenization、masking、access control、retention policy。

資料生命週期包含 collect、store、process、transmit、archive、delete。每個階段要定義 owner、classification、legal basis、retention、access review、backup、destruction。PII、payment data、health data、credentials、trade secrets 都需要不同控制。

### Incident Response

要建立 incident response plan 並演練。自動化 detection、investigation、remediation 可縮短 response time。

Incident response 應包含 preparation、identification、containment、eradication、recovery、lessons learned。不同 incident 類型需要不同 runbook，例如 credential leak、malware、DDoS、data exfiltration、ransomware、privilege escalation。演練可驗證 team 是否知道如何封鎖流量、撤銷 credential、保留 evidence、通知 stakeholder。

## Identity and Access Management

Enterprise user management 通常使用 centralized directory。

技術：

- Active Directory
- LDAP
- AWS Directory Service
- AD Connector
- Simple AD
- Okta / Ping Identity / Centrify
- Federation / SSO

FIM (Federated Identity Management) 讓使用者以既有 identity 存取 service，減少密碼管理負擔。

Identity architecture 要避免每個 application 自己保存 username/password。Enterprise 通常使用 Microsoft Active Directory 或 LDAP 作為 user directory，透過 federation/SSO 讓使用者以既有 credential 存取 SaaS 或 cloud workload。AWS Directory Service 可讓 AWS resources 與 existing Microsoft AD 整合；AD Connector 可把 authentication request proxy 到 on-premises AD；Simple AD 提供 managed directory。

Kerberos 是常見 enterprise authentication protocol，用 ticket 機制讓 client/server 互相驗證。LDAP 是查詢與管理 directory information 的協定。這些 legacy/enterprise identity 技術常在 cloud migration 時成為 hybrid dependency，因此 network、DNS、time sync、domain trust 都要設計清楚。

## Federation, SAML, OAuth, JWT

- SAML：XML-based assertion，常用於 enterprise SSO
- OAuth 2.0：authorization delegation，常用於 customer-facing apps
- OpenID Connect：在 OAuth 上增加 identity layer
- JWT：compact token format，可攜帶 claims，用於 API authorization

大型 customer-facing application 可用 Amazon Cognito 這類 managed identity service 管理 registration、login、MFA、federation。

SAML 常用於 enterprise SSO。Identity Provider (IdP) 驗證 user 後發出 SAML assertion 給 Service Provider (SP)，SP 根據 assertion 建立 session 與權限。OAuth 2.0 重點是 delegated authorization，例如某 app 取得使用者授權讀取 profile，不需要拿到使用者密碼。OpenID Connect 在 OAuth 2.0 上加入 identity layer，透過 ID token 表示 user identity。

JWT 由 header、payload、signature 組成，payload 中有 claims，例如 issuer、subject、audience、expiration、scope/role。JWT 必須驗證 signature、issuer、audience、expiration，不應只 decode 後信任內容。Token lifetime 過長會增加風險，過短則影響 UX，需要 refresh token/session design。

## Web Security

常見攻擊：

- DoS / DDoS
- SQL Injection (SQLi)
- Cross-Site Scripting (XSS)
- Cross-Site Request Forgery (CSRF)
- Credential stuffing
- Bot abuse

防護：

- WAF
- DDoS protection
- Input validation
- Output encoding
- Prepared statements
- Rate limiting
- Bot detection
- Secure cookies
- TLS everywhere

DDoS 目標是耗盡 bandwidth、connection、CPU、thread pool 或 application resource。防護要分層：只公開必要 endpoint、使用 CDN/edge absorb traffic、WAF rate-based rule、AWS Shield、load balancer、Auto Scaling、static content offload、application-level throttling。Auto Scaling 可吸收流量，但也要搭配 cost guardrail，避免攻擊導致成本暴增。

SQLi 通常發生在把 user input 拼接進 SQL query。防護是 prepared statements/parameterized queries、ORM 安全用法、input validation、least-privilege database user、WAF rule。XSS 是 attacker 注入 script 到使用者瀏覽器，防護是 output encoding、Content Security Policy、sanitize HTML、HttpOnly/Secure/SameSite cookie。CSRF 則利用使用者已登入 session 觸發未授權操作，防護是 CSRF token、SameSite cookie、re-auth for sensitive action。

Buffer overflow/memory corruption 常見於低階語言或 native dependency。防護包含安全語言/函式庫、compiler hardening、ASLR/DEP、patching、fuzz testing、least privilege execution。

## Application Security

Application layer 需注意：

- Secure coding
- Dependency scanning
- Secret management
- SAST/DAST
- API validation
- Error handling 不洩漏敏感資訊
- Logging 不寫入 secrets/PII
- Secure session management

Application hardening 包含移除 unnecessary packages/services、關閉未使用 ports、使用安全 default config、限制 process permission、自動 restart failed process、集中 patching。Secure code 要避免 hardcoded secret、insecure deserialization、path traversal、SSRF、command injection、weak crypto。

Secrets 應放在 AWS Secrets Manager、SSM Parameter Store 或等效 secret manager，不應放在 source code、AMI、container image、log、environment dump。Error message 不應回傳 stack trace、SQL statement、internal hostname、token 或 PII。

## Infrastructure and Network Security

建立 trusted boundaries：

- VPC/subnet segmentation
- Security groups / NACLs
- Private subnets for backend
- Bastion 或 SSM Session Manager
- No direct public access to database
- Firewall / WAF
- VPN / Direct Connect
- Network logging

Network security 要定義每一層 trust boundary。Public subnet 應只放需要 public ingress 的 load balancer/bastion；application/database 放 private subnet；database 不直接公開；egress 也要限制與記錄。Security group 採 stateful allow rule，NACL 可作 subnet-level stateless control。

IDS/IPS 可分 host-based 與 network-based。Host-based IDS 在 server/endpoint 上觀察 file/process/login；network-based IDS 在 network path 觀察 traffic pattern。Network-based inspection 若需要解密 TLS，會帶來 performance 與 key management 風險，因此要審慎設計。

## Data Protection

Data states：

- Data at rest
- Data in transit
- Data in use

加密方法：

- Symmetric encryption：同一把 key 加解密，效率高
- Asymmetric encryption：public/private key，適合 key exchange、digital signature
- KMS / key management：集中管理 key lifecycle、rotation、access control

也要處理 backup encryption、data retention、data deletion、PII masking。

Data classification 可依 public、internal、confidential、restricted 或更細分類。分類會決定 encryption、access control、logging、masking、retention、sharing approval。敏感資料應避免出現在 log、analytics export、test environment 與 screenshot。

Data at rest encryption 可用 database/storage native encryption 或 application-level encryption。AWS KMS 使用 envelope encryption：data key 加密資料，master/customer managed key 加密 data key。Key policy、grant、rotation、deletion window、cross-account use 都要管理。

Data in transit 應使用 TLS，並注意 certificate lifecycle、protocol version、cipher policy、mTLS 是否需要。Data in use 可考慮 memory protection、confidential computing 或最小化處理範圍。

## API Security

API 是 modern architecture 的主要攻擊面。

Best practices：

- Authentication/authorization
- Rate limiting
- Input validation
- Schema validation
- TLS
- API gateway
- Logging/auditing
- Token expiration
- Least privilege service roles

API Gateway 可集中處理 authentication、authorization、rate limiting、request validation、WAF integration、logging、throttling、versioning。Service-to-service API 要使用 IAM、mTLS、signed request 或 short-lived token，不應用 long-lived shared secret。

API security 也要防止 mass assignment、broken object level authorization (BOLA)、excessive data exposure、replay attack。每個 request 都應檢查 user 是否能 access 該 object，而不是只檢查已登入。

## Compliance and Shared Responsibility

常見 compliance：

- PCI-DSS
- HIPAA
- GDPR
- SOC
- FedRAMP
- ISO

Cloud 採 shared responsibility model：provider 負責 cloud infrastructure security，customer 負責 workload、data、identity、configuration、application security。

Shared responsibility 會隨服務模型變動。EC2/IaaS 下 customer 負責 OS patch、host firewall、runtime；RDS/PaaS 下 provider 管 database engine infrastructure，但 customer 仍管 schema、access、encryption、backup policy；SaaS 下 customer 多負責 identity、data classification、configuration、user access。

Compliance 不是只看 provider certification。即使 AWS 具備某些 compliance attestations，customer workload 的 IAM、logging、encryption、data retention、network exposure、application controls 仍需符合規範。

## Threat Modeling

Threat model 用來系統性找出 assets、trust boundaries、threats、abuse cases、mitigations。

應回答：

- What are we protecting?
- Who can attack?
- Where are trust boundaries?
- What can go wrong?
- What controls reduce risk?
- How do we detect/respond?

Threat modeling 可用 STRIDE 類方法：Spoofing、Tampering、Repudiation、Information disclosure、Denial of service、Elevation of privilege。輸出不應只是風險清單，也要包含 mitigation、owner、priority、residual risk 與 validation method。設計階段做 threat model 比 production 後補 security control 成本低很多。

## Summary

Security architecture 必須從 identity、network、application、data、API、monitoring、incident response 到 compliance 全面設計。重點是 least privilege、defense in depth、blast radius reduction、security automation、continuous auditing，以及以 threat model 驗證設計。
