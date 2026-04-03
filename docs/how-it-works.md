# How NEXUS Works

Technical methodology documentation. This explains what happens under the hood when you run a simulation — without revealing proprietary prompt architecture or orchestration internals.

---

## Simulation Lifecycle

Every simulation follows the same 7-stage pipeline:

```
Stage 1: Input Analysis          ~5 seconds
Stage 2: Market Research         ~10-15 seconds
Stage 3: Knowledge Graph         ~5 seconds
Stage 4: Agent Generation        ~3-5 seconds
Stage 5: Simulation Rounds       ~3-12 minutes (depends on tier)
Stage 6: Analysis & Scoring      ~30-60 seconds
Stage 7: Report Generation       ~60-90 seconds
```

Total wall time: **4-15 minutes** depending on simulation depth.

---

## Stage 1: Input Analysis

The user provides a business decision as free text (50-50,000 characters) or structured form input across 4 templates: Startup Launch, Marketing Campaign, Pricing Decision, or God Mode (open-ended).

The engine runs Named Entity Recognition (NER) to extract:
- Companies, products, brands mentioned
- Geographic locations and markets
- Financial figures (budget, pricing, revenue targets)
- Competitor names and positioning
- Target audience descriptions

Additionally, the engine generates 3-5 follow-up questions to fill gaps in the user's input. These are optional — users answer what they want, skip what they don't.

---

## Stage 2: Real-Time Market Research

For each simulation, NEXUS performs live data collection:

**Web Search** — queries constructed from extracted entities. For a coffee shop in Barcelona, searches include: competitor pricing, local market size, industry benchmarks, regulatory requirements, recent news about the area.

**Geographic Intelligence** — OpenStreetMap Overpass API queries for business density. "131 cafes within 500m" is a real count from a live API call, tagged with coordinates and timestamp. This data is injected into agent prompts so they argue with real numbers, not guesses.

**Data Freshness** — all market data is collected at simulation time. There is no pre-cached dataset. Each simulation gets fresh data relevant to its specific location, industry, and competitive landscape.

**Missing Data Handling** — if a data point cannot be verified, the report states "data not available" rather than fabricating a number. This is enforced at the prompt level with explicit no-fabrication rules.

---

## Stage 3: Knowledge Graph

Extracted entities and their relationships are structured into a graph:

```
Nodes:  Companies, Products, People, Markets, Regulations
Edges:  COMPETES_WITH, TARGETS, DEPENDS_ON, REGULATES, OPERATES_IN
```

The graph serves two purposes:
1. **Agent generation** — stakeholder roles are derived from graph entities (e.g., a COMPETES_WITH edge generates a Competitor agent)
2. **Report generation** — the ReACT-pattern report agent queries the graph to produce structured analysis

For Standard and Deep tiers, an **enrichment step** adds AI-discovered entities (additional competitors, regulations, market segments) with confidence scores. Only enrichments with >80% confidence are shown to the user, who confirms or removes them before simulation begins.

---

## Stage 4: Agent Generation

Agents are generated from a library of **500+ archetypes** — domain-specific behavioral profiles, not generic personas.

Each agent has 4 layers:

**Layer 1: Demographics** — age, income, location, education. Distributions match real census data for the target region.

**Layer 2: Personality** — Big Five model (Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism). Normal distribution across each axis. This drives behavioral diversity — a high-Agreeableness agent responds differently than a low-Agreeableness agent to the same information.

**Layer 3: Domain Expertise** — role-specific knowledge. An "angel investor evaluating pre-seed food tech" has different decision criteria than a "Series B growth investor evaluating SaaS."

**Layer 4: Social Role** — follower count (power-law distribution), group membership, authority score (0-1), initial opinion seeded from market research data.

### 3-Tier Architecture

Not every agent makes an LLM call. This is how we simulate 1,000+ stakeholders at accessible price points:

| Tier | Count | Intelligence | Cost |
|------|:-----:|-------------|:----:|
| **Leaders** | 5-50 | Full multi-perspective reasoning. 5 lenses: Investor, Critic, Advocate, Customer, Observer. Each produces detailed arguments with supporting evidence. | High |
| **Active** | 10-80 | Quick reactions and opinions. Compressed via clustering — hundreds of unique perspectives represented by a smaller number of LLM calls. | Medium |
| **Crowd** | 35-400+ | Zero LLM calls. Probability-based behavior driven by Big Five personality parameters and Leader influence. Massive scale at zero marginal cost. | Zero |

Agent counts scale by tier:

| Simulation Tier | Leaders | Active | Crowd | Total |
|----------------|:-------:|:------:|:-----:|:-----:|
| Free (0×) | 5 | 10 | 35 | 50 |
| Quick (400×) | 15 | 25 | 100 | 140 |
| Standard (1,200×) | 25 | 40 | 200 | 265 |
| Deep (2,500×) | 50 | 80 | 400 | 530 |

---

## Stage 5: Simulation Rounds

The simulation runs 5-25 rounds (depending on tier). Each round:

### 1. Activation
Not all agents are active every round. A time engine activates ~12% of agents per round, weighted by tier (Leaders activate more frequently than Crowd).

### 2. Content Recommendation
Each active agent sees a curated feed — not the entire world. A recommendation system surfaces relevant posts based on social connections, topic relevance, and recency. This compresses the context window and mirrors how real social networks filter information.

### 3. Agent Actions
Active agents choose from multiple action types: post an opinion, react to existing content (agree/disagree), follow another agent, or do nothing. Leaders produce detailed reasoning. Active agents produce quick reactions. Crowd agents follow probability formulas based on their personality profile and the opinions of their nearest Leaders.

### 4. Quality Filters (every round)

**Format Validation** — every LLM response is checked for structural correctness. Malformed responses trigger retry (up to 2x), then fallback to "do nothing" for that round.

**Consistency Check** — agent actions are checked against their personality profile. A conservative investor agent suddenly buying a high-risk product gets flagged. Extreme opinion shifts without cause get flagged.

**Diversity Monitoring** — Krippendorff's alpha measures inter-agent agreement each round. If agents converge too fast (alpha > 0.8), diversification triggers. If opinions are chaotic (alpha < 0.2), consistency checks fire.

### 5. Adversarial Protocol

**Devil's Advocate** — every 3 rounds, a contrarian agent is activated to challenge whatever consensus is forming. If 70% of agents are negative, the Devil's Advocate argues positive (with specific reasoning). This prevents groupthink and forces the simulation to stress-test its own conclusions.

**Bounded Confidence** (Deffuant-Weisbuch model) — agents only influence each other if their opinions are within a threshold distance. This prevents artificial consensus and allows genuine polarization to emerge.

**Influence Propagation** (PageRank-based) — high-authority agents carry more weight in the network. A skeptical investor's critique propagates further than a neutral observer's comment.

### 6. Early Termination
If information gain drops below a threshold for 3 consecutive rounds (agents are repeating themselves), the simulation terminates early. This saves compute without losing quality.

### 7. Error Handling
If >20% of agents receive "do nothing" fallbacks in a single round (indicating LLM provider issues), the simulation stops and the user receives a full refund.

---

## Stage 6: Analysis & Scoring

After all rounds complete:

**Bayesian Aggregation** — agent opinions are weighted by expertise tier and domain relevance. A SOC 2 auditor's opinion on compliance risk carries more weight than a marketing agent's. Weights are learned from simulation outcomes, not hardcoded.

**Decision Chain Extraction** — the engine traces influence paths through the simulation log. "Agent A posted critique → Agent B reacted negatively → 40 Crowd agents shifted opinion → cascade." These chains appear in the report as event cascades.

**Baseline Comparison** — the same user input is sent as a single prompt to a standard LLM. The simulation result is compared against this baseline. If the simulation didn't surface anything the baseline missed, this is flagged internally as a quality issue.

**Scenario Construction** — final agent opinions are clustered into 2-5 distinct scenarios with probability weights derived from the proportion and authority of agents supporting each cluster.

---

## Stage 7: Report Generation

A ReACT-pattern report agent generates structured sections by querying the Knowledge Graph and simulation results:

| Section | What it contains |
|---------|-----------------|
| **TL;DR Verdict** | 2-3 sentences: overall assessment, #1 risk, first action |
| **Scenarios** | Probability-weighted outcomes with financial projections |
| **Risks** | Severity-ranked with dollar impact and specific recommendations |
| **Action Plan** | Keep / Fix / Add / Remove with priority labels |
| **What Would Change** | Key assumptions, missing data, sensitivity analysis |

### Quality Rules (enforced at generation time)

Every sentence in the report must contain at least one of:
- A specific number
- A named entity
- A concrete action
- A timeframe

Sentences without any of the above are deleted. Additionally, a blocklist prevents filler phrases ("the market is competitive," "there are risks involved," "stakeholders have mixed opinions") — each must be replaced with specific data.

A Report Auditor (single LLM call) reviews the final report for logical contradictions, unsupported claims, and consistency with simulation data.

---

## What-If Analysis (Deep tier)

Users can change any variable and re-run the simulation:

1. User modifies a parameter (e.g., "What if rent is €2,200 instead of €4,500?")
2. The Knowledge Graph is updated with the new parameter
3. Agent profiles are regenerated to reflect the change
4. A new simulation runs with the modified world
5. Results are compared: old sentiment → new sentiment, old scenarios → new scenarios
6. A delta report highlights what changed and why

This is a full re-simulation, not a text transformation. Agents genuinely re-evaluate the decision with new information.

---

## Agent Chat (Standard and Deep tiers)

After the simulation, users can click on any agent and ask questions:

- "Why did you change your mind in round 7?"
- "What would make you support this plan?"
- "What's the biggest risk you see?"

The agent responds in character — drawing on their personality profile, the posts they made during the simulation, and the information they were exposed to. This is not a generic chatbot response; it's a contextualized answer from a specific simulated stakeholder.

---

## Cost Model

Simulation costs scale with the number of LLM calls, which scale with tier:

| Tier | Estimated LLM Calls | Estimated Time |
|------|:-------------------:|:--------------:|
| Free | ~30-50 | ~3-4 min |
| Quick | ~80-120 | ~5-6 min |
| Standard | ~150-250 | ~8-11 min |
| Deep | ~300-500 | ~12-15 min |

Crowd agents (35-400 per simulation) make zero LLM calls — they operate on probability formulas. Agent compression (clustering) further reduces Active tier calls by ~60%. These optimizations are what make large-scale simulation accessible at consumer price points.

---

## Self-Learning Pipeline

Every completed simulation generates data that improves future simulations:

1. **Prompt A/B Testing** — variant prompts are tested across simulations. Performance is measured by report quality metrics (specificity score, scenario diversity, recommendation actionability). Better-performing prompts replace weaker ones.

2. **Archetype Refinement** — when users provide real-world outcome feedback ("the coffee shop failed at month 8"), agent archetypes that predicted correctly are reinforced. Those that missed are adjusted.

3. **Domain Weight Calibration** — the relative importance of different agent tiers is learned per domain. In pricing simulations, financial analyst agents may carry 4x weight. In marketing simulations, consumer agents dominate. These weights emerge from data, not assumptions.

4. **Pattern Recognition** — as the simulation count grows, recurring patterns are detected and surfaced in future reports. "In 85% of food delivery simulations in cities with >1M population, customer acquisition cost exceeded initial projections by month 4."

The engine that runs today is measurably different from the one that ran 3 months ago. This improvement is continuous and accelerating.

---

*Questions about methodology? Run a simulation and see for yourself → [nexus-sim.ink](https://nexus-sim.ink)*
