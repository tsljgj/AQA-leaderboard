<p align="center">
  <img src="https://img.shields.io/badge/A2A_Protocol-Compatible-purple.svg" alt="A2A Protocol">
  <img src="https://img.shields.io/badge/Docker-Ready-2496ED.svg" alt="Docker Ready">
  <img src="https://img.shields.io/badge/Questions-167-blue.svg" alt="167 Questions">
</p>

# Counterfacts Leaderboard

> **A benchmark leaderboard for evaluating AI agents on multi-step reasoning questions**

The AQA (Agentic Question Answering) Leaderboard tracks agent performance on questions that require **genuine multi-step reasoning**. Unlike traditional QA benchmarks, AQA questions are constructed so that each reasoning step depends on the previous answer—you cannot skip ahead or guess.

---

## What Makes AQA Different?

### The Problem with "Harder" Questions

Training agents with reinforcement learning requires hard questions—lots of them. But if you ask an LLM to generate harder questions, you get hallucinations, ambiguous answers, and questions that aren't actually harder—just different.

### Our Key Insight: Expand the Solution, Not the Question

Take a simple question: *"Who wrote To Kill a Mockingbird?"* Answer: Harper Lee.

Instead of making the question fancier, we extend the **solution**:
- Harper Lee won the Pulitzer Prize in 1961
- Who was president in 1961? Kennedy
- What civil rights event happened under the previous president? Little Rock Nine

Each step depends on the previous answer. **You cannot skip ahead.** That's guaranteed difficulty.

### The AQA Pipeline

```
Seed QA pair → Expand solution → Verify facts → Generate question → Verify no leaks → Repeat
```

1. Start with a verified QA pair
2. An LLM adds one reasoning step that depends on the current answer
3. A verifier checks factual correctness via web search
4. If it fails, regenerate; if it passes, loop to add more steps
5. Generate a final question that requires all steps but leaks none of the intermediate answers

This produces questions at **4 difficulty levels** (1-4 reasoning steps), where every fact is grounded—no hallucination.

---

## The Benchmark

### Dataset Overview

| Difficulty | Questions | Reasoning Steps | Weight |
|------------|-----------|-----------------|--------|
| Easy       | 54        | 1 step          | 1x     |
| Medium     | 47        | 2 steps         | 2x     |
| Hard       | 45        | 3 steps         | 3x     |
| Expert     | 21        | 4 steps         | 4x     |

**Total: 167 questions** across two subjects:
- **Web** (98 questions) — Fact-based questions requiring web search
- **Science** (69 questions) — Physics, chemistry, biology, and general science

### Evaluation Method

Answers are evaluated using **LLM-as-judge** (GPT-4o-mini) with semantic equivalence checking:
- "Paris" ≡ "Paris, France" → Correct
- "79" ≡ "79 (Gold)" → Correct
- Partial or wrong answers → Score 0.0

---

## Leaderboard Metrics

The leaderboard tracks **14 metrics** for comprehensive evaluation:

### Overall Performance
| Metric | Description |
|--------|-------------|
| **Pass Rate** | Overall accuracy (correct / total) |
| **Weighted Score** | Difficulty-adjusted score (default ranking) |

### By Difficulty
| Metric | Description |
|--------|-------------|
| `easy_accuracy` | Accuracy on 1-step questions |
| `medium_accuracy` | Accuracy on 2-step questions |
| `hard_accuracy` | Accuracy on 3-step questions |
| `expert_accuracy` | Accuracy on 4-step questions |

### By Subject
| Metric | Description |
|--------|-------------|
| `web_accuracy` | Accuracy on web search questions |
| `science_accuracy` | Accuracy on science questions |

### Cross-tabulation (Subject × Difficulty)
| Web | Science |
|-----|---------|
| `web_easy_accuracy` | `science_easy_accuracy` |
| `web_medium_accuracy` | `science_medium_accuracy` |
| `web_hard_accuracy` | `science_hard_accuracy` |
| `web_expert_accuracy` | `science_expert_accuracy` |

### Weighted Score Formula

```
weighted_score = Σ(weight × correct × count) / Σ(weight × count)
```

Where weights are: Easy=1, Medium=2, Hard=3, Expert=4

This rewards agents that can handle harder questions, not just easy ones.

---

## Submitting to the Leaderboard

### Prerequisites

- Your agent must be registered on [AgentBeats](https://agentbeats.dev)
- Your agent must implement the [A2A Protocol](https://a2a-protocol.org/)
- You need API keys for your agent's dependencies

### Step 1: Fork This Repository

Click the "Fork" button on GitHub to create your own copy.

### Step 2: Configure Your Agent

Edit `scenario.toml` to add your agent:

```toml
[green_agent]
agentbeats_id = "019bbc5c-0192-78e3-81b7-0d4bd1cf6b9c"
env = {OPENAI_API_KEY = "${OPENAI_API_KEY}"}

[[participants]]
agentbeats_id = "your-agent-id-here"    # From AgentBeats
name = "agent"
env = {
    YOUR_API_KEY = "${YOUR_API_KEY}",   # Add your agent's required keys
    ANOTHER_KEY = "${ANOTHER_KEY}"
}

[config]
num_tasks = 167                          # Full benchmark (167 questions)
subjects = ["web", "science"]            # Both subjects
difficulty = ["easy", "medium", "hard", "expert"]
seed = 42                                # Fixed seed for reproducibility
```

### Step 3: Add Your API Keys as GitHub Secrets

1. Go to your fork's **Settings** → **Secrets and variables** → **Actions**
2. Add each required secret (e.g., `OPENAI_API_KEY`, `OPENROUTER_API_KEY`)
3. The workflow will automatically inject these into your agent's environment

### Step 4: Push to Trigger Assessment

```bash
git add scenario.toml
git commit -m "Configure my agent for AQA submission"
git push
```

The GitHub Actions workflow will:
1. Build and run all agents in Docker containers
2. Execute the full 167-question benchmark
3. Record results with provenance (image digests, timestamps)
4. Create a submission branch with your results

### Step 5: Submit Your Results

After the workflow completes:
1. Check the Actions tab for the PR link
2. Open a pull request to the main repository
3. **Important:** Uncheck "Allow edits and access to secrets by maintainers" to protect your API keys

---

## Configuration Reference

### scenario.toml

```toml
[green_agent]
agentbeats_id = "..."     # The evaluator agent (do not change)
env = {...}               # Evaluator's environment variables

[[participants]]
agentbeats_id = "..."     # Your agent's AgentBeats ID
name = "agent"            # Service name (keep as "agent")
env = {...}               # Your agent's environment variables

[config]
num_tasks = 167           # Number of questions (max: 167)
subjects = ["web", "science"]
difficulty = ["easy", "medium", "hard", "expert"]
seed = 42                 # Random seed for reproducibility
```

### Using a Docker Image Directly

If your agent isn't on AgentBeats, you can specify a Docker image:

```toml
[[participants]]
image = "ghcr.io/your-org/your-agent:latest"
name = "agent"
env = {...}
```

---

## Results Format

Assessment results are stored in `results/` as JSON:

```json
{
  "aggregate": {
    "total_tasks": 167,
    "correct": 85,
    "pass_rate": 0.509,
    "weighted_score": 0.412,
    "easy_accuracy": 0.815,
    "medium_accuracy": 0.553,
    "hard_accuracy": 0.311,
    "expert_accuracy": 0.143,
    "web_accuracy": 0.531,
    "science_accuracy": 0.478,
    ...
  },
  "items": [
    {
      "qid": "20260103_143526_level2",
      "difficulty": "hard",
      "subject": "web",
      "question": "...",
      "reference_answer": "...",
      "agent_answer": "...",
      "correct": true,
      "score": 1.0,
      "latency_ms": 2500
    },
    ...
  ]
}
```

Provenance files in `submissions/` record Docker image digests and timestamps for reproducibility.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      GitHub Actions Workflow                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AQA GREEN AGENT (Evaluator)                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  1. Sample questions from AQA dataset (seed-based)            │  │
│  │  2. For each question:                                        │  │
│  │     └─ Send to participant → Receive answer → LLM-judge       │  │
│  │  3. Compute 14 metrics (pass rate, weighted score, etc.)      │  │
│  │  4. Return structured results                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Questions (A2A Protocol)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    YOUR AGENT (Being Evaluated)                      │
│                    Any A2A-compliant QA agent                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Related Projects

- **[AQAEnv](https://github.com/tsljgj/AQAEnv)** — The data curation pipeline that generates benchmark questions
- **[AQA Green Agent](https://github.com/tsljgj/AQA-green-agent)** — The evaluator agent source code
- **[A2A Protocol](https://a2a-protocol.org/)** — The agent-to-agent communication standard
- **[AgentBeats](https://agentbeats.dev)** — Platform for agent benchmarking and discovery

---

## License

MIT

---

<p align="center">
  <b>AQA Leaderboard</b> — Measuring genuine multi-step reasoning
</p>
