# Runbook

What to do with a running pipeline: how to start it, what to watch, what
each complaint means and how to fix it. Architecture and the meaning of
the stages are in
[README](README.md); this document is operations only.

## Startup

### Bare machine

```bash
make dev                      # venv + dependencies + linter and tests
cp config.example.yaml config.yaml
$EDITOR config.yaml           # or keys via GROKBOT_* in the environment
make check-config             # check without starting; secrets in the output are masked
make run
```

### Docker

```bash
cp .env.example .env && $EDITOR .env
mkdir -p config logs state && cp config.example.yaml config/config.yaml
docker compose up -d
docker compose logs -f
```

The `logs/` and `state/` volumes MUST live outside the container: `state/`
holds open positions and daily limits, and losing that file means that after
a restart the pipeline will forget both.

### systemd

```ini
[Unit]
Description=grokbot-pumpfun
After=network-online.target

[Service]
User=grokbot
WorkingDirectory=/opt/grokbot-pumpfun
Environment=GROKBOT_GROK_API_KEY=xai-...
Environment=GROKBOT_HEALTH_PORT=8080
ExecStart=/opt/grokbot-pumpfun/.venv/bin/python -m src.pipeline --config config.yaml
Restart=always
RestartSec=10
KillSignal=SIGTERM
TimeoutStopSec=45            # greater than ops.shutdown_grace_seconds
[Install]
WantedBy=multi-user.target
```

### launchd (macOS)

`~/Library/LaunchAgents/com.grokbot.pumpfun.plist`, the key bits:
`ProgramArguments` — the same invocation, `KeepAlive` — true, `EnvironmentVariables`
— `GROKBOT_GROK_API_KEY`. Stopping via `launchctl unload` sends SIGTERM,
i.e. a clean shutdown that persists state.

## First 24 hours

The order that saves money:

1. `mode: dry-run`, a day of operation, then `make replay`.
2. Watch conversion: if 0 bought out of thousands — the threshold is too high
   or the agents are rejecting; if more than a dozen are bought per hour —
   the threshold is too low.
3. In the reject-reason breakdown, verify that every stage is working. If all
   rejections land on one stage — the rest are not receiving data.
4. In per-component averages, look for a component that is always zero: that
   is a silent agent, not a strict agent.

Only after that does a conversation about `live` make sense.

## What to watch

```bash
curl -s localhost:8080/healthz | jq        # status
curl -s localhost:8080/metrics             # counters for Prometheus
python scripts/dashboard.py logs/trades.jsonl --watch 5
```

`/healthz` returns 200 when `status: ok` and 503 when `degraded` — you can
hang a restart policy on that. Fields:

| field | meaning | when it's bad |
|---|---|---|
| `status` | summary | `degraded` = circuit is open or the feed has stalled |
| `stalled` | no events from the socket for more than 10 minutes | `true` — the socket is dead |
| `breaker` | `closed` / `half-open` / `open` | `open` — Grok is not responding |
| `grok_budget_remaining` | remaining daily calls | 0 — until midnight UTC the agents are silent |
| `halted` | daily loss limit is spent | `true` — there will be no trading today |
| `blind_positions` | positions without quotes | more than 0 — exits on them do not work |
| `open_positions` | open positions | cannot exceed `max_open_positions` |
| `in_flight` | tokens being analyzed | stably at the ceiling — hit the Grok limit |

The `жив: ...` line in the log every `heartbeat_seconds` — the same snapshot,
but in the journal, so you can reconstruct history from it.

## Notifications

When `alerts.webhook_url` is set (preferably via `GROKBOT_ALERT_WEBHOOK` —
the URL usually contains a token) events go to an external channel: `started`,
`stopped`, `buy`, `close`, `rug`, `breaker`, `halted`, `stalled`. The set
is configured in `alerts.events`.

State is reported **on transition**: `breaker` arrives once when the
circuit opens and once when it recovers, not every minute. The stream
is rate-limited by `max_per_minute`; extras are dropped and counted in
`alerts.dropped` in `/healthz`, not queued.

Silence on the channel by itself means nothing — check liveness via
`/healthz`, not by the absence of messages. The `alerts.failed` counter in
`/healthz` is exactly how many notifications failed to send.

## Incidents

### `breaker: open`, log says `цепь Grok разомкнута` (Grok circuit is open)

That many consecutive calls failed. The pipeline stopped calling Grok for
`breaker_cooldown_seconds` and **does not buy the entire time** — every agent
returns a pessimistic result, the checker answers with a reject.

Check: the key is alive (`curl` to api.x.ai), the xAI account is not out of
money, there is no 429. The circuit will close itself after the cooldown; a
probe call will show whether it recovered.

### `grok_budget_remaining: 0`

`ops.max_grok_calls_per_day` is spent. This is protection against a burst of
launches eating the monthly budget in one evening. Until midnight UTC the
agents are not called. If this is normal load — raise the ceiling; if not —
raise `filter.min_total_score` so fewer tokens reach the agents.

### `stalled: true`

No events have arrived from the socket for more than ten minutes. The monitor
reconnects on its own with a growing backoff; if `stalled` persists —
check `data.ws_url` and the network. Open positions are still under
stop-loss watch: it goes over REST, not the socket.

### `blind_positions` greater than zero

For this many open positions, several consecutive passes have not produced a
price. That means **exit rules on them are not working right now**: neither
stop-loss, nor take-profit, nor trailing stop. Check the data provider
(`data.rest_url`), key limits, and the network. While there is no price, the
position lives on its own — this is the case where you should intervene by
hand.

### `halted: true`

The daily loss limit is spent. Nothing to fix; counters reset at
midnight UTC. Open positions continue to be managed by stop-loss.

### `состояние … не читается … отложено в .corrupt` (state cannot be read — set aside as `.corrupt`)

The state file is corrupted (usually — the disk filled up at the moment of
write). The pipeline started from a clean slate: **it does not know about
open positions**. Open `state/pipeline.json.corrupt`, pull the position list
out of it, and deal with them by hand. Until that is done, stop-loss on them
does not work.

### Positions are closing on a different rule than expected

`make replay` prints close reasons. What that means:

* almost everything in `max_hold` — the market is not moving, or
  `max_hold_seconds` is too small for the selected tokens;
* almost everything in `trailing_stop` at a small profit —
  `trailing_stop_pct` is already ordinary memecoin noise, the pullback is
  caught on the first move;
* `take_profit` never fires — the threshold is higher than the selected
  tokens actually reach; look at the `pnl_pct` distribution on closed trades.

### Positions stayed open after shutdown

On a clean shutdown this is normal and a warning is written to the log:
the pipeline does not sell everything on the way out. Stop-loss on these
positions does not work until the process is up again. So a long idle with
open positions is a risk, not a pause.

### `executor_not_implemented` in the log

`mode: live` is on, but `LiveExecutor` is a stub by design. The token passed
all nine stages and was not bought. Either finish the execution path, or
go back to `dry-run`.

### Config rejected at startup

The message lists every problem at once. This is not nitpicking: each one is
either a non-working setting (a zero limit) or an unsafe one (live without
a wallet key). `make check-config` shows the same thing without starting
trading.

## Update

```bash
git pull
make check                # linter, types, tests — before restart, not after
systemctl restart grokbot # or docker compose up -d --build
```

State survives restart: positions are restored, the day's counters and
Grok spend continue from the same place. The state format is versioned
(`version` in the file); on a version mismatch the day's counters reset, and
positions are still read.

## Backup

Back up `state/pipeline.json` (positions are money) and `logs/*.jsonl`
(without them you cannot compute results). Do not back up a config with keys
to shared storage — keys are easier to reissue.

## Checklist before going live

- [ ] a day in `dry-run` completed, `replay` reviewed
- [ ] `LiveExecutor.buy` and `.sell` implemented and tested separately
- [ ] a separate wallet, holding only the amount you can afford to lose
- [ ] `risk.*` rechecked against live numbers, not defaults
- [ ] `state/` on a disk that will survive a machine restart
- [ ] `/healthz` wired into monitoring, alert on 503 configured
- [ ] `--i-understand-the-risk` added to the unit file deliberately
