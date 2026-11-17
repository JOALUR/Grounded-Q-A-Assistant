# Grounded Q&A Assistant — Group 7

Group 7 builds the grounding guardrail: every answer must be supported by a
provided passage and include a citation, or return `not found in the sources`.

This repository currently contains the Python project setup and three sample
passages. Answer generation, validation, and repair are not implemented yet.

## Scope

- [Epic #7](https://github.com/swith004/mellea-grounded-qa/issues/7): supported answers or refusal.
- [Issue #10](https://github.com/swith004/mellea-grounded-qa/issues/10): a lowercase `@generative` function returning a Pydantic answer model.
- [Issue #11](https://github.com/swith004/mellea-grounded-qa/issues/11): verify that the cited passage supports the answer.
- [Issue #12](https://github.com/swith004/mellea-grounded-qa/issues/12): retry within a fixed budget, then refuse if still unsupported.

Group 5 supplies chunking and retrieval, Group 6 supplies evaluation questions
and the baseline, and Group 8 supplies benchmarking, according to the course
lane diagram. GitHub also lists baseline issue #13 under epic #7; confirm its
ownership with the groups before taking on that work.

## Shared interface contract

```text
Passage = {id, doc, text}
retrieve(question, k=5) -> list[Passage]
answer(question) -> Answer
Answer = {text, citation_id, quote, found}
```

The baseline and grounded implementation must return the same answer shape.
Coordinate field types and the representation of empty citation fields on
refusal with the other groups before implementation. Production passage IDs
come from Group 5 and must remain stable after week 2.

## Local setup

### First installation

Requires Python 3.11 or newer. Run these commands from the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e '.[dev]'
python -m pip check
python -c 'from importlib.metadata import version; import grounded_qa; from mellea import generative, start_session; print("Mellea:", version("mellea")); print("Package imports OK")'
```

Mellea is pinned to 0.9.0, verified on PyPI on November 17, 2026. Pydantic and
pytest use version ranges; this setup does not yet lock all transitive
dependencies. Agree on a shared dependency lock before running comparisons.
There are no project tests yet; pytest is included for the upcoming guardrail
tests.

Installation reference: <https://docs.mellea.ai/getting-started/installation>.

### Returning to the project

The initial development machine already has `.venv` and the dependencies
installed. In a new terminal, run:

```bash
cd /Users/joseuribe/Grounded-Q-A-Assistant
source .venv/bin/activate
```

Activation selects the project's Python and tools for that terminal. It is
needed for the unqualified `python` commands below, but not for Git commands.
You can also run `.venv/bin/python` directly without activation. Run
`deactivate` when finished. Other contributors should use their own checkout
path and follow the first-installation steps above.

### Model runtime

Importing the package and inspecting fixtures do not require a running model.
Actual generation needs a backend. Ollama is a local option and is already
installed on the initial development machine.

Check the client, start its server if needed, and inspect installed models:

```bash
ollama --version
ollama serve
```

Leave that terminal running. In another terminal:

```bash
ollama list
```

If the Ollama app already runs the server, skip `ollama serve`. Select a model
explicitly when implementing generation and record its tag for reproducible
comparisons. Installing the Python dependencies alone does not establish that
inference works. The specialized Hugging Face groundedness requirement may need extra
dependencies and hardware; it is not part of this initial setup.

On the initial development machine, Ollama has `granite4.1:3b` installed.
On a new machine, install Ollama from <https://ollama.com> and download the
same model with `ollama pull granite4.1:3b` before running the example.
Use it explicitly rather than relying on Mellea's default model:

```python
from mellea import start_session

m = start_session("ollama", model_id="granite4.1:3b", model_options={"temperature": 0})
result = m.instruct(
    "Answer with only the number. Passage: COP 3530 has 3 credits. "
    "Question: How many credits does COP 3530 have?"
)
print(str(result))
```

This is an inference smoke check, not the typed answer or grounding guardrail.
Save the example to a Python file and run it with the activated environment,
or paste it into an interactive `python` session.

### Verified setup

On November 17, 2026, the initial Apple Silicon development machine passed:

- Python 3.13.9 with Mellea 0.9.0, Pydantic 2.14.3, and pytest 9.2.0.
- `python -m pip check` with no broken requirements.
- Imports of the local package and Mellea's `generative` and `start_session`.
- Local inference through Ollama with `granite4.1:3b`; the example returned `3`.

The virtual environment is ignored by Git. Each contributor installs their
own environment; successful inference does not yet validate grounding.

## Start without waiting for retrieval

The project dataset is the supplied `CSCourseSyllabi.zip`: 159 `.docx` files,
eight older `.doc` files, and one PNG. The three passages below are only the
starting fixture, not the full dataset. Full extraction and chunking are a
later integration with Group 5.

`data/fixtures/passages.jsonl` contains three real excerpts from
`CSCourseSyllabi.zip` → `CSCourseSyllabi/COP_3530.docx` (syllabus dated
May 17, 2024). Paragraph separators were normalized to newlines. These are
development fixtures, not current academic advising information or the final
corpus. The `fixture-` IDs are temporary and are not Group 5's production IDs.

Use all three passages directly as context for issue #10. For example:

- Supported: "How many credits is COP 3530?" → 3, supported by `fixture-cop3530-001`.
- Supported: "What are the prerequisites for COP 3530?" → supported by `fixture-cop3530-003`.
- Unsupported by these fixtures: "Who teaches COP 3530?" → refuse.

Keep the full archive outside Git for this initial setup. Raw local copies can
go in the ignored `data/raw/` directory later.

## Implementation milestones

1. Define `Passage` and `Answer` models and generate a typed answer from the
   three fixture passages (#10).
2. Reject unknown citation IDs, empty supporting quotes, quotes absent from
   the cited passage, and answers the cited passage does not support (#11).
   A matching quote alone does not prove that the answer is supported.
3. Add bounded retries and a consistent refusal result (#12).
4. Replace fixture input with the shared retriever and compare with the
   baseline using the same questions, retrieved passages, and model settings.

Tests should cover supported answers, fabricated citations, real quotes paired
with unsupported claims, and refusal after the retry budget is exhausted.
Lower hallucination rates remain an experimental claim until measured.

