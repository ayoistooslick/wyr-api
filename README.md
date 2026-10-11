# wyr-api

A small, slightly over-engineered **would-you-rather API** written in [PlainScript](https://github.com/ayoistooslick/plainscript), compiled to Express, and deployed on Render.

It started as a list of questions. Now it has templates, a Postgres question registry, fingerprints, per-session history, and enough combinations to keep making new dilemmas for a very long time. Hobby project energy, database-backed behavior.

## live API

**Base URL:** <https://wyr-api-vhz0.onrender.com>

Try it:

```bash
curl https://wyr-api-vhz0.onrender.com/api/wyr
```

## quick examples

Get a random category:

```bash
curl https://wyr-api-vhz0.onrender.com/api/wyr
```

Pick the vibe:

```bash
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=good"
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=spicy"
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=18%2B"
```

Keep a session from seeing the same stored question twice:

```bash
curl "https://wyr-api-vhz0.onrender.com/api/wyr?level=good&sessionId=my-game-123"
```

`sessionId` is optional. Without it, the API still uses the global database registry, but it does not record per-session history. That is useful for anonymous one-off requests. With a session ID, the API atomically marks a question as served for that session.

## endpoints

| Method | Endpoint | What it does |
| --- | --- | --- |
| `GET` | `/` | Tiny intro and usage hint. |
| `GET` | `/api/wyr` | One random-level question. |
| `GET` | `/api/wyr?level=good` | One clean-ish question. |
| `GET` | `/api/wyr?level=spicy` | One flirty/awkward question. |
| `GET` | `/api/wyr?level=18%2B` | One adults-only question. Please use common sense. |
| `GET` | `/api/wyr?level=good&sessionId=abc` | Good question while avoiding repeats for session `abc`. |
| `GET` | `/api/levels` | Supported levels and session hint. |
| `GET` | `/api/health` | Database-backed health check and stored question count. |
| `GET` | `/api/stats` | Counts by level, history row count, and dedupe details. |

## levels

- `good` — everyday, safe-ish questions for basically anyone.
- `spicy` — flirty, awkward, or mildly chaotic questions.
- `18+` — adult relationship/intimacy questions. The endpoint does not perform age verification, so the caller has to use it responsibly.
- no `level` — randomly picks one of the three.
- anything else — `400 Bad Request` with the accepted levels.

## response shape

```json
{
  "id": "7a7cf0c3-7d24-4e52-8a5e-5c9f6d0d4e42",
  "level": "good",
  "question": "Would you rather have perfect Wi-Fi everywhere or have unlimited phone battery?",
  "optionA": "have perfect Wi-Fi everywhere",
  "optionB": "have unlimited phone battery"
}
```

The question is built from a compatible component and a template. The options are kept separately in the response so a UI does not have to parse the prose.

## how the “infinite-ish” part works

The app does not call an AI model and it does not pretend a fixed list is infinite. Instead, `src/app.pln` has structured components grouped by level and category. A component might contain two alternatives, or two alternatives plus consequences. PlainScript combines those pieces with templates and makes a candidate question.

Every candidate gets a SHA-256 fingerprint based on the level and the two normalized options. The database has a unique constraint on that fingerprint, so reversed options are treated as the same dilemma and concurrent requests cannot quietly create duplicates.

The current source also keeps the original curated question bank as a fallback. Dynamic templates are tried first; the curated questions are used if the generator needs a fallback.

## session history and concurrency

The database has two tables:

- `questions` — the global registry of generated and curated questions, keyed by a stable fingerprint.
- `question_history` — `(session_id, level, question_id)` rows recording what a named session has already seen.

For named sessions, the endpoint uses one PostgreSQL statement that inserts the candidate if needed, checks the session history, and marks the selected question. The primary key on `question_history` handles two requests arriving at the same time for the same session.

When a named session has seen everything the generator can currently find, the API returns `409` and suggests another level or a new session ID. Anonymous requests do not share a hidden global session.

## run it locally

You need Node 18+ and a PostgreSQL connection string.

```bash
git clone https://github.com/ayoistooslick/wyr-api.git
cd wyr-api
npm install
export DATABASE_URL="postgres://user:password@localhost:5432/wyr_api"
export PORT=3000
npm run check
npm run build
npm start
```

Then try:

```bash
curl "http://localhost:3000/api/wyr?level=good&sessionId=local-demo"
```

On startup, the PlainScript app creates the tables and indexes if they are not there yet. No migration framework here. It is a tiny hobby API, so `CREATE TABLE IF NOT EXISTS` is enough for now.

## tests

The pure engine rules have a native PlainScript test file:

```bash
npx plainscript test tests/engine.test.pln
```

The production API behavior is database-backed, so after setting `DATABASE_URL` you can use the curl examples above to check random generation, all levels, session repeat prevention, invalid levels, `/api/health`, and `/api/stats`.

## project layout

```text
src/app.pln          PlainScript application, generator, SQL, and routes
tests/engine.test.pln PlainScript unit tests for normalization/fingerprints
render.yaml          Render web service + Postgres blueprint
package.json         build and start scripts
```

All application behavior lives in PlainScript. The generated `dist/app.js` is just the Express output from the compiler and is not the source of truth.

## deploy notes

Render builds with:

```bash
npm ci && npm run build
```

and starts with:

```bash
npm start
```

The service expects `DATABASE_URL` to point at the Render Postgres database. The checked-in `render.yaml` documents the intended `fromDatabase` connection, and the live service uses the `wyr-api-db` Postgres instance in Frankfurt.

The database created for this hobby deployment is on Render’s free Postgres plan, which currently has an expiration window. That is a Render plan limitation, not an application behavior; move the database to a paid/non-expiring plan if this API needs to be kept around long-term.

## tiny disclaimer

This is a hobby API. The wording is subjective, the generator is deliberately simple, and the 18+ level is not an age gate. Please do not use adult questions around people who did not ask for them.
