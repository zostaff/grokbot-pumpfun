# grokbot-pumpfun

[![CI](https://github.com/zostaff/grokbot-pumpfun/actions/workflows/ci.yml/badge.svg)](https://github.com/zostaff/grokbot-pumpfun/actions/workflows/ci.yml)

A memecoin trading pipeline on pump.fun: a stream of new launches passes through nine
stages, four of which are agents on the Grok API, and the rest are computed in code.
The point of the design is the order of the stages: cheap filters sit before expensive
ones, and only a fraction of a percent of the stream reaches the strong model.

**Trade execution is deliberately left as a stub.** The code that would send
transactions with your key is not generated here — see [Executor](#executor).
By default the project runs in `dry-run`.

## Architecture

```
                    stream of new pump.fun tokens
                                │
┌───────────────────────────────▼───────────────────────────────┐
│ 1. MONITOR            WebSocket, filter in code               │
│    ≥5 buyers · curve <40% · has metadata · >2 min             │
└───────────────────────────────┬───────────────────────────────┘
                    drop ~94%   │
┌───────────────────────────────▼───────────────────────────────┐
│ 1.5 CREATOR MEMORY    code, own closed-trade log              │
│    an address whose token already rugged does not proceed     │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│ 2. ANALYZER           REST ×3 in parallel (asyncio.gather)    │
│    top-5 · snipers · diversity · social signals · curve       │
│    cutoff at risk_score > 7/10; unconditional veto if         │
│    the creator holds ≥25% or the top-5 hold ≥80%              │
└───────────────────────────────┬───────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐      ┌────────────────┐      ┌────────────────┐
│ 3. AUDITOR    │      │ 4. NARRATIVE   │      │ 5. TIMING      │
│ grok-4-fast   │      │ grok-4-fast    │      │ grok-4-fast    │
│ coordination, │      │ trend,         │      │ whole market,  │
│ wash, dump,   │      │ virality,      │      │ 15 min cache   │
│ organic       │      │ community      │      │                │
└───────┬───────┘      └────────┬───────┘      └────────┬───────┘
        └───────────────────────┼───────────────────────┘
                                ▼
┌───────────────────────────────────────────────────────────────┐
│ 6. SCORING MATRIX     code, weights from config               │
│    audit·0.30 + narrative·0.25 + timing·0.15 + metrics·0.30   │
│    below min_total_score → skip with a reason                 │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│ 7. CHECKER            grok-4, adversarial                     │
│    looks for reasons NOT to buy: contradictions, missed flags │
│    approve: false — a normal outcome; errors are also false   │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│ 8. RISK GATE          code, five limits                       │
│    trade cap · daily loss · trades/day ·                      │
│    open positions · exits (background task):                  │
│    stop-loss · take-profit · trailing stop · timer            │
└───────────────────────────────┬───────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────┐
│ 9. EXECUTION          dry-run: tx_hash "dry_run"              │
│                       live:    NotImplementedError (stub)     │
└───────────────────────────────┬───────────────────────────────┘
                                ▼
                    JSONL log: buy / skip / close
```

## Structure

```
grokbot-pumpfun/
├── README.md
├── RUNBOOK.md                # operations: startup, incidents, checklists
├── pyproject.toml            # package, ruff, mypy, pytest
├── Makefile                  # make dev / check / run / replay
├── Dockerfile                # image with no keys inside
├── docker-compose.yml
├── requirements.txt
├── requirements-dev.txt
├── config.example.yaml       # template; config.yaml in .gitignore
├── .env.example              # secrets for compose; .env in .gitignore
├── .github/workflows/ci.yml  # ruff + mypy + pytest on 3.11-3.13 + image
├── src/
│   ├── pipeline.py           # orchestrator, lifecycle, entry point
│   ├── models.py             # pydantic models, config, validation, secrets
│   ├── monitor.py            # WebSocket launch monitor (code)
│   ├── analyzer.py           # REST metrics analyzer (code)
│   ├── agents/
│   │   ├── base.py           # shared Grok call mechanics
│   │   ├── auditor.py        # agent 1: wallet audit
│   │   ├── narrative.py      # agent 2: meme potential
│   │   ├── timing.py         # agent 3: market moment (with cache)
│   │   └── checker.py        # agent 4: adversarial check
│   ├── scoring.py            # scoring matrix (code)
│   ├── risk.py               # risk manager and stop-loss
│   ├── state.py              # state that survives restart
│   ├── ops.py                # Grok limiters, metrics, health, heartbeat
│   ├── executor.py           # execution: dry-run works, live is a stub
│   └── log.py                # JSONL logging with rotation
├── tests/                    # pytest, they do not hit the network
└── scripts/
    ├── replay.py             # log replay and statistics
    ├── dashboard.py          # CLI dashboard
    └── tune.py               # weight and threshold tuning from the log
```

## Quick start

```bash
git clone https://github.com/zostaff/grokbot-pumpfun.git
cd grokbot-pumpfun

make dev                     # venv, dependencies, linter and tests
cp config.example.yaml config.yaml
$EDITOR config.yaml          # Grok key; the rest can be left as is

make check-config            # validate the config without starting anything
make run                     # dry-run, the default mode
```

You can skip putting keys in the file entirely — environment variables take precedence over it:

```bash
export GROKBOT_GROK_API_KEY=xai-...
make run
```

In a container:

```bash
cp .env.example .env && $EDITOR .env
mkdir -p config logs state && cp config.example.yaml config/config.yaml
docker compose up -d && docker compose logs -f
```

What happens next: the pipeline subscribes to new launches and writes
`logs/trades.jsonl`. You can watch as it runs:

```bash
python scripts/dashboard.py logs/trades.jsonl --watch 5   # what's happening now
python scripts/replay.py   logs/trades.jsonl              # summary for the period
```

Tests:

```bash
pytest -v
```

## Agents

**Auditor** (`grok-4-fast`) receives the raw trade stream and the holder list —
not aggregates, but the transactions themselves. It looks for what averages lose:
coordinated buys with identical amounts and intervals under five
seconds, wash trading, the creator splitting a position before dumping, batch buys
in the first second of the token's life. It returns four boolean flags and
the share of organic buyers. When data is insufficient it must set a flag,
not give the token a pass.

**Narrative** (`grok-4-fast`) does not look on-chain at all. Its input is the name,
ticker, description, image, and links; its question is whether this will spread.
Four independent scores from 0 to 1: trend fit, virality,
signs of a living community, launch timing. Clones of yesterday's
hype are scored strictly; missing data is a low score, not a middle one.

**Timing** (`grok-4-fast`, 15-minute cache) scores not the token but the backdrop:
Solana sentiment, whether a meme season is on, volumes on pump.fun, anomalies like
a network outage or a cascade rug. The answer is the same for every token within
the window, so it is cached — otherwise each launch would pay for the same
conclusion. The cache is lock-protected: a burst of simultaneous tokens does not fire three
identical parallel requests. Failures are not cached.

**Checker** (`grok-4`, a stronger model) is the last line before money and
the only agent forbidden from looking for reasons to buy. It receives
every prior conclusion and looks for contradictions among them: high meme potential
with low organic flow, a good bonding curve with concentration in the top-5, a strong
score assembled from one component while the rest fail. `approve: false`
is the expected outcome, not a failure.

### Shared rule for all four

All call mechanics live in `agents/base.py`: request assembly, `temperature=0`,
strict JSON parsing, retries with exponential backoff, 30-second timeout.
Prompts live as constants in the agent modules and end with a requirement
to answer in bare JSON with no markdown wrapper.

**On any error the agent returns the most pessimistic result,
not an empty one.** A timeout, a 500, malformed JSON, a response that does not match the schema — for
the auditor that is every risk flag `true` and zero organic flow, for the checker that is
`approve: false`. A broken check equals a reject, never a silent
pass. This is covered by tests (`tests/test_agents.py`).

## Config

Everything is in `config.yaml`, the template is `config.example.yaml`. **`config.yaml` is in
`.gitignore`**; only the template with placeholders lands in the repository.

| section | what it sets |
|---|---|
| `mode` | `dry-run` or `live` |
| `grok` | key, `fast_model` for the three fast agents, `checker_model` for the checker, timeout, retries |
| `solana` | RPC, wallet private key, Jito: block-engine address and tip size |
| `data` | provider key, REST and WS addresses |
| `risk` | five limits: trade cap, daily loss, trades per day, open positions, stop-loss |
| `filter` | base filter thresholds and `min_total_score` |
| `scoring` | weights of the four components and timing cache TTL |
| `logging` | JSONL path and level |

Scoring weights are normalized: if you write 0.5/0.5/0.5/0.5, the proportions
are kept and the result stays in the 0..1 range.

### Risk management

Position size is proportional to scoring, but capped twice: by the
`max_sol_per_trade` ceiling and by 30% of the remaining daily loss limit. So as
the day goes into the red, bets shrink automatically, and on
reaching `daily_loss_limit_sol` the pipeline stops until the next
UTC day.

### Creator memory

The pipeline examines each launch from a clean slate, so the same
deployer can rug us three times in a row — and each time they will be "new".
The auditor will not recognize them either: it sees one token, not the address history.

`src/reputation.py` keeps a book of addresses from **our own closed
trades** — this is not a list from the internet and not a heuristic, but a fact from our own
log. A close worse than `rug_loss_pct` counts as a rug for that address; after
`block_creator_after_rugs` rugs, their tokens are filtered at the entrance, before
a single Grok request. Separately, `one_position_per_creator` applies: two
tokens from the same deployer are one bet, not two; they usually rug
together.

The book lives in `state/creators.json` and survives restart. Clean addresses
are forgotten after `forget_creators_after_days`; addresses with rugs — never:
they are the book's value.

### Position exits

Open positions are driven by a separate background task, polling prices every
`stop_loss_poll_seconds`. There are four rules, and the order among them is the
priority order:

| rule | when it fires | why |
|---|---|---|
| `stop_loss` | price is `stop_loss_pct` below entry | cap the loss |
| `take_profit` | price is `take_profit_pct` above entry | take the profit |
| `trailing_stop` | pullback from the peak of `trailing_stop_pct` | don't give back what already grew |
| `max_hold` | in the position longer than `max_hold_seconds` | a memecoin that hasn't moved in an hour won't |

Trailing stop is only computed above the entry price: below that, the stop-loss owns
the position, otherwise the two rules would fight over the same drawdown. The price peak
is stored in state and survives restart — otherwise after a restart
the trailing stop would start over from the current price. Zero in any of the three new
parameters disables the corresponding rule; with `stop_loss_pct` alone
the behavior is exactly what it was before.

The exit reason lands in `close.reason` in the log and in the
`exit_<reason>` counter in metrics — from those you can see how positions actually
end: taken profit, pullback, or timer.

### Dry-run mode

By default `mode: dry-run`. The pipeline runs every stage, actually
calls the agents and computes scoring, but instead of a transaction it writes a record with
`tx_hash: "dry_run"`. Prices are real, so stop-loss and PnL in
dry-run are computed against the market.

Switching to live is only by an explicit config edit, and on start the pipeline
prints a warning and requires a flag:

```bash
python -m src.pipeline --config config.yaml --i-understand-the-risk
```

Without the flag, a `live` start is rejected.

## Unattended operation

The pipeline is designed to live for days. What was done for that.

**State survives restart.** Open positions, day counters, and Grok spend
live in `state/pipeline.json` and are loaded on start. Without this
a restart would zero the daily loss limit and the protection against buying
the same token again — both limiters would be counted from scratch. The file is written
atomically (a temp file plus `os.replace`), so a half-written JSON never sits
on disk. Positions are always restored;
day counters — only if the file is from today.

**Shutdown is graceful.** SIGTERM and SIGINT do not tear work apart: the pipeline
stops accepting new tokens, lets those already in analysis finish
(up to `shutdown_grace_seconds`), saves state, and closes connections.
Positions are not sold off — and a warning is written to the log that
stop-loss on them does not work until the process is up again.

**Grok spend is limited in three different ways**, because their refusals are
different: a token bucket does not let the API be hammered faster than agreed, a daily
call budget does not let a launch spike eat a month's money in an evening, and
the circuit breaker opens after `breaker_failures` consecutive failures and
stops calling a place that is not answering anyway. While the circuit is open,
agents return the pessimistic result — meaning the pipeline does not buy.

**Important events are reported outward.** With `alerts.webhook_url` set, the pipeline
sends events to a webhook: start and stop, buy, position close,
creator rug, open circuit, daily limit hit, stalled
launch stream. States are reported on transition, not on every tick; the stream
is rate-limited, and any send error stays in the log and does not touch trading.
Off by default.

**Liveness is visible from outside.** With `ops.health_port` greater than zero,
`GET /healthz` comes up (JSON, 200 on `ok` and 503 on `degraded` — you can hang
a restart policy on it) and `GET /metrics` in Prometheus format. Every
`heartbeat_seconds` the same thing goes to the log as a line. No web framework was
stood up for this: two handlers on `asyncio.start_server`.

**Nothing grows without bound.** JSONL rotates by size
(`logging.max_bytes`, `backups`); the monitor buffer and the list of already-seen
mints are length-capped.

**Secrets do not leak into logs.** Keys are `SecretStr`: they are in neither `repr`, nor
the model dump, nor the traceback. `--check` prints the config with
masked keys. The image contains no secrets at all: they arrive
via `GROKBOT_*`, the config is mounted as a volume.

**A bad config does not start.** A zero trade limit, a threshold out of range,
`live` without a wallet key, a leftover placeholder instead of a Grok key —
all of these are startup errors with a list of problems, not a surprise an hour
into trading. Warnings that do not block startup are printed separately.

What to watch in production, what each complaint means, and how to fix it —
[RUNBOOK.md](RUNBOOK.md).

### Environment variables

| variable | what it sets |
|---|---|
| `GROKBOT_GROK_API_KEY` | xAI key |
| `GROKBOT_DATA_API_KEY` | data provider key |
| `GROKBOT_WALLET_PRIVATE_KEY` | wallet private key (needed only in live) |
| `GROKBOT_MODE` | `dry-run` or `live` |
| `GROKBOT_RPC_URL` | Solana RPC |
| `GROKBOT_LOG_PATH`, `GROKBOT_LOG_LEVEL` | log path and level |
| `GROKBOT_STATE_PATH` | state file |
| `GROKBOT_HEALTH_PORT` | health endpoint port, 0 — disable |
| `GROKBOT_ALERT_WEBHOOK` | webhook for notifications (usually contains a token) |

An empty variable value does not overwrite what is in the file: in compose this
is a common mistake.

## Executor

`src/executor.py` is the only place left unfinished on purpose.
`DryRunExecutor` works fully; `LiveExecutor.buy` and `.sell` raise
`NotImplementedError`, and next to them sits a step-by-step list of what needs
to be written: loading the Keypair, bonding curve accounts, buyer ATA,
computing `max_sol_cost` with slippage, the pump.fun program instruction,
ComputeBudget, sending a bundle to Jito with a tip, waiting for confirmation.

Price reads from the bonding curve are shared by both modes and work.

## Logging

JSONL, one record per line, three types:

- `buy` — full decision context: decomposed scoring, answers from all four
  agents, metrics, entry price, size, `tx_hash`;
- `skip` — stage, reason, and detail (for example, which scoring component
  was the weakest);
- `close` — exit price, PnL in SOL and percent, hold time, reason.

`scripts/replay.py` builds a summary from the log: how many considered and bought,
skip-reason breakdown by stage, scoring histogram and component averages, PnL, share of
profitable trades, average hold time.
`scripts/dashboard.py` shows current state: open positions,
today's skips, latest events; with `--watch N` it refreshes itself.

`scripts/tune.py` recomputes scoring from the log with different weights and a threshold
— the components are stored in every record, so the agents do not need to be called again.
It shows how the threshold changes the number of candidates, and which weight sets
would have kept more profit on trades that already happened.

The limitation the script prints itself: what would have happened to a token filtered
by the threshold, the log does not know and cannot know. The table describes what already happened,
not future income, and on a couple dozen trades this is fitting noise, not
tuning. Hence the warning when the sample is under 30 closed trades.

## Tests and checks

```bash
make check      # ruff + mypy + pytest, CI runs the same thing
make test
make cov
```

Covered: the base filter and monitor buffer, the scoring matrix at boundary
values and weight normalization, all five risk limits, position shrink at
the daily budget edge, day rollover, stop-loss, agents on mocked
HTTP — including malformed JSON, timeout, 500, and a response that does not match the schema — config
validation and secret masking, state after restart, Grok spend
limiters, the health endpoint, log rotation, and an end-to-end dry-run pipeline
pass. No test hits the network, except the health endpoint on 127.0.0.1.

CI runs the linter, types, and tests on Python 3.11, 3.12, and 3.13, builds the image,
and separately checks that `config.example.yaml` with placeholders does not
start.

## Disclaimer

This is research code, not a trading product and not financial advice.

Memecoins on a bonding curve go to zero completely, and that is a common
outcome, not a rare one. A substantial share of pump.fun launches are organized
rugs; some of the rest become them. Neither a wallet audit nor an
adversarial check reliably distinguishes a prepared rug from
organic growth — they only lower the share of obviously bad entries.

The five limits in the config cap the rate of losing money, not the probability
of losing it. Automated trading on your own key means that a bug in
the code, a data-provider outage, or a bad prompt costs exactly as much
as sits in the wallet.

Work in `dry-run` until you have read every stage yourself. The live part is
unfinished on purpose: by finishing it, you accept responsibility for what
it does with your funds.
