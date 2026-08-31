# Stage 3 Tutor

A retrieval-grounded, adaptive tutoring app. Students chat with a tutor
that answers from real curriculum content (not just the LLM's own
knowledge), runs a short diagnostic to seed a per-topic mastery
estimate, and adapts how it explains things to each student over time.

## Subjects

**Biology, Chemistry, Computer Science** — sourced live from
[Isaac Science](https://isaacscience.org) and
[Ada Computer Science](https://adacomputerscience.org). No curriculum
content is committed to this repo; it's fetched by the ingest step
below and cached locally in a vector store.

## How it works

- **Retrieval** — a local sentence-transformers embedding model +
  Chroma vector store retrieve relevant curriculum chunks per question,
  reranked by similarity/trust/feedback signals.
- **Diagnostic** — the first time a student opens a topic, a 3-question
  check-in seeds a mastery estimate for that (student, subject, topic).
- **Adaptive explanations** — the tutor tracks which explanation style
  (worked example, analogy, step-by-step, etc.) has worked for a given
  student per concept, and leans on that going forward.
- **Redaction guard** — a fail-closed PII filter sits between any
  external input and the prompt sent to the LLM.

## Stack

- **Backend** — FastAPI, LangChain + Chroma, sentence-transformers,
  SQLite (student state, conversations), Google Gemini as the LLM
  provider (pluggable — see `stage3/llm/client.py`).
- **Frontend** — React + TypeScript + Vite.

## Quick start

See **[RUNNING.md](RUNNING.md)** for full setup and day-to-day run
instructions. Short version, from the repo root:

```powershell
python -m venv .venv && .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env          # then add your Gemini API key
cd frontend && npm install && cd ..
```

Then double-click **`start.bat`** — it ingests curriculum content on
first run, seeds a few demo students, starts both servers, and opens
the browser.

## Layout

```
stage3_rag_tutor/
├── stage3/
│   ├── connectors/        # curriculum source connectors (Isaac, Ada)
│   ├── vectordb/          # Chroma store
│   ├── retriever/         # reranked search
│   ├── student_state/     # per-student SQLite mastery/state
│   ├── conversations/     # chat thread storage
│   ├── tutor/             # prompt assembly, context fusion, session loop
│   ├── llm/               # provider-neutral LLM client
│   └── api/               # FastAPI app
├── frontend/               # React + Vite UI
├── evaluation/             # structured review instrument
└── tests/                  # offline unit tests (no API key needed)
```

## Licensing note

Ingested curriculum content carries its source's own licence — Isaac
Science content is CC BY 4.0, Ada Computer Science content is
CC BY-NC-SA 4.0. The app attributes sources in-context; see
`stage3/tutor/attribution.py`.
