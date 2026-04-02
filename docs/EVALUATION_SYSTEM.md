# Evaluation & Simulation Framework Spec

## Purpose

This framework validates the correctness, robustness, and safety of the Station95 scheduling agent by combining:

- Deterministic test cases (known scenarios)
- LLM-driven simulation (realistic variability)
- Multi-turn conversational testing
- Adversarial testing (failure + escalation paths)

The goal is to ensure **reliable behavior across updates to prompts, models, and code**, and to prevent regressions before deployment.

---

## Core Principles

1. **Deterministic where possible, LLM where necessary**
2. **All tests must be reproducible**
3. **External systems must be isolated and resettable**
4. **Evaluation must produce measurable metrics**
5. **Multi-turn workflows must be explicitly tested**

---
## Deployment
A seperate development environment must be set up for this `EVALUATION SYSTEM`.  

- **Dev/Test Supabase**
- **Mock GroupMe (don’t hit real chat)**
- **Sandbox Calendar Service OR mock**

## Data resetting
A mechanism must be designed to reset the data before (or just after) running all tests.  The database and calendar must be in a known state (a state that is known by the testing framework as well as the human evaluators).  Since requests are contemporary (the meaning of "today" is actually evaluated at the time the request is sent), 

--

## Test Categories

---

### 1. Deterministic Test Suite (Required)

#### Description
Hard-coded scenarios designed by engineers to validate known behaviors.

#### Input
```json
{
  "name": "no_crew_simple",
  "messages": ["42 can't make it tonight"],
  "initial_state": {
    "date": "2026-05-01",
    "schedule": {
      "1800-0600": [42, 35]
    }
  },
  "expected": {
    "actions": [
      {
        "type": "noCrew",
        "squad": 42,
        "date": "20260501"
      }
    ],
    "clarifications": [],
    "errors": []
  }
}
````

#### Validation (Deterministic)

* Calendar actions EXACT match
* No unexpected clarifications
* No incorrect assumptions

#### Coverage Areas

* Add/remove crew
* Already-absent squad
* Multi-day requests
* Date resolution
* Invalid squad IDs

---

### 2. LLM-Driven Scenario Testing (Fuzzing)

#### Description

An LLM generates realistic variations of user messages.

#### Example

Seed:

`"42 can't make it tonight"`

Generated:

* "we're out tonight"
* "no crew for 42 tn"
* "42 unavailable this evening"

#### Flow

1. Generate N variations
2. Send through system
3. Compare outputs to expected behavior

#### Evaluation

* Deterministic checks (actions)
* LLM judge (semantic correctness)

---

### 3. Multi-Turn Conversation Testing

#### Description

Simulates ambiguous conversations requiring clarification.

#### Example Scenario

```json
{
  "messages": [
    "We can't make it tonight",
    "42",
    "Actually also tomorrow"
  ],
  "expected": {
    "actions": [
      {"type": "noCrew", "squad": 42, "date": "20260501"},
      {"type": "noCrew", "squad": 42, "date": "20260502"}
    ],
    "clarifications": [
      "Which squad?"
    ]
  }
}
```




#### Requirements

* Conversation state persists across turns
* Clarifications are appropriate and minimal
* Workflow resumes correctly

#### Validation

* Correct number of clarification steps
* Correct final actions
* No premature execution

---

### 4. Adversarial Testing

#### Description

Simulates edge cases and attempts to break the system.

#### Examples

* Conflicting inputs:

  * "42 is out tonight"
  * "Actually nevermind"
* Ambiguous:

  * "we might not have coverage"
* Noise:

  * "great job tonight"
* Invalid:

  * "99 is covering tonight"

#### Expected Behavior

* No incorrect calendar updates
* Proper clarification OR ignore
* Escalation when needed

#### Escalation Cases

* Exceeds clarification limit
* Conflicting instructions
* Low-confidence interpretation

---

## LLM Judge (Hybrid Evaluation)

#### Role

Evaluate non-deterministic aspects:

* Clarification quality
* Semantic correctness
* Tone / UX

#### Output

```json
{
  "verdict": "correct | incorrect | partial",
  "confidence": 0.0-1.0,
  "reason": "..."
}
```

#### Rule

* Deterministic checks override LLM judge
* LLM judge is advisory, not authoritative

---

## Environment Strategy (CRITICAL)

### Problem

Tests depend on:

* Calendar state
* Dates (relative phrases like "tonight", "next Tuesday")
* External services

### Solution: Isolated Test Environment

#### Required Components

* **Supabase (test instance)**
* **Mock GroupMe client**
* **Mock or sandbox Calendar Service**

---

## Calendar State Management

### Approach: Snapshot + Reset

Each test defines an `initial_state`.

Before test:

1. Reset calendar to baseline
2. Apply test-specific state

#### Option A (Preferred)

Calendar service supports:

```http
POST /test/reset
POST /test/load_state
```

#### Option B

Use database-level reset:

* Replace schedule table with fixture data

---

## Time Control (VERY IMPORTANT)

### Problem

"tonight" changes based on real date

### Solution: Freeze Time

All tests run with:

```python
TEST_CURRENT_DATE = "2026-05-01"
```

#### Implementation Options

* Inject clock service
* Override `datetime.now()`
* Pass date into all LLM prompts

---

## Handling Future Dates

Example:

> "Next month we won't have crew on Tuesdays"

### Strategy

* Expand into concrete dates:

  * 2026-06-02, 2026-06-09, etc.
* Validate generated commands

---

## Execution Flow

```
For each test:
    Reset environment
    Set test date
    Load initial state
    Replay messages (sequentially)
    Capture outputs
    Run validators
    Store results
```

---

## Metrics

| Metric             | Description                |
| ------------------ | -------------------------- |
| Accuracy           | % correct actions          |
| Clarification Rate | # clarifications / test    |
| Escalation Rate    | % escalated workflows      |
| False Positives    | Noise incorrectly acted on |
| Latency            | Optional                   |

---

## Running Tests

### Local

```bash
pytest tests/evaluation/
```

### CI/CD (Recommended)

* Run on every PR
* Block merge if:

  * Accuracy < threshold (e.g., 95%)
  * Any deterministic test fails

### Nightly

* Run full simulation suite (LLM-heavy)

---

## Cost Control

* Deterministic suite → always run
* LLM simulation → sample subset
* Adversarial → nightly only

---

## Optional: Scenario DSL

Define scenarios in YAML/JSON:

```yaml
name: no_crew
messages:
  - "42 can't make it tonight"
expected:
  actions:
    - type: noCrew
      squad: 42
```

---

## Summary

This framework ensures:

* Deterministic correctness for known cases
* Robustness to real-world language variation
* Proper handling of ambiguity and multi-turn workflows
* Protection against regressions and model drift
* Safe interaction with external systems via controlled environments

```

