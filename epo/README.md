# EPO Open Patent Services — MCP Server for n8n

Two n8n workflows exposing the EPO's Open Patent Services (OPS) to Claude, ChatGPT,
or Gemini as MCP tools — worldwide patent search, INPADOC families and legal status,
European Patent Register records, published full text, CPC classification, number
conversion, and OCR of published document pages.

No credentials or API keys are contained in these files.

---

## What this can and cannot do

**Can:** CQL search worldwide and over the EP Register · bibliographic data for 100+
jurisdictions · claims and description (EP/WO) · INPADOC family mapping · INPADOC legal
status · EP Register procedural events and steps · CPC lookup · number conversion ·
OCR of published document pages **including search reports**.

**Cannot: EPO file inspection.** Art. 94(3) communications, applicant replies,
examining-division minutes and opposition briefs are not exposed by OPS or any other
EPO API. The Register tells you an event happened; it does not give you the document.

Three tools therefore hand off a deep link instead of reporting a dead end:

```
https://register.epo.org/application?number=EP{APPNUM}&lng=en&tab=doclist
```

Note this is keyed on the **application** number, not the publication number.
`EP2954056` will not resolve; `EP14749020` will.

---

## Prerequisites

| Requirement | Notes |
| --- | --- |
| **n8n with MCP Server Trigger v2** | v1.1 fails at runtime on every tool that takes input. |
| **EPO OPS credentials** | Free. Register an app at `developers.epo.org` for a Consumer Key + Secret. |
| **Mistral API key** | Paid, roughly $2 per 1,000 OCR pages. Only needed for EPO Read Document. |

> ### ⚠️ MCP Server Trigger v2 is required
>
> On v1.1, every parameter set to Expression mode — which includes every `$fromAI()`
> call — fails with `NodeOperationError: No bridge acquired for this context.` The tool
> list still enumerates correctly, so the server looks healthy while every real call
> dies. These files ship with the trigger at v2.

---

## Setup

### 1. Import both workflows

**Workflows → Import from File.** Import **`EPO_Document_Reader.json` first**, then
`EPO_OPS_MCP_Server.json`. Order matters — step 4 needs the sub-workflow to exist.

### 2. Create the EPO OPS credential

**Credentials → Add credential → `OAuth2 API`** (the generic one).

| Field | Value |
| --- | --- |
| Grant Type | **Client Credentials** |
| Access Token URL | `https://ops.epo.org/3.2/auth/accesstoken` |
| Client ID | your OPS **Consumer Key** |
| Client Secret | your OPS **Consumer Secret** |
| Scope | *leave empty* |
| Authentication | **Send as Basic Auth header** |

Name it **`EPO OPS`**.

Both settings matter. *Client Credentials* is the machine-to-machine grant, so there is
no browser consent step and no redirect URL; n8n fetches and refreshes the token itself,
which is necessary because OPS tokens are short-lived. *Send as Basic Auth header* is
required because OPS expects `Authorization: Basic base64(key:secret)` on the token
request — if n8n posts the credentials in the body instead, the token call fails and
every tool dies at once with an auth error.

### 3. Create the Mistral credential

**Add credential → `Mistral Cloud API`** → paste your key. Only needed for OCR.

### 4. Bind the credentials

**In `EPO OPS MCP Server`** — attach the `EPO OPS` OAuth2 credential to all eleven
HTTP Request Tool nodes:

`EPO Biblio` · `EPO Claims` · `EPO Description` · `EPO Family` · `EPO Legal Status` ·
`EPO Register` · `EPO List Documents` · `EPO Classification` · `EPO Convert Number` ·
`EPO Search Patents` · `EPO Search Register`

**In `EPO Document Reader`** — two nodes, two credentials:

| Node | Credential |
| --- | --- |
| `Download Page` | EPO OPS (OAuth2) |
| `Mistral OCR` | Mistral Cloud API |

### 5. Re-point the sub-workflow reference

Open `EPO OPS MCP Server` → the **`EPO Read Document`** node. Its Workflow field reads
`REPLACE_WITH_YOUR_EPO_DOCUMENT_READER_ID`. Select **EPO Document Reader** from the
dropdown. Leave the three input mappings as they are.

### 6. Publish both

Both must be **published**, not merely saved. A sub-workflow that is only saved cannot
be called from a live MCP server.

### 7. Connect your MCP client

Open the **EPO OPS MCP Server** trigger node → **Production URL** → copy it
(`https://<your-instance>/mcp/epo-ops`). Add it as a custom connector.

### 8. Test

> Get the bibliographic data for EP2954056.

Then exercise the OCR loop, which has the most moving parts:

> List the documents for EP2954056, then read the search report pages.

---

## The tools

| Tool | Input | Returns |
| --- | --- | --- |
| EPO Biblio | publication number | Title, applicants, inventors, IPC/CPC, dates, priorities |
| EPO Claims | publication number | Full claim text (EP/WO only) |
| EPO Description | publication number | Full specification (EP/WO only; very large) |
| EPO Family | publication number | INPADOC family members worldwide |
| EPO Legal Status | publication number | INPADOC legal event timeline |
| EPO Register | EP/PCT publication number | Register biblio, events, procedural steps |
| EPO List Documents | publication number | Document inventory with page counts and section start-pages |
| EPO Read Document | documentLink, pageStart, pageEnd | OCR'd markdown, page-labelled, max 20 pages |
| EPO Classification | CPC symbol | Title, definition, notes (XML) |
| EPO Convert Number | number, formats | original ↔ epodoc ↔ docdb |
| EPO Search Patents | CQL query, range | Worldwide search with bibliographic rows |
| EPO Search Register | CQL query, range | EP Register search with full records |

Publication numbers are accepted in EPODOC form; a trailing kind code (`A1`, `B1`) is
stripped automatically.

---

## OPS quirks these workflows already work around

Each of these was found by running the tools against live data. Don't "clean them up."

**OPS is not uniform on content negotiation.** Every node sends
`Accept: application/json` — except `EPO Classification`, which must not. The CPC
service returns `400 RESTEASY003515: Malformed parameters` if you send it, and returns
XML regardless, so that node is set to `responseFormat: text`.

**Family must not request `/biblio`.** Adding the constituent triggers
`SERVER.LimitedServerResources — please request bibliographic data in smaller chunks`
on any sizeable family. The plain family endpoint returns members only, which is what
family mapping needs.

**Both search tools use `/search/biblio`.** The bare `/search` endpoint returns
publication numbers with no titles or applicants.

**OPS serves ONE page per image request.** The `Range` header does not honour a span —
and it doesn't fail either, it silently returns a different page than you asked for.
`EPO Document Reader` therefore loops one page at a time and concatenates in order.
A 10-page range is 10 OPS calls and 10 OCR pages; the cap is 20.

**Full text is EP/WO only.** A US number to Claims or Description returns
`CLIENT.InvalidCountryCode`. Use USPTO tooling for US text.

**`SERVER.EntityNotFound` on Claims is often correct**, not a fault — an application
withdrawn or refused before grant has no full text.

**Responses are large.** OPS returns XML-mapped JSON — every value under `$`, every
attribute under `@`. A 115-member family came back at 211 KB. Watch your quota: the
free non-paying tier is documented as 4 GB (sources disagree on whether per week or per
month — check your own account page).

---

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `No bridge acquired for this context` | MCP Server Trigger is on v1.1. Upgrade to v2. |
| Every tool fails with an auth error at once | Token request misconfigured — check grant type is Client Credentials and auth is Basic Auth header. |
| `CLIENT.InvalidCountryCode` | Non-EP/WO number sent to Claims or Description. Expected. |
| `SERVER.LimitedServerResources` | Requesting too much in one call; narrow the constituents or range. |
| `SERVER.EntityNotFound` on Claims | No published full text — often withdrawn before grant. Expected. |
| Classification returns a JSON parse error | The Accept header was re-added to that node. Remove it. |
| `EPO Read Document` fails | Sub-workflow not published, or the Workflow reference was never re-pointed (step 5). |
| A tool doesn't appear in your client | MCP clients cache the tool list. Reconnect the connector. |
