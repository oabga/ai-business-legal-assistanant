# Kiến trúc hệ thống chi tiết

> Bổ sung cho `docs/MVP_PRODUCT_ARCHITECTURE.md` — tài liệu đó giải thích
> *tại sao*, file này vẽ *như thế nào* bằng sơ đồ Mermaid (render được trực
> tiếp trên GitHub/GitLab và VS Code preview). Mỗi sơ đồ có đoạn giải thích
> ngắn ngay phía trên.

## Mục lục

1. [Tổng quan thành phần hệ thống](#1-tổng-quan-thành-phần-hệ-thống)
2. [Flow: câu hỏi pháp lý qua Agentic RAG](#2-flow-câu-hỏi-pháp-lý-qua-agentic-rag)
3. [Ví dụ minh hoạ: câu hỏi đa domain](#3-ví-dụ-minh-hoạ-câu-hỏi-đa-domain)
4. [Flow: đồng bộ & xuất bản văn bản pháp luật](#4-flow-đồng-bộ--xuất-bản-văn-bản-pháp-luật)
5. [Mô hình dữ liệu vòng đời văn bản](#5-mô-hình-dữ-liệu-vòng-đời-văn-bản)
6. [Kiến trúc triển khai AWS](#6-kiến-trúc-triển-khai-aws)
7. [Flow CI/CD](#7-flow-cicd)
8. [Flow observability — trace đi đâu](#8-flow-observability--trace-đi-đâu)
9. [Sơ đồ flow cho tính năng mở rộng (vai trò, cộng tác, tích hợp)](#9-sơ-đồ-flow-cho-tính-năng-mở-rộng)

---

## 1. Tổng quan thành phần hệ thống

```mermaid
graph TB
    Browser[Trình duyệt]

    subgraph Edge["Edge"]
        CDN[CloudFront CDN]
        WAF[AWS WAF]
    end

    NextJS["Next.js App<br/>(Vercel hoặc ECS Fargate)"]

    subgraph Backend["Backend — ECS Fargate"]
        API[FastAPI]
        Agent["LangGraph Agent<br/>(Legal Assistant)"]
        Worker["Worker service<br/>(crawler + re-embed)"]
    end

    subgraph Data["Data layer"]
        PG[("PostgreSQL<br/>+ pgvector + tsvector")]
        Redis[("Redis<br/>cache / session")]
        S3[("S3<br/>snapshots, uploads")]
        SQS[["SQS<br/>crawl/embed queue"]]
    end

    subgraph External["Dịch vụ ngoài"]
        LLM["LLM Provider<br/>(Gemini / Claude / GPT)"]
        VBPL["vbpl.vn<br/>(nguồn chính thống)"]
        Langfuse["Langfuse<br/>(self-host, ECS)"]
    end

    Browser --> CDN --> WAF --> NextJS
    NextJS -->|"REST + SSE"| API
    API --> Agent
    Agent --> PG
    Agent --> LLM
    Agent -.trace.-> Langfuse
    API --> Redis
    API --> S3
    Worker --> SQS
    SQS --> Worker
    Worker --> VBPL
    Worker --> PG
    Worker --> S3
```

**Đọc sơ đồ:** người dùng chỉ chạm vào Next.js qua CDN/WAF; mọi nghiệp vụ nằm
trong `Agent` (LangGraph) được FastAPI gọi; `Worker` là service tách biệt lo
crawl + re-embed, không nằm trên đường request của người dùng nên không ảnh
hưởng latency chat. `Langfuse` nhận trace bất đồng bộ (đường nét đứt) — không
chặn response nếu Langfuse chậm/lỗi.

## 2. Flow: câu hỏi pháp lý qua Agentic RAG

Chi tiết hoá graph LangGraph đã mô tả ở `MVP_PRODUCT_ARCHITECTURE.md` mục 3,
vẽ đủ nhánh rẽ (multi-domain, validity, confidence gate):

```mermaid
flowchart TD
    Start(["Người dùng gửi câu hỏi"]) --> Intent["analyze_intent"]
    Intent --> Domain{"Bao nhiêu domain<br/>pháp lý liên quan?"}
    Domain -->|"1 domain"| Prep["prepare_retrieval_query<br/>(rewrite / HyDE)"]
    Domain -->|"nhiều domain"| Fanout["Fan-out song song<br/>theo từng domain"]
    Fanout --> Prep
    Prep --> Retrieve["retrieve<br/>(hybrid: pgvector + BM25/tsvector, RRF)"]
    Retrieve --> Rerank["rerank + llm_filter"]
    Rerank --> Validity{"★ validity_check<br/>Điều còn hiệu lực tại<br/>thời điểm câu hỏi áp dụng?"}
    Validity -->|"còn hiệu lực"| Merge["Hợp nhất kết quả<br/>các domain"]
    Validity -->|"hết hiệu lực"| Flag["Loại khỏi context<br/>hoặc đánh dấu cảnh báo"]
    Flag --> Merge
    Merge --> Generate["generate_answer<br/>(structured citation,<br/>không phải text tự do)"]
    Generate --> Verify{"★ verify_groundedness<br/>citation có khớp<br/>đoạn trích gốc?"}
    Verify -->|"pass + confidence cao"| Respond["Trả lời kèm:<br/>nội dung · Điều + văn bản ·<br/>trạng thái hiệu lực · ngày hiệu lực"]
    Verify -->|"fail / confidence thấp"| Disclaim["Trả lời kèm cảnh báo:<br/>'không đủ căn cứ,<br/>nên tham vấn luật sư'"]
    Respond --> Log["Ghi citation_log (Postgres)<br/>+ đẩy trace (Langfuse)"]
    Disclaim --> Log
    Log --> End(["Trả về frontend qua SSE"])
```

Hai node ★ (`validity_check`, `verify_groundedness`) là phần mới so với RAG
một bước thông thường — xem lý do ở mục 3–4 của
`MVP_PRODUCT_ARCHITECTURE.md`.

## 3. Ví dụ minh hoạ: câu hỏi đa domain

Minh hoạ cụ thể cho pain point "câu hỏi chạm nhiều lĩnh vực luật cùng lúc":

```mermaid
sequenceDiagram
    actor U as Người dùng
    participant FE as Next.js Frontend
    participant API as FastAPI
    participant AG as LangGraph Agent
    participant DB as PostgreSQL (+pgvector)
    participant LLM as LLM Provider
    participant LF as Langfuse

    U->>FE: "Công ty trả lương trễ 2 tháng,<br/>bị phạt gì và phải đóng BHXH thế nào?"
    FE->>API: POST /chat/stream
    API->>AG: invoke(question)
    AG->>LLM: analyze_intent
    LLM-->>AG: domain = [Lao động, BHXH]

    par Nhánh Lao động
        AG->>DB: retrieve hybrid (Lao động)
        DB-->>AG: top-k Điều liên quan
    and Nhánh BHXH
        AG->>DB: retrieve hybrid (BHXH)
        DB-->>AG: top-k Điều liên quan
    end

    AG->>AG: validity_check (loại Điều hết hiệu lực)
    AG->>LLM: generate_answer (structured citation)
    LLM-->>AG: answer + citations
    AG->>LLM: verify_groundedness
    LLM-->>AG: confidence score
    AG->>DB: ghi citation_log
    AG-->>LF: trace toàn bộ bước (bất đồng bộ)
    AG-->>API: answer + citations + trạng thái hiệu lực
    API-->>FE: SSE stream
    FE-->>U: Câu trả lời + trích dẫn bấm được + cảnh báo nếu có
```

Hai nhánh `par` chạy song song — đây chính là lý do cần agentic (fan-out/merge)
thay vì một lần retrieve đơn giản.

## 4. Flow: đồng bộ & xuất bản văn bản pháp luật

Chi tiết hoá pipeline ở mục 2.3 của `MVP_PRODUCT_ARCHITECTURE.md`:

```mermaid
flowchart LR
    Cron["Cron trigger<br/>(EventBridge Scheduler,<br/>hàng ngày/tuần)"] --> Crawl["Crawl job<br/>(ECS task / Lambda)"]
    Crawl --> Fetch["Fetch vbpl.vn<br/>+ cross-check Công báo"]
    Fetch --> Snapshot["Lưu snapshot gốc<br/>(HTML/PDF) → S3"]
    Fetch --> Diff{"Có thay đổi so với<br/>bản đã lưu?"}
    Diff -->|"không"| Stop(["Kết thúc — không làm gì"])
    Diff -->|"có"| Staging["Ghi vào staging corpus<br/>(chưa publish)"]
    Staging --> Notify["Thông báo reviewer"]
    Notify --> Review{"Luật sư/chuyên gia<br/>duyệt qua UI admin"}
    Review -->|"từ chối"| Reject["Đánh dấu rejected<br/>+ lý do"]
    Review -->|"duyệt"| Publish["Publish:<br/>cập nhật legal_document /<br/>legal_article + quan hệ"]
    Publish --> Reembed["Incremental re-embed<br/>(chỉ phần thay đổi)"]
    Reembed --> Invalidate["Invalidate cache liên quan<br/>+ cập nhật vector index"]
    Invalidate --> Done(["Sẵn sàng phục vụ"])
```

**Điểm mấu chốt:** bước "Luật sư/chuyên gia duyệt" là **bắt buộc**, không
phải optional — không có văn bản nào lên production mà chưa qua review thủ
công (xem rủi ro ở mục 2.3 và 14 của tài liệu chính). Sự kiện publish ở đây
cũng là điểm khởi phát cho flow "cảnh báo thay đổi luật" — xem
[§9.6](#96-flow-cảnh-báo-thay-đổi-luật-liên-quan-tính-năng-5).

## 5. Mô hình dữ liệu vòng đời văn bản

Chi tiết hoá schema ở mục 5 của `MVP_PRODUCT_ARCHITECTURE.md`. Mô hình tổ
chức/vai trò (`ORGANIZATION_MEMBER`) chi tiết hoá mục 6 — ERD đầy đủ cho cả
12 tính năng mở rộng (mời thành viên, feedback, watchlist, calculator,
escalation, Zalo, partner...) ở [§9.1–§9.4](#91-erd-mở-rộng-tổ-chức--phân-quyền):

```mermaid
erDiagram
    LEGAL_DOCUMENT ||--o{ LEGAL_ARTICLE : contains
    LEGAL_DOCUMENT ||--o{ LEGAL_DOCUMENT_RELATION : "from / to"
    LEGAL_ARTICLE ||--o{ LEGAL_ARTICLE_VERSION : "có nhiều version"
    LEGAL_ARTICLE_VERSION ||--o{ CITATION_LOG : "được trích trong"
    CONVERSATION ||--o{ MESSAGE : contains
    MESSAGE ||--o{ CITATION_LOG : produces
    ORGANIZATION ||--o{ ORGANIZATION_MEMBER : has
    USER ||--o{ ORGANIZATION_MEMBER : "là thành viên"
    USER ||--o{ CONVERSATION : starts

    LEGAL_DOCUMENT {
        uuid id
        string so_hieu
        string loai_van_ban
        date ngay_ban_hanh
        date effective_from
        date effective_to
        string status
        string source_url
        string source_snapshot_s3_key
    }
    LEGAL_ARTICLE {
        uuid id
        uuid document_id
        string dieu_so
        date effective_from
        date effective_to
        string status
    }
    LEGAL_ARTICLE_VERSION {
        uuid id
        uuid article_id
        text content
        string embedding_model
        date valid_from
        date valid_to
    }
    CITATION_LOG {
        uuid message_id
        uuid article_version_id
        float verifier_score
    }
    ORGANIZATION_MEMBER {
        uuid id
        uuid organization_id
        uuid user_id
        string org_role "OWNER / ADMIN / MEMBER / VIEWER"
        string department "mở — không gate quyền"
        string status
    }
```

`LEGAL_ARTICLE_VERSION` là bảng trả lời được câu hỏi "tại thời điểm X áp dụng
bản nào" — `CITATION_LOG` trỏ thẳng vào version cụ thể, không chỉ số Điều, để
audit được chính xác hệ thống đã dùng bản nào khi trả lời.

## 6. Kiến trúc triển khai AWS

Chi tiết hoá mục 13 của `MVP_PRODUCT_ARCHITECTURE.md`:

```mermaid
graph TB
    User["Người dùng"]

    subgraph AWS["AWS — ap-southeast-1 (Singapore)"]
        CF["CloudFront + WAF"]

        subgraph VPC["VPC"]
            subgraph Public["Public subnet"]
                ALB["Application Load Balancer"]
            end
            subgraph Priv["Private subnet"]
                ECS_API["ECS Fargate<br/>FastAPI backend"]
                ECS_Worker["ECS Fargate<br/>Worker: crawler / re-embed"]
                ECS_LF["ECS Fargate<br/>Langfuse (self-host)"]
            end
            subgraph DataTier["Data subnet"]
                RDS[("RDS PostgreSQL<br/>Multi-AZ + pgvector")]
                Redis[("ElastiCache Redis")]
            end
        end

        S3B[("S3: snapshots,<br/>uploads, static assets")]
        SQSQ[["SQS: crawl/embed queue"]]
        SM["Secrets Manager"]
        CW["CloudWatch"]
    end

    User --> CF --> ALB --> ECS_API
    ECS_API --> RDS
    ECS_API --> Redis
    ECS_API --> S3B
    ECS_API -.trace.-> ECS_LF
    ECS_Worker --> SQSQ
    SQSQ --> ECS_Worker
    ECS_Worker --> RDS
    ECS_Worker --> S3B
    ECS_API --> SM
    ECS_Worker --> SM
    ECS_API --> CW
    ECS_Worker --> CW
```

MVP có thể bỏ Multi-AZ/autoscale (chạy 1 task mỗi service) để tiết kiệm chi
phí — scale lên khi có traffic thật, cấu trúc VPC/subnet giữ nguyên.

## 7. Flow CI/CD

Chi tiết hoá mục 12 của `MVP_PRODUCT_ARCHITECTURE.md`:

```mermaid
flowchart TD
    PR["Mở Pull Request"] --> Lint["Lint: ruff, oxlint<br/>Typecheck: tsc"]
    Lint --> Test["Unit test:<br/>pytest (+Postgres container), vitest"]
    Test --> EvalCheck{"Đổi prompt /<br/>retrieval / model config?"}
    EvalCheck -->|"có"| EvalGate["Eval gate:<br/>F2-score + RAGAS trên golden set<br/>→ đẩy kết quả lên Langfuse"]
    EvalCheck -->|"không"| Build
    EvalGate --> GateCheck{"Giảm quá ngưỡng<br/>đã chốt?"}
    GateCheck -->|"có"| Fail(["CI fail — chặn merge"])
    GateCheck -->|"không"| Build["Build Docker image<br/>backend + frontend"]
    Build --> ECR["Push ECR<br/>(tag = commit SHA)"]
    ECR --> MergeMain["Merge vào main"]
    MergeMain --> Staging["Deploy staging<br/>(ECS service update)"]
    Staging --> Smoke["Smoke test<br/>(scripts/smoke_*.py)"]
    Smoke --> Approval{"Manual approval<br/>(GitHub Environments)"}
    Approval -->|"approve"| Prod["Deploy production<br/>(rolling / blue-green)"]
    Approval -->|"reject"| Hold(["Giữ nguyên ở staging"])
```

Pipeline cập nhật corpus (mục 4 ở trên) chạy **tách biệt** với pipeline CI/CD
code — không gắn vào PR, chạy theo lịch crawl riêng.

## 8. Flow observability — trace đi đâu

```mermaid
flowchart LR
    subgraph Sources["Nguồn log/trace"]
        AgentNode["Mỗi node LangGraph<br/>(intent, retrieve, rerank,<br/>validity_check, generate,<br/>verify_groundedness)"]
        APIErr["Lỗi API/frontend"]
        Infra["Metric hạ tầng<br/>(ECS, RDS, ALB)"]
    end

    AgentNode -->|"Langfuse SDK<br/>callback"| Langfuse["Langfuse<br/>(self-host)"]
    APIErr --> Sentry["Sentry"]
    Infra --> CloudWatch["CloudWatch"]

    Langfuse --> Dash1["Dashboard: trace theo request,<br/>chi phí/latency theo tenant,<br/>RAGAS score theo version prompt"]
    Sentry --> Dash2["Dashboard: lỗi theo release"]
    CloudWatch --> Dash3["Dashboard: health hạ tầng,<br/>alarm autoscale"]
```

Ba hệ thống tách biệt theo đúng mục đích (LLM trace / lỗi ứng dụng / hạ tầng)
— không dồn hết vào một nơi để tránh dashboard quá tải tín hiệu không liên
quan khi debug.

## 9. Sơ đồ flow cho tính năng mở rộng

Chi tiết hoá mục 9 (12 tính năng) của `MVP_PRODUCT_ARCHITECTURE.md` và đặc tả
đầy đủ flow/API contract ở [`FEATURES.md`](FEATURES.md). Phần này chỉ vẽ ERD
mở rộng + các flow nhiều bước đáng vẽ nhất; những tính năng còn lại (dashboard,
calculator, so sánh phương án) là CRUD/aggregation đơn giản, đã đủ rõ ở dạng
API contract trong `FEATURES.md`, không cần thêm sơ đồ.

### 9.1 ERD mở rộng — tổ chức & phân quyền

Chi tiết hoá mục 6 của `MVP_PRODUCT_ARCHITECTURE.md` — thay thế quan hệ
`User 1-N Organization` hiện tại bằng N-N qua `ORGANIZATION_MEMBER`:

```mermaid
erDiagram
    USER ||--o{ ORGANIZATION_MEMBER : "là thành viên"
    ORGANIZATION ||--o{ ORGANIZATION_MEMBER : has
    ORGANIZATION ||--o{ ORGANIZATION_INVITE : "gửi lời mời"
    USER ||--o{ ORGANIZATION_INVITE : "được mời (theo email)"

    USER {
        uuid id
        string email
        bool is_system_admin "tách khỏi org_role — mục 6.1"
    }
    ORGANIZATION {
        uuid id
        string name
        string tax_code
    }
    ORGANIZATION_MEMBER {
        uuid id
        uuid organization_id
        uuid user_id
        string org_role "OWNER / ADMIN / MEMBER / VIEWER"
        string department "mở — không gate quyền"
        string status "active / removed"
    }
    ORGANIZATION_INVITE {
        uuid id
        uuid organization_id
        string email
        string org_role
        string token
        string status "pending / accepted / expired / revoked"
        datetime expires_at
    }
```

### 9.2 ERD mở rộng — tương tác & cộng tác nội bộ

Entity cho tính năng #2 (liên kết hợp đồng↔tuân thủ), #3 (feedback), #4
(nhắc email), #9 (escalation):

```mermaid
erDiagram
    MESSAGE ||--o{ MESSAGE_FEEDBACK : "nhận feedback"
    MESSAGE ||--o{ ESCALATION_REQUEST : "được escalate"
    ORGANIZATION ||--|| NOTIFICATION_SETTING : "cấu hình nhắc hạn"
    COMPLIANCE_TASK }o--|| DOCUMENT_REVIEW : "trích từ hợp đồng (nếu source_type=contract_extracted)"

    MESSAGE_FEEDBACK {
        uuid id
        uuid message_id
        uuid user_id
        string rating "up / down"
        string reason_code
    }
    ESCALATION_REQUEST {
        uuid id
        uuid message_id
        string status "pending / answered / rejected"
        uuid assigned_to
    }
    NOTIFICATION_SETTING {
        uuid organization_id PK
        bool email_reminders_enabled
        int_array remind_days_before "mặc định {7,3,1}"
    }
    COMPLIANCE_TASK {
        uuid id
        string source_type "rule_generated / contract_extracted / calculator / manual"
        uuid source_document_id "nullable"
        uuid source_review_id "nullable"
    }
    DOCUMENT_REVIEW {
        uuid id
        string note "entity đã có sẵn trong hệ thống, không phải mới"
    }
```

### 9.3 ERD mở rộng — theo dõi hiệu lực & tính toán

Entity cho tính năng #5 (cảnh báo thay đổi luật) và #7 (máy tính nghĩa vụ):

```mermaid
erDiagram
    USER ||--o{ LAW_WATCH : theo_dõi
    LAW_CHANGE_EVENT ||--o{ LAW_CHANGE_NOTIFICATION : "sinh thông báo"
    USER ||--o{ LAW_CHANGE_NOTIFICATION : nhận
    USER ||--o{ CALCULATOR_COMPUTATION : thực_hiện
    CALCULATOR_COMPUTATION }o--o| COMPLIANCE_TASK : "có thể lưu thành (save-as-task)"

    LAW_WATCH {
        uuid id
        uuid user_id
        string law_id
        string article "nullable — theo dõi cả văn bản nếu rỗng"
    }
    LAW_CHANGE_EVENT {
        uuid id
        string law_id
        string article
        string change_type "amended / repealed / new_version"
        datetime detected_at
    }
    LAW_CHANGE_NOTIFICATION {
        uuid id
        uuid user_id
        uuid law_change_event_id
        datetime read_at "nullable"
    }
    CALCULATOR_COMPUTATION {
        uuid id
        uuid user_id
        uuid organization_id
        string calculator_code
        jsonb inputs
        jsonb result
        datetime created_at
    }
```

### 9.4 ERD mở rộng — tích hợp ngoài

Entity cho tính năng #10 (Zalo OA) và #12 (API đối tác):

```mermaid
erDiagram
    USER ||--o| ZALO_LINK : "link tài khoản"
    PARTNER ||--o{ PARTNER_API_KEY : có
    PARTNER ||--o{ PARTNER_ORGANIZATION_GRANT : "được cấp quyền"
    ORGANIZATION ||--o{ PARTNER_ORGANIZATION_GRANT : "cấp quyền cho"

    ZALO_LINK {
        uuid id
        uuid user_id
        string zalo_user_id
        datetime linked_at
    }
    INTEGRATION_MESSAGE {
        uuid id
        string channel "zalo / web"
        string external_message_id "unique theo (channel, external_message_id)"
        uuid conversation_id
    }
    PARTNER {
        uuid id
        string name
        string status
    }
    PARTNER_API_KEY {
        uuid id
        uuid partner_id
        string key_hash
    }
    PARTNER_ORGANIZATION_GRANT {
        uuid id
        uuid partner_id
        uuid organization_id
        string org_role "member / viewer — không bao giờ owner/admin"
    }
```

`INTEGRATION_MESSAGE` không có quan hệ vẽ sẵn ở trên vì vai trò của nó là
**chặn trùng** khi Zalo retry webhook (tra theo `external_message_id` trước
khi xử lý), không phải để truy vấn/join thường xuyên.

### 9.5 Flow: mời thành viên vào tổ chức (tính năng #1)

```mermaid
sequenceDiagram
    actor Owner as Owner/Admin
    participant FE as Frontend
    actor Invitee as Người được mời
    participant API as FastAPI
    participant DB as PostgreSQL
    participant Mail as Email service

    Owner->>FE: Nhập email + chọn org_role + department
    FE->>API: POST /organizations/{id}/invites
    API->>DB: Tạo ORGANIZATION_INVITE (token, expires_at)
    API->>Mail: Gửi email có link accept
    Mail-->>Invitee: Email mời tham gia

    alt Invitee đã có tài khoản
        Invitee->>FE: Đăng nhập → bấm Accept
    else Invitee chưa có tài khoản
        Invitee->>FE: Đăng ký kèm token
    end

    FE->>API: POST /invites/accept { token }
    API->>DB: Kiểm tra token hợp lệ + chưa hết hạn
    API->>DB: Tạo ORGANIZATION_MEMBER, đánh dấu invite = accepted
    API-->>FE: 200 OrganizationMemberOut
    FE-->>Invitee: Vào thẳng workspace tổ chức
```

### 9.6 Flow: cảnh báo thay đổi luật liên quan (tính năng #5)

Mở rộng trực tiếp từ flow đồng bộ ở [§4](#4-flow-đồng-bộ--xuất-bản-văn-bản-pháp-luật) —
bắt đầu từ bước "Publish" của sơ đồ đó:

```mermaid
flowchart TD
    Publish(["Publish văn bản mới/sửa đổi<br/>(tiếp theo §4)"]) --> CreateEvent["Tạo LAW_CHANGE_EVENT"]
    CreateEvent --> FindWatch["Join với LAW_WATCH<br/>(theo dõi chủ động)"]
    CreateEvent --> FindCitation["Join với CITATION_LOG<br/>(đã từng trích cho user nào)"]
    FindWatch --> Union["Hợp nhất danh sách<br/>user bị ảnh hưởng"]
    FindCitation --> Union
    Union --> Empty{"Có user nào<br/>bị ảnh hưởng?"}
    Empty -->|"không"| Done1(["Kết thúc"])
    Empty -->|"có"| CreateNotif["Tạo LAW_CHANGE_NOTIFICATION<br/>cho từng user"]
    CreateNotif --> SendMail["Gửi email tóm tắt"]
    CreateNotif --> Banner["Hiển thị banner trong app<br/>khi user đăng nhập lại"]
    SendMail --> Done2(["Kết thúc"])
    Banner --> Click{"User click xem?"}
    Click -->|"có"| ShowDiff["Hiển thị: câu trả lời cũ +<br/>đoạn luật mới + diff"]
    Click -->|"không"| Done2
```

### 9.7 Flow: case builder — node `clarify` (tính năng #6)

Chèn thêm vào đầu graph agentic RAG ở [§2](#2-flow-câu-hỏi-pháp-lý-qua-agentic-rag),
trước `prepare_retrieval_query`:

```mermaid
flowchart TD
    Intent["analyze_intent<br/>(như §2)"] --> NeedClarify{"★ clarify<br/>Câu hỏi có thiếu thông tin<br/>quan trọng không?"}
    NeedClarify -->|"đủ thông tin"| Prep["prepare_retrieval_query<br/>(tiếp tục §2)"]
    NeedClarify -->|"thiếu, chưa hỏi lại lần nào"| AskUser["Sinh 1-3 câu hỏi làm rõ<br/>dạng quick-reply"]
    AskUser --> WaitUser["Trả SSE event<br/>'clarifying_question', dừng chờ"]
    WaitUser --> UserReply["User trả lời<br/>(hoặc bỏ qua)"]
    UserReply --> Merge["Gộp câu trả lời vào context"]
    Merge --> Prep
    NeedClarify -->|"thiếu, nhưng đã hỏi lại 1 lần rồi"| Assume["Trả lời kèm giả định rõ ràng<br/>(không hỏi lại vòng 2)"]
    Assume --> Prep
```

### 9.8 Flow: escalation sang chuyên gia thật (tính năng #9)

```mermaid
sequenceDiagram
    actor U as Người dùng
    participant FE as Frontend
    participant API as FastAPI
    participant DB as PostgreSQL
    actor Expert as Chuyên gia (admin panel)

    Note over U,FE: Confidence thấp (verify_groundedness) → đề xuất escalate,<br/>hoặc user tự bấm "Yêu cầu chuyên gia"
    U->>FE: Bấm "Yêu cầu chuyên gia xem lại"
    FE->>API: POST /conversations/{id}/messages/{id}/escalate
    API->>DB: Tạo ESCALATION_REQUEST (status=pending)
    API-->>FE: 201 — hiển thị "Đang chờ chuyên gia"

    Expert->>API: GET /escalations?status=pending
    API-->>Expert: Danh sách request đang chờ
    Expert->>API: POST /escalations/{id}/respond { answer }
    API->>DB: Tạo MESSAGE mới (role=expert)
    API->>DB: Cập nhật ESCALATION_REQUEST.status = answered
    API-->>FE: (qua poll/SSE) message mới xuất hiện trong conversation
    FE-->>U: Hiển thị câu trả lời chuyên gia (badge khác màu với AI)
```

### 9.9 Flow: Zalo OA integration (tính năng #10)

```mermaid
sequenceDiagram
    actor U as Người dùng (Zalo)
    participant Zalo as Zalo OA
    participant API as FastAPI
    participant DB as PostgreSQL
    participant AG as LangGraph Agent

    U->>Zalo: Nhắn tin hỏi pháp lý
    Zalo->>API: POST /integrations/zalo/webhook
    API->>DB: Tra ZALO_LINK theo zalo_user_id

    alt Chưa link tài khoản
        API-->>Zalo: Trả hướng dẫn link tài khoản + OTP
        Zalo-->>U: Hiển thị hướng dẫn
        Note over U,API: User mở web app, nhập OTP → POST /integrations/zalo/link
    else Đã link
        API->>AG: invoke(question) — cùng pipeline với kênh web
        AG-->>API: answer + citations
        API->>DB: Ghi IntegrationMessage (chặn xử lý trùng nếu Zalo retry)
        API-->>Zalo: Gửi trả lời (rút gọn, không markdown)
        Zalo-->>U: Hiển thị câu trả lời
    end
```

### 9.10 Kiến trúc API đối tác (tính năng #12)

```mermaid
graph LR
    Partner["Hệ thống đối tác<br/>(dịch vụ kế toán/luật)"]
    Partner -->|"X-API-Key"| API[FastAPI]

    subgraph Auth["Resolve quyền truy cập"]
        Check["Kiểm tra PARTNER_API_KEY<br/>+ PARTNER_ORGANIZATION_GRANT"]
    end

    API --> Check
    Check -->|"hợp lệ, scoped theo organization_id"| Endpoints["Endpoint hiện có:<br/>/legal/chat, /compliance/tasks,<br/>/documents..."]
    Check -->|"không có grant"| Reject["403 Forbidden"]

    Owner["Chủ DN (OWNER)"] -->|"POST .../partner-grants"| API
    Owner -.->|"có thể thu hồi<br/>bất kỳ lúc nào"| Check
```

Chủ DN luôn giữ quyền thu hồi (`DELETE .../partner-grants/{partner_id}`) —
Partner không bao giờ sở hữu `Organization`, chỉ được **grant** quyền truy
cập có thể huỷ.
