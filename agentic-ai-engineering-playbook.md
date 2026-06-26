# The Agentic AI Engineering Playbook (2026)

> A practitioner reference for building production agentic AI systems — derived from the
> *"Complete Agentic AI Roadmap 2026"* podcast, then **fact-checked against primary sources**
> and reorganized from a conversation into a build lifecycle.
>
> Companion file: [`complete-agentic-ai-roadmap-2026-transcript.md`](./complete-agentic-ai-roadmap-2026-transcript.md) (the raw source).

---

## How to read this document

This playbook follows the **lifecycle of building an agent** — foundations → tools → knowledge →
reasoning → orchestration → infrastructure → safety → observability → evaluation → production.
Each topic gives you *what it is, why it matters, best practices, pitfalls,* and *sources*.

Where the podcast **overstated or garbled** something, you'll see a **⚠️ Podcast correction** callout.

### Confidence legend

Every load-bearing claim carries a confidence marker so you know how much weight it can hold:

| Marker | Meaning |
|--------|---------|
| ✅ **Verified** | Confirmed by 3-vote adversarial verification against a **primary source** (official docs, spec, OWASP, peer-reviewed paper). Survived an explicit attempt to refute it. |
| 🔹 **Sourced** | Backed by a primary source that was fetched and quoted, but not put through the full adversarial vote. High but not stress-tested. |
| ⚪ **Consensus** | Widely-held practitioner guidance / framework convention. Sensible default, but not anchored to one decisive primary source. |

**Time-sensitivity:** Framework APIs (LangGraph, OpenAI Agents SDK, Google ADK) and the MCP spec
evolve fast. The MCP spec cited is the **2025-11-25** revision; OWASP is the **2025** edition. Both
were current as of **2026-06-26**. Re-check before relying on version-specific details.

---

## Phase 0 — Foundations: what actually makes it an "agent"

**What it is.** An agent is not a chatbot with API keys. The working distinction: an agent uses an
LLM to **decide its own control flow** — it reasons over a goal, chooses tools, observes results,
and loops until done — rather than running a fixed, human-authored script. Anthropic's widely-cited
framing separates *workflows* (LLMs orchestrated through predefined code paths) from *agents* (LLMs
that dynamically direct their own processes and tool usage).

**Why it matters.** This is the mindset shift the whole roadmap rests on: you are building software
that makes runtime decisions you didn't explicitly encode. That buys flexibility and costs you
determinism — which is why every later phase (safety, observability, evaluation) exists.

**Best practices.**
- Start with the **simplest thing that works**. Don't reach for an agent when a single LLM call,
  a retrieval step, or a fixed workflow solves the problem. ⚪
- Add agency only where you genuinely need open-ended decision-making over a variable number of steps. ⚪
- Treat "give the model autonomy" as a dial, not a switch — more autonomy means more surface area to secure and observe. ⚪

**Pitfalls.**
- *Agent-washing*: calling a linear prompt chain an "agent." It adds cost and unpredictability with no benefit.
- Equating model intelligence with system reliability. A smart model in an unguarded system fails faster, not safer.

**Sources.** Anthropic, *Building Effective Agents* (2024) — https://www.anthropic.com/research/building-effective-agents ⚪ (canonical, widely cited).

---

## Phase 1 — Tools: giving the model hands (function calling)

**What it is.** Tool/function calling lets the LLM emit a structured request (a JSON object) that
your code executes — querying a database, calling an API, running code — and returns the result to
the model. The **tool schema** (a JSON Schema) defines each tool's name, description, and typed
parameters.

**Why it matters.** The model never touches your systems directly; it only proposes well-typed
calls against a contract you define. The quality of that contract largely determines reliability.
The podcast's "strict restaurant menu" analogy is apt: the schema is the menu, and the parameters
must be specified flawlessly.

**Best practices.**
- **Constrain hard with the schema.** Use enums, types, ranges, `required`, and
  `additionalProperties: false`. Prefer providers' **strict / structured-output** modes that
  *guarantee* the output conforms to your schema rather than hoping the model complies. 🔹
- **Write the description for the model, not for humans.** Tool/param descriptions are prompt
  surface; ambiguity here is the #1 cause of wrong-tool / wrong-arg calls. ⚪
- **Keep the toolset small and orthogonal.** Overlapping tools cause selection errors. ⚪
- **Validate inputs server-side anyway.** The schema reduces but does not eliminate bad arguments. ⚪

**Pitfalls.**
- ⚠️ **Podcast correction (minor, but real):** the podcast says sending the string `"100"` instead
  of integer `100` means *"the whole system crashes."* It only crashes if **you let it** — i.e. you
  pass unvalidated model output straight to an API. With strict/structured output + server-side
  validation, type mismatches are caught, not fatal. The lesson is *validate*, not *fear*.
- Giant "do-everything" tools with free-form string params — you've just reinvented an unvalidated API.

**Sources.** OpenAI Structured Outputs / function-calling docs; provider strict-mode docs. Overview:
*Structured Outputs with LLMs* — https://towardsdatascience.com/structured-outputs-with-llms-json-mode-function-calling-and-when-to-use-each/ ⚪ (blog; treat schema-design specifics as practitioner consensus).

---

## Phase 2 — Tool execution lifecycle & failure handling

**What it is.** Everything that happens *around* a tool call: validating arguments, executing,
catching errors, retrying, self-correcting, and escalating. The 2026 emphasis is that the
interesting engineering is in the **failure path**, not the happy path.

**Why it matters.** Models hallucinate arguments and pick wrong tools. A production agent must treat
tool errors as normal, recoverable events.

**Best practices.**
- **Feed errors back to the model.** On a `400`/validation error, return the error message into the
  loop so the model can correct its arguments and retry — a bounded self-correction loop. ⚪
- **Bound the retries.** Cap attempts and add backoff; an unbounded retry loop is both a cost
  incident and a potential infinite loop (see Phase 7 — "called a tool 50 times"). ⚪
- **Define escalation.** When retries are exhausted, or the action is high-impact, **stop and hand
  off to a human** rather than guessing (see Phase 6). ⚪
- **Make tools idempotent where possible**, so a retry can't double-charge / double-write. ⚪

**Pitfalls.**
- Retrying without changing anything (no error fed back) — same failure, more money.
- Silent fallback: when a tool fails, the model invents a plausible result to keep going (see
  *silent failures*, Phase 7). This is worse than a loud crash.

**Sources.** Practitioner consensus across framework guides ⚪. Reliability framing corroborated by
OWASP's mediation guidance (Phase 6).

---

## Phase 3 — Knowledge: RAG done right (and the "RAG is dead" fact-check)

**What it is.** Retrieval-Augmented Generation grounds the model in your data: retrieve relevant
content, put it in context, generate an answer tied to it. The canonical *naive* pipeline is three
stages — **index** (chunk → embed → store), **retrieve** (top-K vector similarity), **generate**
(stuff chunks into the prompt). 🔹 *(Gao et al., RAG survey, arXiv:2312.10997.)*

**Why it matters.** Grounding is how you fight hallucination and use private/current data. The
podcast made a strong claim here that deserved — and got — a higher evidentiary bar.

### ⚠️ Podcast correction — the big one: *"basic RAG is completely dead / fundamentally unsafe"*

This was fact-checked with **three independent investigators** (one prosecuting the claim, one
defending RAG, one defining terms). They **converged**:

> **The absolute claim is NOT supported by any Tier-1 primary source.** No major lab, vendor, or
> peer-reviewed survey (Anthropic, OpenAI, Microsoft, AWS, Google, OWASP, NIST, the RAG surveys)
> states that the retrieve-then-generate architecture is inherently broken or should be abandoned.
> ✅ *(convergent finding across all three investigations.)*

Mapped to the four interpretations you should always separate:

| Interpretation | Verdict | Evidence |
|---|---|---|
| **(1) RAG is *inherently* flawed** | **Contradicted** | Every survey frames advanced/modular/agentic RAG as *"a progression and refinement within the RAG family"* — extending naive RAG, not replacing it. 🔹 *(Gao et al. 2312.10997; Singh et al. 2501.09136)* |
| **(2) Weak/naive implementations fail** | **Strongly supported — this is the real claim** | Anthropic's Contextual Retrieval cut retrieval-failure **49%** (and **67%** with reranking) — i.e. naive RAG's failures are *technique-fixable*. 🔹 *(Anthropic, Sept 2024)* |
| **(3) Unsuitable for certain use cases** | **Supported, narrow** | Long-context can substitute for RAG *only* when the whole corpus fits the context window; enterprise KBs routinely exceed it (cost ~1000×, latency 30s+). 🔹 *(Ragie, 2025; Anthropic ~200K-token threshold)* |
| **(4) Newer approaches win for specific requirements** | **Supported, conditional** | *"Neither RAG nor long-context LLMs are a silver bullet; their relative strengths depend on model size, context length, task type…"* 🔹 *(LaRA, ICML 2025, arXiv:2502.09977)* |

**The genuine kernel of truth — security.** The one place the claim leans toward an *architectural*
flaw is safety: retrieved, untrusted text is treated as trusted context.
- **PoisonedRAG**: injecting just **5 malicious texts** into a million-document store achieved a
  **~90% attack success rate**, and existing defenses were "insufficient." ✅ *(USENIX Security 2025, arXiv:2402.07867)*
- **OWASP LLM01:2025** states plainly that RAG and fine-tuning *"do not fully mitigate prompt
  injection vulnerabilities,"* and names poisoned-document retrieval as an attack. ✅

**The defensible, scoped claim** (use this, not the slogan):

> *Naive single-shot, embed→top-K→stuff RAG is increasingly inadequate for complex or high-stakes
> enterprise queries and carries a real, under-mitigated injection/poisoning surface. Production
> systems should move to **hybrid/agentic retrieval with reranking, contextual chunking,
> groundedness checks, citations, and access control** — but RAG as a retrieval-grounding pattern is
> alive, extended rather than replaced, and remains the cheaper, more scalable choice whenever the
> knowledge base exceeds the context window.*

Note: even the headlines that shout "RAG is dead" walk it back in their own body text —
*"RAG is not dead. The architecture most enterprises used to implement it is."* 🔹

**Best practices (the parts of the podcast that hold up ✅-in-spirit).**
- **Generate citations / attribution.** Tie every claim back to the retrieved chunk. Strongly
  recommended; AWS Bedrock, Azure, and others build this in. 🔹
- **Score groundedness / faithfulness**, and **refuse when confidence is low** rather than guessing.
  This is sound — just frame it as *"RAG done well,"* not *"basic RAG is illegal."* 🔹
- **Upgrade retrieval before blaming the model:** hybrid (vector + keyword) search, reranking,
  query rewriting, better chunking. 🔹
- **Secure the retrieval path:** treat retrieved content as untrusted input (Phase 6).

**Sources.** Gao et al. arXiv:2312.10997 · Singh et al. arXiv:2501.09136 · LaRA arXiv:2502.09977 ·
Anthropic *Contextual Retrieval* (2024) · Microsoft *RAG and Generative AI — Azure AI Search*
(updated 2026-06-08) · PoisonedRAG arXiv:2402.07867 · OWASP LLM01:2025.

---

## Phase 4 — The reasoning loop: ReAct

**What it is.** **ReAct = Reason + Act + Observe.** The agent interleaves *reasoning traces*
(thoughts) with *actions* (tool calls) and *observations* (results), looping until the goal is met.
It's the foundational single-agent loop the podcast describes correctly.

**Why it matters.** It's the atom of agentic behavior — most orchestration patterns are ReAct loops
composed together.

**Best practices & limits.**
- Great default for tool-using tasks; keep the loop bounded (max steps) and observable. ⚪
- **Limitation:** long ReAct loops bloat context and can drift or loop — which is the motivation for
  multi-agent decomposition (Phase 5) and durable state (Phase 6 infra). ⚪

**Sources.** Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*, ICLR 2023
(arXiv:2210.03629) ⚪ *(origin is well-established; not independently re-verified in this run — verify the citation if quoting it formally).*

---

## Phase 5 — Orchestration: single agent vs. the "agentic org chart"

**What it is.** When one ReAct loop isn't enough, you decompose into roles: a **router** (directs the
request), **planner** (breaks the goal into steps), **executor** (does the work), **verifier**
(checks the executor), and **human-review** gate (final approval). The podcast's "kitchen brigade"
analogy (head chef / sous chef / expediter) maps cleanly onto planner / executor / verifier.

**Why it matters.** This is the most *contested* design decision in the field — so the playbook
presents both sides rather than picking one.

### The genuine debate (cite both sides)

- **Pro multi-agent (Anthropic).** Their multi-agent research system (lead agent + parallel
  subagents, each with its own context window) **outperformed a single agent by ~90.2%** on
  breadth-first, parallelizable research tasks. ✅ *(Anthropic, "How we built our multi-agent research system")*
- **Anti multi-agent (Cognition).** *"Don't Build Multi-Agents"* — running multiple agents in
  collaboration produces **fragile systems** because *"decision-making ends up being too dispersed
  and context isn't able to be shared thoroughly enough."* ✅ *(Cognition / Walden Yan, 2025)*
- **The reconciling rule (OpenAI).** *"Start with one agent whenever you can. Add specialists only
  when they materially improve capability isolation, policy isolation, prompt clarity, or trace
  legibility."* ✅ *(OpenAI orchestration guide — verified verbatim)*

**Best practices.**
- **Default to a single agent.** Split only when the *contract* changes (a clear capability/policy
  boundary), not because an org chart feels tidy. ✅-anchored
- When you do split, give each agent a **narrow, well-specified role** and minimal tools/permissions
  (this also serves least-privilege — Phase 6).
- For parallelizable, breadth-first work (research, broad search), multi-agent genuinely shines;
  for tightly-coupled, context-heavy tasks, a single agent is often more reliable.

**Pitfalls.**
- ⚠️ **Podcast nuance:** the podcast presents the planner/executor/verifier/router stack as *the*
  way to do it. It's *a* powerful pattern — but the primary sources show multi-agent is a genuine
  tradeoff, not a default. Over-decomposition disperses context and multiplies failure surfaces.

---

## Phase 5b — Does routing actually save tokens?

**What it is.** The podcast claims routing to specialists *saves* tokens vs. one mega-agent, because
you only load the tools/context needed for each micro-task.

**Assessment.** ⚪ **Directionally plausible, context-dependent — not a universal law.** Loading only
the relevant tools/policy per step *can* reduce per-call context vs. stuffing every instruction into
one giant prompt. **But** multi-agent adds its own overhead: handoff/coordination tokens, repeated
context-passing, and (per Anthropic's own report) multi-agent systems can burn **many times** the
tokens of a single agent. So: routing can cut tokens *for a fixed task* by trimming context, while
multi-agent *architectures overall* often cost more. Both can be true. Measure for your workload;
don't treat "routing saves money" as guaranteed.

---

## Phase 6 — Infrastructure: durable execution, handoffs, and MCP

### Durable execution & state — LangGraph
**What/why.** LangGraph provides persistence via **checkpointers** (persist a thread's graph state
as checkpoints — short-term, thread-scoped memory enabling human-in-the-loop, time-travel,
conversation continuity, fault tolerance) and **stores** (long-term, cross-thread memory). This lets
agents **resume after interruption / crash** instead of restarting. ✅
- **Caveat:** the OSS library is single-process; checkpointing is not automatically
  "production-grade distributed durability" at scale. ✅ *(noted critique)*
- The podcast's "server crashes 3 hours into a migration, agent resumes where it left off" example
  is accurate in spirit. ✅

### Handoffs — OpenAI Agents SDK
**What/why.** A **handoff** delegates a task to another agent and **transfers control** (the
specialist owns subsequent turns). It's implemented as a tool the model sees — a handoff to "Refund
Agent" surfaces as `transfer_to_refund_agent`. The alternative pattern is **agents-as-tools** (a
manager stays in control and calls specialists as bounded capabilities). ✅
- Accuracy note: handoffs are **model-mediated tool calls**, not unconditional programmatic transfers. ✅

### Google ADK
**What/why.** Open-source (Apache 2.0) framework with **sequential / parallel / loop / LLM-routing**
workflows plus built-in session, state, event, and memory management (with rewind and migrate);
persistence via `DatabaseSessionService`. ✅

### Model Context Protocol (MCP)
**What/why.** ⚠️ The podcast's *"USB for AI"* framing is **accurate**. MCP — created at **Anthropic**
by David Soria Parra and Justin Spahr-Summers — is an **open standard using JSON-RPC 2.0** that
standardizes how apps share context with and expose tools to LLMs, replacing fragmented custom
integrations with one protocol. ✅
- **Architecture:** three roles — **Hosts** (LLM apps that initiate connections), **Clients**
  (connectors inside the host), **Servers** (services providing context/capabilities). ✅
  *(The simpler Anthropic announcement uses a two-role client/server framing; the full spec adds Host.)*
- **Security — critical:** MCP **cannot enforce security at the protocol level.** It specifies trust
  principles (user consent & control, data privacy, tool safety, sampling controls) that
  *implementors* must enforce — including treating **tool descriptions/annotations as untrusted
  unless from a trusted server.** ✅ This directly feeds Phase 7.

**Sources.** LangGraph docs (durable-execution, persistence) · OpenAI Agents SDK (handoffs,
orchestration, multi_agent) · Google ADK docs · MCP spec 2025-11-25 · Anthropic MCP announcement.

---

## Phase 7 — Safety & security: the part that matters more than the model

**What it is.** Agents *act*, so their failures aren't embarrassing sentences — they're deleted
records, leaked files, wrong trades. The podcast's core mindset claim — *"agent safety matters
infinitely more than chatbot safety"* — is directionally right and worth internalizing.

### ⚠️ Podcast correction — OWASP numbering (the second big one)

> The podcast says *Excessive Agency* is **"LLM08."** **Wrong for the current list.** In the **OWASP
> Top 10 for LLM Applications 2025**, **Excessive Agency is LLM06:2025**. **LLM08:2025 is "Vector and
> Embedding Weaknesses."** The "LLM08" label is left over from the **2023** edition. **Prompt
> Injection is LLM01:2025** (the top risk). ✅ *(verified against the OWASP PDF + GenAI pages)*

**Excessive Agency (LLM06:2025).** Harm from an LLM granted broad capability to call functions /
interface with systems. Three root causes: **excessive functionality, excessive permissions,
excessive autonomy.** ✅

**Prompt Injection (LLM01:2025).** Inputs alter the model's behavior in unintended ways — **direct**
(user input) and **indirect** (from external sources like websites, files, or *retrieved
documents*). ✅ The podcast's "white-text-in-an-email → agent exfiltrates tax docs" scenario is a
textbook **indirect prompt injection** combined with **excessive agency**, and is realistic.

### Mitigations (OWASP-backed ✅)
- **Least privilege:** limit each agent's permissions to the minimum necessary. ✅
- **Minimize functionality:** expose only the tools actually needed. ✅
- **Separate read tools from write tools.** The podcast's advice — a research/read agent must never
  share permissions with one that moves money or modifies databases — is exactly the least-privilege
  + minimized-functionality principle. ✅-aligned
- **Human-in-the-loop approval for high-impact actions** (money movement, data deletion, prod-code
  changes). ✅
- **Complete mediation:** enforce authorization in **downstream systems**, not by trusting the LLM's
  decision. Don't let "the agent decided it was allowed" be your access control. ✅
- Treat all retrieved/tool-returned content as **untrusted input** (ties to MCP + RAG poisoning). ✅

**Sources.** OWASP Top 10 for LLM Applications 2025 (LLM01, LLM06, LLM08 pages + PDF); OWASP Agentic
AI – Threats and Mitigations.

---

## Phase 8 — Observability & tracing: grade the trajectory, not just the answer

**What it is.** For agents you must evaluate the **whole trajectory** — every reason/act/observe
step — not just the final answer. Tracing records the exact node-by-node sequence, tool choices,
errors, recoveries, and token costs.

**Why it matters.** It's how you catch **silent failures** — e.g. the agent couldn't find a document
so it fabricated a statistic; the final output *looks* perfect, but the trace shows where the
reasoning derailed. The podcast describes this well.

**Best practices.**
- Adopt **OpenTelemetry GenAI semantic conventions** for agent/framework spans (standardized
  `invoke_agent` / `create_agent` span schema) so traces are portable across tools. 🔹
- Use a tracing/eval tool (e.g. **Arize Phoenix**, open-source) to inspect inputs/outputs per step,
  tool selection, and cost. 🔹
- Watch for: wrong tool choice, ignored constraints, runaway loops (the "called search 50 times,
  huge bill" failure), and silent fallbacks.

**Sources.** OpenTelemetry GenAI agent-span semantic conventions —
https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/ 🔹 ·
Arize Phoenix — https://github.com/Arize-ai/phoenix 🔹.

---

## Phase 9 — Evaluation & benchmarks

**What it is.** Systematic measurement of agent quality — both end-to-end and trajectory-level.

**Key benchmarks/frameworks (the podcast names these correctly).**
- **SWE-bench** — **2,294** real GitHub issue+PR tasks across **12 Python repos**; success is scored
  by the repo's **own unit tests** (execution-based, not text similarity). The agent clones the repo,
  localizes the bug, patches, and must pass the tests. 🔹 *(Jimenez et al., arXiv:2310.06770)*
- **RAGAS** — **reference-free** evaluation of RAG pipelines; metrics include **answer relevance**
  and **faithfulness** (can every claim be traced to retrieved context?). 🔹 *(arXiv:2309.15217)*
- **LLM-as-a-judge** — strong judge models (e.g. GPT-4) reached **>80% agreement** with humans
  (~human-human level) on MT-Bench/Chatbot Arena. 🔹 *(Zheng et al., arXiv:2306.05685)*
  - **But** trajectory judging is harder: across 12 judges on agent trajectories, **no single judge
    was best across all settings** — judges have biases; don't treat one as ground truth.
    🔹 *(AgentRewardBench, arXiv:2504.08942)*

**Best practices.** Evaluate the trajectory (did the planner understand the goal? did the verifier
catch the executor? was human review triggered at the right threshold?), not just the destination.
Combine execution-based checks (SWE-bench style), grounding metrics (RAGAS style), and LLM-judges —
and cross-check the judges.

**Sources.** arXiv:2310.06770 · arXiv:2309.15217 · arXiv:2306.05685 · arXiv:2504.08942.

---

## Phase 10 — Production maturity: chatbot vs. agentic system

**The takeaway (podcast nails this).** The differentiator in 2026 isn't *"I built an AI agent."*
Setting up a chatbot with API keys is baseline. The mark of engineering maturity is being able to
say:

- I built **advanced RAG with citations and groundedness checks**;
- I built **multi-step orchestration** (and can justify single- vs. multi-agent);
- I implemented **human approval gates** and **least-privilege** tool access;
- I have **node-by-node tracing** and trajectory evaluation;
- **I can explain my failure modes** — I know how and why the system breaks, and I've engineered
  fallback logic to catch failures before they reach the user.

It's the shift from a clever script to **industrial-grade infrastructure**: assume every component
will eventually fail, and design so failure is caught by another component first. ⚪ (sound
engineering philosophy; not a single-source claim.)

---

## Phase 11 — The open question: human oversight at scale

**The tension (podcast's closing thought, and it's a real one).** Human-in-the-loop is the ultimate
safeguard against excessive agency — but as systems generate hundreds of approval requests per day,
the human reviewer risks becoming the bottleneck, and rubber-stamping under fatigue ("normalization
of deviance") erodes the safeguard.

**Current thinking.** ⚪/🔹 There's no settled answer. Directions being discussed: **risk-tiered
approval** (auto-approve low-risk, gate only high-impact actions — money, deletion, prod changes),
**policy-as-code / guardrails** that encode rules so humans review exceptions not everything,
**batch/async review**, and **better trace legibility** so a human can approve quickly *with
understanding*. The honest status: scaling oversight without either bottlenecking throughput or
hollowing out the safeguard is an **unsolved governance problem**, not a solved feature.

**Sources.** Human-in-the-loop governance / normalization-of-deviance discussion 🔹 (secondary);
OWASP LLM06 HITL guidance ✅.

---

## Appendix A — Podcast accuracy scorecard

| # | Podcast claim | Verdict | Correct version |
|---|---|---|---|
| 1 | Agentic AI ≠ chatbots; it's an engineering discipline | ✅ Accurate | — |
| 2 | Tool calling needs strict JSON schema design | ✅ Accurate | — |
| 3 | A string-vs-int mismatch "crashes the whole system" | ⚠️ Overstated | Only if you skip validation; strict mode + server-side validation catches it |
| 4 | "Basic RAG is completely dead / fundamentally unsafe" | ❌ Overstated / slogan | Naive single-shot RAG is *inadequate for complex/high-stakes use* and has a real injection surface; RAG is **extended, not replaced** |
| 5 | ReAct = Reason/Act/Observe | ✅ Accurate (transcribed "React") | ReAct (Yao et al., 2023) |
| 6 | Planner/executor/verifier/router is *the* architecture | ⚠️ Nuance | A strong pattern; multi-agent vs single-agent is a genuine tradeoff — default to one agent |
| 7 | Routing saves tokens | ⚠️ Partly | Can reduce per-task context; multi-agent overall often costs *more* tokens |
| 8 | LangGraph durable execution / state | ✅ Accurate | + caveat: OSS is single-process |
| 9 | "LandGraph" | ✏️ Transcription error | **LangGraph** |
| 10 | OpenAI Agents SDK stateful handoffs | ✅ Accurate | Handoffs = model-mediated control transfer via `transfer_to_*` tools |
| 11 | MCP = "USB for AI," Anthropic, client/server | ✅ Accurate | + Host/Client/Server in full spec; JSON-RPC 2.0; can't enforce security at protocol level |
| 12 | OWASP "excessive agency" is **LLM08** | ❌ Wrong (current list) | **LLM06:2025**; LLM08:2025 = Vector & Embedding Weaknesses; Prompt Injection = LLM01 |
| 13 | White-text email prompt injection → exfiltration | ✅ Realistic | Indirect prompt injection + excessive agency |
| 14 | Least privilege, separate read/write, approval gates | ✅ Accurate | Directly matches OWASP LLM06 mitigations |
| 15 | "Arise Phoenix" / "OpenTelemetry" for tracing | ✅ Accurate (transcribed "Arise") | **Arize Phoenix** |
| 16 | Grade the trajectory, not just the answer | ✅ Accurate | — |
| 17 | "SWEbench" feeds real GitHub bugs | ✅ Accurate | **SWE-bench** — 2,294 tasks, 12 repos, unit-test scored |
| 18 | RAGAS checks relevance & faithfulness | ✅ Accurate | Reference-free RAG eval |
| 19 | Maturity = explainable failure modes + fallback | ✅ Accurate | — |

**Transcription fixes:** LandGraph→**LangGraph**, Arise Phoenix→**Arize Phoenix**, React→**ReAct**,
SWEbench→**SWE-bench**, "Egentic"→**Agentic**, "IRAC/ARAG"→**RAG**.

---

## Appendix B — Primary sources

**Frameworks / protocol (verified ✅):**
- LangGraph durable execution — https://docs.langchain.com/oss/python/langgraph/durable-execution
- LangGraph persistence — https://docs.langchain.com/oss/python/langgraph/persistence
- OpenAI Agents SDK handoffs — https://openai.github.io/openai-agents-python/handoffs/
- OpenAI orchestration guide — https://developers.openai.com/api/docs/guides/agents/orchestration
- Google ADK — https://google.github.io/adk-docs/
- MCP spec (2025-11-25) — https://modelcontextprotocol.io/specification/2025-11-25
- MCP announcement — https://www.anthropic.com/news/model-context-protocol

**Safety (verified ✅):**
- OWASP Top 10 for LLM Apps 2025 — https://genai.owasp.org/llm-top-10/
- LLM06 Excessive Agency — https://genai.owasp.org/llmrisk/llm062025-excessive-agency/
- LLM01 Prompt Injection — https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- OWASP Agentic AI – Threats & Mitigations — https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/

**Multi-agent debate (verified ✅):**
- Anthropic multi-agent research system — https://www.anthropic.com/engineering/multi-agent-research-system
- Cognition, "Don't Build Multi-Agents" — https://cognition.com/blog/dont-build-multi-agents

**RAG fact-check (sourced 🔹):**
- Gao et al., RAG survey — https://arxiv.org/abs/2312.10997
- Singh et al., Agentic RAG survey — https://arxiv.org/abs/2501.09136
- LaRA (ICML 2025) — https://arxiv.org/abs/2502.09977
- Anthropic Contextual Retrieval — https://www.anthropic.com/news/contextual-retrieval
- Microsoft, RAG & Generative AI (Azure AI Search, 2026-06-08) — https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview
- PoisonedRAG (USENIX Security 2025) — https://arxiv.org/abs/2402.07867
- Stanford HAI, legal RAG hallucination study — https://hai.stanford.edu/news/ai-trial-legal-models-hallucinate-1-out-6-or-more-benchmarking-queries

**Evaluation / observability (sourced 🔹):**
- SWE-bench — https://arxiv.org/abs/2310.06770
- RAGAS — https://arxiv.org/abs/2309.15217
- LLM-as-a-judge (MT-Bench) — https://arxiv.org/abs/2306.05685
- AgentRewardBench — https://arxiv.org/abs/2504.08942
- OpenTelemetry GenAI agent spans — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
- Arize Phoenix — https://github.com/Arize-ai/phoenix

---

*Method note: the technical/safety claims marked ✅ passed 3-vote adversarial verification against
primary sources (25/25 confirmed, 0 refuted) in an automated deep-research run; the "RAG is dead"
fact-check was additionally cross-examined by three independent investigators (prosecute / defend /
define) that converged. Claims marked 🔹 are primary-sourced but not adversarially stress-tested;
⚪ marks practitioner consensus. Re-verify version-specific and fast-moving details before relying
on them.*
