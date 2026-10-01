# CLAUDE.md

Hướng dẫn cho Claude Code khi làm việc trong repo này.

## Dự án là gì

Trợ lý pháp lý AI cho doanh nghiệp nhỏ và vừa Việt Nam: hỏi đáp pháp luật có
trích dẫn Điều luật, tra cứu văn bản, soát xét hợp đồng, lịch tuân thủ. Codebase
gốc xuất phát từ bài thi **R2AI Stage 1** (Legal Information Retrieval + QA,
chấm F2-score trên trích dẫn `Điều X` + LLM-as-a-judge) — xem
`docs/KE_HOACH_DU_AN.md`. Phần competition được giữ lại ở `/api/v1/lab/` để
tái tạo số liệu cho khóa luận (`Baocao/`).

## Cấu trúc repo

```
├── backend/            FastAPI + LangGraph — service chính, Python 3.10
│   ├── src/{routers,services,schemas,models,core}
│   ├── alembic/         migration schema PostgreSQL
│   ├── tests/           pytest (chạy trên PostgreSQL thật, không mock)
│   ├── scripts/         run_backend, load_postgres, run_competition, smoke_*
│   └── config.yaml       nguồn sự thật cho mọi config commit được
├── frontend/            React 19 + Vite + TypeScript SPA
├── reference/ui-nextjs/ UI Next.js tham khảo, không phải sản phẩm chính
├── corpus/              script build corpus + manifest 40 văn bản luật
├── data/                base_data.json (3.844 Điều luật) + dataset thi
├── eval/                khung đánh giá retrieval/QA (xem eval/README.md — hiện harness thật vẫn ở backend/scripts/run_competition.py)
├── docs/                tài liệu dự án, kế hoạch, đề cương khóa luận
├── Baocao/              khóa luận LaTeX (không phải code sản phẩm)
└── docker-compose.yml   postgres + backend + frontend(nginx)
```

Mỗi service tự quản lý `src/`/`tests/`/`scripts/` riêng — không có `src/` dùng
chung ở root vì đây là monorepo đa service, không phải một package.

## Lệnh hay dùng

```bash
# Toàn stack qua Docker
cp .env.example .env            # điền LLM_API_KEY, JWT_SECRET_KEY
docker compose up -d --build
docker compose exec backend python scripts/load_postgres.py --truncate
docker compose restart backend

# Backend dev (Python 3.10 — ràng buộc của underthesea)
cd backend
uv sync --frozen
uv run alembic upgrade head
uv run python scripts/load_postgres.py --truncate
uv run python scripts/run_backend.py --reload
uv run pytest                   # cần `docker compose up -d postgres` trước
uv run ruff check .

# Frontend dev
cd frontend
npm ci
npm run dev
npm test
npm run lint                    # oxlint
```

Backend mặc định tại `:8023`, frontend dev server `:5173` (proxy `/api` sang
backend), PostgreSQL host port `:23432`.

## Quy ước quan trọng

- **Config**: `backend/config.yaml` chứa mọi thứ commit được; biến môi trường
  chỉ ghi đè secret/giá trị phụ thuộc môi trường, không ghi đè toàn bộ YAML khi
  chạy qua Docker (xem `backend/README.md`). Thứ tự ưu tiên khi chạy local: env
  → `.env` → `config.yaml` → default code.
- **Path tương đối cố định**: `backend/config.yaml` trỏ `../corpus/...` và
  `../data/...` — khi chạy Docker, `docker-compose.yml` mount `./corpus` và
  `./data` ở root vào container. **Không di chuyển `corpus/` hay `data/`** nếu
  không cập nhật đồng thời cả `docker-compose.yml` lẫn `backend/config.yaml`.
- **Test backend chạy trên PostgreSQL thật**, không SQLite — vì phụ thuộc
  `tsvector`, `unaccent`, `pg_trgm`. Luôn cần `docker compose up -d postgres`
  trước khi `pytest`.
- **Tiếng Việt full-text search**: dùng `to_tsvector('simple',
  immutable_unaccent(...))` + `pg_trgm`; highlight làm ở client trên văn bản
  gốc (không dùng `ts_headline` vì nó trả đoạn trích đã mất dấu).
- **RAG pipeline**: LangGraph 7 node — `analyze_intent →
  prepare_retrieval_query (rewrite/HyDE) → retrieve (hybrid Chroma+BM25, RRF) →
  rerank → llm_filter → generate_answer → format_submission`.
- **Backend sống được khi thiếu LLM API key**: auth/tra cứu/lịch tuân thủ vẫn
  chạy, chỉ hỏi đáp trả 503. Thêm key xong gọi
  `POST /api/v1/admin/corpus/reindex`, không cần restart.
- **Competition vs sản phẩm**: kết quả thi dùng Qwen3-8B (ràng buộc open-source
  < 14B), sản phẩm dùng Gemini — hai số liệu không so sánh trực tiếp.
- Mọi code Python mới tuân `ruff` (`line-length=120`, target py310). Mọi code
  frontend tuân `oxlint`.

## Khi sửa code

- Sửa backend service/router/schema trong `backend/src/`, không sửa trực tiếp
  trong `reference/ui-nextjs/` (đó chỉ là tham khảo).
- Thêm migration mới qua Alembic (`backend/alembic/versions/`), không sửa tay
  schema PostgreSQL.
- Sau khi đổi `embeddings.model`/`embeddings.dimensions` trong config, Chroma
  tự phát hiện qua manifest và rebuild toàn bộ index — không cần xóa tay.
- Script smoke test (`backend/scripts/smoke_*.py`) gọi thẳng API, dùng để kiểm
  tra nhanh một nhóm endpoint mà không cần chạy toàn bộ pytest suite.
