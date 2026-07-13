# The API Contract Block

When a change crosses the backend/frontend boundary, the orchestrator writes
this block during **Stage 1 — Plan**, before anyone builds. It is the device
that makes parallel building safe: both builders implement against the same
literal text instead of guessing at each other's output.

## Format

One block per endpoint, as a fenced JSON snippet:

```json
{
  "endpoint": "GET /api/widgets/{id}/stats",
  "auth": "required (session or token)",
  "request": { "query": { "period": "day|week|month" } },
  "response": {
    "data": {
      "total": "number",
      "series": [{ "date": "YYYY-MM-DD", "value": "number" }]
    }
  },
  "errors": { "404": "widget not found", "422": "invalid period" }
}
```

Field notes:

- **`endpoint`** — method + path, with path parameters in braces.
- **`auth`** — the auth requirement and any permission the caller must hold.
- **`request`** — query/body shape with types or allowed values.
- **`response`** — the exact key names and nesting, with types as values.
- **`errors`** — every non-2xx status the frontend must handle.

## Rules

1. **Defined by the orchestrator at Stage 1 — Plan**, as part of the plan the
   user approves at Gate 1. Builders never author or amend the contract.
2. **Pasted VERBATIM into both builder prompts** at Stage 2 — Build. Not
   summarized, not paraphrased — the same bytes in both prompts.
3. **Any change requires re-approval by the orchestrator** and re-notification
   of **both** builders — even a builder who "just renamed one key" has
   invalidated the other builder's work in progress.
4. **Response keys are law.** The frontend never renames, aliases, or guesses
   a key; the backend never returns a shape that differs from the block. If
   the contract turns out to be wrong, fix the contract first (rule 3), then
   the code.

The single most common failure of parallel AI development is the frontend
guessing a response shape the backend never produced. This block exists so
that the guess is never necessary — and QA (Stage 3 — Review) checks both
sides against it, not against each other.
