
# DataExport — Large Dataset CSV Export in Next.js

A production-quality Next.js application demonstrating how to efficiently export up to **1 million+ rows** as CSV, with:

-  Cursor-based chunked export (never overwhelms the DB)
-  Real-time progress tracking via polling
-  Automatic resume on failure (no rows re-processed)
-  Live streaming download (browser receives CSV as it's written)
-  Paginated table UI with filters (department, role, location, status, salary range, search)

---

## Run in 3 Steps

```bash
# 1. Install dependencies
npm install

# 2. Seed the database (100k rows, ~10 seconds)
npm run seed

# 3. Start the dev server
npm run dev
```

Open **http://localhost:3000** — you'll see the employee table. Apply filters, then click **↓ Export CSV**.

> To test with 1 million rows, edit `scripts/seed.js` and change `TOTAL_ROWS = 100_000` to `TOTAL_ROWS = 1_000_000`, then re-run `npm run seed`.


| Approach | Problem |

| `SELECT * LIMIT 1000 OFFSET 500000` | OFFSET scans 500k rows just to skip them — O(n²) total cost |
| Load all rows into memory | 1M rows × ~200 bytes = 200 MB in RAM per export request |
| Single long-running DB query | Holds a DB connection open for minutes; blocks other traffic |
| No checkpointing | Server restart = export lost, must restart from scratch |

**Why cursor pagination is critical:**
- `WHERE id > cursor ORDER BY id LIMIT 5000` uses the primary key B-tree index
- Each chunk is **O(chunk_size)** regardless of how deep we are in the dataset
- At row 900,000 of a 1M row export, we're still scanning exactly 5,000 rows — not 905,000

### Database Protection

Three mechanisms prevent the export from overwhelming the DB:

1. **Chunking**: Each chunk is a short-lived read (≈5ms), then the connection is free
2. **Inter-chunk delay** (`INTER_CHUNK_DELAY_MS = 50`): Other transactions run between chunks
3. **WAL mode**: SQLite Write-Ahead Logging means readers and writers never block each other

For PostgreSQL/MySQL in production, the same cursor approach applies. Add `READ COMMITTED` isolation level to further reduce lock contention.

### Resume on Failure

The `cursor` column in `export_jobs` is updated after every chunk write. If the server crashes:

```bash
# Resume from last checkpoint:
POST /api/jobs/{id}/resume
```
The file is reopened in **append mode** (`flags: 'a'`), and the cursor query starts from the last successfully persisted `id`. Zero rows are re-processed, zero rows are missed.
## API Reference

| Method | Path | Description |
| `GET` | `/api/data?page=1&department=Engineering&...` | Paginated employee listing |
| `POST` | `/api/data` | Returns distinct filter option values |
| `POST` | `/api/export` | Create export job `{ filters: {...} }` → returns job |
| `GET` | `/api/jobs/:id` | Poll export progress |
| `POST` | `/api/jobs/:id/resume` | Resume a failed export |
| `GET` | `/api/export/:id/download` | Stream CSV (works during processing or after done) |
