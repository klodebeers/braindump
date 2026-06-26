---
name: agentic-ai-playbook
description: >-
  Fact-checked build-lifecycle reference for designing, building, reviewing, or evaluating
  agentic AI / LLM-agent systems. Use when the task involves tool/function calling, RAG and
  grounding, the ReAct loop, single- vs multi-agent orchestration, durable execution
  (LangGraph / OpenAI Agents SDK / Google ADK), the Model Context Protocol (MCP), agent
  safety (OWASP LLM Top 10, excessive agency, prompt injection, least privilege, approval
  gates), observability/tracing, or agent evaluation (SWE-bench, RAGAS, LLM-as-judge).
  Apply it to ground design and review decisions in primary sources rather than hype.
---

# Agentic AI Engineering Playbook

A practitioner reference that turns a 2026 podcast roadmap into a build-lifecycle guide, with
every load-bearing claim checked against primary sources and labeled by confidence tier.

## When to use this skill

Invoke when the user is:
- **Designing** an agentic system (choosing single- vs multi-agent, tool schemas, RAG strategy,
  state/durability, MCP integration).
- **Reviewing** an agent design or PR for safety/reliability gaps (least privilege, approval gates,
  read/write separation, prompt-injection / excessive-agency exposure).
- **Evaluating** an agent (trajectory vs outcome, SWE-bench / RAGAS / LLM-as-judge, observability).
- Asking "how should X be done" for any agentic-AI topic the playbook covers.

Do **not** invoke for unrelated LLM tasks (basic prompting, non-agent chatbots) or non-AI work.

## How to apply it

1. **Read the reference** before advising: [`agentic-ai-engineering-playbook.md`](../../../agentic-ai-engineering-playbook.md)
   (repository root). It is organized as Phases 0–11 (foundations → tools → RAG → ReAct →
   orchestration → infrastructure → safety → observability → evaluation → maturity → governance).

2. **Respect the confidence tiers — do not flatten them.** The playbook marks every claim:
   - ✅ **Verified** — passed 3-vote adversarial verification against a primary source. Safe to assert.
   - 🔹 **Sourced** — primary-sourced and quote-audited, but not adversarially stress-tested. Assert with attribution.
   - ⚪ **Consensus** — practitioner guidance / synthesis. Present as a sensible default, not fact.
   When you cite the playbook, carry the tier through; never upgrade a 🔹/⚪ claim to a certainty.

3. **Carry the two key corrections** (the podcast got these wrong):
   - **OWASP:** *Excessive Agency* is **LLM06:2025** (not LLM08; LLM08 is "Vector and Embedding
     Weaknesses"). *Prompt Injection* is LLM01:2025.
   - **"Basic RAG is dead/unsafe" is a slogan, not a finding.** The defensible claim: naive
     single-shot RAG is inadequate for complex/large-corpus or high-stakes use and carries a real
     injection/poisoning surface — so move to hybrid/agentic retrieval with reranking, citations,
     groundedness checks, and access control — but RAG is *extended, not replaced*.

4. **Default to the contested-debate framing.** On single- vs multi-agent, present both sides
   (Anthropic pro / Cognition con) with OpenAI's "start with one agent" as the reconciling rule —
   don't assert a universal winner.

5. **Check currency.** Framework APIs and the MCP / OWASP specs move fast; the playbook's specifics
   were current as of 2026-06-26. Re-verify version-specific details before relying on them. Two
   known stale spots: the OpenTelemetry GenAI-spans spec has relocated, and Arize Phoenix is
   source-available (Elastic License 2.0), not OSI "open source."

## Provenance & limits

The playbook derives from an unverified Substack podcast; its value is the independent
re-grounding, not the source. The companion transcript (`complete-agentic-ai-roadmap-2026-transcript.md`)
is a capture of Substack-generated captions, **not** an audio-verified transcription. The ⚪ tier is
opinion/consensus by design. See the playbook's own method note for exactly what each marker rests on.
