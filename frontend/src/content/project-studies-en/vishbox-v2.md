> A voice phishing dialogue simulation study that tracks attack strategy and victim state round by round, grounded in crime procedures and persuasion principles.

[ACL 2026 Industry Track paper](https://aclanthology.org/2026.acl-industry.145/) · Earlier research: [VishBox v1](https://github.com/yoonmo01/VP)

## My contribution and implementation

- **Main Agent and tools:** Designed and implemented the Main Agent that coordinates the simulation and the tools it calls. Connected prompt construction, MCP dialogue execution, emotion labeling, guidance generation, and round judgment.
- **MCP Dialogue Agent:** Separated the attacker's procedural planning (Planner), actual utterance generation (Realizer), and victim response so the dialogue advances in stages.
- **Emotion and judgment tools:** Mapped the eight emotions output by Hugging Face's HowRU-KoELECTRA to four HMM observation symbols (N/F/A/E), which the HMM uses to estimate three latent vulnerability states (V1–V3). In the current repository implementation, handled “surprise” according to threat and refusal cues in the utterance. Fed emotion values into the Guidance Tool and extracted emotion-related vulnerabilities in the round judgment tool for the Main Agent's next step.
- **Stable agent input:** Passing all intermediate emotion predictions and HMM outputs to the agent caused malformed JSON responses. I reduced the handoff to the final state values after the calculations.

The experimental results below belong to the **team's complete system**. The overall architecture includes the Tactic Search Agent; my directly implemented scope is described above.

The [published paper's emotion-mapping appendix](https://aclanthology.org/2026.acl-industry.145.pdf) maps Surprise to Excitement. The current repository's cue-based routing differs from that description; the paper's results do not evaluate that routing change separately.

## What changed from v1?

VishBox v1 produced conversations that human participants could not reliably distinguish from real ones. It did not explain **why persuasion worked in a particular round**. VishBox v2 reorganized the system to investigate that question.

| | **v1** (IEEE Access) | **v2** (ACL 2026) |
|---|---|---|
| **Agents** | Manager Agent plus attacker and victim models | **Main Agent** coordinates the **Dialogue Agent** and **Tactic Search Agent** |
| **Attacker design** | Single generation from persona prompts | Separate **planning and realization** |
| **Tactic grounding** | Police crime-analysis reports | Those reports plus **PPSE persuasion principles** and investigation-manual **procedure codes (`proc_code`)** |
| **Emerging tactics** | None | **Tactic Search Agent** collects and summarizes new tactics through web search |
| **Victim state** | Turn-level conviction score | **Emotion signals from utterances → HMM estimate of latent vulnerability** |
| **Evaluation unit** | Whole dialogue | Three levels: **Turn / Round / Case** |
| **Validation** | Identification experiment with 102 participants | Procedural plausibility evaluated by **three active-duty police officers** |

---

## System architecture

![VishBox v2 system architecture](/projects/VP2/fig1-architecture.png)

The Main Agent manages the simulation loop. It delegates dialogue generation to the MCP-isolated Dialogue Agent and emerging-tactic collection to the Tactic Search Agent.

```text
                         +---------------------------+
                         |         Main Agent        |
                         |  Simulation loop          |
                         |  Tool calls and stopping  |
                         +---------+---------+-------+
                           delegate|         |decide extended search
                  +----------------+         +--------------------+
                  v                                               v
       +-----------------------+                      +-----------------------+
       |    Dialogue Agent     |                      |  Tactic Search Agent  |
       |    MCP-isolated       |                      |  Extract keywords     |
       |    Attacker LLM       |                      |  Search the web       |
       |    Victim LLM         |                      |  Summarize tactics    |
       +-----------+-----------+                      +-----------+-----------+
                   | utterances and signals                      | new tactics
                   v                                             v
       +---------------------------------------------------------------+
       | Emotion extraction (KoELECTRA) -> HMM latent vulnerability   |
       | PPSE persuasion labels and proc_code procedure consistency   |
       +---------------------------------------------------------------+
```

### MCP isolation of the Dialogue Agent

The Dialogue Agent is separated from the Main Agent through **Model Context Protocol (MCP)**. The design supports **on-premises deployment** of sensitive dialogue models and isolation of internal resources. Attacker generation is split into planning and realization.

### Tactic Search Agent

When the Main Agent requests extended search after **two consecutive failed persuasion attempts within a case**, this agent extracts keywords, searches the web, and synthesizes a compact tactic report. The result informs attack planning in the next round.

### Estimating victim state

The system extracts emotion signals from victim utterances with `HowRU-KoELECTRA`, then uses an **HMM estimator** to infer latent vulnerability that is not directly observable. This makes changes in psychological risk across rounds easier to analyze.

### Evaluation units

| Unit | Definition |
|---|---|
| **Turn** | One attacker–victim message exchange. |
| **Round** | One continuous phishing attempt; ends when the victim explicitly refuses or high-risk behavior is observed. |
| **Case** | The complete simulation episode; ends after **five rounds** or when **`risk_level=Critical`** confirms phishing success, such as a transfer or disclosure of sensitive information. |

---

## Validation and outcomes

The study focused on **law-enforcement impersonation** and generated and analyzed **181 cases / 571 rounds**.

### 1. Evaluation by three active-duty police officers (five-point scale)

| Criterion | Score |
|---|---:|
| **Plausibility** | **4.43** |
| Realism | 4.11 |
| Diversity | 4.16 |

The experts also noted repetitive expressions and threats that were less intense than in real cases.

### 2. Procedural position mattered more than the tactic alone

In the six-stage procedure represented by `proc_code`, success rates differed sharply between patterns that developed a stage and patterns that skipped ahead.

| Procedure pattern | Success rate |
|---|---:|
| Refinement within a stage (`6-1 → 6-2 → 6-3`) | **73.8%** |
| Skipping stages (`2-1 → 3-1 → 4-1`) | **4.9%** |
| Aggregate comparison | **52.3%** vs **4.4%** |

The gap remained large when comparing 3-grams with similar frequency. Skipping trust building and moving directly to a demand rarely worked.

### 3. High-risk rounds used fewer tactics more persistently

![Tactic diversity by round and tactic density by risk group](/projects/VP2/fig2-ppse.png)

| Risk group | Tactic diversity (`H_norm`) | Tactic density |
|---|---:|---:|
| Low + Medium | 0.209 | 9.53 |
| **High + Critical** | **0.129** | **17.08** |

**High + Critical rounds had lower tactic variety and higher tactic density** than Low + Medium rounds (*p* < .001, Hedges' *g* = 1.30).

### 4. Compliant neutrality was a stronger warning signal than fear

![HMM latent vulnerability distribution and correlation with risk score](/projects/VP2/fig3-hmm-vulnerability.png)

The study compared round-level emotion with the HMM-estimated probability of latent vulnerability state V3.

| Emotion | Correlation with V3 probability |
|---|---:|
| **Neutrality** (compliant neutrality) | **r = 0.519** |
| Fear | r = 0.255 |

Both have *p* < .001. In these simulated dialogues, compliant neutrality had a stronger correlation with the estimated V3 state than fear. Fear alone would miss this pattern.

### 5. The web-search paradox

![PPSE label distribution for emerging tactics found through web search](/projects/VP2/fig4-websearch-ppse.png)

Web search activates after two consecutive failed persuasion attempts within a case.

| Web search | Rounds | Successes | Success rate |
|---|---:|---:|---:|
| ON | 164 | 18 | **10.98%** |
| OFF | 407 | 205 | **50.37%** |

Rounds that brought in newer tactics had lower success rates. **Search was only activated in already difficult situations**, so this comparison cannot establish a causal effect of search. It also showed a procedural issue: introducing fake official apps, deceptive URLs, or deepfake identity checks **before establishing trust could raise suspicion instead of compliance**. Knowing a new tactic and using it at the right stage are different things.

Within the **web-search-augmented tactic categories**, **A5 (pressure and threat)** and **A1 (authority)** accounted for roughly **93–96%** of PPSE labels. This percentage does not describe every round in the study.

---

## Technology

| Layer | Technology |
|---|---|
| **Backend** | FastAPI · SQLAlchemy · Pydantic |
| **Agents** | MCP server (`vp_mcp`) · ReAct orchestrator · external web-search integration |
| **Emotion and state** | HowRU-KoELECTRA classifier · HMM vulnerability estimator (`app/services/emotion`, `app/services/hmm`) |
| **LLM** | GPT-family models configured separately for attacker, victim, manager, and agent roles |
| **Database** | PostgreSQL |
| **Frontend** | React · Vite |
| **Other** | TTS synthesis · manager summaries · prevention guidance generation |

> The attacker model in the paper's experiments was GPT-4o-mini.

---

## Limitations and reflection

This is a synthetic-dialogue study of a specific impersonation type. Search activates only after persuasion stalls, so the ON/OFF success-rate difference is not a causal estimate of the search feature. Recording procedural steps and state changes was as important as the final transcript.

Figure source: Figures 1–4 from the [VishBox v2 paper](https://aclanthology.org/2026.acl-industry.145/), CC BY 4.0.
