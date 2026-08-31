# Running Stage 3 locally

Quick reference for starting the backend + frontend and confirming
everything still works. See `README.md` for the project overview; this
file is just the day-to-day runbook.

## Quick start (recommended)

Once the one-time setup below has been done, **`start.bat` alone gets you
to a fully populated "running demo"** — no separate ingest/seed step:

1. Double-click **`start.bat`** at the repo root. Before starting either
   server it automatically: ingests curriculum content if `data/chroma`
   is empty (first run only, ~1-2 min, needs network), and seeds the same
   three demo students used throughout this write-up (every run — cheap,
   offline, idempotent, so the dropdown and mastery bars are never empty).
   Then it starts both servers, waits for them to come up (first backend
   load is ~20-30s — loading the embedding model), and opens the browser
   automatically. Leave the console window it opens alone; the servers
   keep running after it's closed.
2. Double-click **`stop.bat`** when done — shuts both down cleanly,
   including anything left over from a previous manual start.

The real logic is in `scripts/start.ps1` / `scripts/stop.ps1` (PowerShell,
for proper PID tracking); the `.bat` files are double-click shims, since
Windows' default execution policy usually blocks a bare `.ps1` from
running on double-click. `start.ps1` still fails loud with a clear
message if the *one-time* setup (venv/npm/`.env`) hasn't been done —
see below — rather than guessing; ingest/seed failures (e.g. no network)
only warn, since the servers can still start without them.

The manual two-terminal steps below still work exactly as before, and are
the useful fallback when something needs debugging directly. The manual
ingest/seed commands below are also how you force a re-ingest or add more
demo state later — `start.bat` only auto-ingests when `data/chroma` is
completely empty.

## One-time setup

```powershell
# Backend (from repo root)
python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env

# Frontend
cd frontend
npm install
```

Node.js/npm must be installed (this machine has it via
`winget install OpenJS.NodeJS.LTS`, at `C:\Program Files\nodejs`). If a
fresh terminal doesn't recognise `node`/`npm`, open a new terminal window —
PATH is set at the OS level after install and should pick up automatically.

## Ingest curriculum content (done automatically by `start.bat` on first run)

```powershell
python -m stage3.ingest --source isaac_science
python -m stage3.ingest --source isaac_chemistry
python -m stage3.ingest --source ada_computer_science
```

`data/chroma/` (the vector DB) is deliberately gitignored — it's a ~50MB
binary blob, and rebuilding it from the same public APIs is cheap and
keeps every clone's corpus honestly reproducible rather than
committing a snapshot. **Without this step the app still runs and
chats still start, but every answer is ungrounded** (no curriculum
chunks to retrieve) — this is the single most likely reason a fresh
clone "doesn't work the same" as a machine where ingest was already run
once. Safe to re-run any time — ingest is idempotent (same chunk ids
get updated, not duplicated).

## Seed some demo students (done automatically by `start.bat` every run)

```powershell
python scripts/seed_demo_students.py
```

Creates three clearly-synthetic students (`demo-student-1/2/3`) with a
varied spread of topics/mastery already seeded — so the student picker
and the mastery indicator have something to show immediately, instead of
starting completely empty. **Fabricated data, never real** — see the
script's own docstring (`source="demo_seed"` in `student_state.db`, kept
distinct from real `source="diagnostic"` rows). Safe to re-run (skips
already-seeded topics); `--reset` wipes just the `demo-*` rows first.

## Enable the Stage 1 signal (optional)

```powershell
cp data/stage1/stage1_profiles.synthetic.csv data/stage1/stage1_profiles.csv
```

Without this, `profile_to_note`/`attainment_band_to_prior` just have no
profile to read (`get_profile` returns `None`, clean no-op — see
`profiles/stage1_loader.py`) — the app runs fine, this step is purely to
exercise the Stage 1 wiring locally. `stage1_profiles.synthetic.csv` is a
committed fixture (confirmed safe — see local dev notes for the
verification trail); the copy target, `stage1_profiles.csv`, is where a
real future export would land and is gitignored, same convention as
`.env.example` → `.env`. Restart the backend after copying (profiles load once at
startup). Try `SYN0001`/`SYN0008`/`SYN0010` in the student picker on a
fresh **Biology** topic (e.g. Biochemistry) to see three different
`flag_status`/`attainment_band` combinations — `none`/`above`,
`provisional`/`in_line`, `confirmed`/`well_below` respectively.

## Start the service (every time)

Two terminals, left running side by side:

**Terminal 1 — backend**, from the repo root:
```powershell
python -m uvicorn stage3.api.main:app --reload
```
- Serves `http://127.0.0.1:8000`.
- **First request after each restart takes ~20-30s** — it loads the
  sentence-transformers embedding model into memory on first import. This
  is normal, not a hang.

**Terminal 2 — frontend**, from `frontend/`:
```powershell
npm run dev
```
- Serves `http://localhost:5173`. Open that URL in a browser.

Stop either with Ctrl+C.

## Smoke test

### 1. Automated offline tests (fastest, no servers needed)

```powershell
python -m unittest discover tests
```
Expect **215 passing tests** (and growing), instantly. This checks wiring,
the prompt/redaction guards, and conversation-store logic offline — it
does **not** touch the live API, the embedding model, or the UI.

### 2. Backend API check (needs the backend running)

```powershell
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/subjects
curl http://127.0.0.1:8000/subjects/biology/topics
curl http://127.0.0.1:8000/students
```
Expect:
- `/health` → `{"ok":true}`
- `/subjects` → `{"subjects":["biology","chemistry","computer_science"]}`
  (one entry per file in `data/topics/` — mathematics/english were
  removed 2026-08-14: provisional placeholders added before subject
  scope was confirmed, dropped once it was)
- `/subjects/biology/topics` → the Biochemistry/Cell Biology/Genetics/... list
- `/students` → every id that has ever started a conversation — `start.bat`
  seeds `demo-student-1/2/3` automatically, so expect those on the list
  rather than an empty one (only a bare `python -m uvicorn ...` start, per
  the manual two-terminal steps, skips seeding) — not a real roster either
  way, see `stage3/api/students.py`'s module docstring.

### 3. UI check (needs both servers running)

1. Open `http://localhost:5173`.
2. Click the Student ID field — it's a searchable dropdown, not a bare
   text box. Type `demo` and pick `demo-student-1` (or `-2`/`-3` — each
   has a different mastery spread). Or type any new ID, e.g.
   `test-student-1`, and either press Enter or click the "Use new ID" row.
3. Click a subject in the left sidebar — every topic for that subject is
   listed immediately and is already its own chat. There is no separate
   "New Chat" step; topics ARE the chats (one continuous thread per
   (student, subject, topic) — see `stage3/conversations/store.py`'s
   module docstring). If the selected student has mastery data, a small
   coloured bar appears next to each subject (rollup) and topic (per-topic
   estimate) — red/low to green/high, grey/empty for "no check-in yet".
4. Click a topic to open/continue its thread. If no Student ID is set
   yet, an inline hint appears instead of nothing happening.
5. **The first time you open any topic, the tutor asks 3 short questions
   before normal tutoring starts** — a diagnostic to seed a mastery
   baseline (see `stage3/tutor/diagnostic.py`). A banner shows "Quick
   check-in — question N of 3". Answer them (any answer works — right,
   wrong, or nonsense are all fine, just to see the flow) and the tutor
   transitions to normal tutoring after the 3rd. A "Re-check my
   understanding" button in the chat header restarts this on the same
   topic/thread any time.
6. Type a message and send it.
7. All three subjects (**biology**, **chemistry**, **computer_science**)
   have real curriculum content ingested (see "Subjects" in `README.md`)
   — so expect a real grounded answer, both for diagnostic questions and
   normal tutoring.
8. The mastery bars in the sidebar/topic list (see step 3) update after
   each diagnostic round completes — or inspect `data/student_state.db`'s
   `mastery` table directly.
9. **Explanation-method selection has no UI surface** — it's a
   backend/prompt mechanism only. To see it happening: after the
   diagnostic, ask a normal question on any subject (all three have
   `concept_id` populated as of the 2026-08-16 re-ingest), then check
   `data/student_state.db`'s `method_interactions` / `method_posterior`
   tables directly. The
   student never sees a method name or an `[[UNDERSTANDING: ...]]`
   marker — both are stripped before display.

## Known current limitations (not bugs)

- **Subject scope is deliberately just three** — biology, chemistry,
  computer_science (Isaac Science / Ada Computer Science). Mathematics
  and English were provisional placeholders added before subject scope
  was confirmed; removed 2026-08-14 (three real,
  well-grounded subjects is a defensible scope on its own — adding more
  isn't a goal unless a real need for them emerges). `data/stage1/`
  (learner profiles) is also still empty — personalisation has nothing to
  draw on regardless of subject.
- **Mastery only updates via the diagnostic**, never during ordinary
  tutoring turns — a deliberate scope decision, not a bug (see
  `stage3/tutor/diagnostic.py`'s module docstring). The
  foundation-tier trigger that would eventually *read* mastery isn't
  built yet either.
- Only `biology`, `chemistry`, and `computer_science` have seeded topic
  lists (`data/topics/*.json`); add more `data/topics/<subject>.json`
  files to add subjects — but see the scope note above before doing so.
- ~~Explanation-method selection only activates for Computer Science~~
  **Resolved 2026-08-16** — the 2026-08-16 re-ingest populated `concept_id`
  for Biology/Chemistry too, so explanation-method selection is now live
  for all three in-scope subjects.
- **Gemini free-tier quota is small** (`generate_content_free_tier_
  requests`, 20/day observed on `gemini-3.7-flash` for this key) — heavy
  manual testing (several diagnostic rounds in one session) can exhaust
  it. `GeminiLLM.generate` retries `max_retries` times (see
  `llm/client.py`) then raises `LLMGenerationError` (2026-08-14 — it used
  to return `""` and show a blank tutor message; that's fixed), which
  the API turns into a `503` the UI shows as a normal "Failed to send"
  error with a Retry
  button. Not a bug in this project's code; wait for the quota window to
  reset (or use `LLM_PROVIDER=null` / `NullLLM` for wiring checks that
  don't need real answers).
