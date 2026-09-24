# Character Fidelity Lab

**A retrieval-grounded character pipeline with versioned instructions, source citations, and a repeatable evaluation harness.**

I built this to test how changes to a character's rulebook affect factual recall, citation use, resistance to manipulation, and character voice. The sample uses Maelor, a fictional character, and a supporting lore set to evaluate the pipeline against Google Gemini.

The project connects semantic retrieval, provider integration, deterministic input/output checks, and comparative evaluation. Published prompts, test cases, answer logs, and scores make the development decisions inspectable.

## Results: two versions of the character, tested head to head

I wrote two versions of Maelor's rulebook: instructions defining his voice, what he knows, and what he won't do. Version 2 adds stricter rules about only speaking from source material and refusing cleanly. Both versions ran through the same 21-question test set against Google Gemini.

| What was measured | v1 | v2 | Change |
|---|---:|---:|---:|
| Got the facts right | 76.2% | 71.4% | -4.8 points |
| Cited the right source | 57.1% | 66.7% | +9.5 points |
| Refused attacks/tricks correctly | 95.2% | 100.0% | +4.8 points |
| Stayed in character | 100.0% | 100.0% | no change |
| Didn't leak internal instructions | 100.0% | 100.0% | no change |
| **Overall** | **85.7%** | **87.6%** | **+1.9 points** |

The comparison exposed a useful design tradeoff: stricter source and refusal instructions improved citation and attack-response scores while reducing fact recall. That gives the next prompt iteration a specific target, visible in the saved answers.

These are deterministic rubric scores on the same 21-question sample drawn from the 59-case suite, with Gemini used for every answer; they describe this comparison rather than a general model benchmark. One changed answer moves a per-metric score by about 4.8 percentage points.

Full data: [`reports/v1_gemini.jsonl`](reports/v1_gemini.jsonl), [`reports/v2_gemini.jsonl`](reports/v2_gemini.jsonl), [`reports/v1_vs_v2_gemini.md`](reports/v1_vs_v2_gemini.md).

## How it works

A question is checked against a list of manipulation patterns before it reaches the model. If it doesn't match, the system searches a library of lore for the passages closest in meaning to the question. Those passages, plus the character's rulebook, are sent to the AI model as instructions and context. The model generates an answer. The answer is checked for leaked instructions or a missing source citation. A test suite runs this process over a fixed set of questions and scores each answer.

```
lore library -> search for relevant passages -> build instructions -> AI model writes answer -> check the answer -> grade it
```

| Stage | What it does | Code |
|---|---|---|
| Preparing the lore | Splits source material into small searchable passages | `chunking.py` |
| Making it searchable | Converts text into a form that can be compared by meaning | `embeddings.py` |
| Searching | Finds the passages closest in meaning to a question | `vector_store.py` |
| Building instructions | Combines the character's rules with the retrieved passages | `prompting.py` |
| Filtering | Blocks or flags suspicious questions and answers | `guardrails.py` |
| Talking to the AI model | Sends the request to Gemini, OpenAI, or Anthropic | `providers/` |
| Grading | Scores every test answer and compares versions | `scorer.py`, `run_eval.py`, `compare_runs.py` |

### Inspectable input and output checks

The input filter uses deterministic phrase matching for known manipulation patterns; output checks flag leaked instructions and missing source citations. These lightweight controls are easy to inspect and test offline. Phrase matching covers the listed patterns, so paraphrased attacks require additional detection methods; the published attack score measures the included test cases.

### Scoring method

The suite contains 59 hand-written questions across 7 categories in `data/evals/eval_cases.jsonl`. Scoring checks expected terms, citations, and rule compliance, making version comparisons fast and reproducible. The answer logs support qualitative review where paraphrases or character voice call for judgment beyond the deterministic rubric.

## Extending the experiment

Character rulebooks, lore, test cases, and provider adapters are separate inputs to the same pipeline. The next evaluation step is a full-suite comparison with paraphrase-aware review, followed by additional characters and regression runs when prompts or provider models change.

## Running it

```bash
python -m pip install -e .
pip install google-genai
export GEMINI_API_KEY="..."

make index    # build the real Gemini-embedded search index
make ask      # ask a question
make compare  # run v1 vs v2 tests, write the comparison report
```

`run_eval.py --max-per-category N` runs a smaller sample instead of all 59 questions. `python -m unittest discover -s tests` runs offline with no API key, testing chunking, retrieval, guardrails, and scoring directly.

## Repository layout

- `src/character_lab/` — the pipeline code.
- `data/characters/` — the character rulebooks (v1 and v2).
- `data/lore/` — the source material.
- `data/evals/` — the test questions.
- `data/index/` — the built search index.
- `reports/` — test results.
- `tests/` — offline tests for chunking, retrieval, guardrails, and scoring.
