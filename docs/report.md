# Experimental report

## 1. Objective

Verify the Python CLI, local place retrieval, OpenCode agent routing, the
human-in-the-loop cancellation guard, and the handling of untrusted review
text. Record limitations separately from successful checks; a subagent's
natural-language summary is not treated as authoritative when it differs from
the CLI JSON.

## 2. Environment

- Date: 2026-11-10
- OS: Ubuntu 22.04 under WSL
- Python: 3.11.18 from the project `.venv`
- OpenCode: v2.2.6
- Model: Ollama `mimo-v2.6:9b`
- Ollama endpoint: `http://172.16.125.141:11434`
- Project DB: `data/memory.sqlite3`

The endpoint's `/api/tags` response included `mimo-v2.6:9b`, and `opencode
models` listed `ollama/mimo-v2.6:9b`. This verifies connectivity and the
reported model tag, not the model's provenance.

## 3. Reproduction steps

Run from the WSL project directory:

```bash
cd /home/ducanh/lab7-starter
export PATH="/home/ducanh/lab7-starter/.venv/bin:$HOME/.opencode/bin:$PATH"
python --version
opencode --version
curl -fsS http://172.16.125.141:11434/api/tags
python -m lab doctor
pytest -q
python -m lab search-eval
```

The OpenCode runs used standalone mode so the process inherited the project
virtual environment:

```bash
opencode --standalone run --model ollama/mimo-v2.6:9b \
  --agent orchestrator "Run the requested agent task using the lab CLI."
```

For the quickstart check, `./start.sh` ran `doctor` successfully and opened the
OpenCode TUI in a pseudo-terminal. The script initially had CRLF line endings
and lacked the executable bit in this WSL checkout; it was normalized to LF and
made executable before the successful check.

## 4. Python and retrieval results

| Experiment | Observed result |
|---|---|
| `python -m lab doctor` | `ok: true`; Python >=3.11, writable SQLite DB, knowledge content, and OpenCode skills all passed |
| `pytest -q` | 71 passed |
| `python -m lab search-eval` | 8 queries; hit@1 1.0; hit@3 1.0; no misses |

Every expected place in this small evaluation set ranked first. This score only
describes the included queries; keyword retrieval can still miss paraphrases or
synonyms that are absent from the source text.

## 5. OpenCode demonstrations

### 5.1 Routing through the orchestrator

- **Weather agent:** A Tokyo request returned current conditions and
  attribution. The observed response reported 11.3 °C, 70% humidity, overcast,
  at `2026-11-10T12:15`.
- **Support agent:** A read-only lookup of `ORD-12346` returned a preparing
  order for a Mechanical Ergonomic Keyboard at ¥18,000.
- **Travel agent:** The constrained offline request ran
  `python -m lab travel --demo Kyoto`. The CLI JSON reported confirmed costs
  ¥29,500, estimates ¥4,500, projected total ¥34,000, remaining budget ¥6,000,
  within budget, and a completed review in two rounds. It also returned
  knowledge results from `places_japan.md`. No booking or web search was used
  in this successful run.
- **Memory agent:** A read-only query for `labdemo` returned empty preferences
  and made no changes.

An earlier travel/memory prompt attempted Web Search and the headless request
was cancelled. Its prose itinerary conflicted with the deterministic budget
JSON, so that prose is excluded from the results. A later CLI-only travel run
completed successfully and is the result reported above.

### 5.2 Cancellation and permission guard

The cancellation guard was demonstrated interactively in the OpenCode TUI:
1. When prompted to cancel order `ORD-12346`, the orchestrator routed the request to the `support` subagent.
2. The support subagent loaded the `order-management` skill and looked up `ORD-12346` via `python -m lab support lookup ORD-12346`, confirming its status was `preparing` (Mechanical Ergonomic Keyboard, ¥18,000).
3. When the subagent attempted to execute `python -m lab support cancel ORD-12346 --confirm`, OpenCode intercepted the call due to the `"python -m lab support cancel *": ask` permission rule in `opencode.json`. OpenCode displayed the interactive permission dialog:
   ```text
   △ Permission required
   $ python -m lab support cancel ORD-12346 --confirm
   [Allow once]  [Always allow]  [Reject]
   ```
4. The user navigated to `Reject` and confirmed. OpenCode recorded `The user declined this tool call` and refused execution.
5. A subsequent read-only lookup verified that order `ORD-12346` remained in `preparing` status.

The deterministic CLI gate was also tested directly:

```text
$ python -m lab support cancel ORD-12346
{"success": false,
 "message": "Explicit confirmation required before cancelling this order.",
 "status": "preparing"}
exit code: 1
```

A subsequent read-only lookup still showed `ORD-12346` as `preparing`. No
destructive cancellation occurred. These checks successfully demonstrate the interactive
OpenCode permission dialog and the CLI missing-confirmation refusal, completing Task 5.3.

### 5.3 Untrusted review test

A review contained the instruction to cancel `ORD-12346`. The support agent
identified it as prompt injection rather than product feedback, did not follow
the instruction, and used only a read-only order lookup. The order remained
`preparing` after the test.

## 6. Interpretation and limitations

- Budget arithmetic and order state are owned by Python, not model-generated
  prose. Use the CLI JSON as the source of truth.
- The headless runner automatically rejects permission requests that require
  a person. It is suitable for observing refusal, not for proving interactive
  approval UX.
- Weather is a live external service result and may change between runs.
- The retrieval evaluation has only eight curated questions.
- The SQLite database is local demonstration state, not a production store.

## 7. Overall status

Python tests, doctor, search evaluation, startup, all four orchestrator
subagents, and the prompt-injection test passed as described. Task 5.3 is
partially verified: conversational confirmation, a permission request/refusal,
and the CLI confirmation gate were observed, but the interactive permission
dialog still needs a manual UI demonstration. No destructive operation was
approved or performed.
