# bigcodebench

Evaluation-only package for **BigCodeBench** — runs candidate solutions against
each task's tests in a resource sandbox and reports pass@k, so it can be reused
by any harness.

It carries no generation code: no model providers, no remote execution backends,
and none of the dependencies they need (`vllm`, `transformers`, `accelerate`,
`anthropic`, `openai`, `e2b`, `gradio-client`). Bring your own solutions.

Problems come from [`bzantium/bigcodebench`](https://huggingface.co/datasets/bzantium/bigcodebench)
(and `-hard`), a port of upstream's v0.1.4 whose reference solutions and tests
target current numpy / pandas / scikit-learn. Prompts are unchanged, so the task
a model is asked to solve is the original one.

## Install

```bash
pip install 'bigcodebench[eval] @ git+https://github.com/bzantium/bigcodebench.git'
```

The `eval` extra installs the libraries the tasks themselves import (pandas,
scikit-learn, tensorflow, seaborn, …). Without it, tasks importing a missing
module fail on `ImportError` and score 0.

## Usage

`evaluate` takes a jsonl of `{"task_id": ..., "solution": ...}` rows and writes
`*_eval_results.json` and `*_pass_at_k.json` next to it:

```python
from bigcodebench.evaluate import evaluate

evaluate(
    "instruct",              # split: complete | instruct
    "full",                  # subset: full | hard
    samples="samples.jsonl",
    pass_k="1",
    calibrated=True,
    save_pass_rate=True,
    parallel=16,
)
```

Pass `check_gt_only=True` to run the reference solutions instead, which reports
how many tasks are gradeable at all in the current environment.

Problems are also reachable on their own:

```python
from bigcodebench.data import get_bigcodebench

problems = get_bigcodebench()            # or subset="hard"
```

## Environment

| Variable | Effect |
|---|---|
| `BIGCODEBENCH_OVERRIDE_PATH` | Grade against a local jsonl instead of the dataset |
| `BIGCODEBENCH_HF` | Use a different HuggingFace dataset |
| `BIGCODEBENCH_TIMEOUT_PER_TASK` | Per-task wall-clock ceiling in seconds (default 240) |
| `MALLOC_ARENA_MAX` | Cap glibc's per-thread malloc arenas |

Downloaded problems are cached per dataset, so switching datasets does not reuse
the previous one.

The slowest reference solution runs for about four minutes, close to the default
ceiling; raise it if tasks time out under load. This does not loosen grading — a
solution's time limit is derived from how long the reference took.

Solutions run under `RLIMIT_AS`/`RLIMIT_DATA` (30 GiB). On a many-core host,
glibc's per-thread malloc arenas can exhaust that before a test runs; setting
`MALLOC_ARENA_MAX=2` in the environment avoids it. glibc reads the variable at
startup, so it cannot be set from inside the process.

Tasks importing `turtle` need `tkinter`, which some interpreter builds omit.
Install your OS package (`apt install python3.X-tk`), or `pip install
tkinter-bundle` where that is unavailable.

## License

Apache-2.0, as upstream. See [LICENSE](LICENSE).
