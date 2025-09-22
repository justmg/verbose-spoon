# verbose-spoon

## GAO Bid Protest Monitor workflow

This repository contains an n8n workflow (`workflows/gao-bid-protest-monitor.json`) that watches the Government Accountability Office (GAO) bid protest decision page, captures newly posted decisions, downloads the associated PDF, and generates a structured briefing for contracting and GovCon professionals.

### Key capabilities

- Polls the GAO “Recent Bid Protest Decisions” listing on a daily cadence (configurable via the `Daily Schedule` Cron node).
- Normalizes the listing feed, tracks previously processed decisions using workflow static data, and only processes genuinely new items after the initial bootstrap run.【F:workflows/gao-bid-protest-monitor.json†L7-L91】【F:workflows/gao-bid-protest-monitor.json†L93-L191】
- Retrieves each decision detail page, extracts usable narrative text, resolves the PDF download URL, and prepares a rich prompt for summarization.【F:workflows/gao-bid-protest-monitor.json†L193-L305】
- Calls OpenAI’s Chat Completions API (via the built-in credential type) to draft a briefing that highlights: an executive summary, why the decision matters for contracting professionals, why it matters for the broader GovCon community, and interesting tidbits.【F:workflows/gao-bid-protest-monitor.json†L307-L359】
- Downloads the official GAO decision PDF (when available) and merges it with the generated summary so downstream nodes can deliver both the narrative and the source document.【F:workflows/gao-bid-protest-monitor.json†L361-L480】
- Crafts a ready-to-send email that includes the AI briefing, helpful metadata, and the GAO PDF when available, then sends it through the Email Send node using your SMTP credential.【F:workflows/gao-bid-protest-monitor.json†L482-L575】

### Importing the workflow into n8n

1. Clone or download this repository and open the `workflows/gao-bid-protest-monitor.json` file.
2. In your n8n instance, navigate to **Workflows → Import from File**, select the JSON file, and confirm the import.
3. Open the imported workflow and supply an OpenAI credential on the **“Call OpenAI for Summary”** HTTP Request node. The node is preconfigured to use the `openAiApi` credential type; create or select a credential that has access to the Chat Completions endpoint.【F:workflows/gao-bid-protest-monitor.json†L307-L337】
4. Adjust the **Daily Schedule** Cron node if you prefer a different run time or frequency.【F:workflows/gao-bid-protest-monitor.json†L9-L31】
5. (Optional) Update the summarization model, temperature, or token budget by editing the `jsonBody` expression on the OpenAI node.【F:workflows/gao-bid-protest-monitor.json†L317-L333】

### First run bootstrap & resetting state

- The workflow uses the workflow-level **global static data** store to remember which GAO decision IDs have already been processed. On the very first execution it records the current listing and exits without generating summaries so that you start from a clean slate.【F:workflows/gao-bid-protest-monitor.json†L93-L191】
- To reprocess all decisions (or recover from stale state) open the **Identify New Decisions** Function node and set `staticData.bootstrapComplete` back to `false` via the “Execute Node → Reset” option, or run the workflow once with the “Execute Workflow → Reset workflow data” command in n8n. This clears the cache and causes the next scheduled run to rebuild it from scratch.

### Configuring email delivery

The workflow finishes by emailing each new decision as soon as it is summarized:

1. Open the **Craft Decision Email** Function node to adjust the default sender (`fromEmail`), recipient (`to`), and any CC/BCC values. You can also tweak the subject template or body layout in this node.【F:workflows/gao-bid-protest-monitor.json†L482-L548】
2. Attach an SMTP credential to the **Send Decision Email** node. It uses the built-in `smtp` credential type; create or pick a credential that is authorized to send from the address configured above.【F:workflows/gao-bid-protest-monitor.json†L550-L575】
3. (Optional) If you need to route the content elsewhere (e.g., Slack, Teams, an archive) branch from the Craft node before the email send.

### Customizing downstream delivery

The exported workflow stops after the **Send Decision Email** node. You can branch before or after the email step to deliver the enriched JSON (summary, metadata, OpenAI usage) and binary PDF (when available) to additional systems such as Slack, Notion, Google Drive, or databases.【F:workflows/gao-bid-protest-monitor.json†L361-L575】

### Error handling and resilience tips

- The GAO site occasionally changes markup. If the workflow stops finding PDF links or summary text, adjust the extraction logic inside the **Prepare Decision Content** Function node (look for the `collectStrings` helper and candidate field arrays).【F:workflows/gao-bid-protest-monitor.json†L217-L303】
- If GAO blocks automated requests, consider throttling the Cron schedule, adding an n8n `Wait` node between requests, or proxying through a compliant network that follows GAO’s terms of service.
- When OpenAI returns an error, the HTTP Request node will surface it in the execution log. You can set `response.response.neverError` to `true` if you prefer to handle non-2xx responses downstream.【F:workflows/gao-bid-protest-monitor.json†L311-L333】

### Security considerations

- The OpenAI credential lives entirely within n8n’s credential vault. No API keys are stored in the workflow JSON exported from this repository.【F:workflows/gao-bid-protest-monitor.json†L307-L337】
- The workflow does not persist GAO content outside of the execution data. If you need a longer-lived archive, add storage nodes after the merge step.

### Next steps

- Attach notification nodes (e.g., Slack, email) to automatically deliver daily digests.
- Feed the summary into a knowledge base or CRM so capture managers and counsel can search historical protests.
- Swap the OpenAI node for another LLM provider if you prefer different pricing or hosting—only the HTTP Request node needs to change.

