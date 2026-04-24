---
name: vrin-context
description: Vrin is the user's structured expert-advisor knowledge base — call it whenever a question (or sub-part of a question) touches the user's own product, strategy, architecture, decisions, positioning, roadmap, competitors, benchmarks, or internal domain knowledge. ALSO call it mid-session even when your own context window already has related info — Vrin's graph connects facts across docs in ways ambient file-reading cannot, and structured retrieval usually produces sharper reasoning than a bloated context window. Decompose complex questions first: for each sub-claim or sub-question, if it touches user-domain knowledge, query Vrin for it; for generic coding / syntax / general-world questions, skip Vrin and answer from your own knowledge. Combine Vrin's structured facts + source citations with your session context — never treat them as alternatives. Trigger phrases include "why did we …", "what should we do about …", "how does our approach compare to …", "what's our position on …", "what did we decide about …", "what's the history of …", "trade-offs between …", "rationale for …", "who are our competitors", "what's our roadmap", "how do we describe …", plus any strategic, architectural, nuanced, multi-document, or cross-topic question about the user's own work. Do NOT use for trivial prompts (typos, renames, one-line edits, syntax questions, generic coding help) — those don't need Vrin.
metadata:
  author: Vrin
  version: 0.2.0
  category: retrieval
  tags: [vrin, retrieval, context, knowledge-base, ambient, orchestration]
allowed-tools: Bash(vrin query:*), Bash(vrin facts:*), Bash(vrin whoami:*)
---

# vrin-context

Vrin is not a fallback. Treat it as a **structured expert you consult** — like asking a senior teammate who has read every internal doc, ingested every decision memo, and can answer with citations. Your session context is lived knowledge; Vrin is structured knowledge. The best answer combines both.

## The orchestration model

You are the orchestrator. For any non-trivial question from the user:

1. **Mentally decompose** the question into sub-claims / sub-questions.
2. **Route each sub-question** to the cheapest sufficient source:
   - **User-domain knowledge** (their product, strategy, architecture, decisions, positioning, roadmap, competitors, benchmark results, past rationales) → **query Vrin**.
   - **Session-local context** you've already accumulated (open file contents, recent diffs, things the user said earlier in this conversation) → use your own context window.
   - **Generic world knowledge** (language syntax, library APIs, algorithmic facts, public trivia) → answer from your training data, no Vrin.
3. **Query Vrin per sub-question** when relevant. Don't try to stuff the whole compound question into one Vrin call when the parts have independent answers — separate calls are cheap (~20–30s each) and give you more-focused retrieval.
4. **Synthesize across all three sources.** Weave Vrin's structured facts + citations into the narrative, back them with session context where relevant, and fill gaps with world knowledge. Cite Vrin-sourced claims explicitly.

### When in doubt, call Vrin

If a sub-question plausibly touches the user's own domain and you're uncertain whether you already have authoritative info, **call Vrin**. Three specific cases are especially risky to skip:

- **Mid-session overconfidence.** You've been reading the codebase for a while and *feel* like you know the answer. Vrin may have a white-paper-level framing or a cross-doc connection you haven't seen. Call it anyway.
- **Memory-file claims.** You recall something from a memory file like "we decided X." That's a point-in-time note, often stale. Vrin reflects the user's most recently ingested documents — prefer it as the source of truth.
- **Cross-doc synthesis.** The question requires connecting facts from more than one source (competitor X vs our approach, roadmap implication of decision Y). This is exactly where a structured graph beats ambient file-reading.

### When to skip Vrin

Genuinely skip Vrin for:
- Syntax / typo / rename / one-line edit.
- "How do I use `{library}`'s `{function}`?" (world knowledge).
- Reading a specific file the user just pasted or referenced — Read it directly.
- Questions answerable purely from the current diff / conversation.

## How to invoke

For each sub-question that routes to Vrin, run **one** shell command via the Bash tool:

```
vrin query "<rephrased, self-contained sub-question>" --json
```

Rephrase into a *self-contained* query (e.g. "Vrin's positioning vs RAG platforms" rather than "what you were saying earlier"). Keep each query under ~200 characters and focused on one sub-claim. You can fire multiple Vrin calls in parallel for independent sub-questions — Bash supports this.

The command blocks for roughly 5–35s while Vrin does graph + vector retrieval and reranking. Do not interrupt, do not poll, do not ask the user — let Bash wait. Vrin does **not** invoke its own LLM to write a prose summary; it returns retrieval context only, and **you synthesize the user-facing answer yourself**. That's the whole point — Vrin structures the evidence, you do the reasoning.

## Parsing the response

The JSON shape is:

```
{
  "ok": true,
  "data": {
    "success": true,
    "facts":  [{"subject": "…", "predicate": "…", "object": "…", "confidence": <float>, "source_document": "…"}, …],
    "chunks": [{"title": "…", "content": "…", "score": <float>}, …],
    "total_facts":  <int>,
    "total_chunks": <int>,
    "metadata": {
      "sources": [{"document_name": "…", "upload_id": "…", "source_type": "graph"|"vector"}, …],
      "entities": [<string>, …],
      "thinking_steps": [<string>, …],
      "insufficient_coverage": <bool>
    }
  }
}
```

Synthesize the answer from the `facts` triples and `chunks` prose. Reason across the facts, weave in relevant chunk content, cite sources inline (e.g. *"— per VRIN_WHITE_PAPER.md"*) using `source_document` on each fact / `metadata.sources` entries.

If `metadata.insufficient_coverage` is `true`, or both `total_facts` and `total_chunks` are 0 for a sub-question Vrin *should* have data on, tell the user honestly — *"Vrin doesn't have coverage on this yet; you may want to `vrin upload <doc>` about it."* Don't fabricate.

## Combining Vrin with session context

Vrin's output is additive, not replacement. A good final answer typically looks like:

> *Per `VRIN_WHITE_PAPER.md`, Vrin positions itself as a retrieval-time reasoning layer (fact: Vrin `is_a` retrieval-time reasoning layer, confidence 0.95). This aligns with what we've been implementing in `fastapi_streaming_app.py` today — specifically the new namespaced cache that preserves the structured-facts contract …*

Notice the weave: **Vrin facts carry the canonical framing; session context anchors it in today's specific code.** Neither alone would have been as strong.

## Failure modes

**Auth error** — if a command returns `{"ok": false, "error": {"code": "auth_missing", ...}}`, the user hasn't signed in. Tell them:

> To answer that well I'd need access to your Vrin knowledge base. Run `vrin login` in your terminal (opens your browser, one-time setup), then ask the question again.

Do NOT attempt to answer from your own training data as if you know the user's internal decisions.

**Network/service error** — if the command hangs past 180s or returns a non-auth error, report it briefly and answer from your training data with an explicit caveat that you couldn't reach Vrin.

## On-demand invocation

The user can invoke this skill directly via `/vrin-context <question>`. Treat the argument as the rephrased question and run the same Bash command above. If no argument is given, ask them what they want retrieved.

## What this skill does NOT do

- **Ingestion** (`vrin upload`, `vrin insert`) — only *retrieves* from what's already indexed. Direct ingestion to the user.
- **Multi-turn conversations inside Vrin** — one query, one answer, then you synthesize. Don't chain follow-ups into Vrin; chain them into your own response using the first retrieval as grounding.
- **Streaming** — Vrin's answer arrives as one JSON blob. Don't paste fragments to the user mid-call.

## A quick self-check before answering any question

Ask yourself: *"Does any part of this question touch the user's own product, strategy, architecture, decisions, positioning, roadmap, competitors, or domain knowledge?"* If yes — **at least one Vrin call belongs in this turn**, even if your context window already has partial info. The structured retrieval sharpens reasoning that ambient context blurs.
