# bigcodebench

Evaluation-only port of [bigcode-project/bigcodebench](https://github.com/bigcode-project/bigcodebench)
intended for use as a third-party submodule by harnesses such as
[nemo-skills-harness](https://github.daumkakao.com/lmt/nemo-skills-harness)
(BFCL-eval pattern).

## What's here

| Module | Purpose |
|---|---|
| `bigcodebench.evaluate` | `evaluate(...)` — runs solutions against the task tests locally and writes pass@k |
| `bigcodebench.data` | Problem loading and the samples/results jsonl helpers |
| `bigcodebench.eval` | Test execution under the resource sandbox, plus the special-case oracles |
| `bigcodebench.gen.util` | `trusted_check`, used to time the reference solutions |

## What's NOT here (vs. upstream)

- Generation (`generate.py`, `provider/`, `gen/util/*_request.py`) — the consumer
  harness drives generation
- Remote execution — the `gradio` and `e2b` branches of `evaluate()`, along with
  their endpoint arguments. Only `local` remains, so `execution` is gone as a
  parameter
- The `bigcodebench.evaluate` console script and its `fire` CLI
- `sanitize.py`, `syncheck.py`, `inspect.py` — post-processing the harness does
  itself

Because generation is gone, so are its dependencies: `vllm`, `transformers`,
`accelerate`, `anthropic`, `google-genai`, `mistralai`, `openai`, `e2b` and
`gradio-client`. That matters for a harness that pins those versions for its own
inference stack — upstream's install would fight those pins.

## Problem data

Problems come from [`bzantium/bigcodebench`](https://huggingface.co/datasets/bzantium/bigcodebench)
(and `-hard`), a port of upstream's v0.1.4 whose reference solutions and tests
target current numpy / pandas / scikit-learn. Prompts are unchanged from
upstream, so the task a model is asked to solve is the original one.

Upstream's own solutions are written against numpy 1.x and pandas 2.0; on a
current stack 67 of the 1140 fail, meaning a correct model answer scores zero.
Against the ported data the groundtruth pass rate is 1.000.

Set `BIGCODEBENCH_OVERRIDE_PATH` to a jsonl to grade against your own problems
instead, or point `BIGCODEBENCH_HF` at another dataset. Downloaded problems are
cached under a filename carrying the dataset owner, so switching datasets does
not silently reuse the previous one — upstream keyed the cache on the version
string alone, which meant a fork sharing `v0.1.4` was graded against whatever
had been downloaded first.

## Install

```bash
pip install -e '.[eval]'
```

The `eval` extra installs the libraries the tasks themselves import (pandas,
scikit-learn, tensorflow, seaborn, …), unpinned. Without it, tasks importing a
missing module fail on `ImportError` and score 0.

## Usage

```python
from bigcodebench.evaluate import evaluate

evaluate(
    "instruct",              # split: complete | instruct
    "full",                  # subset: full | hard
    samples="samples.jsonl", # rows of {"task_id": ..., "solution": ...}
    pass_k="1",
    calibrated=True,
    save_pass_rate=True,
    parallel=16,
)
```

Results are written next to the samples file as `*_eval_results.json` and
`*_pass_at_k.json`.

To check the reference solutions without a model, pass `check_gt_only=True`.

## Environment

| Variable | Effect |
|---|---|
| `BIGCODEBENCH_OVERRIDE_PATH` | Grade against a local jsonl instead of the dataset |
| `BIGCODEBENCH_TIMEOUT_PER_TASK` | Per-task wall-clock ceiling in seconds (default 240) |
| `MALLOC_ARENA_MAX` | Cap glibc's per-thread arenas (see below) |

The slowest reference solution takes ~235s, so the default ceiling leaves little
room once workers contend; raise it if tasks time out. Doing so does not loosen
grading — a solution's own limit is derived from how long the reference took.
Upstream read this variable as a string and only applied it to the outer process
join, so it had no effect on the limit the tests ran under.

Generated code runs under `RLIMIT_AS`/`RLIMIT_DATA` (30 GiB by default). On a
many-core host, glibc's per-thread malloc arenas can exhaust that before a test
runs; export `MALLOC_ARENA_MAX=2` if you hit it. It must be set before the
process starts — glibc reads it at startup.

Tasks that import `turtle` need `tkinter`, which is not part of a pip install.
Use your OS package (`apt install python3.X-tk`), or `pip install tkinter-bundle`
where that is unavailable.

## License

Apache-2.0, as upstream. See [LICENSE](LICENSE).
