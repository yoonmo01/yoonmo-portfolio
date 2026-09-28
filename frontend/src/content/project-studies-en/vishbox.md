> A research system that studies voice phishing interactions by generating synthetic conversations conditioned on victim profiles without using real call recordings.

[IEEE Access paper](https://doi.org/10.1109/ACCESS.2026.3667823) · Follow-up research: [VishBox v2](https://github.com/yoonmo01/VP2)

## My contribution and implementation

- **Agent and MCP transition:** Implemented the Manager Agent's calls to scenario, prompt, and analysis tools and moved dialogue execution into a separate MCP server.
- **Live progress interface:** Sent progress events from a FastAPI SSE endpoint and connected the frontend stream handler so dialogue and analysis logs appear as the simulation runs.

The research evaluation figures below describe the **team's complete simulation**, not an individual component.

## Why we built it

Voice phishing education often relies on fixed examples, while real attacks change in response to the victim. Actual call recordings are difficult to use in research because of investigative and privacy concerns.

VishBox uses **three LLM agents—attacker, victim, and manager**—to reproduce these interactions with synthetic dialogue. Conversations change with the victim profile, and risk is evaluated at each turn.

---

## System architecture

A central **Manager Agent** coordinates three stages.

![VishBox system architecture](/projects/VP/fig1-architecture.png)

| Stage | Processing |
|---|---|
| **1. User configuration** | Select a scenario type and assemble a victim profile. |
| **2. Dialogue simulation** | Attacker Agent ↔ Victim Agent in an isolated MCP environment. |
| **3. Analysis and evaluation** | Score turn-level risk and generate tailored prevention strategies. |

The Manager Agent calls Scenario Generator, Prompt Builder, Analysis Engine, and Prevention Generator as tools. It delegates dialogue generation itself to the MCP server.

### Victim profile

The profile combines three dimensions informed by Korean empirical studies.

| Dimension | Contents |
|---|---|
| **Demographics** | Age-group characteristics: 20s, 30s–40s, and 60s or older. |
| **Digital financial literacy** | Knowledge, behavior, and attitude, calibrated with national survey data. |
| **Personality** | The five OCEAN (Big Five) factors. |

These values affect the chance of calling an official number to verify a claim, the tendency to check procedures, and emotional vulnerability.

### Three scenario types

| Type | Tactic | Main target |
|---|---|---|
| **Loan impersonation** | Lure of a lower-interest refinancing offer | People in their 30s–40s |
| **Authority impersonation** | Prosecutor or financial regulator impersonation and urgency | People in their early 20s |
| **Family impersonation** | Fabricated family emergency and emotional panic | People aged 60 or older |

Every scenario follows four stages: **contact → trust building → control → extraction**. The structure and detailed tactics are based on the Korean National Police Agency's *Monthly Phishing* crime-analysis reports.

### MCP-isolated dialogue

![MCP-isolated dialogue simulation architecture](/projects/VP/fig2-mcp-dialogue.png)

Dialogue generation runs inside the **MCP server (`vp_mcp/`)**, separate from the external agent system that includes the Manager Agent. The attacker and victim models interact only inside the simulation; standardized prompt exchanges and structured logs pass across the boundary. This supports reproducible simulations and controlled experiments.

### Behavioral metadata

Each victim turn includes internal-state metadata alongside the utterance.

```json
{
  "utterance": "I'll call the bank's official number to check first.",
  "is_convinced": 2,
  "thoughts": "They keep repeating the same thing. Something feels wrong; I should verify it with the bank first."
}
```

The Manager Agent reads these annotations to update risk state for each round.

---

## Validation and outcomes

The team evaluated the system through a blind identification task with **102 participants** and by comparing simulation outcomes with real victimization statistics from Korea's Financial Supervisory Service (FSS).

### 1. Participants could not reliably distinguish generated dialogue

![Accuracy of distinguishing human and generated conversations](/projects/VP/fig4-accuracy.png)

Participants identified real versus generated conversations with **48.04%** accuracy, close to chance level (50%).

| Test | Result |
|---|---|
| Gender | χ² = 0.00, *p* = 1.000 |
| Age group | χ² = 0.86, *p* = 0.835 |
| Education | χ² = 0.56, *p* = 0.756 |
| Logistic regression (gender, age, education) | χ² = 0.98, *p* = 0.980 |

**No demographic group performed significantly better.** This supports the finding that the generated conversations were not convincing only to one group.

### 2. Age-related vulnerability and real victimization patterns

![Simulation success rates by age group and scenario](/projects/VP/fig6-age-scenario.png)

| Age group | Most vulnerable scenario | Simulation | Real FSS statistics |
|---|---|---|---|
| 20s | Authority impersonation | **80%** | Same type accounts for **82.1%** of losses among people under 30. |
| 30s | Loan impersonation | **80%** | **58.1%** of losses in this age group. |
| 60s | Family or acquaintance impersonation | **100%** | **51.0%** of losses and **75.6%** of cases. |

Results varied even within the same age group. In the simulation, family impersonation reached 80% success for people in their 20s, while its share of actual losses was under 2%. This variation comes from simulating **personas that combine age, personality, and financial literacy**. Age statistics alone are not enough to design prevention education.

### 3. Risk rose as the conversation progressed

A linear mixed-effects analysis found a **significant upward trend** in perceived risk across dialogue stages (β = 0.505, *p* < .001).

| Dialogue stage | Mean perceived risk |
|---|---:|
| Stage 1 | 3.27 |
| Stage 2 | 3.88 |
| Stage 3 | **4.28** |

Every pair of stages differed significantly (Mann–Whitney U, *p* < 0.001). This supports the intended accumulation of pressure in the four-stage crime script.

---

## Technology

| Layer | Technology |
|---|---|
| **Backend** | FastAPI · SQLAlchemy · Pydantic |
| **Agent** | MCP server (`vp_mcp`) · ReAct orchestrator · Tavily web search |
| **LLM** | GPT-4.1-mini (attacker) · Gemini 2.5 Flash Lite / GPT (victim) · o4-mini (manager) |
| **Database** | PostgreSQL |
| **Frontend** | React · Vite |
| **Other** | TTS synthesis · dialogue splitter |

---

## Limitations and reflection

The **48.04%** figure is accuracy on the real-versus-generated dialogue identification task. It is not system accuracy or evidence of prevention effectiveness. Results from synthetic conversations should not be generalized to real-world prevention outcomes. VishBox v2 followed this work by adding stage-level tactics and state analysis.

Figure source: Figures 1, 2, 4, and 6 from the [VishBox paper](https://doi.org/10.1109/ACCESS.2026.3667823), CC BY 4.0.
