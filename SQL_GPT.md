This is a clean text-to-SQL-over-local-files architecture, and it maps well to a diagram. Let me sketch it.Here's the high-level design. I'll show it in two views: the overall architecture (the two pipelines and how they share storage), then a zoom into the query execution loop, since that's where the Ollama calls actually happen.

<img width="1440" height="832" alt="image" src="https://github.com/user-attachments/assets/aa8ba80e-34d1-42d8-ab42-291943df2fc3" />


The two pipelines run at different times and only touch through storage. The write path normalizes files once; the read path never sees raw rows. The orchestrator is the brain — it holds the schema, talks to Ollama for SQL, and runs that SQL through DuckDB. The color coding: purple is the LLM (Ollama), teal is the deterministic data and query engine, gray is I/O and coordination.

Here's what each component is responsible for:

- **Upload + ingestion service** — accepts CSV/XLSX, sniffs delimiters and encodings, cleans headers (strips merged-cell junk, normalizes column names), infers types, and writes one Parquet file per table. xlsx and CSV are slow and untyped; converting once means every later query is fast and type-stable. DuckDB can do the conversion itself in a few lines (`COPY (SELECT * FROM read_xlsx('f.xlsx')) TO 'f.parquet'`).
- **Storage** — the Parquet files plus a small schema catalog (table names, columns, types, a few sample rows, key relationships). The catalog is what you feed Ollama as grounding; it's tiny and keeps the prompt bounded regardless of data size.
- **Query orchestrator** — the only stateful piece. Given a question, it pulls the relevant schema, prompts Ollama, validates the returned SQL, executes it on DuckDB, and optionally sends the small result back to Ollama for a plain-English summary.
- **Ollama** — runs locally, used twice: once to turn the question into SQL, once to phrase the result. A code-tuned model (e.g. `sqlcoder`, or a general instruct model with a tight schema prompt) does the SQL step better than a general chat model.
- **DuckDB** — does the actual filtering, joining on your common keys, and aggregation. This is the part that must be deterministic; the LLM never computes over rows itself.

Now the query execution loop in detail — this is the sequence the orchestrator runs for every question:Two design points worth deciding early, since they shape the whole build:

<img width="1440" height="1136" alt="image" src="https://github.com/user-attachments/assets/c253c8c4-6484-4c06-99e5-0e3b1880392b" />

The **validate step** is not optional. A local model will occasionally emit a hallucinated column, a write statement, or malformed SQL. Before executing, parse the SQL and check that every referenced table and column exists in the catalog, and reject anything that isn't a `SELECT`. DuckDB also supports read-only connections, so you get a second layer of safety for free. This catches most failure modes before they hit the engine; on a validation failure you can loop the error back to Ollama for one retry.

The **summarize step** is where you control token cost. Have DuckDB return aggregates or a top-N slice, not the full result set, and only that small payload goes to Ollama for phrasing. For a question like "total sales by region last quarter," DuckDB returns ~10 rows and the LLM never sees the millions. For some queries you may want to skip summarization entirely and just hand back the result table — cheaper and exact.

A reasonable minimal stack: FastAPI for the upload and query endpoints, DuckDB for both conversion and querying, Parquet on disk (or object storage), a JSON or SQLite schema catalog, and Ollama running locally for the two LLM calls. That's a single-box deployment you can stand up quickly and scale later by moving Parquet to object storage and DuckDB queries to a worker pool.

One thing to decide based on your query patterns: whether Ollama emits **raw SQL** (flexible, handles open-ended questions, needs the validation layer) or **structured query params** that your code compiles into SQL (safer, fewer hallucinations, but only covers query shapes you anticipated). For exploratory "ask anything" usage the raw-SQL route is worth the validation cost; for a fixed set of question types the structured route is more robust.

Want me to turn this into a written HLD document, or drop down a level into the actual FastAPI endpoint contracts and the ingestion/query code?
