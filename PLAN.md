# SellerAssistBot — Design Plan

## System Overview
Agentic sales assistant that ingests live seller↔customer conversations, scores customer KPIs, generates real-time seller recommendations, and runs a feedback loop to improve recommendation quality within and across sessions.

---

## Design Checklist

```
[x] Section 1: Agent Design & Boundaries
    [x] 1.1 Customer Agent — KPI scoring, state detection, promotes index on client arrival
    [x] 1.2 Seller Agent — recommendation usage detection, semantic LLM match
    [x] 1.3 Generator Agent — Sonnet, recommendations from trends + evaluator feedback
    [x] 1.4 Evaluator Agent — KPI delta, AB state ownership, failure history
    [x] 1.5 Summarizer — Skill (not agent), deterministic, orchestrator-triggered
    [x] 1.6 Orchestrator — AB state machine owner, trigger routing
    [x] 1.7 Cold start — no KPIs/recommendations until first client message (turn index 0)
    [x] 1.8 Seller agent cold start — no evaluation until turn index 1 minimum

[x] Section 2: Memory & State Design
    [x] 2.1 Shared conversation store (customer + seller agents write)
    [x] 2.2 KPI trend store — separate lightweight file, generator reads
    [x] 2.3 Index = speaker change, client arrival promotes index
    [x] 2.4 Consecutive same-speaker prompts = same index
    [x] 2.5 Summarization every 10 prompts, keep last 1, background skill, non-blocking
    [x] 2.6 Contradiction detection — generator responsibility via context, known risk

[x] Section 3: Latency & Concurrency Model
    [x] 3.1 Pre-compute on client arrival (hidden in seller reading time ~3-8s)
    [x] 3.2 Stream generator output to recommendation panel
    [x] 3.3 Cache+delta: skip regeneration below weighted threshold
    [x] 3.4 Hard triggers regardless of weight: topic shift, intent=leave
    [x] 3.5 Weights externalized in config, tunable, every decision logged
    [x] 3.6 Evaluator runs parallel but evaluates previous turn (1-turn lag — accepted)
    [x] 3.7 Generator at turn N uses evaluator output from turn N-2

[x] Section 4: LLM Stack
    [x] 4.1 Customer Agent — claude-haiku-4-5-20251001
    [x] 4.2 Seller Agent — claude-haiku-4-5-20251001
    [x] 4.3 Evaluator Agent — claude-haiku-4-5-20251001
    [x] 4.4 Generator Agent — claude-sonnet-4-6
    [x] 4.5 Summarizer Skill — claude-haiku-4-5-20251001
    [x] 4.6 Reflection Loop — Different provider (GPT-4o/Gemini), escapes Claude bias
    [x] 4.7 Structured output — Anthropic tool_use primary
                              — Prompt instruction + Pydantic + single retry for fallback provider
    [x] 4.8 Response envelope — {status, response, time_to_first_token, time_to_last_token}
                              — status is orchestrator-only health signal, never exposed to UI
    [x] 4.9 Prompt storage — txt files on disk, generator caches 1-2 active versions
                           — older versions retained for audit/replay

[x] Section 5: Feedback Loop & Improvement
    [x] 5.1 States: stable | ab (no other states needed)
    [x] 5.2 stable→ab trigger: KPI degradation OR consecutive seller ignores
    [x] 5.3 ab→stable trigger: configurable # loops, winner = version that continues
    [x] 5.4 Min loops before AB eligible (cold start guard, configurable)
    [x] 5.5 AB = 2 parallel generator calls, same agent code, different prompt version
    [x] 5.6 Failure memory: evaluator stores failed strategies, reflection loop excludes them
    [x] 5.7 Reflection loop needs failure WHY not just THAT — evaluator stores interpretation
    [x] 5.8 Fallback: different provider generates prompt with full failure context
    [x] 5.9 Session ends mid-AB: discard version B, version A already validated, no special handling
    [x] 5.10 Evaluator JSON schema (flat array):
             turns[]: {index, used_flag, kpi_impact, ab: false | {version, strategy}}

[x] Section 6: UI / Display
    [x] 6.1 Rich terminal output for POC
    [x] 6.2 Deployment model deferred — post-POC UX decision (desktop/web/CRM plugin)

[x] Section 7: Storage & Persistence
    [x] 7.1 Conversation store — in-session only, discarded after session ends
    [x] 7.2 KPI trend store — in-session only, separate lightweight file
    [x] 7.3 Evaluator JSON — persisted after session (turns + AB history)
    [x] 7.4 Prompt files — persisted, last validated version kept
                         — mid-AB session end: discard B, keep A (already validated)
    [x] 7.5 Session output — full session record persisted for post-analysis
```

---

## Pipeline Sequence (per turn)

```
Client message arrives
  → Customer Agent scores KPIs              [Haiku, every turn]
  → Evaluator runs on previous turn         [Haiku, parallel, 1-turn lag]
  → Generator fires with current KPIs       [Sonnet, pre-computed, streamed]
      └─ reads last 5 evaluator feedback records
      └─ uses active prompt version (1 or 2 during AB)
  → Recommendation displayed to seller
  → Seller reads, responds
  → Seller Agent evaluates usage            [Haiku]
  → Next client message → cycle repeats

Every 10 prompts:
  → Summarizer Skill runs in background     [Haiku, non-blocking]
  → Keeps last 1 raw prompt, discards rest
```

---

## AB State Machine

```
stable → ab:   KPI degradation OR consecutive seller ignores
ab → stable:   configurable # loops completed, winner = version that continues

Cold start guard: min # loops in stable before AB eligible (configurable)
Failure memory:   evaluator JSON stores failed strategies per AB cycle
                  reflection loop reads history, excludes failed strategies
Fallback:         if new prompt not better → different provider generates next attempt
                  with full failure context + interpretation of WHY it failed
```

---

## Known Risks & Deferred Items

**Risks:**
- Counterfactual problem: unused recommendations unmeasurable during session
  → Mitigation: offline pre-launch validation on real session data
- Cold start prompt: needs general-purpose, topic-agnostic baseline
- Contradiction detection: depends on summary quality, known accuracy risk

**Deferred post-POC:**
- Deployment model (desktop / web / CRM plugin)
- Long-term cross-session learning
- strategy_alignment field in seller agent (needs real data to define)
- Weight tuning (start equal + hard triggers, tune from logged decisions)
