# Reliability and enforcement boundaries

> This document describes the local revised workflows before the public export was sanitized. The JSON in `workflows/` has credential references, workflow IDs, pinned data and instance metadata removed. The 37 mocked checks below were run on the pre-sanitization files; a fresh import and live test are required for the published copies.

The architecture is Chat → Orchestrator → Research Agent → Analyst → Report writer → Markdown/HTML → Gotenberg PDF. The writer can additionally call Research Agent. Research Agent has no path back to writer, Analyst or Orchestrator, so there is no recursive workflow cycle. Agents can still repeat their available tools within an invocation.

## What code/connections enforce

| Area | Enforced behavior |
|---|---|
| Agent architecture | One AI agent in each of four workflows; exactly one Chat Trigger and three Execute Workflow triggers. |
| Parsing | Accept an object, JSON string, or one complete outer JSON fence. Reject malformed/truncated/prose-prefixed JSON rather than guessing a substring or inventing defaults. Tool decoding supports specific JSON/single-item/response/content wrappers only. |
| Provenance | A quote and all context snippets must occur in the same extracted page, after whitespace/Unicode normalization. Search snippets cannot verify a final passage. Rejected quote content is not forwarded. |
| Numeric observations | A source numeric token must match the numeric value. Subject/unit/year/measure/scope strings must occur in verified context. Pipe tables require matching headers, rows and selected columns; year-bearing headers must match the observation year. |
| Analysis | Numeric findings must reference a direct accepted observation and copy its subject/value/unit/year/measure/scope/URL/quote. Invalid findings are excluded with a limitation. Qualitative direct excerpts remain available separately. |
| Chart | Requires 2–20 distinct accepted findings. Labels and values are built from those findings. Mixed unit/measure/scope fields are rejected; cross-subject charts require matching periods, trends require one subject and distinct periods. No missing values are filled with zero. |
| Handoff audit | Specialist inputs are returned unchanged as audit fields. Before PDF generation, code compares Researcher output to Analyst input and Analyst output to writer input, including the chart. The original chat question is checked too. This detects changes after execution; it does not prevent the receiving agent from processing an altered input first. |
| Orchestrator sequence | Actual returned intermediate steps must contain one Researcher, then one Analyst, then one Report_writer call. Wrong order, missing stages or extra specialist calls block publication afterward. |
| Writer follow-up | More than one recorded Researcher call blocks publication afterward. A malformed/failed supplemental result contributes no evidence and is reported as a limitation. Additional evidence does not modify the original numerical findings/chart. |
| Report | Citations are restricted to accepted source URLs. Numerical tokens outside the allowed supplied material are rejected. Raw HTML and arbitrary images are rejected. Verified data, excerpts and limitations are appended independently of the writer's prose. |
| PDF | Full-document HTML validation precedes conversion. Runtime conversion errors cannot produce a success reply. A returned binary must start with `%PDF-`; this checks file type, not PDF layout or substantive correctness. |

## What remains prompt-based or needs human judgment

- **Exact tool-call budgets are not pre-call enforced.** Research is instructed to use at most 3 searches and 4 extractions per invocation. Writer is instructed to request research at most once. Avoiding duplicate queries/failed URLs, deciding that a gap is essential, and stopping when evidence is sufficient are prompt rules. Post-call audits cannot refund or prevent calls already made.
- `maxIterations` is configured as 8 for Orchestrator/Analyst, 12 for Research Agent, and 4 for the writer. n8n's agent iteration bound is not an exact per-tool call quota. The writer's optional Research Agent invocation has its own iteration budget.
- Source authority, independence, freshness and completeness are agent judgments. An extracted page can be wrong. Matching a quote proves where its text came from, not that the information is true.
- Literal metadata matches do not prove semantic alignment. A paragraph may contain several values; a table can have complex merged headers or footnotes. The Research Agent must select the correct context, and the Analyst must reject ambiguity. Flattened numerical PDF tables are deliberately not accepted as structured chart observations.
- Relevance labels and `comparison.comparable` originate with the agents. Code adds strict field-consistency checks but cannot establish that two publishers use equivalent methods simply because their labels match.
- Narrative checks are not a theorem prover or full fact-checker. A writer could attach an allowed source to the wrong statement, reuse an allowed number with the wrong subject, omit an important nuance, or make an unsupported nonnumeric inference. Prompts prohibit these; review the narrative against the authoritative appended table and linked source excerpts. Do not treat a successful validation flag as a guarantee of factual accuracy.
- Recorded limitations are preserved, but agents can miss an unrecognized gap. The workflow does not have a deterministic list of every subject/requirement in an arbitrary natural-language question.

## Deliberate conservative choices

Numeric findings and charts use directly stated source values. This version does not publish newly calculated values, ratios or unit conversions: the original free-form calculation contract did not reliably tie every computed value to validated inputs. The Analyst remains an AI agent, performs source/context/comparability assessment and selects findings; its original Calculator tool remains connected for checking. A later calculation feature should use explicit operand IDs and operations verified by code.

If a source only has ambiguous units/year/scope, the workflow may produce a partial report or no chart. Broader coverage should come from better source extraction and complete context, not from weakening the validators or accepting model guesses. Original country/period wording is retained in the verified table rather than normalized by the model.

Inline SVG removes remote chart loading as a PDF failure source. The `chart_url` field is retained for compatible handoff naming but contains a data URI. The writer does not copy it into Markdown; the HTML styling node inserts it from its validated input. Long labels, large tables and page breaks still need visual inspection in a real PDF.

The four tool references must be remapped after import. Credential references are preserved. No storage service, persistent cross-execution cache, extra workflow, or live-instance change is required. Intermediate steps contain source evidence and tool inputs; retain execution logs according to your own data-handling requirements.

## Official references checked

- [n8n Tools Agent options](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/tools-agent/): maximum iterations and returning intermediate steps.
- [n8n Call Workflow Tool](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/): subworkflow inputs and returned output.
- [n8n Chat Trigger source](https://github.com/n8n-io/n8n/blob/master/packages/%40n8n/nodes-langchain/nodes/trigger/ChatTrigger/ChatTrigger.node.ts): `options.responseMode: lastNode`.
- [Official Tavily node](https://github.com/tavily-ai/tavily-n8n-node): the original community package and extraction functionality.
- [Gotenberg HTML-to-PDF](https://gotenberg.dev/docs/convert-with-chromium/convert-html-to-pdf): Chromium HTML conversion and the `index.html` upload contract.

These references help check configuration; they do not substitute for testing your installed versions.
