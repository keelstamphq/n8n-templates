# Hash-chained log of every AI agent run (n8n template)

Run an AI agent for paying clients and log every run, whether answered, failed or refused, per client in a SHA-256 hash chain. A daily check and a verify endpoint show whether an entry was changed, removed or reordered afterwards.

File: `agent-run-log-v2.json` (workflow name: "Log every AI agent run per client in a hash chain and verify it daily"). Import it in n8n with **Workflows → Import from file**, or paste the JSON onto the canvas.

## What it does

**1. Run and log:** `POST /webhook/agent-run-log/run` (Header Auth) with `{ "client_id": "acme", "prompt": "..." }`.

1. `client_id` picks the chain. An invalid `client_id` gets HTTP 400, and a client that is not in `allowed_clients` (if you fill it in) gets 403. Neither is logged. A client in `paused_clients` (403) and an empty or too long prompt (422) are refused **and logged** with the outcome `refused`.
2. **Client AI agent** (AI Agent with an OpenAI Chat Model and a Calculator tool) runs with Max Iterations and a token limit from **Settings**. Agent errors are logged too, with the outcome `error`.
3. The run becomes one entry in that client's chain in the `agent_run_log` Data table: SHA-256 of the prompt and the answer, model, number of model and tool calls, outcome, error, `prev_hash` and `entry_hash`. Raw text is stored only if you set `store_text` to true, and then only the first `store_text_max_chars` characters.
4. **Parallel runs.** The run reads the newest entry of the client, links to it, appends its entry and reads again. If a parallel run took the same `seq`, the entry with the lowest row id wins, and this run removes its attempt, waits briefly and retries, up to `max_log_attempts` times.
5. The caller gets HTTP 200 with the answer and a receipt (`client_id`, `seq`, `run_id`, `logged_at`, `outcome`, `prompt_sha256`, `answer_sha256`, `prev_hash`, `entry_hash`), or 502 with a receipt if the agent failed. If the entry can't be written, the caller gets 503 **without** the answer and the owner gets an email.

**2. Every day at 03:15** (instance time zone): every entry of every client is recomputed, and each chain is compared with its last saved anchor. If something looks broken, the check waits `recheck_after_seconds` and looks again, so a retry that is still in progress does not raise a false alarm. Intact chains get a new anchor row in `agent_run_anchors`. You get an email with the first break per client, or, if `email_when_intact` is true, with each chain's latest hash as your copy outside n8n. If the tables can't be read, you get a separate email.

**3. Verify endpoint:** `GET /webhook/agent-run-log/verify?client_id=acme` (same Header Auth) returns `status` (`intact`, `broken` or `empty`), the number of entries, the newest hash, the first problem, the problem count and the last anchor. Add `&seq=<n>&entry_hash=<hash>` to check one receipt, or `&from=<date>&to=<date>` to count runs per outcome in a period. It returns hashes and counts only, never text, and only for that one client.

### How the hash is computed

```
entry_hash = SHA-256( prev_hash + "\n" + JSON.stringify([
  "agent-run-log/1", client_id, seq, run_id, logged_at, model, outcome,
  prompt_sha256, answer_sha256, model_calls, tool_calls, error, prompt_text, answer_text ]) )
```

`seq`, `model_calls` and `tool_calls` are numbers; every other field is a string (empty if missing). Each client's chain starts with a `prev_hash` of 64 zeros. `answer_sha256` is empty when the run was not answered, and a count of -1 means not known (the agent failed).

The Code nodes use Node's `crypto` module where the instance allows it (the n8n docs list it as available on n8n Cloud) and otherwise a built-in JavaScript SHA-256 with identical output. On self-hosted n8n, `require('crypto')` is blocked unless you set `NODE_FUNCTION_ALLOW_BUILTIN=crypto` (on the task runners if you run them as a separate service). Both paths were tested and give the same hashes.

## Setup in 5 steps

1. **Create the two Data tables.**
   - `agent_run_log`: `client_id`, `run_id`, `logged_at`, `model`, `outcome`, `prompt_sha256`, `answer_sha256`, `error`, `prompt_text`, `answer_text`, `prev_hash`, `entry_hash` as **string**; `seq`, `model_calls`, `tool_calls` as **number**.
   - `agent_run_anchors`: `anchored_at`, `client_id`, `last_hash` as **string**; `entries`, `last_seq` as **number**.

   Keep `logged_at` and `anchored_at` as strings: a date column reformats the value and breaks the hash. The Data table nodes look tables up by name with a *contains* match, so don't keep another table whose name contains `agent_run_log` (a backup copy, for example), or pick both tables from the list after import.
2. **Add credentials:** OpenAI on **Chat model**, Header Auth on **Agent request** and **Verify request**, and SMTP on the three email nodes.
3. **Fill in Settings and Daily check settings:** model (preset to `gpt-5-mini`), system message, `max_output_tokens`, `max_iterations`, `max_prompt_chars`, `store_text`, `store_text_max_chars`, `allowed_clients`, `paused_clients` (comma-separated), `max_log_attempts`, the owner and sender addresses, `email_when_intact` and `recheck_after_seconds`.
4. **Publish and test:**

```bash
curl -X POST https://<your-n8n>/webhook/agent-run-log/run \
  -H "X-Api-Key: <your webhook key>" -H "Content-Type: application/json" \
  -d '{"client_id":"acme","prompt":"What is 6 times 7?"}'

curl "https://<your-n8n>/webhook/agent-run-log/verify?client_id=acme" \
  -H "X-Api-Key: <your webhook key>"
```

   In the editor, **Agent request** has pinned example data for **Execute workflow**.
5. **Get the first anchor:** run **Every day at 03:15** by hand once.

### Tamper test (2 minutes)

1. Log three runs for `acme` with the curl above.
2. Open `agent_run_log` and change one cell of entry 2, for example `outcome` from `ok` to `error`.
3. Call the verify URL: `status` is `broken` and the first problem is at seq 2.
4. Change the cell back: `intact` again.

You can also delete entry 2 or swap the `seq` of two rows. Deleting the newest entries leaves no gap: only an earlier anchor catches that.

## Limitations (please read)

- **Detects edits, does not prevent them.** Anyone who can edit the table can change rows. A change shows up for whoever holds a later hash, such as a receipt or an anchor email. The chain does not prove who wrote an entry.
- **A rewrite needs an anchor to be caught.** Deleting the newest entries, or rewriting the chain from some point with every hash recomputed, leaves a chain that is valid on its own. Only a saved anchor or receipt catches that.
- **Parallel runs.** Data tables have no atomic append. Runs that collide retry up to `max_log_attempts`; past that the caller gets 503. Tested with 10, 20 and 40 parallel calls on the default SQLite backend, not on Postgres. For high volume, use a database with a unique key on (client_id, seq).
- **Hashes are not encryption.** A short or guessable prompt can be confirmed by guessing. Treat hashes of personal data as personal data.
- Deleting old rows breaks the chain. `logged_at` is the n8n server clock, not a trusted third-party timestamp.

## Tested with

Self-hosted n8n 2.41.6 (Node.js 24, SQLite) against a local mock of the OpenAI Chat Completions API and a local SMTP catcher: normal runs, a run with a tool call, an agent error, an empty prompt, non-ASCII text and emoji, Max Iterations reached, an invalid `client_id`, a missing key, a client outside `allowed_clients`, a paused client, stored text, 10, 20 and 40 parallel calls on one client and 20 spread over 4 clients (all chains intact), retries used up (503 without the answer, owner email), tables that can't be read or written (503), the daily check with intact and broken chains, a temporary duplicate removed during the recheck (no false alarm), one run by the real scheduler with the time set a minute ahead, and the verify endpoint. A changed field, a changed `entry_hash`, a deleted entry, swapped `seq`, a copied row, a deleted newest entry and a full rewrite were all detected; the last two only through the anchor. Every hash was recomputed independently in Python with the same result.

Not tested: the real OpenAI API, n8n Cloud (including whether `crypto` is allowed there), Postgres, queue mode, real email delivery, the 03:15 run itself, and editing cells in the Data table view in a browser (the tamper tests changed rows through n8n's REST API).

## Nodes used

Webhook, Edit Fields (Set), Code, Switch, If, Data table, AI Agent, OpenAI Chat Model, Calculator, Wait, Send Email, Respond to Webhook, Schedule Trigger, Sticky Note. All built into n8n; no community nodes.

## License

MIT. See [LICENSE](../LICENSE).

This template works on its own; you do not need Keelstamp to use it. It was built by the team behind [Keelstamp](https://keelstamp.com), which is in development.

Prepared with AI assistance. Editorial responsibility: PowerQuant ApS.
