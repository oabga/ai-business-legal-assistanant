# Trợ lý Pháp lý AI — Từ bài thi đến sản phẩm có thể scale

> Tài liệu brainstorm kiến trúc + roadmap, viết lại bối cảnh: sản phẩm thật nhắm tới chủ doanh nghiệp, kế toán, nhân sự SME Việt
> Nam. Mục tiêu MVP trước, nhưng thiết kế để scale lên mà không phải viết lại.
>
> Sơ đồ chi tiết (Mermaid, đầy đủ các flow) xem tại
> [`docs/architecture.md`](architecture.md) — tài liệu này tập trung vào *tại
> sao*, file kia vẽ *như thế nào*. Flow + API contract chi tiết cho từng tính
> năng mở rộng (mục 9) xem [`docs/FEATURES.md`](FEATURES.md).

## 1. Định vị sản phẩm — tại sao không phải "chatbot pháp luật" thông thường

Thị trường đã có nhiều chatbot hỏi-đáp luật (thường là RAG một bước: embed câu
hỏi → lấy top-k đoạn văn bản → LLM trả lời). Vấn đề của nhóm này:

1. **Không biết văn bản mình trích có còn hiệu lực hay không.** Luật Việt Nam
   sửa đổi/bổ sung/thay thế liên tục, thường chỉ sửa một vài Điều/Khoản trong
   một văn bản còn lại vẫn hiệu lực — rất dễ trích nhầm bản cũ.
2. **Không phân biệt được hiệu lực theo thời điểm.** "Tại thời điểm ký hợp đồng
   năm 2022" và "tại thời điểm hiện tại" có thể áp dụng hai Điều khác nhau.
3. **Không nói rõ độ tin cậy.** Trả lời chắc nịch dù retrieval yếu — nguy hiểm
   với nghiệp vụ pháp lý/thuế vì người dùng có thể hành động sai và bị phạt.

**Khác biệt hoá của sản phẩm này = "độ tin cậy có thể kiểm chứng", không phải
độ mượt của hội thoại:**

- Mọi câu trả lời đi kèm **trạng thái hiệu lực + ngày hiệu lực** của từng Điều
  được trích, không chỉ trích nội dung suông.
- Có **pipeline đồng bộ liên tục** với nguồn chính thống, không phải corpus
  tĩnh nạp một lần như hiện tại.
- Có **lớp xác minh (verifier)** kiểm tra câu trả lời có thực sự được chứng
  minh bởi đoạn văn bản trích hay không, trước khi trả về người dùng.
- Khi không đủ căn cứ hoặc văn bản đang tranh chấp hiệu lực, hệ thống **chủ
  động nói "không chắc, cần luật sư xem"** thay vì đoán.

## 2. Pain point cốt lõi cần giải — "độ tươi" của pháp luật

Đây là rủi ro lớn nhất, nên thiết kế toàn bộ hệ thống xoay quanh nó thay vì
coi corpus là dữ liệu tĩnh.

### 2.1 Nguồn dữ liệu chính thống có thể móc nối

| Nguồn | Vai trò | Ghi chú |
| --- | --- | --- |
| `vbpl.vn` (Cơ sở dữ liệu quốc gia văn bản pháp luật — Bộ Tư pháp) | Nguồn chính thống số 1, có gắn nhãn "Tình trạng hiệu lực", liệt kê văn bản liên quan (sửa đổi/thay thế/hướng dẫn) | Không có API công khai → cần scraper tuân thủ robots.txt, chạy định kỳ |
| `congbao.chinhphu.vn` (Công báo) | Nguồn xác nhận ngày có hiệu lực chính thức, dùng để cross-check | Là căn cứ pháp lý mạnh nhất khi có tranh chấp |
| `quochoi.vn` | Văn bản Luật, Nghị quyết Quốc hội | |
| `gdt.gov.vn`, `molisa.gov.vn`, `ipvietnam.gov.vn`... | Thông tư/hướng dẫn chuyên ngành (thuế, lao động, SHTT) | Mỗi ngành một cổng, cần scraper riêng hoặc theo dõi RSS/thông báo mới nếu có |
| `thuvienphapluat.vn` (không phải chính thống nhưng dữ liệu có cấu trúc tốt) | Tham khảo chéo, có thể mua API/data thương mại | Nhiều legal-tech VN dùng làm nguồn thực tế vì tagging hiệu lực/quan hệ văn bản tốt hơn site chính phủ; vẫn nên verify lại qua `vbpl.vn`/Công báo trước khi publish |

**Khuyến nghị MVP:** đừng tự xây crawler cho tất cả cổng cùng lúc. Bắt đầu với
`vbpl.vn` làm nguồn chính (chính thống, đủ cấu trúc để lấy "tình trạng hiệu
lực" + danh sách văn bản liên quan), và cân nhắc mua data/API từ
`thuvienphapluat.vn` để rút ngắn thời gian xây dựng pipeline — scraping ở quy
mô production có rủi ro pháp lý/ToS cần review riêng.

### 2.2 Vòng đời văn bản phải model được, không chỉ "văn bản → đoạn text"

```
Văn bản (Luật/Nghị định/Thông tư)
 ├─ effective_from, effective_to, status (còn hiệu lực / hết hiệu lực / chưa hiệu lực)
 ├─ amends / amended_by / replaces / replaced_by  (quan hệ cấp văn bản)
 └─ Điều
     ├─ effective_from, effective_to  (một Điều có thể hết hiệu lực trước cả văn bản)
     ├─ amended_by (Điều nào trong văn bản khác sửa Điều này)
     └─ nội dung (có thể có nhiều version theo thời gian)
```

Không lưu "một bản text mới nhất đè lên bản cũ" — lưu **lịch sử version theo
khoảng thời gian hiệu lực** (temporal table), để:

- Trả lời đúng câu hỏi "theo luật tại thời điểm X" (hợp đồng ký 2022, tranh
  chấp xảy ra 2024 — áp dụng luật nào tại thời điểm ký hay thời điểm xử lý?).
- Có audit trail: với bất kỳ câu trả lời nào trong quá khứ, biết chính xác hệ
  thống đã dùng version văn bản nào → quan trọng nếu khách hàng khiếu nại vì
  làm theo tư vấn sai.

### 2.3 Pipeline đồng bộ (không phải nạp một lần như hiện tại)

Sơ đồ đầy đủ (kèm nhánh reject, cache invalidation): xem
[`architecture.md` §4](architecture.md#4-flow-đồng-bộ--xuất-bản-văn-bản-pháp-luật).

```
[Scheduled crawler] → [Change detection / diff] → [Staging corpus]
        │                                                 │
        │                                   [Human review − luật sư/chuyên gia duyệt]
        │                                                 │
        └──────────────────────────────→ [Publish: incremental re-embed + update graph quan hệ]
```

- Crawl định kỳ (hàng ngày/hàng tuần tuỳ nguồn) → chỉ re-embed phần **thay
  đổi**, không rebuild toàn bộ index mỗi lần (đã làm một phần việc này ở
  backend hiện tại qua manifest hash, cần mở rộng thành incremental thật).
- **Bắt buộc có bước duyệt thủ công** trước khi văn bản mới/sửa đổi lên
  production — đây là domain có thể gây thiệt hại tài chính thực nếu sai, nên
  "human-in-the-loop" là yêu cầu sản phẩm, không phải tuỳ chọn.
- Lưu snapshot gốc (HTML/PDF) vào object storage (S3) cho mọi lần crawl — làm
  bằng chứng/audit, và để backfill nếu parser có bug.
- **Agent xác minh real-time** (tuỳ chọn, cho câu hỏi nhạy cảm): khi câu trả
  lời phụ thuộc nặng vào một văn bản cụ thể, agent có thể gọi tool fetch trực
  tiếp trang "tình trạng hiệu lực" của văn bản đó trên `vbpl.vn` tại thời điểm
  trả lời, làm lớp double-check ngoài batch sync — tốn thêm latency nên chỉ
  bật cho câu hỏi high-stakes hoặc khi confidence thấp.
- Pipeline này cũng là nơi phát sinh sự kiện cho tính năng **#5 Cảnh báo thay
  đổi luật** (mục 9) — publish xong thì đối chiếu `citation_log`/watchlist để
  biết user nào bị ảnh hưởng.

## 3. Agentic RAG — có nên dùng không, và dùng ở đâu

**Có**, nhưng không phải vì "agentic" là trend — vì bài toán thực sự cần nhiều
bước suy luận mà single-shot RAG không làm được:

- Câu hỏi có thể chạm **nhiều lĩnh vực luật cùng lúc** (thuế + lao động +
  doanh nghiệp) — cần route/fan-out sang nhiều domain rồi hợp nhất.
- Cần **kiểm tra hiệu lực theo thời gian** trước khi coi một đoạn trích là
  đáng tin — một bước suy luận riêng, không phải retrieval thuần.
- Cần **tự phê bình (self-critique)** trước khi trả lời — đối chiếu câu trả
  lời với đoạn trích gốc để chặn hallucination, đây là lý do chính đáng nhất
  để có thêm một bước "agent" sau khi generate.

Backend hiện tại đã có graph LangGraph 7 node (`analyze_intent → rewrite/HyDE
→ retrieve hybrid → rerank → llm_filter → generate → format`). Mở rộng thành
agentic RAG = thêm các node sau, không phải viết lại từ đầu. Sơ đồ flowchart
đầy đủ + ví dụ sequence diagram minh hoạ câu hỏi đa domain: xem
[`architecture.md` §2–3](architecture.md#2-flow-câu-hỏi-pháp-lý-qua-agentic-rag).

```
analyze_intent
   │
   ├─ single-domain ──────────────────────────┐
   └─ multi-domain → fan-out theo domain ──────┤
                                                ▼
                            prepare_retrieval_query (rewrite/HyDE)
                                                ▼
                               retrieve (hybrid: vector + BM25)
                                                ▼
                                         rerank + llm_filter
                                                ▼
                        ★ validity_check  — lọc/đánh dấu Điều đã hết hiệu lực,
                                             xếp theo thời điểm áp dụng câu hỏi
                                                ▼
                               (nếu multi-domain) merge kết quả các nhánh
                                                ▼
                                         generate_answer
                                            (buộc output có citation structured,
                                             không phải text tự do)
                                                ▼
                        ★ verify_groundedness — đối chiếu từng claim/citation
                                             với đoạn trích gốc, confidence score
                                                ▼
                   confidence thấp hoặc văn bản đang tranh chấp hiệu lực?
                      → trả lời kèm cảnh báo / đề nghị tư vấn luật sư
                      → ngược lại trả lời bình thường kèm nguồn + ngày hiệu lực
```

Hai node đánh dấu ★ (`validity_check`, `verify_groundedness`) là phần **mới**,
chính là lớp tạo ra sự khác biệt so với chatbot thông thường. Đừng xây agent
tự trị kiểu "swarm" nhiều agent độc lập tự quyết định gọi nhau — với domain
pháp lý, graph tường minh (LangGraph) dễ audit/debug/kiểm soát chi phí hơn
nhiều so với agent framework tự do, và team chấm/khách hàng cần giải thích
được hệ thống suy luận thế nào.

Tính năng **#6 Case builder** (mục 9) thêm một node `clarify` trước
`prepare_retrieval_query` — chi tiết trong `FEATURES.md`.

## 4. Chống hallucination — nguyên tắc bắt buộc

1. **Ép citation có cấu trúc** qua structured output/tool-calling (schema:
   `{dieu, van_ban, effective_status, effective_date, quote}`), không để LLM
   tự do chèn "Điều X" vào văn bản tự nhiên — dễ hallucinate số Điều.
2. **Verifier pass**: so khớp từng citation trong câu trả lời với đoạn trích đã
   retrieve (NLI-style hoặc LLM-as-judge chuyên biệt) — citation nào không
   khớp bị loại trước khi trả về.
3. **Ngưỡng tin cậy rõ ràng**: retrieval score thấp hoặc verifier fail → trả
   lời "không đủ căn cứ để khẳng định, nên tham vấn luật sư/kế toán", không
   đoán cho đủ câu.
4. **Luôn hiển thị ngày hiệu lực + trạng thái của Điều được trích**, kể cả khi
   câu trả lời đúng — để người dùng tự đánh giá độ tin cậy thay vì tin mù
   quáng vào AI.

## 5. Mô hình dữ liệu gợi ý (rút gọn)

Sơ đồ ER đầy đủ (core legal + mở rộng tổ chức/cộng tác): xem
[`architecture.md` §5](architecture.md#5-mô-hình-dữ-liệu-vòng-đời-văn-bản).

```
legal_document(id, so_hieu, loai_van_ban, ngay_ban_hanh, effective_from,
               effective_to, status, source_url, source_snapshot_s3_key)

legal_document_relation(from_document_id, to_document_id,
                         relation_type: amends|replaces|guides|repeals)

legal_article(id, document_id, dieu_so, parent_article_id,
              effective_from, effective_to, status)

legal_article_version(article_id, content, embedded_at, embedding_model,
                       valid_from, valid_to)

-- business/app layer — mô hình tổ chức/vai trò đầy đủ ở mục 6
organization(id, name, tax_code, industry, ...)
organization_member(id, organization_id, user_id, org_role, department, status)
conversation / message / citation_log(message_id, article_version_id, verifier_score)
```

`citation_log` là bảng quan trọng cho audit: mọi câu trả lời lưu lại chính xác
`article_version_id` đã dùng, không chỉ số Điều — để trả lời được câu "hệ
thống đã dựa vào bản luật nào khi tư vấn việc này". `citation_log` cũng là nền
cho tính năng #5 (cảnh báo thay đổi luật, mục 9).

## 6. Mô hình vai trò & phân quyền

Role hiện tại trong code (`UserRole`: `owner`, `accountant`, `hr`, `admin`)
đúng là **quá hẹp** nếu nhìn sang sản phẩm thật — và việc liệt kê thêm role
theo từng chức danh (pháp chế, mua hàng, kinh doanh, vận hành...) không phải
cách giải đúng, vì nó mắc 2 vấn đề:

1. **Cứng theo chức danh, không scale.** Mỗi khi có khách hàng với chức danh
   mới, lại thêm 1 giá trị enum + rải logic `if role == X or role == Y...`
   khắp backend — rối dần và dễ sót quyền.
2. **Lẫn 2 khái niệm khác nhau vào 1 field.** `UserRole.ADMIN` hiện tại vừa
   dùng cho "quản trị toàn hệ thống" (sửa corpus, xem mọi user — xem
   `backend/src/routers/admin.py`) vừa ngầm định là vai trò trong 1 tổ chức.
   Hai thứ này cần tách: *system admin* (vận hành nền tảng, hiếm, nội bộ công
   ty bạn) khác hẳn với *admin của 1 doanh nghiệp khách hàng* (quản lý thành
   viên công ty họ).

### 6.1 Nguyên tắc: tách "quyền truy cập" khỏi "chức danh"

- **`OrgRole`** (nhỏ, ổn định, quyết định *được làm gì*): `OWNER`, `ADMIN`,
  `MEMBER`, `VIEWER`. Đây là 4 *tier* quyền, không phải chức danh — không
  tăng theo số phòng ban/vị trí công việc.
- **`Department`** (mở, chỉ để hiển thị/lọc/route task — **không gate API**):
  `executive` (ban giám đốc), `accounting`, `hr`, `legal`, `procurement`,
  `sales`, `operations`, `other`. Doanh nghiệp có vị trí gì cứ gắn tag, không
  ảnh hưởng logic phân quyền ở bất kỳ đâu — khác hẳn thêm 1 `OrgRole` mới.
- **`is_system_admin`** (bool trên `User`, tách khỏi `OrgRole`): quản trị nền
  tảng (corpus, toàn bộ user hệ thống) — độc lập hoàn toàn với vai trò trong
  bất kỳ tổ chức khách hàng nào.
- Một `User` có thể thuộc **nhiều `Organization`** với `OrgRole` khác nhau ở
  mỗi nơi (bảng nối N-N thay vì `organization_id` 1-N trên `User` như hiện
  tại) — cần cho tính năng **#12 API đối tác** (kế toán dịch vụ phục vụ nhiều
  khách hàng) và hợp lý hơn về lâu dài (một người có thể vừa là owner công ty
  mình vừa được mời làm viewer ở công ty đối tác).

### 6.2 Ma trận quyền theo module

| Module | OWNER | ADMIN | MEMBER | VIEWER |
| --- | --- | --- | --- | --- |
| Chat hỏi đáp pháp lý | Full | Full | Full | Full (chỉ đọc/hỏi, không sửa dữ liệu nên không cần hạn chế) |
| Tra cứu văn bản | Full | Full | Full | Full |
| Soát xét hợp đồng | Full | Full | Upload + review | Chỉ xem kết quả |
| Lịch tuân thủ | Full | Full | Tạo/sửa/hoàn thành task | Chỉ xem |
| Mời/quản lý thành viên | Full | Mời + đổi role (trừ owner) | — | — |
| Xoá tổ chức / billing | Full | — | — | — |
| Quản trị corpus / hệ thống | *(ngoài phạm vi `OrgRole`, xem `is_system_admin`)* | | | |

### 6.3 Thay đổi data model so với hiện tại

```
-- Hiện tại (backend/src/models/user.py)
User(id, email, ..., role: UserRole, organization_id)   -- 1 user = 1 org, role lẫn system+org

-- Đề xuất
User(id, email, ..., is_system_admin: bool)
Organization(id, name, tax_code, ...)                     -- giữ nguyên, vẫn là hồ sơ DN cho compliance
OrganizationMember(id, organization_id, user_id,
                    org_role: OWNER|ADMIN|MEMBER|VIEWER,
                    department: executive|accounting|hr|legal|procurement|sales|operations|other,
                    status: active|invited|removed,
                    invited_by, joined_at)
```

Chi tiết migration + API cho flow mời thành viên: tính năng **#1** ở mục 9 và
[`docs/FEATURES.md`](FEATURES.md#1-mời-thành-viên-vào-tổ-chức).

## 7. Tech stack đề xuất

Giữ đề xuất của bạn (Next.js / FastAPI / Postgres) làm xương sống, bổ sung các
mảnh còn thiếu cho một sản phẩm scale-up:

| Lớp | MVP | Khi scale | Ghi chú |
| --- | --- | --- | --- |
| Frontend | Next.js (App Router) + TS + Tailwind + shadcn/ui, streaming qua SSE/Vercel AI SDK | Giữ nguyên, thêm i18n nếu mở rộng ngoài VN | Đổi từ Vite SPA hiện tại sang Next.js hợp lý nếu cần SSR cho SEO (tra cứu văn bản public nên index được trên Google) |
| Backend | FastAPI + LangGraph (đã có) | Thêm module routing multi-domain, validity/verifier node | Giữ abstraction `LLMClient` hiện tại để đổi provider không sửa code |
| DB nghiệp vụ | PostgreSQL (đã có) | RDS Multi-AZ, thêm bảng lifecycle ở mục 5 | |
| Vector store | Cân nhắc **chuyển sang `pgvector`** trong chính Postgres thay vì Chroma rời | pgvector + partitioning, hoặc Qdrant managed nếu cần filter phức tạp ở scale lớn | Gộp vector + full-text (đã dùng `tsvector`) trong 1 DB giảm số service vận hành, backup/HA đơn giản hơn trên RDS. Giữ Chroma nếu muốn tránh churn ở MVP — không bắt buộc đổi ngay |
| Lexical search | Postgres `tsvector` + `pg_trgm` (đã có) | OpenSearch nếu cần phân tích tiếng Việt nâng cao ở corpus rất lớn | |
| LLM | Model routing: model rẻ (Gemini Flash) cho rewrite/classify, model mạnh hơn (Claude/GPT-class) cho generate + verifier | Giữ nguyên, thêm fallback provider | Bước verifier nên dùng model khác/tách prompt với model generate để giảm thiên lệch tự-xác-nhận |
| Queue/worker (crawler, embedding) | APScheduler/cron + 1 worker container | SQS + ECS worker, hoặc Step Functions cho crawl pipeline nhiều bước | |
| Cache | Redis (session, rate limit, hot query) | ElastiCache | |
| Auth & phân quyền | JWT hiện tại + `OrganizationMember` (mục 6) | Audit log mọi thay đổi role/member | Không hard-code role theo chức danh — xem mục 6 |
| Observability | Langfuse (self-host) cho LLM/agent trace, Sentry cho lỗi backend/frontend | Thêm OpenTelemetry Collector → CloudWatch cho hạ tầng (ECS/RDS), giữ Langfuse cho tầng LLM | Chi tiết ở mục 11 |
| Eval | `eval/` + RAGAS trên tập golden set | CI chặn merge khi RAGAS/F2-score/groundedness regression | Chi tiết ở mục 10 |

## 8. MVP vs V1/V2 — phạm vi theo giai đoạn

### MVP (ưu tiên launch nhanh, nhưng không đánh đổi an toàn trích dẫn)

- 3–4 domain pháp lý đã có data: Doanh nghiệp, Thuế, Lao động, Hợp đồng.
- Agent graph hiện tại + **2 node mới bắt buộc**: `validity_check` (chỉ cần
  status hiệu lực + ngày, chưa cần multi-version phức tạp) và
  `verify_groundedness` (citation có cấu trúc + verifier pass đơn giản).
- Crawler cho **một nguồn** (`vbpl.vn`) chạy định kỳ, có review thủ công
  trước khi publish — chưa cần real-time check agent.
- Multi-tenant theo mô hình `OrgRole` ở mục 6 (`OWNER/ADMIN/MEMBER/VIEWER` +
  `department` mở) — không giới hạn theo 3 chức danh cố định, chưa cần
  billing/plan.
- Tính năng Tier 1 ở mục 9 (#1–#4): mời thành viên, liên kết hợp đồng↔tuân
  thủ, feedback câu trả lời, nhắc deadline qua email.
- Giữ nguyên các tính năng đã xây: tra cứu văn bản, soát xét hợp đồng, lịch
  tuân thủ.
- Eval set nhỏ (50–100 câu có đáp án đã biết) chạy trong CI trước mỗi lần đổi
  prompt/model.

### V1 — sau khi có người dùng thật

- Thêm nguồn crawl (Công báo, thông tư chuyên ngành theo domain khách hàng
  yêu cầu nhiều nhất).
- Real-time verification agent cho câu hỏi high-stakes.
- Multi-version theo thời gian đầy đủ (trả lời được "tại thời điểm X").
- Tính năng Tier 2 ở mục 9 (#5–#8): cảnh báo thay đổi luật, case builder,
  máy tính nghĩa vụ, dashboard rủi ro.
- Audit log đầy đủ cho mọi thao tác trên corpus và trên `OrganizationMember`.
- Model routing + cost tracking theo tenant.

### V2 — scale

- Tách embedding/crawler pipeline thành service riêng, autoscale độc lập với
  API backend.
- Tính năng Tier 3 ở mục 9 (#9–#12): escalation chuyên gia thật, Zalo OA, so
  sánh phương án, API đối tác B2B2C.
- Cân nhắc fine-tune/prompt-cache theo domain nếu volume đủ lớn để tối ưu chi
  phí LLM.
- Mở rộng ngoài 4 domain ban đầu theo nhu cầu khách hàng.

## 9. Tính năng mở rộng — 12 ý tưởng giá trị

Flow chi tiết từng bước, API contract (request/response), và thay đổi data
model cho cả 12 tính năng: xem **[`docs/FEATURES.md`](FEATURES.md)**. Dưới
đây là tóm tắt giá trị + độ ưu tiên; `OrgRole` tối thiểu ghi theo mục 6.

### Tier 1 — build trên nền đã có, effort thấp, giá trị cao ngay

1. **Mời thành viên vào tổ chức** (`OWNER`/`ADMIN`) — hoàn thiện mô hình vai
   trò ở mục 6: mỗi tổ chức thực sự nhiều người dùng chung, không còn 1
   user = 1 org như hiện tại.
2. **Liên kết Hợp đồng ↔ Lịch tuân thủ** (`MEMBER`+) — soát hợp đồng xong, đề
   xuất tạo compliance task từ nghĩa vụ phát sinh trong hợp đồng (ngày thanh
   toán, gia hạn...). Hai service đã có sẵn, chỉ thiếu cầu nối.
3. **Feedback trên câu trả lời** (mọi role) — 👍/👎 + lý do, vừa là UX vừa nuôi
   golden set cho RAGAS (mục 10) bằng dữ liệu production thật.
4. **Nhắc deadline qua email** (`OWNER`/`ADMIN` cấu hình) — compliance task
   sắp đến hạn hiện chỉ nằm trong app, chủ DN không mở app thì không biết.

### Tier 2 — giá trị cao, cần thêm hạ tầng/thiết kế

5. **Cảnh báo thay đổi luật liên quan** (mọi role, auto theo `citation_log`)
   — tính năng khác biệt hoá rõ nhất, phụ thuộc trực tiếp pipeline đồng bộ
   (mục 2.3) + `citation_log` (mục 5).
6. **Case builder — agent hỏi lại khi thiếu ngữ cảnh** — thêm node `clarify`
   vào agent graph (mục 3), giảm rủi ro tư vấn sai do câu hỏi mơ hồ.
7. **Máy tính nghĩa vụ** (thuế TNCN, BHXH, phạt chậm nộp...) — rule engine xác
   định (không dùng LLM tính số), nâng cấp từ "nhắc lịch" lên "tính con số",
   kèm căn cứ Điều.
8. **Dashboard rủi ro tổng thể** (`OWNER`/`ADMIN`, `MEMBER` xem) — gộp tín
   hiệu từ 3 module thành 1 màn hình quản trị rủi ro pháp lý cho chủ DN.

### Tier 3 — mở rộng kênh/mô hình kinh doanh

9. **Escalation sang chuyên gia thật** (mọi role yêu cầu, `ADMIN` xử lý hàng
   đợi) — điểm nối giữa AI tư vấn sơ bộ và dịch vụ trả phí.
10. **Zalo OA integration** — kênh tiếp cận SME Việt Nam thực tế hơn web
    riêng; rủi ro chủ yếu ở phê duyệt/ToS Zalo (ngoài phạm vi kỹ thuật).
11. **So sánh phương án dạng bảng** — mở rộng format câu trả lời (không phải
    entity mới), phù hợp câu hỏi ra quyết định ("hộ kinh doanh hay TNHH").
12. **API cho đối tác (B2B2C)** — dịch vụ kế toán/luật outsource quản lý
    nhiều `Organization` khách hàng qua API, tận dụng trực tiếp mô hình
    multi-org của `OrganizationMember` ở mục 6.

## 10. Eval & quality gate

Đúng là bản trước **chưa có bước evaluation cụ thể** — mục 8 cũ chỉ nói chung
chung "đo F2/groundedness", chưa chọn framework, chưa có metric cụ thể, chưa
nói tool nào chạy. Bổ sung lại đầy đủ ở đây, dùng **RAGAS** làm framework
chính vì nó đã chuẩn hoá bộ metric RAG phổ biến nhất và tích hợp sẵn với
LangChain/LangGraph (đang dùng) lẫn Langfuse (mục 11).

### 10.1 Hai lớp eval, không trộn lẫn

| Lớp | Chạy khi nào | Mục đích |
| --- | --- | --- |
| **Online verifier** (`verify_groundedness` ở mục 3) | Mỗi request thật, real-time | Chặn hallucination trước khi trả lời — phải nhanh, rẻ, không thể dùng full RAGAS vì tốn thêm LLM call làm tăng latency |
| **Offline eval** (RAGAS + F2-score, mục này) | CI/CD, theo lịch (nightly), hoặc thủ công trước khi đổi prompt/model | Đo chất lượng hệ thống trên diện rộng, phát hiện regression trước khi ảnh hưởng người dùng thật |

### 10.2 Bộ metric RAGAS áp dụng cho domain pháp lý

RAGAS (`ragas` — Python, chạy được với LangChain/LangGraph trace trực tiếp)
cho 2 nhóm metric, dùng LLM-as-judge + embeddings để chấm, không cần nhãn
người tuyệt đối cho mọi câu (trừ `answer_correctness`/`context_recall` cần câu
trả lời/ngữ cảnh tham chiếu):

| Metric RAGAS | Đo gì | Map sang yêu cầu sản phẩm |
| --- | --- | --- |
| `faithfulness` | Claim trong câu trả lời có được context hỗ trợ không | Bản tự động hoá của nguyên tắc chống hallucination ở mục 4 — chạy offline để có con số theo dõi xu hướng, khác với verifier online |
| `answer_relevancy` | Câu trả lời có đúng trọng tâm câu hỏi không | Chất lượng QA tổng quát |
| `context_precision` | Trong các đoạn retrieve, bao nhiêu % thực sự liên quan (và xếp hạng đúng vị trí) | Trục Precision — bổ sung cho F2-score (F2 ưu tiên đúng số Điều, context_precision đánh giá cả đoạn văn bản) |
| `context_recall` | Context retrieve có phủ hết thông tin cần để trả lời đúng không (so với ground-truth answer) | Trục Recall — cần tập có đáp án mẫu, không áp dụng được cho toàn bộ 2.000 câu thi cũ (chỉ có đáp án `Điều X`, chưa có câu trả lời mẫu) |
| `answer_correctness` | So khớp ngữ nghĩa + factual giữa answer và ground-truth answer | Gần với trục "QA Quality" (LLM-as-Judge 5 tiêu chí) trong tài liệu thi cũ — cần xây/giữ lại tập có đáp án mẫu được chuyên gia duyệt |

**Giới hạn cần biết:** RAGAS không hiểu khái niệm "Điều X của Luật Y" hay
"hiệu lực pháp lý" — nó chỉ chấm theo ngữ nghĩa chung. Vì vậy **không thay
thế** F2-score trích dẫn (vẫn là ground truth cứng, exact-match theo số Điều)
mà dùng **song song**:

- F2-score trích dẫn → vẫn là metric chính cho retrieval (tái dùng logic
  chấm của bài thi cũ, đã có sẵn).
- RAGAS (`faithfulness`, `context_precision/recall`, `answer_relevancy`,
  `answer_correctness`) → tín hiệu chất lượng ngữ nghĩa rộng hơn, chấm được cả
  khi câu hỏi không có đáp án `Điều X` rõ ràng (câu hỏi tình huống, tư vấn mở).

### 10.3 Metric tự xây — RAGAS không có sẵn, nhưng bắt buộc cho domain này

- **`stale_citation_rate`**: % câu trả lời trích Điều đã hết hiệu lực mà không
  cảnh báo — test bằng case cố ý hỏi về văn bản đã biết hết hiệu lực
  (`validity_check` ở mục 3 phải bắt được).
- **`temporal_accuracy`**: với câu hỏi có mốc thời gian cụ thể ("tại thời điểm
  ký hợp đồng 2022"), kiểm tra hệ thống chọn đúng version Điều đang hiệu lực
  tại thời điểm đó, không phải version hiện tại.
- **`citation_format_validity`**: % câu trả lời có citation đúng schema cấu
  trúc (mục 4) — việc này gần như phải đạt 100% vì ép bằng structured output,
  nên dùng làm smoke check hơn là metric chất lượng.

### 10.4 Vận hành

- Golden set: 100–200 câu ban đầu (tái dùng một phần câu hỏi thi R2AI đã có
  đáp án `Điều X`), **bổ sung thêm câu trả lời mẫu** do chuyên gia pháp lý
  duyệt cho nhóm metric cần ground-truth (`context_recall`,
  `answer_correctness`) — tập thi cũ chỉ đủ cho F2-score, chưa đủ cho RAGAS
  đầy đủ.
- Chạy `ragas.evaluate()` trong CI (xem mục 12) trên golden set, ghi kết quả
  có thể đẩy thẳng lên Langfuse (có integration sẵn: Langfuse hiển thị RAGAS
  score theo từng trace/run, so sánh giữa các lần chạy).
- Ngưỡng chặn merge: chốt baseline từ lần chạy đầu, không cho giảm quá X%
  (ví dụ 3–5 điểm phần trăm) ở `faithfulness` và F2-score — đây là hai metric
  "an toàn", ưu tiên chặn cứng hơn các metric còn lại.
- Review mẫu thủ công định kỳ (hàng tuần) trên một tập nhỏ câu trả lời thật từ
  production — LLM-judge (kể cả RAGAS) vẫn có thể sai, cần người có chuyên môn
  pháp lý kiểm tra chéo, nhất là trước khi tăng ngưỡng tự động hoá.

## 11. Observability & Tracing cho agent

Bản trước chỉ nhắc "Langfuse/LangSmith" lướt qua trong bảng tech stack —
dưới đây là so sánh và lựa chọn cụ thể. Sơ đồ flow trace: xem
[`architecture.md` §8](architecture.md#8-flow-observability--trace-đi-đâu).

### 11.1 So sánh lựa chọn

| Công cụ | Open-source / self-host | Tích hợp LangGraph | Điểm mạnh | Điểm cần cân nhắc |
| --- | --- | --- | --- | --- |
| **Langfuse** | Có (MIT phần core), self-host bằng Docker | Có callback handler chính thức cho LangChain/LangGraph | Self-host được (quan trọng vì dữ liệu hỏi-đáp pháp lý/doanh nghiệp nhạy cảm, PDPL), có tích hợp RAGAS sẵn, theo dõi chi phí theo tenant/user, quản lý version prompt | Tính năng enterprise (SSO, RBAC nâng cao) trả phí nếu dùng Langfuse Cloud |
| **LangSmith** | Không (SaaS, self-host chỉ ở gói Enterprise đắt) | Tích hợp sâu nhất vì cùng đội LangChain làm | UI mạnh, eval framework tốt, debug LangGraph trực quan nhất | Dữ liệu mặc định đi qua server LangChain (US) — cần cân nhắc với dữ liệu pháp lý/doanh nghiệp nhạy cảm của khách Việt Nam |
| **Arize Phoenix** | Có, OpenTelemetry-native | Qua OTel instrumentation, không có callback riêng cho LangGraph | Mạnh về phân tích embedding/retrieval (cluster câu hỏi, phát hiện vùng retrieval yếu) | Hệ sinh thái eval/UI non hơn Langfuse cho use case "theo dõi câu trả lời + chi phí" |
| **OpenLLMetry / Traceloop** | Có (thư viện OTel instrumentation) | Instrument một lần, export sang bất kỳ backend OTel nào (kể cả Langfuse/Phoenix) | Tránh vendor lock-in, chuẩn OTel dùng chung với tracing hạ tầng | Không phải nền tảng hiển thị — cần backend khác để xem (dùng kèm, không thay thế) |

### 11.2 Khuyến nghị

**Langfuse self-host**, lý do cụ thể cho dự án này:

1. **Data residency**: dữ liệu trace chứa câu hỏi pháp lý/thông tin doanh
   nghiệp của khách — self-host trong VPC AWS cùng hạ tầng hiện tại (mục 13)
   tránh gửi dữ liệu nhạy cảm ra SaaS bên thứ ba, hợp với lo ngại PDPL đã nêu
   ở mục 14.
2. **Tích hợp sẵn với RAGAS** (mục 10) — chạy eval xong đẩy score vào
   Langfuse, xem trend theo thời gian, so sánh giữa các version prompt ngay
   trên cùng dashboard thay vì ghép hai hệ thống rời.
3. **Theo dõi chi phí theo tenant** — cần thiết một khi có multi-tenant (mục
   6, 8) và model routing nhiều provider (mục 7), Langfuse group được
   cost/latency theo user/session/tenant.
4. Chi phí vận hành thấp hơn LangSmith Enterprise ở giai đoạn MVP/V1 khi chưa
   có doanh thu ổn định.

Triển khai: thêm Langfuse (Postgres + ClickHouse backing, có thể chạy chung
docker-compose lúc dev, tách service ECS riêng lúc deploy) + Langfuse SDK gắn
callback vào LangGraph tại điểm khởi tạo agent — trace tự động từng node
(`analyze_intent`, `retrieve`, `rerank`, `validity_check`,
`verify_groundedness`, `generate_answer`...), kèm input/output, latency, token
cost mỗi bước mà không cần sửa logic agent.

Nếu sau này cần tránh lock-in, có thể thêm lớp **OpenLLMetry** instrument một
lần rồi export song song cả Langfuse lẫn CloudWatch/Datadog — nhưng với quy mô
MVP/V1, tích hợp Langfuse SDK trực tiếp là đủ, không cần thêm lớp trừu tượng
OTel ngay.

## 12. CI/CD

Sơ đồ flowchart đầy đủ: xem [`architecture.md` §7](architecture.md#7-flow-cicd).

```
PR mở
 ├─ lint (ruff, oxlint) + typecheck (tsc)
 ├─ unit test (pytest với Postgres service container, vitest)
 ├─ eval regression gate (mục 10: F2-score + RAGAS trên golden set, kết quả
 │  đẩy lên Langfuse) — chỉ chạy khi đổi prompt/retrieval/model config, fail
 │  nếu faithfulness hoặc F2-score giảm quá ngưỡng đã chốt
 └─ build Docker image (backend, frontend) → push ECR (tag theo commit SHA)

Merge vào main
 ├─ deploy staging tự động (ECS service update, Terraform-managed)
 ├─ smoke test staging (dùng lại backend/scripts/smoke_*.py hiện có)
 └─ deploy production: manual approval gate (GitHub Environments) → rolling/blue-green update ECS

Pipeline riêng: cập nhật corpus
 ├─ scheduled crawl job
 ├─ diff + staging corpus
 ├─ manual review (luật sư/chuyên gia duyệt qua UI admin)
 └─ publish → incremental re-embed + invalidate cache liên quan
```

Tool: GitHub Actions (đã dùng GitHub). IaC: Terraform cho VPC/ECS/RDS/S3/ALB —
dễ review bằng PR, phù hợp team nhỏ hơn CDK/CloudFormation thuần.

## 13. Kiến trúc triển khai AWS

Sơ đồ chi tiết (có subnet/VPC, luồng trace sang Langfuse): xem
[`architecture.md` §6](architecture.md#6-kiến-trúc-triển-khai-aws).

```
                         CloudFront (CDN)
                               │
        ┌──────────────────────┴───────────────────────┐
        │                                                │
  Next.js (Vercel cho MVP, hoặc ECS Fargate+ALB sau)   S3 (static assets,
        │                                                 document snapshots,
        │ REST/SSE                                        uploaded contracts)
        ▼
      ALB
        │
  ECS Fargate (FastAPI backend, autoscale theo CPU/queue depth)
        │
   ┌────┼─────────────┬───────────────┬────────────────┐
   ▼    ▼              ▼               ▼                ▼
 RDS   ElastiCache   SQS (crawl/embed  Secrets Manager   Langfuse (ECS,
Postgres  Redis       job queue) →      (API keys, DB     self-host, mục 11)
(+pgvector,           ECS worker        creds)            — nhận trace từ
 Multi-AZ)            service (crawler,                    backend qua SDK
                       re-embed job)
```

- Region: `ap-southeast-1` (Singapore) — gần Việt Nam nhất trong AWS, cân nhắc
  yêu cầu lưu trữ dữ liệu nếu khách hàng là tổ chức nhà nước (PDPL — Nghị định
  13/2023 về bảo vệ dữ liệu cá nhân cần review riêng với luật sư, ngoài phạm
  vi tài liệu kỹ thuật này).
- WAF trên ALB/CloudFront — domain tài chính/pháp lý nên có baseline chống
  bot/scan ngay từ đầu.
- Backup: RDS automated snapshot + PITR, S3 versioning cho snapshot văn bản
  gốc (dùng làm bằng chứng nếu có tranh chấp).
- MVP có thể chạy single ECS service + RDS single-AZ để tiết kiệm chi phí,
  bật Multi-AZ/autoscale khi có traffic thật — đừng over-provision trước khi
  có người dùng.

## 14. Rủi ro cần xử lý ngoài phạm vi kỹ thuật

- **Pháp lý của việc scraping** các cổng thông tin chính phủ — cần xác nhận
  điều khoản sử dụng, cân nhắc nguồn dữ liệu thương mại (API trả phí) để giảm
  rủi ro thay vì tự scrape ở quy mô lớn.
- **Trách nhiệm pháp lý của "tư vấn sơ bộ"** — cần ToS/disclaimer rõ ràng
  (không thay thế luật sư), có thể cần ý kiến pháp lý trước khi launch công
  khai cho khách hàng trả phí. Liên quan trực tiếp tính năng #9 (escalation)
  — cần làm rõ ranh giới trách nhiệm giữa AI và chuyên gia thật trả lời.
- **PDPL (Nghị định 13/2023)** nếu lưu dữ liệu cá nhân/doanh nghiệp của khách
  hàng — review riêng, không nằm trong phạm vi tài liệu kiến trúc này. Áp
  dụng cả với dữ liệu trace Langfuse và dữ liệu chia sẻ qua API đối tác (#12).
- **Chính sách Zalo OA** (#10) — cần tuân thủ điều khoản gửi tin nhắn broadcast
  của Zalo, tránh bị khoá OA vì spam/không đúng mục đích đăng ký.

## 15. Thứ tự làm gợi ý (không phải lịch cố định)

1. Chốt schema vòng đời văn bản (mục 5) + mô hình vai trò `OrganizationMember`
   (mục 6) — cả hai là nền tảng, đổi sau sẽ tốn kém.
2. Xây crawler + staging review cho `vbpl.vn` (một nguồn trước), tích hợp vào
   pipeline nạp corpus hiện có.
3. Gắn Langfuse SDK vào LangGraph (mục 11) **trước khi** thêm node mới — có
   trace ngay từ đầu giúp debug 2 node mới ở bước 4 dễ hơn nhiều.
4. Thêm 2 node `validity_check` + `verify_groundedness` vào agent graph hiện
   tại — tái dùng tối đa service đã có (`services/agents/legal_assistant`).
5. Ép structured citation ở `generate_answer`, cập nhật frontend hiển thị
   trạng thái hiệu lực kèm trích dẫn.
6. Dựng golden set nhỏ, tích hợp RAGAS (mục 10), gắn vào CI làm regression
   gate trước khi mở rộng domain.
7. Triển khai tính năng Tier 1 (mục 9, #1–#4) — trong đó **#1 mời thành viên**
   là bước hiện thực hoá mô hình vai trò mới, nên ưu tiên sớm.
8. Triển khai AWS theo mục 13, bắt đầu cấu hình tối giản, scale dần theo
   traffic thật.
9. Tính năng Tier 2 (#5–#8) sau khi có người dùng thật phản hồi, Tier 3
   (#9–#12) khi cần mở rộng kênh/mô hình kinh doanh.
