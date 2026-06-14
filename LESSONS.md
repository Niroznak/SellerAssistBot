# Agentic System Design — Lessons

Learned during SellerAssistBot design session.

---

## 1. Agent vs Skill — Know the Difference

**Agent** = state-aware, autonomous, decides what to do based on context.
**Skill** = deterministic, triggered explicitly, no decision-making of its own.

> The summarizer runs every 10 prompts on a fixed rule. It has no autonomy — it's a skill.
> The generator decides *what* to recommend based on state — it's an agent.

**Rule:** Don't make something an agent just because it uses an LLM. If the trigger and behavior are deterministic, it's a skill. Over-agentifying adds complexity without benefit.

---

## 2. Reduce Agent Count — Complexity Grows Fast

Default to fewer agents. Each agent adds:
- Its own memory and state management
- Coordination overhead with orchestrator
- A new failure mode

Only split into separate agents when concerns are genuinely independent in state, timing, and ownership. Three questions to test a split:
- Do they run at different times?
- Do they own different data?
- Would combining them create a god-object with mixed responsibilities?

---

## 3. Separate Agents by Concern, Not by LLM Call

Bad split: "one agent per LLM call"
Good split: "one agent per distinct responsibility"

> Customer scoring, seller evaluation, recommendation generation, and impact assessment are 4 different concerns — even if they all call LLMs.

---

## 4. The Orchestrator Owns the State Machine

Agents produce signals. The orchestrator decides what those signals mean and what happens next. Never let agents trigger other agents directly — it creates hidden coupling and makes sequencing impossible to reason about.

> Evaluator detects degradation → writes signal → orchestrator reads it → orchestrator initiates AB.
> Not: evaluator initiates AB directly.

---

## 5. Pre-Compute to Hide LLM Latency

Don't race the LLM against a 1-second clock. The real budget is "time from input event to user action." Start processing the moment input arrives — hide the latency inside the human's natural reading/thinking time (typically 3-8s).

> Generator fires the moment client message arrives. By the time seller finishes reading, recommendation is ready.

This is the standard pattern in real-time assist products (Gong, Chorus, Einstein).

---

## 6. Feedback Lag is a Feature, Not a Bug

In real-time systems, you often can't wait for evaluation before generating the next output. Accept the lag explicitly, document it, and design around it.

> Evaluator at turn N uses KPI from turn N+1 (next client message). Generator at turn N uses evaluator output from turn N-2. This is a known, accepted one-turn lag — not a bug.

Naming the tradeoff prevents future confusion about "why is the feedback delayed?"

---

## 7. Weighted Thresholds + Hard Triggers

Not all signals should have equal weight. Pattern:
- **Weighted sum of deltas** for soft changes (slight topic drift, minor sentiment drop)
- **Hard triggers** for critical events regardless of weight (topic change, intent=leave, price objection)

Externalize weights in config. Log every trigger decision. You can't tune what you don't log.

---

## 8. The Counterfactual Problem in Recommendation Systems

You can only measure recommendations that were used. Ignored recommendations are invisible — you can't know if they would have worked.

**Mitigations:**
- Semantic proximity: did seller approximate the recommendation even if not verbatim?
- Strategy alignment: did seller follow the recommended approach even with different words?
- Aggregate signal: single session is noisy, patterns emerge across many sessions

**Most important mitigation:** Offline pre-launch validation on real session data. Don't try to bootstrap from bad prompts during live sessions. The feedback loop refines already-decent prompts — it doesn't replace prompt quality.

---

## 9. Different Provider for Self-Improvement

If your system uses an LLM to improve its own prompts, using the same model introduces circular bias. The model can't escape its own blind spots.

> Reflection loop uses a different provider (GPT-4o/Gemini). Different model = different reasoning patterns = genuine diversity in improvement attempts.

This is a production BKM in LLM systems that do prompt self-optimization.

---

## 10. Failure Memory Needs WHY, Not Just THAT

When storing failed attempts for future reference:
- Don't just store "strategy X failed"
- Store WHY it failed (evaluator's interpretation: ignored + KPI dropped on price topic, etc.)

Without WHY, the reflection loop generates variations on the same failed approach. With WHY, it can reason about genuinely different strategies.

---

## 11. AB Testing — Winner is Implicit

You don't need an explicit "winner" field. After AB concludes, the version that continues IS the winner. Store what's needed to detect failure history, not what's derivable from behavior.

---

## 12. Offline Validation Before Online Learning

For recommendation systems, establish a quality baseline before going live:
- Run existing session data through the system offline
- Validate prompt performance per topic
- Build a prompt library covering known topics
- Define a general-purpose cold start prompt for unknown topics

Online learning (within-session improvement) refines a good baseline. It cannot replace one.

---

## 13. Schema Discipline — Don't Add Fields Until You Know What They Contain

Don't add `strategy_alignment` to a schema because it sounds useful. Wait until you have real session data that tells you what strategy dimensions actually matter. Premature fields in schemas create noise and implementation debt.

---

## 14. Cold Start is a Real Failure Mode

Every component that depends on history needs a cold start definition:
- What's the minimum data needed before it produces output?
- What does it do before that threshold?
- Is the threshold configurable?

> AB testing needs min # loops before it's eligible. Generator needs at least 1 client message. Evaluator needs at least 1 recommendation to evaluate.

Define these explicitly at design time — not when you hit a null reference in production.
