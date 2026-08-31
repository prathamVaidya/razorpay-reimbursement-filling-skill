# razorpay-reimbursement-filling-skill

An [agent skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches a coding agent
to file and track expense claims on **RazorpayX Payroll** (`payroll.razorpay.com`).

RazorpayX Payroll ships no public reimbursement API. It is a server-rendered HTML app, but its
form endpoints are perfectly drivable with `curl` and the session cookie you already have in your
browser. This skill documents that surface — the endpoints, the field formats, the type ids, and
the handful of silent failure modes that will otherwise eat an afternoon.

> Not affiliated with, endorsed by, or supported by Razorpay. It documents observed behaviour of a
> web app that can change without notice. Verify before trusting.

## What it does

Given a folder of receipts, the agent will read every one of them, group them into claims, learn
your house style from your own claim history, file them, and verify each row landed.

1. **Unlock the session** — one "Copy as cURL" of the listing page. O(1); everything after is automated.
2. **Read the listing** — existing claims, statuses, what is already pending.
3. **Discover token and type map** — CSRF token and expense-type ids scraped per tenant, never hardcoded.
4. **Learn your style** — mirrors the phrasing your approvers already accept, and checks for duplicates.
5. **POST the claim** — multipart, with real receipt bytes attached.
6. **Verify and report** — re-reads the listing and reports the claim id, because the POST response carries none.

## Install

Drop it wherever your agent looks for skills, e.g.:

```bash
git clone https://github.com/<you>/razorpay-reimbursement-filling-skill.git \
  ~/.claude/skills/razorpay-reimbursements
```

Then just ask — *"file these receipts on razorpay"* — and the skill loads on its own.

## Gotchas it encodes

The reason this exists as a skill rather than a snippet. Each of these fails **silently**:

| Trap | What actually happens |
|---|---|
| **A stray query param blanks the page** | Unknown params return `200 OK` with a zero-byte body — no error. An empty response is far more often a bad query string than a dead session. Strip back to `?filter=all`. |
| **The default view hides pending claims** | The page you copy is usually `filter=ap` (Approved & Paid). Submit a claim, check that view, and the row is missing — looking exactly like a failed write. Verify with `filter=p`. |
| **Guessed filter values don't error** | An unrecognised value returns the unfiltered list or an empty table. Scrape the real ones off the `<select>`. |
| **`100 Continue` masquerades as failure** | curl sends `Expect: 100-continue` on multipart, so a header dump holds two status lines. `grep -m1 '^HTTP/'` reports `100`. Trust `-w '%{http_code}'`. |
| **Firefox strips file bytes on "Copy as cURL"** | A captured upload shows `attachment[]` with a filename and an empty body. Replay it verbatim and you file a claim with no receipt. |
| **Two `<select name="type">` on one page** | One is the listing filter, one is the new-claim form. The form's is authoritative — identify it by its leading blank option. |
| **The POST returns no claim id** | The response body is empty. Re-read the listing to learn the id. |
| **Live sessions refresh their cookie** | A `Set-Cookie` with a future `Max-Age` proves the session is alive and tells you how long is left — the clean way to tell "bad URL" from "expired". |

## Judgment it encodes

Mechanics are the easy half. The skill also carries the decisions that matter:

- **Read every receipt.** Filenames lie, and so do the dates in them.
- **Never invent a number.** Every amount traces to a document or a statement.
- **Foreign-currency claims use what the card actually settled**, not today's spot rate. A vendor
  receipt showing `$100.00` carries no rupee figure at all; only the statement substantiates one.
- **The expense date sets the policy month**, and a receipt dated outside it is fine when the
  reason explains why — a subscription paid on the 3rd can still be claimed for the month it covers.
- **Check for duplicates before filing.** A prepared sheet is not evidence nothing on it was already submitted.
- **Surface personal-vs-business ambiguity instead of resolving it silently.** Miscategorising is
  the user's problem, not the agent's, so it gets raised rather than guessed.

## Safety

- Claims are financial records filed under your name with your employer. The skill requires an
  explicit go-ahead per claim and will not batch-submit on its own initiative.
- Your session cookie is a credential. It stays in a scratch directory, is never written into the
  project, never echoed back, and never sent anywhere but `payroll.razorpay.com`.
- Cookies here are short-lived, so a session lasts hours, not days.
- Pending claims have Edit and Delete controls, so mistakes caught early are recoverable.

## Contributing

Field ids and formats are observed on a single tenant and may differ on yours — the skill scrapes
them at runtime rather than trusting the table. If your tenant differs, or you hit a failure mode
not listed above, a PR describing the observed behaviour is welcome.

Please do not include cookies, tokens, employee or user ids, claim ids, real claim text, or
receipt contents in issues or PRs.

## License

MIT
