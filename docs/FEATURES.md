# Đặc tả 12 tính năng mở rộng — flow, API contract, data model

> Chi tiết hoá mục 9 của [`MVP_PRODUCT_ARCHITECTURE.md`](MVP_PRODUCT_ARCHITECTURE.md).
> Mỗi tính năng: mô tả ngắn → flow từng bước → API contract → thay đổi data
> model. `OrgRole` tham chiếu mô hình vai trò ở
> [`MVP_PRODUCT_ARCHITECTURE.md` mục 6](MVP_PRODUCT_ARCHITECTURE.md#6-mô-hình-vai-trò--phân-quyền)
> (`OWNER`/`ADMIN`/`MEMBER`/`VIEWER`, không phải chức danh cố định).
>
> Sơ đồ Mermaid cho các flow nhiều bước: xem
> [`architecture.md` §9](architecture.md#9-sơ-đồ-flow-cho-tính-năng-mở-rộng).

## Mục lục

**Tier 1** — [#1 Mời thành viên](#1-mời-thành-viên-vào-tổ-chức) ·
[#2 Hợp đồng↔Tuân thủ](#2-liên-kết-hợp-đồng--lịch-tuân-thủ) ·
[#3 Feedback](#3-feedback-trên-câu-trả-lời) ·
[#4 Nhắc email](#4-nhắc-deadline-qua-email)

**Tier 2** — [#5 Cảnh báo luật thay đổi](#5-cảnh-báo-thay-đổi-luật-liên-quan) ·
[#6 Case builder](#6-case-builder--agent-hỏi-lại-khi-thiếu-ngữ-cảnh) ·
[#7 Máy tính nghĩa vụ](#7-máy-tính-nghĩa-vụ-calculators) ·
[#8 Dashboard rủi ro](#8-dashboard-rủi-ro-tổng-thể)

**Tier 3** — [#9 Escalation](#9-escalation-sang-chuyên-gia-thật) ·
[#10 Zalo OA](#10-zalo-oa-integration) ·
[#11 So sánh phương án](#11-so-sánh-phương-án-dạng-bảng) ·
[#12 API đối tác](#12-api-cho-đối-tác-b2b2c)

---

## #1 Mời thành viên vào tổ chức

**Giá trị:** hiện thực hoá mô hình vai trò mới (mục 6) — mỗi tổ chức thực sự
nhiều người dùng chung thay vì 1 user = 1 org như hiện tại.

### Flow

1. `OWNER`/`ADMIN` mở trang "Thành viên" → nhập email + chọn `org_role` +
   `department` → gửi lời mời.
2. Backend tạo `OrganizationInvite` (token ngẫu nhiên, hết hạn 7 ngày), gửi
   email chứa link `/invite/accept?token=...`.
3. Người được mời:
   - Đã có tài khoản → đăng nhập → bấm accept → tạo `OrganizationMember`.
   - Chưa có tài khoản → đăng ký kèm `token` → tự động accept sau khi tạo
     user thành công.
4. `OWNER`/`ADMIN` xem danh sách thành viên + trạng thái (đã tham gia/đang
   chờ), có thể thu hồi lời mời, đổi `org_role`, hoặc remove member (không
   cho remove `OWNER` cuối cùng của tổ chức).

### API contract

```
POST /api/v1/organizations/{org_id}/invites
  Role yêu cầu: OWNER, ADMIN
  Request:  { email: string, org_role: "admin"|"member"|"viewer", department: string }
  Response: 201 { id, email, org_role, department, status: "pending", expires_at }

GET /api/v1/organizations/{org_id}/invites?status=pending
  Role yêu cầu: OWNER, ADMIN
  Response: 200 { items: InviteOut[] }

DELETE /api/v1/organizations/{org_id}/invites/{invite_id}
  Role yêu cầu: OWNER, ADMIN
  Response: 204

POST /api/v1/invites/accept
  Yêu cầu: đã đăng nhập (JWT)
  Request:  { token: string }
  Response: 200 OrganizationMemberOut

GET /api/v1/organizations/{org_id}/members
  Role yêu cầu: mọi thành viên (xem danh sách đồng nghiệp)
  Response: 200 { items: [{ user_id, email, full_name, org_role, department, status, joined_at }] }

PATCH /api/v1/organizations/{org_id}/members/{user_id}
  Role yêu cầu: OWNER (đổi role bất kỳ), ADMIN (không đổi được OWNER)
  Request:  { org_role?: string, department?: string }
  Response: 200 OrganizationMemberOut

DELETE /api/v1/organizations/{org_id}/members/{user_id}
  Role yêu cầu: OWNER, ADMIN (không remove được OWNER cuối cùng)
  Response: 204
```

### Data model

```
OrganizationInvite(id, organization_id, email, org_role, department,
                    token, status: pending|accepted|expired|revoked,
                    invited_by, expires_at, created_at)

OrganizationMember(id, organization_id, user_id, org_role, department,
                    status: active|removed, invited_by, joined_at)
```

Thay thế `User.role` + `User.organization_id` hiện tại (xem mục 6.3 của
`MVP_PRODUCT_ARCHITECTURE.md`) — cần migration dữ liệu: mỗi `User` hiện có
chuyển thành 1 `OrganizationMember` với `org_role` suy ra từ `UserRole` cũ
(`owner→OWNER`, `accountant/hr→MEMBER`, `admin→` giữ `is_system_admin=true`
trên `User`, không map vào `OrgRole`).

---

## #2 Liên kết Hợp đồng ↔ Lịch tuân thủ

**Giá trị:** hai module đã tồn tại độc lập (`contracts`, `compliance`) —
nối chúng biến "soát hợp đồng" từ việc đọc-xong-rồi-thôi thành nguồn tạo
nghĩa vụ theo dõi được.

### Flow

1. Sau khi review hợp đồng xong (`ReviewStatus.DONE`), agent trích các "nghĩa
   vụ phát sinh" từ nội dung + kết quả review (ngày thanh toán, ngày gia hạn,
   ngày hết hạn, nghĩa vụ bảo hành...).
2. Hệ thống hiển thị danh sách đề xuất cho user xác nhận — **không tự tạo
   task mà không duyệt**, tránh sinh task sai/spam.
3. User (`MEMBER`+) xác nhận từng mục → tạo `ComplianceTask` với
   `source_type=contract_extracted`, liên kết `source_document_id`.
4. Compliance task hiển thị badge "từ hợp đồng «tên file»", bấm vào mở lại
   review gốc.

### API contract

```
POST /api/v1/documents/{document_id}/reviews/{review_id}/extract-obligations
  Role yêu cầu: MEMBER+
  Response: 200 { items: [{ title, description, suggested_due_date, amount? }] }
  (đề xuất, chưa lưu DB — idempotent, có thể gọi lại nhiều lần)

POST /api/v1/compliance/tasks/from-document
  Role yêu cầu: MEMBER+
  Request:  { document_id, review_id, obligations: [{ title, due_date, amount? }] }
  Response: 201 { items: ComplianceTaskOut[] }
```

### Data model

```
ComplianceTask: thêm cột
  source_type: rule_generated|contract_extracted|manual   (default rule_generated)
  source_document_id: uuid NULL  (FK -> documents.id)
  source_review_id: uuid NULL    (FK -> document_reviews.id)
```

---

## #3 Feedback trên câu trả lời

**Giá trị:** nguồn dữ liệu thật đầu tiên cho golden set RAGAS (mục 10 kiến
trúc) — hiện không có cách nào thu thập tín hiệu chất lượng từ production.

### Flow

1. Dưới mỗi message `role=assistant`, hiển thị nút 👍/👎.
2. Bấm 👎 → hiện modal chọn nhanh lý do (`trích dẫn sai`, `thiếu thông tin`,
   `không liên quan`, `khác` + free text).
3. Lưu `MessageFeedback`. Không giới hạn 1 feedback/message (user có thể đổi
   ý — `PATCH` ghi đè).
4. Admin/dashboard nội bộ (không phải UI khách hàng) xem tổng hợp câu trả
   lời bị downvote nhiều để ưu tiên review.

### API contract

```
POST /api/v1/conversations/{conversation_id}/messages/{message_id}/feedback
  Role yêu cầu: mọi role (member tự feedback câu trả lời mình nhận)
  Request:  { rating: "up"|"down", reason_code?: string, comment?: string }
  Response: 201 MessageFeedbackOut (hoặc 200 nếu ghi đè feedback cũ)

GET /api/v1/admin/feedback?rating=down&from=...&to=...
  Role yêu cầu: is_system_admin
  Response: 200 { items: [...], aggregate: { down_rate, top_reason_codes } }
```

### Data model

```
MessageFeedback(id, message_id, user_id, rating: up|down,
                 reason_code: string NULL, comment: text NULL,
                 created_at, updated_at)
  UNIQUE(message_id, user_id)
```

---

## #4 Nhắc deadline qua email

**Giá trị:** compliance task sắp đến hạn hiện chỉ nằm trong app — chủ DN
không chủ động mở app thì không biết.

### Flow

1. Worker job chạy hàng ngày (xem `architecture.md` §4 pattern tương tự crawl
   job), quét `ComplianceTask` có `due_date` trong N ngày tới
   (`remind_days_before`, mặc định `[7, 3, 1]`) và `status != done`.
2. Gom theo `organization_id`, gửi 1 email tổng hợp (không spam từng task một
   email riêng) cho các `OrganizationMember` có bật `email_reminders_enabled`.
3. User có thể tắt/chỉnh ngày nhắc trong Cài đặt tổ chức.

### API contract

```
GET /api/v1/organizations/{org_id}/notification-settings
  Role yêu cầu: OWNER, ADMIN
  Response: 200 { email_reminders_enabled: bool, remind_days_before: int[] }

PATCH /api/v1/organizations/{org_id}/notification-settings
  Role yêu cầu: OWNER, ADMIN
  Request:  { email_reminders_enabled?: bool, remind_days_before?: int[] }
  Response: 200 NotificationSettingOut
```

Không có API cho worker job (chạy nội bộ, không expose endpoint).

### Data model

```
NotificationSetting(organization_id PK, email_reminders_enabled bool default true,
                     remind_days_before int[] default '{7,3,1}', updated_at)
```

---

## #5 Cảnh báo thay đổi luật liên quan

**Giá trị:** tính năng khác biệt hoá rõ nhất trong toàn bộ roadmap — không
chatbot RAG thông thường nào làm được vì cần hạ tầng hiệu lực theo thời gian
(mục 2.2, 5 kiến trúc) làm nền.

### Flow

1. Watchlist hình thành 2 cách: **tự động** (mọi `Điều` đã từng xuất hiện
   trong `citation_log` của user được coi như đang "theo dõi ngầm") và
   **chủ động** (user bấm "theo dõi" 1 văn bản/Điều cụ thể ở trang tra cứu).
2. Khi pipeline đồng bộ (mục 2.3, `architecture.md` §4) publish thay đổi cho
   1 Điều, hệ thống tạo `LawChangeEvent`, rồi join với `LawWatch` +
   `citation_log` để tìm user bị ảnh hưởng.
3. Tạo `LawChangeNotification` cho từng user, gửi email tóm tắt + hiển thị
   banner trong app ("3 câu trả lời trước đây liên quan Điều 5 Luật X vừa sửa
   đổi — xem lại").
4. User click vào xem: câu trả lời cũ, đoạn luật mới, diff với bản cũ.

### API contract

```
POST   /api/v1/laws/{law_id}/watch                          -> theo dõi cả văn bản
POST   /api/v1/laws/{law_id}/articles/{article}/watch        -> theo dõi 1 Điều
DELETE /api/v1/laws/{law_id}/articles/{article}/watch
GET    /api/v1/me/watchlist
  Response: 200 { items: [{ law_id, law_name, article?, watched_since }] }

GET /api/v1/me/law-change-notifications?unread=true
  Response: 200 { items: [{ id, law_id, article, change_type, detected_at,
                             related_message_id?, read_at }] }

PATCH /api/v1/me/law-change-notifications/{id}
  Request:  { read: true }
  Response: 200
```

### Data model

```
LawWatch(id, user_id, law_id, article NULL, created_at)

LawChangeEvent(id, law_id, article, change_type: amended|repealed|new_version,
                old_article_version_id, new_article_version_id, detected_at)

LawChangeNotification(id, user_id, law_change_event_id,
                       related_citation_log_id NULL, read_at NULL, created_at)
```

Phụ thuộc trực tiếp `legal_article_version` + `citation_log` (mục 5 kiến
trúc) — không build được nếu chưa có version lịch sử theo thời gian.

---

## #6 Case builder — agent hỏi lại khi thiếu ngữ cảnh

**Giá trị:** giảm rủi ro tư vấn sai do câu hỏi mơ hồ — vấn đề cố hữu của RAG
một bước (agent cứ trả lời dựa trên giả định ngầm thay vì hỏi lại).

### Flow (mở rộng agent graph ở mục 3 kiến trúc)

1. Sau `analyze_intent`, nếu agent nhận diện câu hỏi thiếu thông tin quan
   trọng để trả lời chính xác (loại hợp đồng, mốc thời gian, địa bàn áp
   dụng...) → node `clarify` sinh 1–3 câu hỏi làm rõ dạng quick-reply.
2. Frontend hiển thị câu hỏi làm rõ dưới dạng chip bấm chọn hoặc nhập nhanh.
3. Câu trả lời làm rõ được gộp vào context → tiếp tục pipeline retrieve như
   bình thường.
4. **Giới hạn tối đa 1 vòng hỏi lại** — tránh vòng lặp làm phiền user; nếu
   sau 1 vòng vẫn thiếu, agent trả lời kèm giả định rõ ràng ("giả sử bạn đang
   hỏi về hợp đồng lao động không xác định thời hạn...").

### API contract

Mở rộng response của `/legal/chat/stream` hiện có (không tạo endpoint mới):

```
POST /api/v1/legal/chat/stream
  Request: {
    conversation_id,
    message,
    clarification_answers?: [{ question_id: string, answer: string }]
  }

  SSE event types mới:
    event: clarifying_question
    data: { question_id, question, quick_replies: string[] }

    event: answer          (như hiện tại, khi không cần/đã đủ làm rõ)
    data: { ... }
```

### Data model

Không cần bảng mới — lưu câu hỏi làm rõ + câu trả lời của user trong cột
`Message.metadata` (JSONB, cần thêm cột này nếu chưa có) thay vì tạo entity
riêng, vì đây là dữ liệu phụ trợ 1 lần cho 1 message, không cần query độc
lập.

---

## #7 Máy tính nghĩa vụ (calculators)

**Giá trị:** nâng từ "nhắc lịch" lên "tính con số" — giá trị thực dụng cao
với kế toán, nhóm sẵn sàng trả phí hơn cho tính năng này so với chat thuần.

### Flow

1. Danh sách calculator có sẵn (thuế TNCN, BHXH bắt buộc, phạt chậm nộp
   thuế...), mỗi calculator có form input khai báo qua `input_schema`.
2. User điền form → backend chạy **rule engine xác định** (code Python
   thuần, cùng pattern với `compliance/seed.py` hiện có — **không dùng LLM để
   tính số**, LLM không đáng tin cho phép tính).
3. Trả kết quả + breakdown từng bước + Điều luật căn cứ cho từng bước.
4. User có thể "Lưu vào lịch tuân thủ" → tạo `ComplianceTask`
   (`source_type=calculator`) với `due_date` + `amount` từ kết quả.

### API contract

```
GET /api/v1/calculators
  Response: 200 { items: [{ code, name, description, input_schema: JSONSchema }] }

POST /api/v1/calculators/{calculator_code}/compute
  Request:  { inputs: { ... theo input_schema ... } }
  Response: 200 {
    result: { total: number, unit: "VNĐ" },
    breakdown: [{ step: string, amount: number, legal_basis: Citation[] }],
    computed_at
  }

POST /api/v1/calculators/{calculator_code}/save-as-task
  Request:  { computation_id, due_date }
  Response: 201 ComplianceTaskOut
```

### Data model

```
CalculatorComputation(id, user_id, organization_id, calculator_code,
                       inputs JSONB, result JSONB, created_at)
```

Logic tính (`CalculatorDefinition`) là code, không phải bảng DB — nhưng mỗi
calculator cần field `effective_from`/`effective_to` ở mức *version rule*
(giống vấn đề hiệu lực văn bản ở mục 2.2) vì công thức tính thuế/BHXH cũng
thay đổi theo thời gian ban hành quy định mới.

---

## #8 Dashboard rủi ro tổng thể

**Giá trị:** biến sản phẩm từ "tra cứu" thành "quản trị rủi ro pháp lý" —
gộp tín hiệu từ 3 module đã có thành 1 màn hình cho chủ DN.

### Flow

1. Trang Dashboard load 1 lần khi vào app, hiển thị:
   - Số hợp đồng rủi ro CAO chưa xử lý (từ `document_reviews`).
   - Số compliance task quá hạn / sắp đến hạn 7 ngày.
   - Số câu hỏi gần đây có confidence thấp từ `verify_groundedness` (mục 3)
     chưa có câu trả lời chắc chắn.
   - Điểm "sức khỏe tuân thủ" tổng hợp (composite score, trọng số đơn giản).
2. Click vào từng số liệu điều hướng sang trang chi tiết tương ứng (đã có
   sẵn: `/contracts`, `/compliance`, `/chat`).

### API contract

```
GET /api/v1/organizations/{org_id}/dashboard/summary
  Role yêu cầu: mọi thành viên (VIEWER chỉ xem, không điều hướng sang trang
                sửa được)
  Response: 200 {
    contracts: { high_risk_pending: int, total_reviewed: int },
    compliance: { overdue: int, due_soon: int, completion_rate: float },
    qa: { low_confidence_recent: int },
    health_score: float   // 0-100
  }
```

### Data model

Không cần bảng mới — aggregation query trên các bảng đã có. **Cache ở Redis**
(TTL ngắn, vd 5 phút) vì tổng hợp nhiều bảng cùng lúc, tránh query nặng mỗi
lần load dashboard.

---

## #9 Escalation sang chuyên gia thật

**Giá trị:** điểm nối tự nhiên giữa "AI tư vấn sơ bộ" và dịch vụ trả phí,
đồng thời là lối thoát an toàn thay vì AI cố trả lời khi không chắc.

### Flow

1. Kích hoạt khi: `verify_groundedness` confidence thấp (tự động đề xuất) HOẶC
   user chủ động bấm "Yêu cầu chuyên gia xem lại" trên 1 câu trả lời.
2. Tạo `EscalationRequest` (`status=pending`), thông báo cho đội ngũ chuyên
   gia (MVP: 1 hàng đợi nội bộ xử lý thủ công qua admin panel, chưa cần
   marketplace nhiều luật sư).
3. Chuyên gia trả lời qua giao diện riêng (admin panel) → câu trả lời được
   gắn vào conversation dạng message mới `role=expert` (khác màu/badge so
   với AI).
4. (V1/V2) Có thể gắn phí dịch vụ ở bước tạo escalation — ngoài phạm vi đặc
   tả kỹ thuật này.

### API contract

```
POST /api/v1/conversations/{conversation_id}/messages/{message_id}/escalate
  Role yêu cầu: MEMBER+
  Request:  { note?: string }
  Response: 201 EscalationRequestOut { id, status: "pending", created_at }

GET /api/v1/escalations?status=pending
  Role yêu cầu: is_system_admin (hoặc role "expert" nếu tách riêng sau)
  Response: 200 { items: EscalationRequestOut[] }

POST /api/v1/escalations/{id}/respond
  Role yêu cầu: is_system_admin / expert
  Request:  { answer: string }
  Response: 200 { message: MessageOut }   // message mới role=expert trong conversation
```

### Data model

```
EscalationRequest(id, message_id, conversation_id, requested_by_user_id,
                   status: pending|answered|rejected, assigned_to NULL,
                   note, created_at, answered_at NULL)

MessageRole: thêm giá trị EXPERT (bên cạnh USER, ASSISTANT hiện có)
```

---

## #10 Zalo OA integration

**Giá trị:** kênh tiếp cận SME Việt Nam thực tế hơn web riêng — nhiều chủ DN
dùng Zalo hàng ngày hơn là mở 1 web app riêng.

### Flow

1. User chat qua Zalo Official Account → Zalo gửi webhook đến backend.
2. Backend map `zalo_user_id` ↔ `User` nội bộ qua bước **link tài khoản**
   (user thực hiện 1 lần trong app: nhận mã OTP, nhập vào Zalo chat để xác
   thực).
3. Nếu đã link → chạy cùng agent pipeline (mục 3) như kênh web, trả lời dạng
   text rút gọn (Zalo không render markdown đẹp như web) gửi lại qua Zalo
   Send API.
4. Nếu chưa link → bot trả lời hướng dẫn link tài khoản trước.

### API contract

```
POST /api/v1/integrations/zalo/webhook
  (Zalo gọi vào — xác thực bằng chữ ký request theo tài liệu Zalo OA,
   không dùng JWT thông thường)
  Request: theo schema webhook của Zalo OA
  Response: 200 (luôn trả nhanh, xử lý bất đồng bộ qua queue nếu cần)

POST /api/v1/integrations/zalo/link
  Role yêu cầu: đã đăng nhập (JWT)
  Request:  { otp: string }
  Response: 200 { zalo_user_id, linked_at }
```

### Data model

```
ZaloLink(id, user_id, zalo_user_id, linked_at)

IntegrationMessage(id, channel: zalo|web, external_message_id,
                    conversation_id, created_at)
  UNIQUE(channel, external_message_id)   -- chặn xử lý trùng khi Zalo retry webhook
```

**Rủi ro ngoài kỹ thuật:** cần tuân thủ chính sách gửi tin nhắn của Zalo OA
(giới hạn broadcast, nội dung đúng mục đích đăng ký) — xem mục 14 kiến trúc.

---

## #11 So sánh phương án dạng bảng

**Giá trị:** phù hợp câu hỏi ra quyết định của chủ DN ("nên chọn hộ kinh
doanh hay công ty TNHH") — câu trả lời dạng bảng dễ quét hơn văn xuôi dài.

### Flow

1. `analyze_intent` (mục 3) nhận diện thêm intent `compare` bên cạnh intent
   hiện có.
2. Nếu `compare`, `generate_answer` dùng structured output schema riêng cho
   dạng này thay vì văn xuôi tự do.
3. Frontend render `comparison_table` thành bảng so sánh thay vì bong bóng
   chat text thông thường.

### API contract

Mở rộng response schema hiện có của `/legal/chat` và `/legal/chat/stream`
(không cần endpoint riêng):

```
ChatResponse {
  ...existing fields,
  answer_format: "text" | "comparison_table",
  comparison_table?: {
    criteria: string[],
    options: [{
      name: string,
      pros: string[],
      cons: string[],
      legal_basis: Citation[]
    }]
  }
}
```

### Data model

Không cần bảng mới — đây là format trình bày của `generate_answer`, không
phải entity mới. Có thể lưu `answer_format` trong `Message` nếu cần biết lại
sau này message nào từng render dạng bảng.

---

## #12 API cho đối tác (B2B2C)

**Giá trị:** mở kênh phân phối qua dịch vụ kế toán/luật outsource — họ quản
lý nhiều khách hàng SME qua API thay vì UI, tận dụng trực tiếp mô hình
multi-org của `OrganizationMember` (mục 6).

### Flow

1. Đối tác đăng ký làm `Partner`, được cấp `PartnerApiKey`.
2. Chủ DN (khách hàng của đối tác) **chủ động grant quyền** cho Partner truy
   cập `Organization` của mình (không phải Partner tự tạo org và sở hữu —
   chủ DN luôn giữ full control, có thể thu hồi quyền bất kỳ lúc nào).
3. Partner gọi các endpoint hiện có (chat/compliance/contracts) kèm header
   `X-API-Key`, scoped theo `organization_id` đã được grant — không khác gì
   user thường gọi, chỉ khác cơ chế auth.

### API contract

```
POST /api/v1/partners/api-keys
  Role yêu cầu: is_system_admin (tạo cho đối tác đã duyệt)
  Response: 201 { key: string (chỉ hiện 1 lần), key_id, created_at }

POST /api/v1/organizations/{org_id}/partner-grants
  Role yêu cầu: OWNER (chủ DN chủ động cấp quyền)
  Request:  { partner_id, org_role: "member"|"viewer" }
  Response: 201 PartnerOrganizationGrantOut

DELETE /api/v1/organizations/{org_id}/partner-grants/{partner_id}
  Role yêu cầu: OWNER
  Response: 204

GET /api/v1/partners/organizations
  Auth: X-API-Key
  Response: 200 { items: [{ organization_id, name, org_role }] }

-- Các endpoint hiện có (/legal/chat, /compliance/tasks, /documents...)
-- chấp nhận thêm X-API-Key header, resolve thành (partner, organization_id)
-- thay vì (user, organization_id) từ JWT.
```

### Data model

```
Partner(id, name, contact_email, status: pending|active|suspended, created_at)

PartnerApiKey(id, partner_id, key_hash, created_at, revoked_at NULL)

PartnerOrganizationGrant(id, partner_id, organization_id,
                          org_role: member|viewer, granted_at, revoked_at NULL)
```

`org_role` cho Partner giới hạn tối đa `member` (không thể là `owner`/`admin`
— đối tác không được quản lý thành viên hay xoá tổ chức của khách hàng).
