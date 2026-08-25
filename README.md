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
echo "..." > PROMPT.md      # what to do each iteration — see below
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

`PROMPT.md` is the whole configuration. A workable skeleton — copy
[`PROMPT.example.md`](PROMPT.example.md) and edit:

1. **Orient** — read the plan, the last progress entry, recent commits.
2. **Health check** — lint, typecheck, test. Red `main` is the only task.
3. **Pick ONE task** — smallest change that is fully verifiable this iteration.
4. **Do it, prove it** — tests green, evidence for user-facing behavior.
5. **Commit and push.**
6. **Hand off** — update the plan, append one line to the progress log, then exit.

The two rules that matter: never leave `main` broken, and write down anything you noticed but did
not do. Everything else is taste.

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
