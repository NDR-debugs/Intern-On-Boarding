# OpenAPI Spec — README

**File:** `bookswap-openapi.yaml` · **Format:** OpenAPI 3.1.0

**What it is:** the *contract* for the BookSwap REST API. It describes every URL,
what you send, and what you get back — a human-readable and machine-readable
agreement that front-end and back-end devs can both build against.

---

## How to read an OpenAPI file (beginner version)

Four building blocks:
- **`paths`** — the URLs you can call (the *resources*, always nouns).
- **`get` / `post` / `patch`** under a path — the *action* (the HTTP verb).
- **`responses`** — the status codes each call can return, with their meaning.
- **`components`** — reusable pieces (data shapes, parameters, headers) written
  **once** and pointed to everywhere with `$ref`, so there's no copy-paste.

**Golden rule we followed:** URLs are **nouns**, actions are **HTTP verbs**.
→ `POST /books` (correct), not `POST /createBook` (wrong).

---

## The endpoints at a glance

| Resource | Verb | What it does | Key status codes |
|----------|------|--------------|------------------|
| `/books` | GET | list / search the catalogue (paginated) | 200, 401, 429 |
| `/books` | POST | list a new book | 201, 400, 401, 422 |
| `/books/{id}` | GET | get one book | 200, 401, 404 |
| `/books/{id}` | PATCH | edit my book (owner only) | 200, 401, 403, 404 |
| `/books/{id}/borrow-requests` | POST | ask to borrow | 201, 401, 404, 409 |
| `/books/{id}/loans` | GET | borrower history for a book (paginated) | 200, 401, 404 |
| `/borrow-requests/{id}` | PATCH | owner accepts / declines | 200, 401, 403, 404, 409 |
| `/loans` | GET | list loans by status (paginated) | 200, 401 |
| `/loans/{id}` | PATCH | mark a loan returned | 200, 401, 403, 404, 409 |

**Status codes in plain English:** 200 OK · 201 Created · 400 bad JSON ·
401 not logged in · 403 logged in but not allowed · 404 doesn't exist ·
409 clashes with current state (e.g. book already out) · 422 valid JSON but fails
a rule (e.g. missing title) · 429 too many requests.

---

## Design rules this spec follows (the grading checklist)

- **Nouns + correct verbs** — resources are nouns; verbs map to actions.
- **9 distinct status codes** used where they fit (the brief asks for 4+).
- **Every list endpoint is paginated** with `page` + `pageSize` (capped at 100).
- **Schemas defined once and reused via `$ref`** — e.g. `PageMeta` is shared by
  `BookPage` and `LoanPage` through `allOf`; every error points to one `Error` shape.
- **Privacy built in** — the `Book`/member shapes never expose address or phone.
- **Correlation** — every response returns an `X-Request-Id` header so logs and
  clients can be matched up.

**One 3.1 detail worth knowing:** OpenAPI 3.1 **removed** the old `nullable: true`
keyword. To say a field can be null we write `type: [string, "null"]` (used on
`Loan.returnedAt`). Using `nullable: true` in a 3.1 file is what a strict auditor
flags as a semantic error.

---

## How to validate and mock it

```bash
# 1) validate — should print "... is valid"
npx @apidevtools/swagger-cli validate bookswap-openapi.yaml

# 2) run it as a fake server on port 4010
npx @stoplight/prism-cli mock bookswap-openapi.yaml --port 4010
```

See `../decisions/mock-report.md` for the smoke tests run against the mock.

---

## What to say when explaining this file

- "URLs are nouns, actions are verbs — that's REST hygiene."
- "Status codes are specific, not everything-is-200."
- "Shapes live in `components` and are reused with `$ref`, so there's no duplication."
- "It validates cleanly, and I mocked it to prove the design behaves."
- "The one decision to defend: IDs are UUIDs — that's why a non-UUID id returns 422."
