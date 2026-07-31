# USPTO Open Data Portal — MCP Server for n8n

Two n8n workflows that expose the USPTO Open Data Portal (ODP) Patent File Wrapper
API to Claude, ChatGPT, or Gemini as MCP tools — eleven tools covering
bibliographic data, prosecution history, continuity, assignments, patent term
adjustment, attorney of record, the file wrapper index, full-text search, and
OCR of any file-wrapper document.

No credentials or API keys are contained in these files.

---

## Prerequisites

| Requirement | Notes |
| --- | --- |
| **n8n with MCP Server Trigger v2** | See the warning below — v1.1 does not work. |
| **USPTO ODP API key** | Free. Requires a USPTO.gov account verified through ID.me. Get one at `data.uspto.gov` → Manage API Key. |
| **Mistral API key** | Paid, roughly $2 per 1,000 OCR pages. Only needed for the document-reader tool. |

> ### ⚠️ MCP Server Trigger v2 is required
>
> On **v1.1**, every tool parameter that uses an expression — which includes every
> `$fromAI()` call, and therefore every tool that accepts input — fails at runtime with:
>
> ```
> NodeOperationError: No bridge acquired for this context. Call acquire() first.
> ```
>
> Tools with fully static parameters still work, and the tool list still enumerates
> correctly over MCP, so the server looks healthy while every real call fails. The
> execution log is not much help either: the trigger records `success` and the failing
> tool node's error is only visible in the MCP client's response.
>
> These files ship with the trigger at **v2**. If you build this from scratch rather
> than importing, check the trigger's version before debugging anything else.

---

## Setup

### 1. Import both workflows

In n8n: **Workflows → Import from File**.

Import **`USPTO_Document_Reader.json` first**, then `USPTO_ODP_MCP_Server.json`.
Order matters — step 4 needs the sub-workflow to already exist.

### 2. Create two credentials

**Credentials → Create Credential**

| Credential | Type | Configuration |
| --- | --- | --- |
| USPTO API Key | **Header Auth** | Name: `X-API-KEY` &nbsp;·&nbsp; Value: your ODP key |
| Mistral | **Mistral Cloud API** | API key: your Mistral key |

The header name must be exactly `X-API-KEY`. ODP does not accept `Authorization: Bearer`.

### 3. Bind the credentials

Credential bindings are deliberately not included in the export, so attach them
after import.

**In `USPTO ODP MCP Server`** — attach the USPTO Header Auth credential to all ten
HTTP Request Tool nodes:

`USPTO Application Data` · `USPTO Application Metadata` · `USPTO Continuity` ·
`USPTO Transactions` · `USPTO Documents` · `USPTO Foreign Priority` ·
`USPTO Assignment` · `USPTO Patent Term Adjustment` · `USPTO Attorney` ·
`USPTO Search Applications`

**In `USPTO Document Reader`** — two nodes, two different credentials:

| Node | Credential |
| --- | --- |
| `Download Document` | USPTO Header Auth |
| `Mistral OCR` | Mistral Cloud API |

### 4. Re-point the sub-workflow reference

Open `USPTO ODP MCP Server` → the **`USPTO Read Document`** node. Its Workflow
field still carries the workflow ID from the instance this was exported from,
which does not exist in yours.

Re-select **USPTO Document Reader** from the dropdown. Leave the four input
mappings (`applicationNumberText`, `documentIdentifier`, `pageStart`, `pageEnd`)
exactly as they are.

### 5. Publish both workflows

Both must be **published**, not merely saved. A sub-workflow that is only saved
cannot be called from a live MCP server.

### 6. Connect your MCP client

Open the **MCP Server Trigger** node → **Production URL** tab → copy the URL.
It will look like `https://<your-instance>/mcp/uspto-odp`.

Add it as a custom connector (in Claude: Settings → Connectors → Add custom
connector → Remote MCP server URL). Leave the advanced OAuth fields blank.

### 7. Test

Ask your assistant:

> What is the status of patent application 17/123,456?

You should get live USPTO data back. Then exercise the OCR chain, which is the
part with the most moving pieces:

> List the file wrapper for 17/123,456 and OCR the final rejection.

---

## The tools

| Tool | Input | Returns |
| --- | --- | --- |
| USPTO Application Data | application number | Bibliographic core — title, dates, status, applicant, inventor, art unit, examiner, CPC, patent/publication numbers |
| USPTO Application Metadata | application number | The `applicationMetaData` block only |
| USPTO Continuity | application number | Parent/child continuity and provisional priority |
| USPTO Transactions | application number | PAIR transaction history |
| USPTO Documents | application number | File wrapper index with `documentIdentifier` for each document |
| USPTO Foreign Priority | application number | § 119(a)–(d) claims |
| USPTO Assignment | application number | Recorded assignments, reel/frame, conveyance |
| USPTO Patent Term Adjustment | application number | A/B/C delay, applicant delay, overlap, total |
| USPTO Attorney | application number | Attorney/agent of record, customer number |
| USPTO Search Applications | query, offset, limit | Search over `applicationMetaData.*` with a compact digest projection |
| USPTO Read Document | application number, documentIdentifier, pageStart, pageEnd | OCR'd markdown of one file-wrapper document |

Application numbers may be passed with or without punctuation — `17/123,456` and
`17123456` both work.

`USPTO Read Document` requires a `documentIdentifier`, which comes from
`USPTO Documents`. It cannot find a document by description.

---

## Notes and gotchas

**The MCP endpoint is unauthenticated by default.** The trigger ships with no
authentication set, so anyone with the Production URL can spend your USPTO quota
and your Mistral OCR budget. If that matters, set Bearer or Header auth on the
trigger node before publishing.

**OCR costs money.** `USPTO Read Document` defaults to pages 1–30. A long Office
Action with exhibits can run to hundreds of pages; pass `pageStart`/`pageEnd` to
bound it.

**`Application Data` is deliberately trimmed** to a bibliographic projection
(`fieldsToInclude: selected`) so it does not return the full record — event history,
attorney bags, and continuity would otherwise flood the model's context on every
status lookup. Those live in their own tools. To restore the full record, set
`fieldsToInclude` to `all` on that node.

**Do not "fix" the regex in the URL expressions.** They use character classes
(`[^0-9]`, `[^A-Za-z0-9]`) rather than `\d` on purpose; the escaping breaks in
n8n expressions.

**Do not replace the base64 helper with an expression.** `PDF to Base64` uses
`this.helpers.getBinaryDataBuffer()` because expression-based base64 conversion
fails under n8n's filesystem binary mode.

**Do not type a literal newline in `Format Output`.** It builds its page separator
with `String.fromCharCode(10)`; a newline inside a quoted string throws a
`SyntaxError` in the Code node.

**ODP account requirement.** Since 18 June 2026 the Open Data Portal requires a
USPTO.gov account. A further requirement for four additional profile fields takes
effect 18 August 2026 — set them in your USPTO.gov account under "Open Data
Portal" or API access may break.

---

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `No bridge acquired for this context` | MCP Server Trigger is on v1.1. Upgrade the node to v2. |
| `401` / `403` from USPTO | Header name is not exactly `X-API-KEY`, or the key is inactive. |
| Tool list appears but calls fail | Workflow saved but not published, or a credential is unbound. |
| `USPTO Read Document` fails | Sub-workflow not published, or the Workflow reference was never re-pointed (step 4). |
| A new tool doesn't appear in your client | The MCP client caches the tool list. Reconnect the connector. |
