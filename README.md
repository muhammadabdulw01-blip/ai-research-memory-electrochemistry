# Catalyst Memory — backend prototype

A real (not browser-only) implementation of the three components in the project
brief: research memory, AI-assisted extraction, and knowledge-driven
experimentation, using electrocatalysis as the test domain.

| Project brief component | Implementation here |
|---|---|
| Research memory | `app/models.py` + PostgreSQL/SQLite — structured, queryable, versioned storage with provenance (`extraction_method`, `notes_raw` kept verbatim, per-field confidence) |
| AI-assisted extraction | `app/llm_extraction.py` — Claude, called with forced structured tool-output, replacing the earlier regex prototype |
| Knowledge-driven experimentation | `app/recommender.py` — Gaussian Process + Expected Improvement (real Bayesian optimization), not a rule-based heuristic |

This is intentionally still a **prototype**: SQLite by default, a 2D (potential,
pH) parameter space for the recommender, and no auth/multi-user layer yet. Each
of those is a clearly scoped next step (see below) rather than a rewrite.

## Setup

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env              # then edit .env and add your ANTHROPIC_API_KEY
python seed_data.py               # optional: populate a few sample experiments
uvicorn app.main:app --reload
```

Then open **http://localhost:8000/docs** — FastAPI generates an interactive
API explorer automatically, so you can try every endpoint from the browser
without writing a frontend yet.

## Endpoints

- `POST /experiments` — log a structured experiment
- `GET /experiments?q=...&reaction=OER` — search/filter the memory
- `POST /extract` — send raw lab-note text, get back structured fields + per-field confidence, extracted by Claude
- `GET /recommend?reaction=OER` — get the next (potential, pH) condition to test, chosen by maximizing Expected Improvement over a Gaussian Process fit to prior runs
- `GET /health` — liveness check

## Example: extraction

```bash
curl -X POST localhost:8000/extract \
  -H "Content-Type: application/json" \
  -d '{"raw_text": "Tested IrO2 NP for OER in 0.5M H2SO4, pH ~0.3. At 1.6 V vs RHE got 10.2 mA/cm2, FE around 95%. Stability 12h, some degradation."}'
```

## Example: recommendation

```bash
curl "localhost:8000/recommend?reaction=OER"
```

Needs at least 3 logged OER experiments with potential, pH and Faradaic
efficiency filled in — run `seed_data.py` first, or log a few via `/experiments`.

## Honest limitations, and what each one becomes as a next step

- **SQLite → PostgreSQL + pgvector.** Fine for one user's laptop; swap
  `DATABASE_URL` in `.env` for real multi-user deployment and concurrent writes.
- **No semantic search yet.** `embed_text()` in `llm_extraction.py` is a stub —
  wire it to a real embedding model (e.g. Voyage AI) and rank by cosine
  similarity against the `embedding` column for "find experiments like this
  one" search, not just keyword match.
- **Recommender is 2D (potential, pH) only.** Real electrocatalysis screening
  has more dimensions (catalyst loading, electrolyte concentration,
  temperature, catalyst identity itself as a categorical/structural feature).
  Past 2–3 continuous dimensions, move from scikit-learn's GP to BoTorch/Ax,
  which handle mixed categorical+continuous spaces properly.
- **No auth.** Fine for a single-user prototype; add e.g. FastAPI's OAuth2
  password flow or an existing lab SSO before multi-user deployment.
- **No frontend yet.** The `/docs` page is usable for testing and demos;
  a real frontend (React, or reusing the earlier HTML prototype's UI wired to
  these endpoints instead of localStorage) is the natural next piece once the
  backend is validated.
- **Recommender is unvalidated.** The real scientific contribution is testing
  whether GP-recommended conditions actually outperform expert/random choices
  *in the lab* — that closed-loop validation is future PhD work, not something
  code alone can establish.

## Suggested order to build this out further

1. Validate extraction: run it against ~30 real notes from your lab, compare
   to what a human curator would enter, report accuracy per field.
2. Add the frontend back on top of this API.
3. Extend the recommender to more parameter dimensions with BoTorch.
4. Run one real closed-loop cycle: get a recommendation, run the experiment,
   log the result, see if the model's prediction was close.
