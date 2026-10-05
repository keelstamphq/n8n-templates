# AI drafts, the client approves (n8n template)

Draft replies to customer messages with an AI agent and send them only after the client's approver says yes. The approver needs no n8n account.

File: `client-approval-v2.json` (workflow name: "Draft replies with an AI agent and send them after client approval"). Import it in n8n with **Workflows → Import from file**, or paste the JSON onto the canvas.

## What it does

1. **Receive and check.** Your helpdesk, form or another workflow sends `POST /webhook/client-approval/draft` (Header Auth) with `client_id`, `task` (the customer's message) and `recipient_email`, plus optional `subject` and `reference`. A bad request gets HTTP 400, an unknown client 404, an inactive client 403 and a client without an approver email 409. Otherwise the request gets a row in `approval_requests` and the caller gets 202 with a `request_id`. If a Data table can't be read or written, the answer is 503 and nothing is drafted.
2. **Draft.** The AI Agent drafts the reply in the client's voice, using the system message from **Settings** and the client's `instructions`. Max Iterations and the token limit also come from **Settings**. Later rounds include the previous draft and the approver's requested changes. The draft gets a SHA-256 fingerprint.
3. **Ask the approver and wait.** The approver gets an email with the customer's message, the draft and a link to a form (Wait node, On Form Submitted). In the form they can approve, edit and approve, request changes or reject, and add a comment. Opening the link only shows the form; only submitting it decides, so mail scanners and link previews can't approve. A reminder goes out `reminder_before_hours` before the deadline. No answer by the deadline means **expired**, never approved.
4. **Record the decision.** The row gets the status, the decision, who the link was sent to, the time, the comment, the round and the SHA-256 of the approved text. If that write fails, nothing is sent. **Request changes** goes back to the agent, up to `max_rounds` times; after that the request is closed as `revision_limit`. Rejected, expired and revision-limit requests email you.
5. **Do the approved action.** The example emails the approved reply to the customer. Just before sending, the text to send is fingerprinted again and compared with the approved fingerprint. If they differ, nothing is sent and you get an email. After sending, the fingerprint of the sent text is stored.
6. **If anything fails.** Model errors, empty drafts and email or Data table failures close the request as failed. The approved action is not run, and you get an email.
7. **What is waiting?** `GET /webhook/client-approval/pending` (Header Auth, optional `?client_id=`) returns the open requests with their deadlines. Every morning at 08:00 (instance time zone) one digest email lists the waiting requests. No email is sent when nothing is waiting.

Clients with `approval_required` set to false skip the form. The decision is still recorded, as `approved_by_setting`.

## Setup in 5 steps

1. **Create the two Data tables.**
   - `approval_clients` (one row per client): `client_id`, `client_name`, `approver_email`, `instructions` (string), `expiry_hours` (number), `approval_required`, `active` (boolean). An empty `approval_required` counts as true. `active` = false blocks the client.
   - `approval_requests` (written by the workflow): `request_id`, `client_id`, `reference`, `subject`, `status`, `decision`, `action_status`, `approver`, `decided_by`, `decided_at`, `expires_at`, `draft_sha256`, `approved_sha256`, `sent_sha256`, `comment`, `history` (string), `round` (number), `edited_by_approver` (boolean).
2. **Add credentials:** OpenAI on **Chat model**, SMTP on the five email nodes, and Header Auth on **New draft request** and **Pending requests**.
3. **Fill in Settings and Digest settings:** company name, sender, your email, `default_expiry_hours`, `reminder_before_hours`, `max_rounds`, `max_task_chars`, `max_iterations`, `max_output_tokens` and the system message. The file ships with a 48-hour deadline, a reminder 12 hours before it and 3 revision rounds. A client's `expiry_hours` overrides the deadline. **Chat model** is preset to `gpt-5-mini`; change it to the model you use.
4. **Publish the workflow.** The form links only work while it is published.
5. **Test it:**

```bash
curl -X POST https://<your-n8n>/webhook/client-approval/draft \
  -H "X-Api-Key: <your webhook key>" -H "Content-Type: application/json" \
  -d '{"client_id":"acme","task":"Hi, are you open on Saturday? Sam","recipient_email":"sam@example.com","subject":"Opening hours","reference":"TICKET-1042"}'
```

You get HTTP 202 with a `request_id`, and the approver email arrives a few seconds later. **New draft request** has pinned sample data for a manual run in the editor.

To act on approval in another way, replace **Send the approved reply** with your own step (Gmail, a helpdesk reply, a CMS post). Keep **Fingerprint the text to send** and **Same text as approved?** in front of it.

## Limitations (please read)

- **Anyone who has the link can answer.** `decided_by` is the address the link was sent to, not a verified login. For more control, set Basic Auth on **Wait for the approver** and share the password separately.
- The deadline is checked about once a minute, so expiry can come up to a minute late.
- An answer from an older version of the form is ignored, and the current version keeps waiting.
- If the same form is submitted twice at the same moment, one answer is recorded and the other gets an error page from n8n.
- Each waiting request is an open execution in n8n's database until it ends.
- If the send step fails, the reply may still have reached the recipient. You get an email saying so.
- The reply field in the form shows only about two lines. The Wait node has no option to change that.
- The agent can misread a message. That is what the approval step is for.

## Tested with

Self-hosted n8n 2.41.6 (Node.js 24, SQLite) against a local mock of the OpenAI Chat Completions API and a local SMTP catcher: approve unchanged, edit and approve, request changes, an answer from an older form, the revision limit, reject, deadline with reminder and expiry, two simultaneous submissions, link-preview requests with browser and bot user agents (no decision), a changed or missing link signature (401), invalid requests and unknown or inactive clients, model errors, `approval_required` = false, the status endpoint, the digest, text changed after approval (blocked), a refused SMTP send, missing Data tables, HTML in the customer message (escaped) and an empty draft.

Not tested: the real OpenAI API, n8n Cloud, Postgres, queue mode, restarting n8n while an approval is waiting, real email delivery and how mail clients show the email, real link scanners (only simulated user agents), a person clicking in a real browser (the form was rendered in headless Chrome and submitted with curl), and the 08:00 digest on the real scheduler.

## Nodes used

Webhook, Edit Fields (Set), Code, If, Switch, Data table, AI Agent, OpenAI Chat Model, Crypto, Send Email, Wait, Respond to Webhook, Schedule Trigger, Sticky Note. All built into n8n; no community nodes.

## License

MIT. See [LICENSE](../LICENSE).

This template works on its own; you do not need Keelstamp to use it. It was built by the team behind [Keelstamp](https://keelstamp.com), which is in development.

Prepared with AI assistance. Editorial responsibility: PowerQuant ApS.
