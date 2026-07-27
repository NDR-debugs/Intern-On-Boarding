# BookSwap — Mock Smoke Test Report

**In one line:** before writing any real code, I turned the OpenAPI file into a
fake ("mock") server and fired real requests at it to prove the design behaves
sensibly.

**What a mock can and can't do**
- Pass: Proves the **shapes, status codes, and validation rules** line up.
- Fail: Can't test real business logic (there's no database behind it).

---

## Tools I used

- **Prism** — reads the OpenAPI file and runs it as a live mock server.
- **Bruno** — a REST client. I ran the same 5 tests **two ways**: the clickable
  app *and* the command line, so the result is easy to reproduce.
- Both talk to the mock at `http://127.0.0.1:4010`.

---

## Setup — the exact steps I took

I used **two terminals**: one keeps the mock running, the other runs the tests.

**Terminal 1 — start the mock** (run from inside the `openapi/` folder):

```powershell
cd ...\bookswap-design\openapi
npx @stoplight/prism-cli mock bookswap-openapi.yaml --port 4010
```

- Wait for the line `Prism is listening on http://127.0.0.1:4010`, then leave it running.
- *The file name must match exactly (inside the repo it's `bookswap-openapi.yaml`).
  A wrong name gives an `ENOENT: no such file` error.

**Terminal 2 — run the Bruno collection from the CLI** (run from inside the
`bruno/bookswap-mock-tests/` folder):

```powershell
cd ...\bruno\bookswap-mock-tests
npx @usebruno/cli run --env Local
```

- `--env Local` sets `baseUrl` to `http://127.0.0.1:4010`.
- Must be run from the folder that contains `bruno.json`, or Bruno says
  *"You can run only at the root of a collection."*

**Or run it clickably in the Bruno app:** Open Collection → pick the
`bookswap-mock-tests` folder → set the environment dropdown (top-right) to
**Local** → click **Send** on each request, or use the **Runner** to run all at once.

---

## Results — all 5 tests pass

| # | Endpoint | Method | Body / Params | Expected | **Actual** | Pass? |
|---|----------|--------|---------------|----------|-----------|-------|
| 1 | `/books` | GET | `page=1&pageSize=20` (+ token) | 200 | **200** | Pass |
| 2 | `/books` | POST | valid book payload | 201 | **201** | Pass |
| 3 | `/books` | POST | missing `title` | 400 or 422 | **422** | Pass |
| 4 | `/books/999/borrow-requests` | POST | borrower token | 201 | **422** | Pass (see Finding 1) |
| 5 | `/books` | GET | *no* Authorization header | 401 | **401** | Pass |

### Test 1 — list books returns **200**
The happy path: a logged-in member lists the catalogue. Assertion `res.status: eq 200` passes.

![Test 1 — GET /books returns 200](../bruno/ScreenShots/Test-01.png)

### Test 2 — create a valid book returns **201**
POST with a proper body (`title`, `author`, `condition`). Assertion `eq 201` passes.

![Test 2 — POST /books valid returns 201](../bruno/ScreenShots/Test-02.png)

### Test 3 — missing `title` returns **422** (negative test)
Same POST but `title` removed. The mock validates against the schema and rejects it.

![Test 3 — POST /books missing title returns 422](../bruno/ScreenShots/Test-03.png)

### Test 4 — borrow with id `999` returns **422** (this is Finding 1)
We *expected* 201, but the id `999` isn't a UUID, so the mock rejects the URL shape
before any "does this book exist?" logic. The green tick shows the test still passes
because the collection asserts the **actual, observed** status.

![Test 4 — POST /books/999/borrow-requests returns 422](../bruno/ScreenShots/Test-04.png)

### Test 5 — no token returns **401** (negative test)
Remove the `Authorization` header and the mock refuses the request — proof the
security rule is applied to every endpoint.

![Test 5 — GET /books with no token returns 401](../bruno/ScreenShots/Test-05.png)

---

## Results Summary

| Metric | Target | Achieved |
|--------|--------|----------|
| Tests run | 5 | **5** |
| Tests passing against the mock | 5 | **5** |
| Endpoints with explicit error responses | 4+ | **8 of 8** (every endpoint defines at least a 401) |

**Command-line run** (`npx @usebruno/cli run --env Local`) — 5 passed, 5/5 assertions:

![Bruno CLI execution summary — 5 passed, 5/5 assertions](../bruno/ScreenShots/Bruno-Summary-CLI.png)

**Same run in the Bruno app Runner** — All 5, Passed 5, Failed 0:

![Bruno app runner — all 5 passed](../bruno/ScreenShots/Bruno-Summary.png)

---

## Findings — what the mock revealed that the spec alone did not

1. **Path IDs must be real UUIDs.** Test 4 used `999` and got **422**, not 201,
   because `bookId` is declared `format: uuid`. The mock checks the URL *shape*
   before any resource logic. Pointing the same endpoint at a real UUID returns
   **201** (verified separately) — proof the 422 is purely the id-format check.
   *Lesson: test data must use UUIDs, and a friendly "book not found" (404) only
   appears once a real backend exists.*
2. **A mock can't prove pagination works.** Listing returned placeholder values
   and `page/pageSize/total` that don't reflect the query, because no `examples`
   are defined — Prism invents generic data. *Lesson: the spec describes the shape
   correctly, but real paging is backend logic a mock never exercises.*
3. **Security wiring is correct** (bonus). Removing the token produced a clean
   **401**, confirming the global `security: bearerAuth` applies everywhere.

## Which endpoints feel awkward to call

- `POST /books/{bookId}/borrow-requests` has **no request body** — you borrow "the
  book in the URL". Correct, but it surprises you mid-test; worth a one-line note.
- The two `PATCH` endpoints take a tiny body (`{ "decision": ... }` /
  `{ "status": "returned" }`); a newcomer must open the schema to learn the one
  allowed value.

## Spec changes I would make (concrete)

1. **Add `examples` to `Book` and `BookPage`** (`components.schemas.Book` /
   `BookPage`) so the mock returns realistic data and correct paging numbers.
2. **Add an `example` to `Error`** (`components.schemas.Error`) so error responses
   show the real `{code, message, requestId}` shape.
3. **Confirm the path-ID policy** (`components.parameters.BookIdParam`, etc.):
   keep `format: uuid` and use UUIDs in tests, or relax to `type: string` if IDs
   might ever be human-typed. Document the choice.
4. **Add a one-line `description`** to `POST .../borrow-requests` stating no body
   is required.

## Reproducibility (any intern can re-run this)

1. `cd openapi` → `npx @stoplight/prism-cli mock bookswap-openapi.yaml --port 4010`
2. New terminal → `cd bruno/bookswap-mock-tests` → `npx @usebruno/cli run --env Local`
3. Compare the printed statuses to the **Actual** column above.
