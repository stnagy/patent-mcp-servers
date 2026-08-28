# Prior Art First Look — MCP Server for n8n

Three n8n workflows exposing a **first-look (knock-out) prior art search** to Claude,
ChatGPT, or Gemini as a single MCP tool — worldwide bibliographic search, examiner-cited
references with their X/Y/A relevance categories, US claims full text, and non-patent
literature, across five free sources.

No credentials or API keys are contained in these files.

> ### ⚠️ This retrieves. It does not judge.
>
> Nothing in this kit scores relevance. Every row it returns was handed back by an API
> for the terms you supplied; none has been compared against the asserted novelty. The
> output is a **reading list, not a result set**, and the failure mode is not an empty
> answer — it is a full page of confident, irrelevant results that looks exactly like
> work product.
>
> This is not a clearance, validity, freedom-to-operate or patentability search, and it
> does not replace a professional searcher.

---

## What this can and cannot do

**Can:** multiply one disclosure into many CQL variants · worldwide bibliographic search
via EPO OPS · automatic CPC-scoped broadening when every keyword variant returns nothing ·
expansion of the best hits through **examiner-cited references**, carrying the examiner's
own X / Y / A / P / E categories · US **claims full text** via Google's
`patents-public-data` on BigQuery · OpenAlex and PubMed for non-patent literature ·
cross-leg deduplication on normalised publication numbers · a per-run verdict, coverage
statement, known-gap list and verification log.

**Cannot:** score, rank or filter for relevance · search US *description* text · search
foreign-language original text · reach any commercial database, chemical structure search,
or sequence homology · tell you that no art exists. It can only tell you what the terms
you supplied returned.

### The two failure modes, stated plainly

Both were reproduced in live testing and both are reported in the run output.

**Zero results does not mean there is no art.** Recall here is governed by vocabulary
match, not data coverage. The same structure described as "stepped" and as "gradient"
returns completely different result sets — in testing, one phrasing found the art and the
other found soil chemistry. A variant returning nothing means your wording missed.

**Thirty results does not mean thirty relevant results.** On a total vocabulary miss the
patent legs correctly return empty, but OpenAlex and PubMed — driven by those same wrong
terms — return a full page of real, on-nothing literature. The run verdict degrades and
says so, but nothing in the pipeline can prevent it.

---

## Prerequisites

| Requirement | Notes |
| --- | --- |
| **n8n with MCP Server Trigger v2** | v1.1 fails at runtime on every tool that takes input. See the root README. |
| **EPO OPS credentials** | Free. Register an app at `developers.epo.org` for a Consumer Key + Secret. The same `EPO OPS` credential the [`epo/`](../epo/) server uses works here. |
| **Google Cloud project with billing enabled** | Required for BigQuery. The free tier is 1 TiB of query processing per month; billing must still be enabled on the project. |
| **A service account with two IAM roles** | `roles/bigquery.jobUser` **and** `roles/bigquery.dataEditor`. Missing `jobUser` is the most common setup failure. |
| **A subset build per CPC area you work in** | The US claims-text leg returns nothing until you run the Subset Builder for that CPC subclass. See step 6. |

> ### ⚠️ This endpoint is authenticated — and that is not the whole control
>
> Unlike the EPO and USPTO servers in this repository, this trigger ships with
> **n8n OAuth2** authentication set, because this is the server that can spend money
> rather than merely quota. Your MCP client authorises against your n8n instance instead
> of carrying a static token.
>
> That bounds *who* can call it. It does not bound *how much* they can spend: any
> authorised caller can start a ~157 GB subset build by searching an unbuilt CPC area.
> Set a project-level daily query-bytes quota as well.

> ### ⚠️ Do not point the search at `patents-public-data` directly
>
> It works on the first try, and it scans roughly **157 GB per run** — about six searches
> inside the 1 TiB monthly free tier, for a tool whose premise is that you run it early
> and often. The Subset Builder exists to pay that scan once per CPC area. After a subset
> is built, a search scans a small clustered table and `maximumBytesBilled` at 5 GB
> becomes a real ceiling rather than a formality.
>
> **The n8n BigQuery node's `dryRun` option is not a cost guard.** Tested against live
> billing: with `dryRun: true` set, the node still executed and billed the query. It
> returned a real `jobId`, `cacheHit: false`, and 157 GB of `totalBytesProcessed`.
> `maximumBytesBilled` is the guard here — BigQuery refuses an over-ceiling job without
> charge — but that behaviour has not been independently verified in this build. Set a
> project-level daily query-bytes quota before your first run. A quota prevents; a budget
> alert only tells you afterwards.

---

## Setup

### 1. Import the three workflows

**Workflows → Import from File.** Import **`Prior_Art_Retrieval_Engine.json` first**, then
`Prior_Art_MCP_Server.json`. Order matters — step 5 needs the engine to exist.
`BigQuery_Subset_Builder.json` is independent; import it whenever.

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

Name it **`EPO OPS`**. If you already built the [`epo/`](../epo/) server, reuse it.

### 3. Create the Google service-account credential

**Add credential → `Google Service Account API`.** Paste the service account's email and
its private key. In the Google Cloud console, grant that service account **both**
`roles/bigquery.jobUser` and `roles/bigquery.dataEditor` on the project.

`jobUser` is the one people miss. Without it every BigQuery call fails with
`Access Denied: ... bigquery.jobs.create` — which this build handles gracefully, routing
the failure to a diagnostic and letting the other four sources finish, so the run looks
fine until you read `coverage.sourcesNotRun`.

### 4. Bind the credentials

**In `Prior Art First Look — Retrieval Engine`:**

| Node | Credential |
| --- | --- |
| `EPO OPS Search` | EPO OPS (OAuth2) |
| `EPO Biblio And Cited References` | EPO OPS (OAuth2) |
| `Broadened Classification Sweep` | EPO OPS (OAuth2) |
| `Check Subset Coverage` | Google Service Account |
| `BigQuery US Full-Text Search` | Google Service Account |

**In `BigQuery Prior-Art Subset Builder`** — all five BigQuery nodes take the Google
Service Account credential.

OpenAlex and PubMed need no credential. They do ask for a contact email, which is
politeness protocol, not authentication — see step 7.

### 5. Re-point the sub-workflow reference

Open `Prior Art First Look MCP Server` → the **`Prior Art First Look`** node. Its Workflow
field reads `REPLACE_WITH_YOUR_RETRIEVAL_ENGINE_ID`. Select **Prior Art First Look —
Retrieval Engine** from the dropdown. Leave the five input mappings as they are.

Then open `Prior Art First Look — Retrieval Engine` → the **`Start Subset Build`** node.
Its Workflow field reads `REPLACE_WITH_YOUR_SUBSET_BUILDER_ID`. Select **BigQuery
Prior-Art Subset Builder**. Leave the `cpcPrefix` input mapping as it is. This is the
auto-build branch — if you skip it, a search for an unbuilt CPC area fails at that node
instead of starting a build.

### 6. Set your GCP project and build a subset

Every BigQuery node in both workflows has its Project ID set to
`REPLACE_WITH_YOUR_GCP_PROJECT_ID`, and the same placeholder appears inside the SQL of
each query — including the new `Check Subset Coverage` node in the engine. Replace it in **all of them** — the resource-locator field *and* the SQL body.

Then open `BigQuery Prior-Art Subset Builder` → **`Subset Config`** → set `cpcPrefix` to a
CPC **subclass** (four characters: `C25B`, `A61K`, `C12N`, `H01M`) and run it manually.
It creates the `prior_art` dataset and the clustered `us_claims_by_cpc` table, clears that
prefix, populates it from `patents-public-data`, and reports coverage.

Each build is a one-time ~157 GB scan. Budget roughly six subclasses per month inside the
free tier. Re-running the same prefix rebuilds only that prefix.

> A CPC subclass is the four-character stem, not the full symbol. A search for
> `C25B11/032` looks for the `C25B` subset. Both sides normalise to four characters, so
> you build once per subclass, not once per group.

#### Auto-build on a cache miss — proof of concept

You do not have to build a subset by hand first — step 6's manual run is a convenience,
not a prerequisite. If a search names a CPC subclass with no subset, the engine starts the
Subset Builder for that subclass itself and returns immediately.

**The coverage check runs first.** `Check Subset Coverage` is a `COUNT(*)` against the
clustered subset table, scoped to your CPC prefix, so it costs almost nothing. Only if it
comes back non-zero does the full-text search run at all; on a miss the search is skipped
entirely rather than executed and discarded.

That check fires the build in **both** of the states that mean "no subset here": the table
exists but holds no rows for that prefix, and the table or dataset does not exist at all —
which is what a genuinely fresh install looks like, since the Subset Builder is what
creates them. The two arrive by different routes (a count of zero versus a BigQuery
`Not found` error), and the run output reports which one via `autoBuildTrigger`.

It deliberately does **not** fire on any other BigQuery failure. An over-ceiling refusal
means the table is too large, and a missing `bigquery.jobs.create` permission means it is
unreachable; in neither case is the table absent, and starting a build would not help.

**It does not wait for the build and then search.** That would be the obvious design, and
it is the wrong one here: a build is a multi-minute scan against a 60-second MCP timeout,
and a timeout loses the whole run — including the EPO and non-patent legs that had already
succeeded. Returning immediately with an honest "US claims text was not searched, re-run
in a few minutes" costs you one re-run; blocking costs you everything.

**It does not wait for the build.** A build is a one-time ~157 GB scan taking minutes,
and the MCP caller times out at 60 seconds. The run that triggered the build still
reports honestly that US claims text was not searched — `autoBuildStarted: true`,
`autoBuildPrefix`, and a reason telling you to re-run. **Re-run the same search a few
minutes later** to pick up the new subset.

> ### ⚠️ Three things this spends, and one it does not guard
>
> - **Each auto-build is ~157 GB.** That is roughly six inside the 1 TiB monthly free
>   tier, now spent without asking you first.
> - **There is no in-flight lock.** Re-running before a build finishes fires a *second*
>   build for the same prefix — two calls, ~314 GB. Wait for the first to finish.
> - **Any authorised caller can trigger one.** The trigger ships with n8n OAuth2 set, so
>   a stranger with the URL cannot; anyone you have authorised can. Authentication is not
>   a spend control — the daily query-bytes quota is.
>
> The one guard that *is* in place: the Subset Builder refuses any prefix that is not a
> four-character CPC subclass. `STARTS_WITH(cp.code, '')` is true for every row, so an
> empty or malformed prefix would have inserted the entire US corpus and billed for it.
> `Validate CPC Prefix` throws instead.

### 7. Set your contact details

Open the **`OpenAlex Search`**, **`PubMed Search`** and **`PubMed Summaries`** nodes and
replace `REPLACE_WITH_YOUR_EMAIL` with your own address, and `REPLACE_WITH_YOUR_TOOL_NAME`
with your own identifier.

Do this before your first run. NCBI enforces rate limits and blocks on the `tool`/`email`
pair, and OpenAlex uses `mailto` the same way. Left unchanged, your traffic is attributed
to whoever the placeholder names.

### 8. Publish

All workflows that get called must be **published**, not merely saved. A sub-workflow that
is only saved cannot be called from a live MCP server — and n8n runs the *published*
version, so republish after every edit or your changes will not take effect.

### 9. Connect your MCP client

Open the **Prior Art First Look MCP Server** trigger node → **Production URL** → copy it
(`https://<your-instance>/mcp/prior-art-first-look`). Add it as a custom connector.

The trigger ships with **n8n OAuth2** authentication, so your client will run an
authorisation flow against your n8n instance on first connect rather than accepting a
token you paste in. If you would rather use a static Bearer token, change
`Authentication` on the trigger node before publishing — the workflow does not depend on
which one you pick.

### 10. Test

> Do a first-look prior art search on a PEM electrolyzer with a graded-porosity titanium
> porous transport layer. CPC C25B11/032.

A good call from the model supplies several synonyms and a CPC symbol. A careless one
supplies two synonyms and no CPC, and returns a materially worse sweep that looks
identical. Read `runVerdict` first.

---

## The tool

One tool, `Prior Art First Look`. Recall depends far more on these inputs than on
anything in the workflow.

| Input | What to supply |
| --- | --- |
| `conceptTerms` | Comma-separated core structural or functional nouns — the things the invention **is**. e.g. `porous transport layer, electrolysis` |
| `synonymTerms` | Comma-separated **alternative wordings** for the distinguishing feature. The single most important input. `stepped, graded, gradient, bimodal` are the same structure and return different art. |
| `cpc` | A CPC symbol, e.g. `C25B11/032`. Strongly recommended: it rescues a vocabulary mismatch, it is the zero-result fallback, and its subclass selects the US claims-text subset. Without it the US full-text leg cannot run. |
| `dateBefore` | `YYYYMMDD` priority or filing cutoff. Optional. |
| `disclosure` | The inventor's own words. Echoed back as the asserted novelty; never rewritten. |

### What comes back

| Field | Meaning |
| --- | --- |
| `runVerdict` | `COMPLETE` / `PARTIAL` / `DEGRADED` / `UNRELIABLE`. **Read this first.** `UNRELIABLE` means no patent data was searched at all. `DEGRADED` means every variant missed and the non-patent results are unvalidated. |
| `coverage` | Sources run, not run, and **answered-with-zero**, variants run, zero-result and errored variants with fault codes, whether broadening fired, seed counts, cross-leg merges, per-source status for BigQuery, OpenAlex and PubMed |
| `coverage.knownGaps` | The standing limitations, including that nothing scores relevance |
| `autoBuildStarted` / `autoBuildPrefix` / `autoBuildTrigger` | Present in `diagnostics` when this run started a subset build. `autoBuildTrigger` says whether the table was empty for that prefix or absent entirely. US claims text was **not** searched on this run — re-run in a few minutes. |
| `patentHits` | Deduplicated on normalised publication number; `sources` and `alsoSeenAs` show where each came from |
| `nplHits` | OpenAlex and PubMed, plus examiner-cited non-patent literature |
| `verificationLog` | Every record in, kept, and **dropped** — with the reason |
| `scopeLimitation` | First-look only; reproduce it when reporting results |
| `dutyOfDisclosureNote` | Finding art creates obligations — 37 C.F.R. § 1.56 |

---

## Quirks these workflows already work around

Each was found by running against live data. Don't "clean them up."

**A green run is not evidence that anything happened.** This bit three separate times in
one build session. `neverError: true` on the OPS nodes turned an XML fault into a
"successful" response the parser read as an empty result set — so a `CLIENT.RobotDetected`
403 was reported as fifteen honest zero-result queries. The nodes now use `fullResponse`
and classify every reply as **ok / empty / throttled / errored** before anything else runs.

**`SERVER.EntityNotFound` and HTTP 404 mean "empty", not "broken".** EPO uses 404 for an
empty result set. Classified as empty deliberately — with the consequence that a genuine
404 fault would be indistinguishable from an empty hit list.

**OPS throttles on call volume, not just rate.** Up to 40 calls per run at ~150 ms was
enough to trigger robot detection. Variants are capped at 4 and seeds at 4, with 3-second
`Wait` nodes between OPS calls. The kit is deliberately slower than it could be.

**Seeds are ranked by jurisdiction, WO → EP → US → GB/DE/FR/CA/AU → rest.** Truncating to
the first four seeds unranked once dropped citation recall from 24 hits to zero, because
the first four happened to be CN publications, which rarely carry search-report citations.
This is a proxy for "carries a search report", **not** a relevance ranking.

**Google drops the leading zero on US publication numbers.** A shared normaliser
re-pads them so the BigQuery and OPS legs can be deduplicated against each other.

**OPS serialises a missing `@cited-phase` as the literal string `"undefined"`.** Guarded.

**`CREATE TABLE` is DDL and returns zero rows**, and n8n skips downstream nodes on an empty
branch — so the first Subset Builder run reported success and created nothing. Every
BigQuery node in that workflow has `alwaysOutputData` set. Don't remove it.

**The subset query returns a coverage row.** That is what lets "zero hits in a built
subset" be told apart from "no subset exists for this CPC area" — which would otherwise
be an indistinguishable, and dangerous, zero.

**A source that answered zero has answered.** Every leg emits its own status record —
`ok` / `empty` / `errored` — and the report reads that, rather than inferring "ran" from
whether any rows came out. Inferring it from the row count reported a legitimate zero as
a source that failed, which is the exact distinction the rest of this kit exists to draw.
PubMed is where it bites: it is a biomedical index driven by the same concept and synonym
terms as the patent legs, so a mechanical or materials disclosure matching nothing there
is the correct answer, not a fault. Such a run is `COMPLETE`, with the source listed in
`coverage.sourcesZeroResult`.

**A zero-result PubMed search still issues the esummary call with an empty id list**, and
NCBI refuses it. That refusal is expected on this path and is classified as `empty`, not
`errored`.

---

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `No bridge acquired for this context` | MCP Server Trigger is on v1.1. Upgrade to v2. |
| `runVerdict: UNRELIABLE`, fault code `CLIENT.RobotDetected` | OPS is throttling. Wait, then re-run. Don't remove the `Wait` nodes. |
| Every OPS call fails with an auth error at once | Token request misconfigured — check grant type is Client Credentials and auth is Basic Auth header. |
| `Access Denied: ... bigquery.jobs.create` | Service account is missing `roles/bigquery.jobUser`. |
| `ProjectId must be non-empty` aborts the whole run | A BigQuery node still holds the placeholder. Parameter validation happens before execution, so `onError` cannot catch it. |
| BigQuery reports "no subset covers this CPC area" | Expected on the first search of a CPC area. A build should have started automatically — look for `autoBuildStarted: true` and re-run in a few minutes. |
| `autoBuildStarted` never appears, and the run errors at `Start Subset Build` | That node still holds `REPLACE_WITH_YOUR_SUBSET_BUILDER_ID`. See setup step 5. |
| `Access Denied: ... bigquery.jobs.create`, and no build started | Correct. A permission failure is not a missing subset — fix the IAM role, then re-run. |
| Query refused over `maximumBytesBilled`, and no build started | Also correct. The table is too large for the ceiling, not absent. Raise the ceiling or narrow the CPC area. |
| `Refusing to build. '' is not a valid CPC subclass` | The search supplied no CPC symbol, or one that is not a four-character subclass. Deliberate — an empty prefix would insert the entire US corpus. |
| Two builds ran for the same CPC area | You re-ran before the first finished. There is no in-flight lock; each build is ~157 GB. |
| Subset Builder reports success but the table is empty | `alwaysOutputData` was removed from the BigQuery nodes. |
| `runVerdict: DEGRADED` and off-topic literature | Your vocabulary missed. That is the tool working — the NPL results are unvalidated, as the verdict says. |
| PubMed listed in `coverage.sourcesZeroResult` | It ran and matched nothing. Expected for any disclosure that is not biomedical — PubMed is searched with your patent vocabulary. Not a fault, and not a reason to change the contact email or tool name. |
| Changes to a workflow have no effect | You saved but did not publish. Sub-workflow calls run the published version. |
| The tool doesn't appear in your client | MCP clients cache the tool list. Reconnect the connector. |
| The client cannot connect, or asks to authorise repeatedly | The trigger uses n8n OAuth2. Re-add the connector and complete the authorisation flow; a token pasted as a Bearer credential will not work unless you change `Authentication` on the trigger node. |

---

## Attribution and licence conditions

**Google Patents Public Data** (`patents-public-data.patents.publications`) is licensed
**CC BY 4.0** by IFI CLAIMS Patent Services and Google. Attribution is required wherever
you reproduce results derived from it. The workflow carries the attribution string in
`coverage.bigQuery.attribution`; keep it when you copy results out.

**EPO OPS** is used under your own key and the EPO's fair-use terms — currently a 4 GB
weekly threshold on the free tier. Verify against `epo.org` before relying on the figure.

**OpenAlex** data is CC0; the underlying articles are not. **PubMed** records are public
domain; the underlying articles are not. Check each source before reuse.

Not affiliated with, endorsed by, or supported by the EPO, the USPTO, Google, or the NIH.
