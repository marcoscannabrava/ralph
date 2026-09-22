# ralph

The grug [ralph loop](https://ghuntley.com/ralph/), in one bash file that just works.

Feed `PROMPT.md` to `claude` over and over until the work is done. Every iteration is a fresh
session with no memory, so all continuity lives in the repo — your plan file, your progress log,
git history, issues. That constraint is the point: the agent has to write down what it did, or
the next iteration will not know.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/marcoscannabrava/ralph/main/ralph -o ~/.local/bin/ralph
chmod +x ~/.local/bin/ralph
```

One file. Needs `bash >= 4.4`, `git`, [`claude`](https://claude.com/claude-code), and `jq` (without
`jq` it still runs, just prints raw JSON).

## Use

```sh
cd your-repo
echo "..." > PROMPT.md      # what the project is — see below
ralph                       # loop until stopped
ralph 5                     # at most 5 iterations
```

It runs in the git repo you launch it from, at the top level whatever subdirectory you are in.

**Stopping.** `Ctrl+C` stops the session in flight (again to force). `touch RALPH_STOP` also stops
it — that is the agent's escape hatch when it runs out of work, and the file is never removed for
you, so it doubles as an operator brake.

**Logs.** `logs/<utc-timestamp>/` at the repo root: `run.log` plus one `iter-NNN.jsonl` of raw
session events per iteration, with `logs/latest` pointing at the newest run. Add `logs/` and
`RALPH_STOP` to your `.gitignore`.

## The prompt

Every iteration is fed two things: the built-in loop contract, then your `PROMPT.md`.

The contract ([`PROMPT.template.md`](PROMPT.template.md), embedded in `ralph`) is the part that
makes the loop work. It tells the fresh session that it is mid-project, not mid-request: orient on
the diff and the progress log first, finish what the last run left cut off, pick one verifiable
task, prove it, commit, hand off. Without it an agent handed a bare `PROMPT.md` reads it as a
one-shot request, sees the repo already has an answer to it, and reports nothing to do — which is
how a run that hit its turn limit turns into a loop that does nothing.

So `PROMPT.md` only has to say what the project is. Goal, constraints, what done looks like, and
anything specific to your stack:

```md
Build a CLI that renders Markdown tables to aligned plain text.

- Rust, no dependencies outside std.
- `cargo fmt`, `cargo clippy -- -D warnings` and `cargo test` must pass.
- Done: reads stdin, handles alignment markers, `--width`, and ships a README.
```

`RALPH_TEMPLATE=0` turns the contract off when you want the prompt sent verbatim.

## Knobs

All optional, all environment variables.

| Variable | Default | What it does |
| --- | --- | --- |
| `RALPH_PROMPT` | `PROMPT.md` | prompt fed to every iteration |
| `RALPH_MODEL` | account default | `--model` for claude |
| `RALPH_PERMISSION_MODE` | CLI default | `--permission-mode`; use `bypassPermissions` for unattended runs |
| `RALPH_TIMEOUT` | `3600` | per-iteration wall clock, seconds (`0` = no limit) |
| `RALPH_SLEEP` | `5` | breather between iterations, seconds |
| `RALPH_MAX_FAILURES` | `3` | consecutive failures before giving up |
| `RALPH_MAX_COST_USD` | `0` | stop once cumulative cost exceeds this (`0` = no cap) |
| `RALPH_PULL` | `1` | `git pull --rebase --autostash` between iterations |
| `RALPH_TEMPLATE` | `1` | prepend the built-in loop contract (`0` = your prompt verbatim) |
| `RALPH_ARGS` | — | extra arguments appended to the `claude` invocation |

```sh
RALPH_PERMISSION_MODE=bypassPermissions RALPH_MAX_COST_USD=20 ralph
```

Unattended runs want `bypassPermissions`. Without it, any tool call needing approval is
auto-denied — `claude -p` cannot answer a prompt, so the agent quietly loses hands.

## Why it looks like this

Three things in here are load-bearing and non-obvious:

- **Streaming.** Each iteration runs `--output-format stream-json` and renders it with an embedded
  `jq` program. The default text output prints nothing until the session ends, which makes a
  working loop look frozen for minutes at a time.
- **Ctrl+C actually stops it.** The session runs as an async job, not a foreground pipeline: bash
  defers trap handlers until the running foreground command returns, so a foreground session
  swallows the interrupt for as long as it lasts. `wait` is interruptible, so the handler fires the
  instant the signal lands.
- **Stopping means gone.** The loop walks the process tree pid by pid, `TERM`s it, and escalates to
  `KILL` if the session ignores it. A session that outlives its loop gets reparented to init and
  goes on editing the repo — which is how you end up with two agents committing to one working
  tree. Observed, not hypothesized.

`test/ralph_test.sh` holds all of it in place, against a stub `claude` in a throwaway repo. Two
seconds, no API calls:

```sh
./test/ralph_test.sh
```

## License

MIT
