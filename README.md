# n8n templates

Three free workflow templates for [n8n](https://n8n.io). They are for agencies and freelancers who run an AI agent for several paying clients and need three things: a budget per client, the client's approval before a reply goes out, and a record of every agent run that can be checked later. Each folder contains one workflow file and a README with setup steps, what was tested and known limitations.

| Folder | What it does |
|---|---|
| [`01-per-client-spend-cap`](01-per-client-spend-cap/) | Runs an OpenAI agent with a monthly budget per client. Each run reserves its worst-case usage before the model is called and books the measured token usage afterwards. An unknown or paused client gets HTTP 403, a client without enough budget left gets HTTP 429, and every other client keeps working. |
| [`02-client-approval-wait-link`](02-client-approval-wait-link/) | An AI agent drafts a reply in the client's voice. The client's approver gets the draft by email with a link to a form and can approve, edit, ask for changes or reject, without an n8n account. Only approved text is sent, and its SHA-256 fingerprint is checked again just before sending. |
| [`03-hash-chained-agent-run-log`](03-hash-chained-agent-run-log/) | Logs every AI agent run (answered, failed or refused) per client in a SHA-256 hash chain in a Data table. A daily check recomputes every chain and emails you the first changed, missing or reordered entry. A verify endpoint returns one client's chain status. |

## Import into n8n

1. Download the `.json` file from the folder (open the file on GitHub and use **Download raw file**, or clone this repository).
2. In n8n, choose **Workflows → Import from file** and select the file. You can also copy the JSON and paste it onto an empty canvas.
3. Follow the setup steps in the folder's README and in the yellow sticky note on the canvas: create the Data tables, add your own credentials, fill in the **Settings** nodes, then publish the workflow.

The files contain no credentials, API keys or personal data. Email addresses and URLs in them are placeholders. Every webhook is set to Header Auth, so you create that credential in your own n8n instance before you publish.

## Tested with

Self-hosted n8n 2.41.6 on Node.js 24 with the default SQLite database, against a local mock of the OpenAI Chat Completions API and a local SMTP catcher. The files were imported with the n8n command line (`n8n import:workflow`). Not tested: the real OpenAI API, n8n Cloud, Postgres, queue mode, real email delivery and import through the editor. All three templates need an n8n version with Data tables. All nodes are built into n8n; no community nodes are needed. Each folder's README lists what was tested.

## Keelstamp

The templates work on their own; you do not need Keelstamp to use them. They were made by the team behind [Keelstamp](https://keelstamp.com), which is in development.

## License

MIT. See [LICENSE](LICENSE).

Prepared with AI assistance. Editorial responsibility: PowerQuant ApS.
