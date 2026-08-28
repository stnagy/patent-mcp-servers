# Patent MCP Servers for n8n

n8n workflows that expose the **EPO Open Patent Services** and the **USPTO Open Data
Portal** to Claude, ChatGPT, or Gemini as MCP tools — 24 tools covering worldwide
patent search, prosecution history, families, legal status, continuity, assignments,
patent term adjustment, full text, OCR of file-wrapper and published documents, and a
composed **first-look prior art sweep** across five free sources.

Import the JSON, attach your own credentials, publish, and point your MCP client at
the resulting URL.

**No credentials or API keys are contained in any file in this repository.**

---

## The three servers

The first two wrap an API. The third composes several of them into one search and is a
different kind of thing — read its caution before installing it.

| | [EPO OPS](epo/) | [USPTO ODP](uspto/) | [Prior Art First Look](prior-art/) |
| --- | --- | --- | --- |
| **Covers** | Worldwide — 100+ jurisdictions, EP Register | US applications and patents | Worldwide patents + non-patent literature |
| **Tools** | 12 | 11 | 1 |
| **Search** | CQL, worldwide and over the EP Register | Full-text over `applicationMetaData.*` | Many CQL variants, CPC broadening, examiner-citation expansion |
| **Full text** | Claims and description, **EP/WO only** | Via file-wrapper OCR | US **claims** via BigQuery, per CPC subclass built |
| **Prosecution** | Register events and procedural steps | PAIR transactions, continuity, assignments, PTA, attorney | — |
| **OCR** | Published document pages, incl. search reports | Any file-wrapper document | — |
| **API key** | Free — `developers.epo.org` | Free — `data.uspto.gov` (needs an ID.me-verified USPTO.gov account) | EPO OPS, plus a Google Cloud project with **billing enabled** |

Each directory has its own README with full setup, the tool table, the API quirks
each workflow already works around, and a troubleshooting table. **Read the one for
the server you're installing** — the setup steps are not interchangeable, and both
have ordering requirements that will bite you if skipped.

They are independent. Install any one alone, or all three. Prior Art First Look reuses
the same `EPO OPS` credential as the EPO server if you already have it.

---

## Things that apply to more than one

> ### ⚠️ MCP Server Trigger v2 is required
>
> On v1.1, every parameter in Expression mode — which is every `$fromAI()` call, and
> therefore every tool that takes input — fails at runtime with:
>
> ```
> NodeOperationError: No bridge acquired for this context. Call acquire() first.
> ```
>
> The tool list still enumerates correctly, so the server looks healthy while every
> real call dies. The trigger records `success` in the execution log; the actual error
> is only visible in the MCP client's response. These files ship with the trigger at v2.

> ### ⚠️ The MCP endpoint is unauthenticated by default
>
> All three triggers ship with no authentication set, so anyone with the Production URL
> can spend your USPTO/EPO quota, your Mistral OCR budget, and — with Prior Art First
> Look — your BigQuery bytes. If that matters, set Bearer or Header auth on the trigger
> node before publishing.
>
> Prior Art First Look raises the stakes on this: it now builds a missing CPC subset
> automatically, so an unauthorised caller can trigger a ~157 GB BigQuery scan simply by
> searching an art area you have not built yet.

> ### ⚠️ Retrieval is not judgment
>
> None of these tools scores relevance, and Prior Art First Look is the one where that
> matters most: it returns a reading list, not a result set. Zero results means your
> vocabulary missed, not that no art exists — and a full page of results is not evidence
> that any of them are on point. See [`prior-art/README.md`](prior-art/README.md).

---

## Quick start

1. **Import the sub-workflow first**, then the server workflow. Order matters — the
   server references the sub-workflow by ID.
2. **Create and bind your credentials.** Bindings are deliberately excluded from these
   exports, so every HTTP node needs its credential attached after import.
3. **Re-point the `Read Document` node** at your own copy of the sub-workflow. The
   exported reference is a placeholder or a foreign workflow ID.
4. **Publish both workflows** — not merely save. A sub-workflow that is only saved
   cannot be called from a live MCP server.
5. **Copy the trigger's Production URL** and add it to your MCP client as a custom
   connector.

Per-server specifics — credential types, exact node lists, test prompts — are in
[`epo/README.md`](epo/README.md), [`uspto/README.md`](uspto/README.md) and
[`prior-art/README.md`](prior-art/README.md).

Prior Art First Look adds two steps of its own: replace `REPLACE_WITH_YOUR_GCP_PROJECT_ID`
everywhere it appears (the project field **and** inside the SQL), and put your own contact
address in place of `REPLACE_WITH_YOUR_EMAIL` before your first run — NCBI and OpenAlex
attribute traffic to whatever that field says.

---

## Cost

The EPO and USPTO APIs are free, subject to quota. Two things are not.

**OCR.** Both document readers use Mistral OCR at roughly **$2 per 1,000 pages**, and the
USPTO reader defaults to pages 1–30. Bound long documents with `pageStart`/`pageEnd`. If
you don't need OCR, skip the document-reader workflow and the Mistral credential entirely
— the other tools work without it.

**BigQuery.** Prior Art First Look reads Google's `patents-public-data`, which is free to
query up to 1 TiB of processed bytes per month and billed per terabyte after that. Queried
naively it scans about **157 GB per search** — roughly six searches a month. The included
Subset Builder pays that scan once per CPC subclass and leaves later searches small. Set a
project-level daily query-bytes quota before your first run; the n8n node's `dryRun`
option is **not** a cost guard and was observed executing and billing the query anyway.

Searching an art area with no subset now **starts that build automatically** — a ~157 GB
scan, without a confirmation step, triggerable by anyone who can reach the endpoint. The
daily query-bytes quota is the control that actually bounds this. Set it first.

---

## What these cannot do

**EPO file inspection is not available from any EPO API.** Art. 94(3) communications,
applicant replies, examining-division minutes, and opposition briefs are not exposed by
OPS. The Register tells you an event happened; it does not give you the document. Three
EPO tools hand off a deep link to the Register doclist instead of reporting a dead end.

**EPO full text is EP/WO only.** A US number sent to Claims or Description returns
`CLIENT.InvalidCountryCode`. Use the USPTO server for US text.

**Nothing here scores relevance.** These are retrieval tools. They surface documents an
API returned for the terms supplied; deciding whether any of them matter is the reading
you still have to do. Prior Art First Look is a first look — not a clearance, validity,
freedom-to-operate or patentability search, and not a substitute for a professional
searcher.

---

## License

MIT — see [LICENSE](LICENSE).

Not affiliated with, endorsed by, or supported by the EPO, the USPTO, Google, or the NIH.
API behaviour, quotas, and account requirements are theirs to change.

Results from Prior Art First Look's BigQuery leg derive from **Google Patents Public
Data**, licensed CC BY 4.0 by IFI CLAIMS Patent Services and Google. Attribution is
required wherever you reproduce them.

Nothing in this repository is legal advice, and using it creates no attorney-client
relationship.
