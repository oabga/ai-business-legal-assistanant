# Trợ lý Pháp lý AI cho Doanh nghiệp nhỏ và vừa

Ứng dụng web giúp chủ doanh nghiệp, kế toán và nhân sự tra cứu nhanh các quy định
pháp luật Việt Nam (Luật Doanh nghiệp, thuế, lao động, hợp đồng), hỏi đáp tình
huống cụ thể và nhận tư vấn sơ bộ. **Mọi câu trả lời đều kèm trích dẫn Điều luật**
từ kho văn bản chính thống để người dùng tự kiểm chứng.

- **Frontend**: React 19 + Vite + TypeScript + Tailwind CSS 4 + TanStack Query + React Router + Zustand
- **Backend**: FastAPI + LangGraph (RAG 7 node) + SQLAlchemy 2.0 async + Alembic
- **Dữ liệu**: PostgreSQL 17 (nghiệp vụ + full-text search) + ChromaDB (vector) + BM25 (lexical)
- **Mô hình**: Google Gemini qua endpoint tương thích OpenAI (`gemini-2.5-flash` + `gemini-embedding-001`)

## Tính năng

| Tính năng | Mô tả |
| --- | --- |
| Hỏi đáp pháp lý | Chat có streaming token, hiện quá trình suy luận của agent theo từng bước, kèm danh sách căn cứ pháp lý bấm được |
| Tra cứu văn bản | Danh mục theo lĩnh vực, cây Chương → Điều, full-text search không cần gõ dấu, viện dẫn chéo giữa các Điều |
| Soát xét hợp đồng | Tải PDF/DOCX/TXT, tách theo điều khoản, chấm mức rủi ro và đề xuất sửa, mỗi phát hiện đều có Điều luật căn cứ |
| Lịch tuân thủ | Sinh nghĩa vụ định kỳ (thuế GTGT, TNDN, BHXH, báo cáo lao động, BCTC) theo hồ sơ doanh nghiệp |
| Quản trị | Thống kê corpus, nạp thêm văn bản, dựng lại vector index, quản lý người dùng và phân quyền |

## Chạy nhanh bằng Docker

Cần Docker Compose và một API key Gemini (lấy miễn phí tại
[aistudio.google.com/apikey](https://aistudio.google.com/apikey)).

```bash
cp .env.example .env
# Mở .env, điền LLM_API_KEY và JWT_SECRET_KEY (openssl rand -hex 32)
docker compose up -d --build
```

Lần đầu, nạp corpus pháp luật vào PostgreSQL rồi dựng vector index:

```bash
docker compose exec backend python scripts/load_postgres.py --truncate
docker compose restart backend
```

| Thành phần | Địa chỉ |
| --- | --- |
| Ứng dụng web | http://localhost:5173 |
| API + Swagger | http://localhost:8023/docs |
| PostgreSQL | localhost:23432 |

Tài khoản đầu tiên đăng ký sẽ tự động nhận vai trò `admin` (khi cơ sở dữ liệu
còn trống) để có người quản trị corpus.

Dừng nhưng giữ dữ liệu: `docker compose down`. Xóa cả dữ liệu:
`docker compose down -v`.

### Vẫn chạy được khi chưa có API key

Backend cố tình **không** coi việc dựng vector index là điều kiện sống: thiếu API
key thì đăng nhập, tra cứu văn bản và lịch tuân thủ vẫn hoạt động bình thường,
chỉ riêng hỏi đáp trả về 503 kèm thông báo giải thích. Sau khi bổ sung key, gọi
`POST /api/v1/admin/corpus/reindex` là chức năng hỏi đáp bật lại, không cần
restart.

## Chạy development trên máy

```bash
# 1. PostgreSQL
docker compose up -d postgres

# 2. Backend (Python 3.10 do ràng buộc của underthesea)
cd backend
cp .env.example .env          # điền LLM_API_KEY, JWT_SECRET_KEY
uv sync --frozen
uv run alembic upgrade head
uv run python scripts/load_postgres.py --truncate
uv run python scripts/run_backend.py --reload

# 3. Frontend (terminal khác)
cd frontend
npm ci
npm run dev
```

Vite proxy `/api` sang `http://127.0.0.1:8023` nên frontend gọi đường dẫn tương
đối, giống hệt khi deploy sau nginx. Đổi backend target bằng
`VITE_BACKEND_ORIGIN`.

## Cấu hình

Nguyên tắc: `backend/config.yaml` chứa mọi thứ commit được (host/port, tên model,
tham số retrieval, ngưỡng rerank); **secret và giá trị phụ thuộc môi trường đọc từ
biến môi trường và ghi đè YAML**.

Thứ tự ưu tiên: biến môi trường → `.env` → `config.yaml` → default trong code.

| Biến | Ý nghĩa |
| --- | --- |
| `LLM_API_KEY` | API key Gemini cho chat model (dùng luôn cho embeddings nếu bỏ trống `EMBEDDINGS_API_KEY`) |
| `LEGAL_DATABASE_URL` | DSN PostgreSQL |
| `JWT_SECRET_KEY` | Khóa ký JWT, bắt buộc đổi trước khi deploy |
| `CORS_ALLOW_ORIGINS` | Danh sách origin, phân tách bằng dấu phẩy |
| `LLM_BASE_URL` / `EMBEDDINGS_BASE_URL` | Đổi sang vLLM local hoặc provider khác |

Danh sách đầy đủ: [`.env.example`](.env.example) (Docker) và
[`backend/.env.example`](backend/.env.example) (chạy trực tiếp).

Đổi `embeddings.model` hoặc `embeddings.dimensions` là đổi số chiều vector, nên
Chroma sẽ tự phát hiện qua manifest và dựng lại toàn bộ index.

## Kiến trúc

```text
                  ┌──────────────────────────────┐
   Trình duyệt ──►│ nginx (SPA + proxy /api)      │
                  └───────────────┬──────────────┘
                                  │ REST + SSE
                  ┌───────────────▼──────────────┐
                  │ FastAPI                       │
                  │  auth · chat · laws           │
                  │  documents · compliance       │
                  │  admin · lab                  │
                  └───┬───────────┬───────────┬───┘
                      │           │           │
        ┌─────────────▼──┐  ┌─────▼─────┐  ┌──▼──────────┐
        │ PostgreSQL      │  │ ChromaDB  │  │ Gemini      │
        │ users, chat,    │  │ vector    │  │ LLM +       │
        │ corpus + FTS    │  │ + BM25    │  │ embeddings  │
        └─────────────────┘  └───────────┘  └─────────────┘
```

Pipeline RAG là graph LangGraph 7 node:
`analyze_intent → prepare_retrieval_query (rewrite/HyDE) → retrieve (hybrid) →
rerank → llm_filter → generate_answer → format_submission`. Retrieval là hybrid
Chroma + BM25 hợp nhất bằng weighted RRF.

Tiếng Việt không có từ điển full-text trong PostgreSQL, nên tra cứu văn bản dùng
`to_tsvector('simple', immutable_unaccent(...))` kèm `pg_trgm` để cứu truy vấn
gõ sai. Việc bôi đậm từ khóa làm ở client trên văn bản gốc, vì `ts_headline`
chạy trên cột đã bỏ dấu sẽ trả về đoạn trích mất hết dấu.

## Cấu trúc repo

```text
├── backend/            FastAPI + LangGraph
│   ├── src/{routers,services,schemas,models,core}
│   ├── alembic/        migration schema
│   ├── tests/          pytest
│   └── config.yaml
├── frontend/           Vite + React SPA
├── reference/ui-nextjs/ UI Next.js tham khảo
├── corpus/             crawler + manifest 40 văn bản
├── data/               base_data.json (3.844 Điều luật)
├── eval/               khung đánh giá retrieval/QA (xem eval/README.md)
├── docs/               tài liệu dự án + khóa luận (xem mục Tài liệu bên dưới)
├── Baocao/             khóa luận LaTeX
└── docker-compose.yml
```

Chi tiết quy ước dự án (config, path cố định, quy trình dev) xem [`CLAUDE.md`](CLAUDE.md).

## Tài liệu & định hướng sản phẩm

Codebase hiện tại là bài thi R2AI đã chuyển hướng thành sản phẩm thật. Các
tài liệu dưới đây là **kế hoạch/kiến trúc mục tiêu**, chưa phải mô tả trạng
thái hiện tại — xem bảng Tính năng ở trên để biết cái gì đã chạy thật.

| Tài liệu | Nội dung |
| --- | --- |
| [`docs/MVP_PRODUCT_ARCHITECTURE.md`](docs/MVP_PRODUCT_ARCHITECTURE.md) | Định vị sản phẩm, agentic RAG, mô hình vai trò/phân quyền, eval (RAGAS), observability (Langfuse), roadmap MVP→V1→V2, triển khai AWS |
| [`docs/architecture.md`](docs/architecture.md) | Sơ đồ Mermaid chi tiết: component diagram, flow agentic RAG, pipeline đồng bộ văn bản, ERD, CI/CD, flow cho từng tính năng mở rộng |
| [`docs/FEATURES.md`](docs/FEATURES.md) | Flow + API contract + data model cho 12 tính năng mở rộng (mời thành viên, cảnh báo thay đổi luật, escalation, Zalo OA, API đối tác...) |
| [`docs/KE_HOACH_DU_AN.md`](docs/KE_HOACH_DU_AN.md) | Kế hoạch gốc cho cuộc thi R2AI Stage 1 |
| [`docs/CUONG_LINH_KHOA_LUAN.md`](docs/CUONG_LINH_KHOA_LUAN.md) | Đề cương khóa luận |

## Kiểm thử

```bash
cd backend && uv run pytest          # 53 test: auth, hội thoại, tra cứu, tuân thủ, hợp đồng, hỏi đáp 503
cd frontend && npm test              # 24 test: hook SSE, tiện ích highlight
```

Test backend chạy trên **PostgreSQL thật** (database `legal_assistant_test` tự
tạo rồi tự xóa) chứ không phải SQLite, vì phần tra cứu dựa hẳn vào `tsvector`,
`unaccent` và `pg_trgm`. Cần `docker compose up -d postgres` trước khi chạy.

Ngoài ra có các script smoke test gọi thẳng từng nhóm API:

```bash
cd backend
uv run python scripts/smoke_auth.py
uv run python scripts/smoke_laws.py
uv run python scripts/smoke_compliance.py
uv run python scripts/smoke_contracts.py
uv run python scripts/smoke_admin.py
```

## Competition mode

Codebase gốc là bài thi; phần này được giữ lại ở `/api/v1/lab/` (yêu cầu vai trò
`admin`) để tái tạo số liệu cho chương đánh giá của khóa luận. Định dạng
`results.json` không đổi.

```text
POST /api/v1/lab/competition          # trả JSON cuối
POST /api/v1/lab/competition/stream   # stream tiến độ từng câu
```

Giao diện tương ứng ở `/lab/competition`: tải bộ test JSON, xem tiến độ, tải
`results.json`. Chạy ngầm không cần mở UI:

```bash
cd backend
uv run python scripts/run_competition.py --file path/to/test.json
```

Kết quả ghi vào `backend/outputs/competition_<run_id>_<status>.json`, kèm
`report.log` append sau mỗi câu.

Lưu ý cho khóa luận: kết quả thi đấu dùng Qwen3-8B (thỏa ràng buộc open-source
< 14B của cuộc thi), còn sản phẩm ứng dụng dùng Gemini. Hai con số không so sánh
trực tiếp được.
