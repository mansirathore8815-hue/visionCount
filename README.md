# Toolfinder: AI Tools & Technologies Analytics

Ask "which tool is best for editing video?" and get a ranked answer from a MySQL catalog of 37 AI tools and 12 deep-learning techniques. Every question is logged and summarised on a live dashboard.

## Run it

1. Install Node 18+ and MySQL 8 (or MariaDB 10.5+), and make sure MySQL is running.
2. `npm install`
3. `cp .env.example .env` and put your MySQL password in `.env`
4. `npm run setup`  (creates `ai_tools_db`, tables and seed data; safe to re-run)
5. `npm start`  then open http://localhost:3000
6. Optional: `npm run demo` fills the dashboard with **synthetic** traffic. It deletes existing `query_logs`, so skip it on real data.

`npm test` runs offline ranking checks.

## Layout

    backend/server.js    Express API, MySQL pool, analytics queries
    backend/ranker.js    question -> ranked tools (the "brain")
    database/schema.sql  ai_tools, ai_technologies, tool_technologies, query_logs
    database/seed.sql    catalog data
    frontend/            index.html, style.css, script.js (no build step)
    scripts/             setup-db, seed-demo, ranker tests

## API

- `POST /api/ask` `{ "question": "..." }` returns ranked tools, or a technology explanation
- `GET /api/tools?category=` and `GET /api/categories`
- `GET /api/analytics` totals, per-category counts, 14-day trend, top and unanswered questions, top tools
- `GET /api/health`

## What was wrong in the original spec, and what changed

| Problem | Why it fails | Fix |
|---|---|---|
| Frontend sent the raw question as the `category` | `LIKE '%which tool is best...%'` matches nothing, so the main use case returned zero rows | Backend interprets the question; the client only sends text |
| `searched_category` stored free text | `GROUP BY` on typed sentences makes every question its own "category"; the chart is meaningless | Category is detected server-side from the top result, so the chart counts real categories |
| Search used `category LIKE ? OR tool_name LIKE ?` | Plain substring match, no ranking, no synonyms ("edit" vs "retouch") | Concept-based TF-IDF ranking (`ranker.js`) |
| Zero-result searches indistinguishable in logs | No way to see what the catalog is missing | `results_count` column and an "unanswered questions" panel |
| `innerHTML` with database/user text | Stored XSS | All rendering uses `textContent` |
| Hardcoded root password, `localhost:3000` in fetch, open CORS | Breaks on any other host; credentials in source | `.env`, relative URLs, same-origin (no CORS needed) |
| Anyone can POST unlimited searches | Analytics can be flooded | Per-IP rate limit and length limits |
| Logging was fire-and-forget | Silent failures | Log is awaited, errors return a clean 500 |
| "Deep learning" in the brief | The original had none | See below |

## About "deep learning"

The ranker is classical information retrieval, not a neural network. Calling it deep learning would be false. What the project does cover:

- `ai_technologies` / `tool_technologies` store the deep-learning techniques (transformers, diffusion, GANs, CNNs, RAG, ...) and which tools use them. Questions like "what is a diffusion model?" are answered from MySQL.
- To make matching genuinely neural, replace `scoreTools` with embedding similarity (store a vector per tool, embed the question, rank by cosine). The API shape and UI would not change.

## Data caveats

Seed data is hand-written. `popularity_score` is an editorial rating, not a measurement, and tool versions go stale fast. Technologies are mapped to a tool only where the architecture is publicly documented.
