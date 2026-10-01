# eval/

Khung đánh giá chất lượng retrieval + QA cho dự án, tách khỏi code chạy sản
phẩm (`backend/src/`). Hiện thư mục này **chưa chứa harness thật** — harness
competition hiện tại vẫn sống ở `backend/scripts/run_competition.py` (gọi qua
`backend/src/services/competition/`) vì nó cần import trực tiếp các service
backend (agent, retrieval store, LLM client).

## Dự kiến dùng cho

- **Retrieval eval**: F2-score trên trích dẫn `Điều X` (trục chấm điểm tự động
  của cuộc thi R2AI Stage 1) — xem `docs/KE_HOACH_DU_AN.md`.
- **RAGAS**: `faithfulness`, `answer_relevancy`, `context_precision`,
  `context_recall`, `answer_correctness` trên golden set — chạy song song với
  F2-score, không thay thế (RAGAS không hiểu khái niệm "Điều X"/hiệu lực pháp
  lý). Chi tiết: `docs/MVP_PRODUCT_ARCHITECTURE.md` mục 8.
- **Metric tự xây** (RAGAS không có sẵn): `stale_citation_rate`,
  `temporal_accuracy`, `citation_format_validity` — xem cùng tài liệu, mục 8.3.
- **QA eval**: LLM-as-a-judge theo 5 tiêu chí (căn cứ pháp lý, chính xác nội
  dung, đầy đủ, thực tiễn, rõ ràng).
- So sánh biến thể pipeline (rewrite/HyDE, có/không rerank, ngưỡng filter...)
  trên cùng một bộ câu hỏi, không chạy qua UI.
- Kết quả eval nên đẩy lên Langfuse (mục 9 cùng tài liệu) để so sánh theo thời
  gian/giữa các version prompt trên cùng dashboard với trace production.

## Khi nào nên chuyển code thật vào đây

Nếu/khi tách được phần eval khỏi runtime import của backend (ví dụ gọi qua
HTTP API thay vì import trực tiếp service), chuyển
`backend/scripts/run_competition.py` + `backend/src/services/competition/`
vào đây theo cấu trúc:

```
eval/
├── datasets/     # bộ câu hỏi test + đáp án mẫu (golden set cho RAGAS,
│                 #  không chỉ Điều X — cần câu trả lời mẫu cho context_recall/
│                 #  answer_correctness)
├── run.py        # harness chạy qua HTTP API, không import backend/src
├── metrics/      # F2-score, RAGAS wrapper, metric tự xây (stale_citation_rate...)
└── results/      # output chấm điểm, không commit (gitignore)
```

Cho đến lúc đó, dùng trực tiếp:

```bash
cd backend
uv run python scripts/run_competition.py --file path/to/test.json
```

Kết quả: `backend/outputs/competition_<run_id>_<status>.json`.
