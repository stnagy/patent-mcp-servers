# Patent MCP Servers for n8n

n8n workflows that expose the **EPO Open Patent Services** and the **USPTO Open Data
Portal** to Claude, ChatGPT, or Gemini as MCP tools — 23 tools covering worldwide
patent search, prosecution history, families, legal status, continuity, assignments,
patent term adjustment, full text, and OCR of file-wrapper and published documents.

Import the JSON, attach your own credentials, publish, and point your MCP client at
the resulting URL.

**No credentials or API keys are contained in any file in this repository.**

---

## The two servers

| | [EPO OPS](epo/) | [USPTO ODP](uspto/) |
| --- | --- | --- |
| **Covers** | Worldwide — 100+ jurisdictions, EP Register | US applications and patents |
| **Tools** | 12 | 11 |
| **Search** | CQL, worldwide and over the EP Register | Full-text over `applicationMetaData.*` |
| **Full text** | Claims and description, **EP/WO only** | Via file-wrapper OCR |
| **Prosecution** | Register events and procedural steps | PAIR transactions, continuity, assignments, PTA, attorney |
| **OCR** | Published document pages, incl. search reports | Any file-wrapper document |
| **API key** | Free — `developers.epo.org` | Free — `data.uspto.gov` (needs an ID.me-verified USPTO.gov account) |

Each directory has its own README with full setup, the tool table, the API quirks
each workflow already works around, and a troubleshooting table. **Read the one for
the server you're installing** — the setup steps are not interchangeable, and both
have ordering requirements that will bite you if skipped.

They are independent. Install either one alone, or both.

---

## Two things that apply to both

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
> Both triggers ship with no authentication set, so anyone with the Production URL can
> spend your USPTO/EPO quota and your Mistral OCR budget. If that matters, set Bearer
> or Header auth on the trigger node before publishing.

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
[`epo/README.md`](epo/README.md) and [`uspto/README.md`](uspto/README.md).

---

## Cost

The EPO and USPTO APIs are free, subject to quota. OCR is not: both document readers
use Mistral OCR at roughly **$2 per 1,000 pages**, and the USPTO reader defaults to
pages 1–30. Bound long documents with `pageStart`/`pageEnd`. If you don't need OCR,
skip the document-reader workflow and the Mistral credential entirely — the other
tools work without it.

---

## What these cannot do

**EPO file inspection is not available from any EPO API.** Art. 94(3) communications,
applicant replies, examining-division minutes, and opposition briefs are not exposed by
OPS. The Register tells you an event happened; it does not give you the document. Three
EPO tools hand off a deep link to the Register doclist instead of reporting a dead end.

**EPO full text is EP/WO only.** A US number sent to Claims or Description returns
`CLIENT.InvalidCountryCode`. Use the USPTO server for US text.

---

## License

MIT — see [LICENSE](LICENSE).

Not affiliated with, endorsed by, or supported by the EPO or the USPTO. API behaviour,
quotas, and account requirements are theirs to change.
