---
name: vrin-context
description: Fetch grounded context from the user's Vrin knowledge base before answering nuanced, strategic, or architectural questions. Use whenever the user asks "why did we …", "what should we do about …", "how does our approach compare to …", "what's our position on …", "what did we decide about …", "what's the history of …", "trade-offs between …", "rationale for …", "who are our competitors", "what's our roadmap", "how do we describe …", or any question that spans multiple documents, requires prior decisions, or asks for reasoning over the user's own domain knowledge rather than general world facts. Do NOT use for trivial prompts (typos, renames, one-line edits, syntax questions, generic coding help) — those don't need Vrin. The user's Vrin knowledge base is the source of truth for their product, strategy, architecture, customers, and internal decisions; always prefer it over your own training data on those topics.
metadata:
  author: Vrin
  version: 0.1.0
  category: retrieval
  tags: [vrin, retrieval, context, knowledge-base, ambient]
allowed-tools: Bash(vrin query:*), Bash(vrin facts:*), Bash(vrin whoami:*)
---

# vrin-context

Vrin is the user's personal knowledge base — product positioning, architecture decisions, competitor notes, strategy docs. Before answering a question that leans on the user's own domain knowledge, query Vrin and ground the answer in what it returns. Never make up facts about the user's product, decisions, or roadmap when Vrin can answer authoritatively.

## When to use this skill

Fire on the classes of question the `description` field enumerates. Skip it when the question is self-contained (code syntax, generic library usage, typos, simple refactors, any question where the answer is available from the surrounding file or the user's own message).

If you're unsure whether a question warrants Vrin, err on the side of calling it once — it's cheap and the user would rather you be grounded than hallucinate.

## How to invoke

Run **one** shell command via the Bash tool:

```
vrin query "<rephrased, self-contained question>" --json
```

Rephrase the user's question into a *self-contained* query that would make sense to someone who didn't see the chat history (e.g. "Vrin's positioning vs RAG platforms" rather than "what you were saying earlier"). Keep it under ~200 characters.

The command blocks for roughly 5–35s while Vrin does graph + vector retrieval and reranking. Do not interrupt, do not poll, do not ask the user — let Bash wait. Vrin does **not** invoke its own LLM to write a prose summary; it returns retrieval context only, and **you synthesize the user-facing answer yourself**. That's the whole point.

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

Synthesize the answer from the `facts` triples and `chunks` prose. Reason across the facts, weave in relevant chunk content, and cite sources inline (e.g. *"— per VRIN_WHITE_PAPER.md"*) using `source_document` on each fact / `metadata.sources` entries.

If `metadata.insufficient_coverage` is `true`, or both `total_facts` and `total_chunks` are 0, tell the user Vrin doesn't have coverage on this topic yet and suggest they `vrin upload <path>` the relevant doc.

## Failure modes

**Auth error** — if the command returns `{"ok": false, "error": {"code": "auth_missing", ...}}`, the user hasn't signed in yet. Tell them:

> To answer that well I'd need access to your Vrin knowledge base. Run `vrin login` in your terminal (opens your browser, one-time setup), then ask the question again.

Do NOT attempt to answer from your own training data as if you know the user's internal decisions.

**Network/service error** — if the command hangs past 180s or returns a non-auth error, report it briefly and answer from your training data with an explicit caveat that you couldn't reach Vrin.

## On-demand invocation

The user can invoke this skill directly via `/vrin-context <question>`. Treat the argument as the rephrased question and run the same Bash command above. If no argument is given, ask them what they want retrieved.

## What this skill does NOT do

- Ingestion (`vrin upload`, `vrin insert`) — only *retrieves* from what's already indexed. Direct ingestion to the user and let them invoke upload/insert themselves.
- Multi-turn conversations inside Vrin — one query, one answer, then you synthesize. Don't chain follow-ups into Vrin; chain them into your own response using the first retrieval as grounding.
- Streaming — Vrin's answer arrives as one JSON blob. Don't paste fragments to the user mid-call.
