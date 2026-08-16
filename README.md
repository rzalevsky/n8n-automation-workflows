# n8n Automation Workflows

Automation scenarios built in self-hosted n8n, each wrapping an LLM step around a process
that normally eats hours of manual work. Workflows are exported as JSON and import directly.

**What makes these different from most n8n demos: the language model runs locally.** No
invoice, no customer message and no internal document leaves the network. Every scenario here
points at an OpenAI-compatible endpoint served by [LM Studio](https://lmstudio.ai/) on my own
hardware — which is the only version of this that a finance or HR department can actually
approve.

Built by [Roman Zalevsky](https://linkedin.com/in/roman-zalevsky) — fifteen years of ERP
process automation, now applied with low-code tooling and language models.

| # | Scenario | Status |
|---|---|---|
| 1 | [Invoice intake — mailbox to structured data](#1-invoice-intake--from-mailbox-to-structured-data) | ✅ Working |
| 2 | Request routing — classify, route, answer | 🚧 In progress |
| 3 | Source monitoring — digest instead of feed | 🚧 In progress |

---

## Quick start

```bash
git clone https://github.com/rzalevsky/n8n-automation-workflows
cd n8n-automation-workflows
cp .env.example .env        # fill in your values
docker compose up -d
```

Then open http://localhost:5678, import a workflow, and connect its credentials.
Full walkthrough below.

---

## 1. Invoice intake — from mailbox to structured data

**Problem.** Supplier invoices arrive as PDF attachments. Someone opens each one, reads off
the counterparty, amount, currency, due date and invoice number, and re-types them into a
spreadsheet. It is slow, it is boring, and typos propagate straight into reporting — where
they are expensive to find and awkward to explain.

**Solution.** A Gmail trigger picks up mail matching an invoice filter. The PDF attachment is
extracted to text and passed to a local LLM, which is constrained to return a fixed JSON
schema. A code node validates that result, the row is appended to Google Sheets, and a
threshold check decides whether the notification is routine or needs a human to look at it
before payment.

```
Gmail trigger → fetch message + attachment → PDF to text
      → local LLM (LM Studio) extracts fields as JSON
      → parse & validate  → Google Sheets row
      → amount > threshold ?  → Telegram alert
                              → Telegram routine notification
```

**Result.** Manual re-typing is out of the loop. Invoice data lands structured and queryable
the moment the mail arrives, and anything above the threshold is surfaced immediately rather
than discovered at month-end.

### Design decisions worth explaining

**The model runs locally.** `qwen2.5-7b-instruct-1m` on LM Studio, reached over an
OpenAI-compatible API. A 7B model is more than enough for structured extraction from a
document whose layout barely changes, and running it locally means invoice data — counterparty
names, amounts, payment terms — never reaches a third party. Swapping in a hosted model is a
one-line change if you want it; the point is that you do not have to.

**The parse step fails loudly instead of guessing.** The model is instructed to return raw
JSON, but models occasionally wrap output in a markdown fence or add a sentence of commentary.
The code node strips the fence, then checks that every required field is present and that the
amount is numeric — including the case where a Polish-trained model returns `123,45` instead
of `123.45`. If any check fails it throws with the raw model output attached.

That is deliberate. A failed execution gets noticed; a silently wrong amount in an accounting
sheet does not. In this domain a visible failure is the cheaper outcome, and the error message
carries what you need to diagnose it.

**The threshold check reads from the parsed data, not from the Sheets response.** Google Sheets
returns column values as strings, so comparing them numerically is fragile. The condition
references the validated output of the parse node, where `amount` is guaranteed to be a number.

**Nothing environment-specific is committed.** Sheet ID, Telegram chat ID, alert threshold and
the LLM endpoint are all read as `{{ $env.NAME }}`, supplied by Docker Compose from a
git-ignored `.env`. The mail filter in the exported JSON is a generic placeholder — my own runs
against a specific sender, but that belongs in my instance, not in a public repository.

### Known limitations

- `Extract from File` reads the first attachment (`attachment_0`). Mail carrying several
  attachments, or the PDF in second position, needs a loop or a filter node ahead of it.
- Password-protected PDFs — common with some providers' e-invoices — are not handled.
- Text-layer PDFs only. A scanned invoice needs an OCR step before extraction.
- Polling runs hourly. For higher volume, a push-based trigger beats polling on both latency
  and API quota.

### What it looks like

![Invoice intake workflow on the n8n canvas](01-invoice-intake/screenshot.png)

*The full graph after a successful run — item counts on the connections show the invoice
travelling from the mailbox through extraction and validation into the sheet, then down the
routine branch because the amount was below the alert threshold.*

*The output: structured rows appended to the sheet, and the notification that lands the moment
an invoice arrives. Test data.*

### Files

- [`01-invoice-intake/workflow.json`](01-invoice-intake/workflow.json) — importable workflow
- [`01-invoice-intake/screenshot.png`](01-invoice-intake/screenshot.png) — the canvas after a run

---

## Running the stack

### 1. Configure

```bash
cp .env.example .env
```

Fill in `.env`:

| Variable | What it is |
|---|---|
| `GOOGLE_SHEET_ID` | The part of your sheet's URL between `/d/` and `/edit` |
| `TELEGRAM_CHAT_ID` | Message your bot, then read it from `https://api.telegram.org/bot<TOKEN>/getUpdates` |
| `INVOICE_ALERT_THRESHOLD` | Amount above which the alert branch fires. Defaults to `500` |
| `LLM_BASE_URL` | Your OpenAI-compatible endpoint — see the networking note below |
| `GENERIC_TIMEZONE` | Defaults to `Europe/Warsaw` |

`.env` is listed in `.gitignore`. Nothing you put there reaches the repository.

### 2. Start n8n

```bash
docker compose up -d
docker compose logs -f n8n     # follow start-up
```

n8n comes up on http://localhost:5678.

The compose file uses `network_mode: host`, which keeps the container on the host's network
stack. That matters for reaching a local model:

| Setup | `LLM_BASE_URL` |
|---|---|
| `network_mode: host`, model on this machine | `http://localhost:1234/v1` |
| `network_mode: host`, model on another machine | `http://<lan-ip>:1234/v1` |
| Bridge networking (`ports: ["5678:5678"]`), model on the host | `http://host.docker.internal:1234/v1` |

`host.docker.internal` does **not** resolve under host networking — it is a bridge-mode
facility. Getting this wrong is the most common reason the LLM node times out.

Compose substitutes `${VAR}` from `.env` at start-up, and those values become environment
variables inside the container, where the workflow reads them as `{{ $env.VAR }}`. That chain
is why no sheet ID, chat ID or endpoint appears in the workflow JSON.

For expressions to see them, n8n needs `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. That is already
n8n's default; the compose file sets it explicitly so the dependency is visible rather than
implied.

### 3. Serve a local model

[LM Studio](https://lmstudio.ai/) or [Ollama](https://ollama.com/), with the OpenAI-compatible
server enabled. Any instruction-tuned 7B model handles structured extraction; this one uses
`qwen2.5-7b-instruct-1m`.

Check it is reachable from the container:

```bash
docker compose exec n8n sh -c 'wget -qO- $LLM_BASE_URL/models'
```

The quoting matters: it defers expansion to the container's shell, where `LLM_BASE_URL`
actually exists. A list of models means the endpoint is reachable; a hang or a connection
refusal means the address is wrong for your networking mode.

### 4. Import the workflow

In n8n: **Workflows → Import from File** → pick `01-invoice-intake/workflow.json`.

### 5. Connect credentials

Gmail OAuth2, Google Sheets OAuth2 and a Telegram bot token are configured in n8n's own
credential store, not in this repository. n8n encrypts them at rest and never writes them into
an exported workflow — which is why the JSON here carries credential *names* but no secrets.

Adjust the Gmail trigger filter to match your own invoices. The committed value
(`invoices@example.com`, `has:attachment filename:pdf`) is a placeholder.

## Stack

n8n (self-hosted, Docker Compose) · LM Studio — local OpenAI-compatible LLM · Gmail API ·
Google Sheets API · Telegram Bot API

## Licence

MIT — see [LICENSE](LICENSE).
