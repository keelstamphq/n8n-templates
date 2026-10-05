# Monthly AI budget per client (n8n template)

Run an OpenAI agent for several clients, each with its own monthly budget, a pause switch and a usage ledger. A run that would take a client over budget is refused before the model is called, while every other client keeps working.

File: `per-client-spend-cap-v2.json` (workflow name: "Run an OpenAI agent with a monthly budget per client using Data tables"). Import it in n8n with **Workflows → Import from file**, or paste the JSON onto the canvas.

## What it does

The workflow has four entry points.

**1. Run the agent:** `POST /webhook/client-ai/run` (Header Auth) with `{ "client_id": "acme", "prompt": "..." }`.

1. **Validate request** checks the input. A missing or invalid `client_id`, an empty prompt or a prompt longer than `max_prompt_chars` gets HTTP 400.
2. The client's `monthly_budget`, `paused` switch and `instructions` are read from the `ai_clients` Data table, and this month's rows from `ai_usage_ledger`. A month is a calendar month in UTC.
3. **Check budget** refuses an unknown client (403), a paused client (403), a client without a valid `monthly_budget` (503) and a client whose remaining budget does not cover a worst-case run (429).
4. **Reserve, then check again.** The run writes its worst-case amount as its own ledger row (`reserved`), reads the ledger again and backs off with 429 (`released`) if parallel runs used the budget first. Runs never update each other's rows, and the earliest reservations win.
5. **Client AI agent** (AI Agent with an OpenAI Chat Model and a Calculator tool) runs with Max Iterations and a token limit from **Settings**. The client's `instructions` are added to the system message.
6. **Read token usage** reads n8n's own record of the running execution through the n8n API. **Settle the run** books the prompt and completion tokens of every model call on the run's row (`settled`). If usage can't be read, the full reservation is charged, never zero. Failed runs are booked too.
7. The caller gets HTTP 200 with the answer and a usage summary, or 502 if the agent failed. An email goes out when a client crosses `alert_at_percent`, and when what is left no longer covers another run.
8. Fail closed: if a Data table can't be read or written, the caller gets 503 and the model is not called.

**2. Every hour:** reservations still `reserved` after `reservation_ttl_minutes` (after a crash or timeout) become `expired`. They keep counting until you review them.

**3. On the 1st of each month:** each month starts at zero because ledger rows are keyed by month. Last month's totals per client are emailed, then rows older than `keep_months` are deleted.

**4. Usage endpoint:** `GET /webhook/client-ai/usage?period=YYYY-MM&client_id=acme` (same Header Auth) returns budget, used, settled, reserved, expired, percent and remaining per client. Both parameters are optional; the default is the current month and all clients.

Usage is counted in **tokens** by default. If you set `input_cost_per_1m_tokens` and `output_cost_per_1m_tokens` in **Settings**, budgets and usage are counted in your own cost unit instead. The workflow contains no prices: both rates are 0 until you enter your own.

## Setup in 5 steps

1. **Create the two Data tables.**
   - `ai_clients`: `client_id` (string), `monthly_budget` (number), `paused` (boolean), `instructions` (string).
   - `ai_usage_ledger`: `client_id`, `period`, `run_id`, `status`, `note` (string) and `amount`, `tokens_in`, `tokens_out`, `model_calls` (number). One row per run; `status` is `reserved`, `settled`, `released` or `expired`.
2. **Add credentials:** OpenAI on **Chat model**, an n8n API key on **Read token usage**, Header Auth on **Client request** and **Usage request**, and SMTP on **Email budget alert** and **Email monthly summary**.
3. **Fill in Settings, Sweep settings and Monthly settings:** the system message, `max_output_tokens`, `max_iterations`, `max_prompt_chars`, `tool_overhead_tokens`, the two optional cost rates, `alert_at_percent` and the alert addresses; `reservation_ttl_minutes`; `keep_months` and the summary addresses. **Chat model** is preset to `gpt-5-mini`; change it to the model you use.
4. **Keep Save execution progress on** in the workflow settings (it is on in the file). **Read token usage** needs it to see the model calls of the running execution.
5. **Add a client row, publish and test.** With `client_id` = `acme`, `monthly_budget` = `20000` and `paused` = false:

```bash
curl -X POST https://<your-n8n>/webhook/client-ai/run \
  -H "X-Api-Key: <your webhook key>" -H "Content-Type: application/json" \
  -d '{"client_id":"acme","prompt":"Draft a short reply about opening hours."}'
```

You get 200 with the answer and usage. Lower the budget to `1` and repeat: 429. Set `paused` to true: 403. In the editor, **Execute workflow** uses the pinned sample request on **Client request**.

## Limitations (please read)

- **The reservation is an estimate** (about 4 characters per token, plus `tool_overhead_tokens`). Large tool results can push a run past it. The booked amount is always the measured one.
- **Exact usage needs the n8n API** credential and Save execution progress. Without them, each run is charged its full reservation. According to the n8n docs, the n8n API is not available during the n8n Cloud free trial.
- **No atomic increment.** Data tables have none. Reserve-then-recheck stops parallel runs from overspending, but under contention runs may be refused early, and alerts can be duplicated or missed.
- **Volume.** Above about 1,000 runs per client per month, reads page through the table and can fail while rows are being added; the run is then refused. Use a database at that volume.
- **Row order.** The reservation order relies on row ids becoming visible in increasing order. This was tested on SQLite, not on Postgres.
- This template meters what passes through it. Calls your clients make elsewhere are not counted.

## Tested with

Self-hosted n8n 2.41.6 (Node.js 24, SQLite) against a local mock of the OpenAI Chat Completions API and a local SMTP catcher: wrong webhook key, empty `client_id`, paused and unknown client (403, 400, 403, 403); a normal run and a run with a tool call, where the booked usage matched the usage the mock reported; a used-up budget (429 and one email); the alert threshold email; model errors and Max Iterations reached (502; tokens already used were booked); a missing Data table (503, model not called); 10 and 20 parallel calls with budget for 3 and 5 runs (exactly 3 and 5 got 200, the rest 429); the hourly sweep, the monthly job and the usage endpoint, started by hand.

Not tested: the real OpenAI API, n8n Cloud, Postgres, queue mode, real email delivery, and the hourly and monthly schedules on the real scheduler.

## Nodes used

Webhook, Edit Fields (Set), Code, If, Data table, AI Agent, OpenAI Chat Model, Calculator, n8n, Send Email, Respond to Webhook, Schedule Trigger, Sticky Note. All built into n8n; no community nodes.

## License

MIT. See [LICENSE](../LICENSE).

This template works on its own; you do not need Keelstamp to use it. It was built by the team behind [Keelstamp](https://keelstamp.com), which is in development.

Prepared with AI assistance. Editorial responsibility: PowerQuant ApS.
