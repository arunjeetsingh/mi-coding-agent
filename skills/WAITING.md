# Waiting for a PR without burning tokens

The canonical wait contract for `open-pr`, `address-review-comments` and `review-pr`.
Those skills reference this file; none of them restates it.

## The rule

**Whatever waits must be a process, not a prompt.**

- A scheduler that enqueues a **prompt** — a cron job, a scheduled task, an
  `automation_update` heartbeat (Codex) — starts a fresh model session on every tick. It
  loads the skill, calls the API, concludes nothing happened, and exits. Cost is
  proportional to **ticks**.
- A background **watch** runs a **command**. The shell/process does the polling; only a
  match becomes a notification. Cost is proportional to **events**.

So put the predicate in a background process and let the model run only when the
predicate is true.

The difference is not marginal. A 10-minute heartbeat is 144 model runs a day per PR. Each
one loads this skill (~5.5k tokens) before it can even decide, so a floor of ~8k tokens a
tick is roughly **1.2M tokens a day per monitored PR**, essentially all no-ops, whether the
PR is active or asleep. The same watch as a background process costs nothing until
something actually happens.

## What it costs GitHub

A response can arrive on three streams — issue comments, reviews, and review replies — and
each can paginate, so the loop reads all three every tick. Conditional `If-None-Match`
requests would be free (a 304 does not consume rate limit), but an ETag is per endpoint per
page and cannot cover a paginated multi-stream read, so that optimisation is gone. At one
tick every 30 minutes over three single-page streams the loop spends **6 REST requests an
hour** against a 5000/hour limit — around a tenth of one percent. Even a long PR with
several pages per stream stays far inside it. Do not go back to one unpaginated stream to
save more, which is what makes a watch miss responses in the first place.

**Why 30 minutes, and it is not about the API budget.** The interval is set for an
UNATTENDED loop: arm this before stepping away and don't expect the PR to move quickly
while you're gone. A one-minute tick buys nothing against that — a review loop is not a
build watch, nobody is sitting on the result, and a batch that lands is equally actionable
half an hour later. If the goal is a hard ceiling on unattended work rather than a slower
tick, stop the watch instead and re-arm later.

**What the interval does NOT do.** It caps LATENCY, not activity. It does not limit how
many review rounds happen while you're away — that's set by how fast the reviewer posts
batches, and each batch still wakes a full round.

Also read the PR state each tick (`gh pr view`) — that's GraphQL, metered on a SEPARATE
5000/hour bucket from REST. So the real cost is ~6 REST requests and ~2 GraphQL queries an
hour, each against its own limit. If you ever measure rate limit, read the response header
(`X-RateLimit-Used`), never `gh api rate_limit` — that endpoint can read stale on some
tokens.

## The tradeoff, stated plainly

**A background watch tied to a session dies when the session ends.** There is no
zero-token durable option on every host: once the session/task ends, something has to
invoke a model to re-arm the watch on hosts without a durable background-process
mechanism. Do not pretend otherwise, and do not paper over it with a raw cron workaround
that re-invokes the whole skill on a fixed timer without a predicate.

What follows from that:

- Re-arm on the next invocation of the skill rather than trying to survive the gap, on
  hosts where the watch does not durably survive.
- **Never report a watch as armed when it is not.** Say which PR is watched, and say
  plainly whether it survives the session ending.
- If the watch genuinely must survive the session on a host without a durable
  process-watch primitive, that is a deliberate trade — accept the periodic-heartbeat cost
  (see the host mapping below) rather than faking a free option.

## Host mapping

**The zero-token watch needs a host that can run a background command, and not every host
can.** These skills run on OpenClaw, Claude Code, and Codex, so this file says which
mechanism applies where instead of naming one host's tool and leaving the others silently
unarmed.

| | OpenClaw | Claude Code | Codex |
|---|---|---|---|
| mechanism | `cron` tool, `isolated` agentTurn job with a `trigger.script` gate that polls the predicate, only firing the payload when it returns `{fire:true}` | `Monitor`, `persistent: true`, running a shell poll | `automation_update` heartbeat re-invoking the skill on a timer |
| create | `cron` `action:"add"` with `schedule.kind:"every"` (e.g. 30 min) + `trigger.script` running the predicate below | arm the loop with `PR`/`REPO`/`MARKER`/`EXCLUDE` bound into the command | `mode=create`, `kind=heartbeat`, `destination=thread`, `rrule=FREQ=MINUTELY;INTERVAL=30`, `status=ACTIVE`, self-contained prompt |
| find again | `cron` `action:"list"` / `action:"get"` by job name | the task id the arming call returned | enumerate active/paused automations, match the immutable ownership marker |
| update | `cron` `action:"update"` | re-arm (a Monitor is not editable in place) | `mode=update` with the ID, preserving every existing field |
| pause / resume | `cron` `action:"update"` with `enabled:false/true` | `TaskStop`, then arm again | pause / reactivate that ID |
| delete | `cron` `action:"remove"` | `TaskStop` with the task id | `mode=delete` with the ID |
| survives the session | **yes** — cron jobs are durable gateway state | **no** | yes |
| cost while quiet | near-zero — the `trigger.script` gate runs headlessly and only invokes the model payload when it returns `fire:true` | **nothing** — only a matching line wakes the model | a fresh model session every tick, ~48/day/PR at 30 min, nearly all no-ops |

**Under Codex the cost is inherent, not a bad choice.** With no way to run a background
predicate there, the heartbeat is the only durable mechanism and its cost is the price of
durability. Under Claude Code the shell loop does the same job for nothing, so use it and
do not add a scheduled task on top. **Under OpenClaw, prefer the `cron` `trigger.script`
gate** — it gets the durability of a scheduled job with (close to) the cost profile of a
shell-side predicate, because the script poll runs headlessly and only invokes the agent
payload when it actually matches.

On the Codex/heartbeat path, put this stable ownership marker at the start of the
heartbeat prompt so a later run can find the job again, and keep the prompt
self-contained — each run starts with no memory of the conversation that created it:

`[<skill-name>-heartbeat:v1 workflow_id=WORKFLOW_ID repo=OWNER/REPO pr=NUMBER reviewer=LOGIN]`

### Heartbeat / durable-job lifecycle (Codex and OpenClaw cron)

**Never use "not armed in this session" as evidence that no durable watch exists.** A
heartbeat or cron job survives the task/session that created it, while each later
invocation is a fresh run. Before any GitHub write that would require monitoring, and
again before re-arming or stopping, inspect the durable job inventory (Codex:
`$CODEX_HOME/automations/*/automation.toml`; OpenClaw: `cron` `action:"list"`). Include
both active and paused/disabled jobs. Match the exact immutable marker above at the start
of the prompt, not a display name or a mutable failure counter.

1. If exactly one match belongs to the current PR/workflow, keep its ID. Leave an active
   job alone; reactivate a paused/disabled one with the same ID and its full existing
   fields.
2. If more than one matching record exists and their marker, repository, PR, and reviewer
   all agree, retain one and delete only the exact owned duplicates. Treat any other
   multi-record case as ambiguous.
3. If a match belongs to another workflow, or ownership is ambiguous, create nothing and
   make no new GitHub write. Report the existing watch/conflict instead of racing it.
4. Create a job only when the lookup found no owner.
5. On a successful snapshot, reset any consecutive-failure counter on that same job. On a
   failed snapshot, increment it; after three consecutive failures pause/disable it and
   report the blocked watch. Never create a replacement merely because a read failed.

## The rules, on every host

1. Resolve the host's mechanism before the first GitHub write that would require
   watching. If none is available, report the blocker instead of pretending the loop is
   active.
2. One watch per PR. Keep whatever identifies it (task id, automation ID, or cron job id)
   — that's what stopping needs, and how you know a watch for this PR is already armed
   instead of arming a second on top of it.
3. **The predicate is a CANDIDATE filter, not the trigger rule.** `MARKER` is a regex, and
   it must cover every response shape the waiting skill promises to accept — which for
   `review-pr` is the token *or* an unambiguous natural-language acknowledgement. A
   predicate matching only the token sleeps through a reply that says "the complete
   latest feedback is fully addressed":

   ```
   MARKER='codex-addressed|addressed|fixed|handled'
   EXCLUDE='<!-- review-pr-state'
   ```

   `EXCLUDE` drops your own batches, which otherwise wake you on your own writing — the
   reviewer and the author can post under the same GitHub login, so content is the only
   thing that separates them.

   **It must be the HTML comment, never the bare string.** `<!-- review-pr-state ... -->`
   is something only this workflow emits; the bare word appears in any reply that
   discusses the watch, including your own.

   | skill | `MARKER` | `EXCLUDE` |
   |---|---|---|
   | `open-pr`, `address-review-comments` | `<!-- review-pr-state` | — |
   | `review-pr` | `codex-addressed\|addressed\|fixed\|handled` | `<!-- review-pr-state` |

   This filter is deliberately looser than the trigger rule, and it wakes on `not
   addressed` too. That is the design and not a defect: the shell/predicate cannot judge
   negation, quoting, or scope — the skill's own trigger rule can, and a wasted wake costs
   one model run while a missed one costs the whole loop.

## Reference shell implementation (Claude Code `Monitor`, or any host that can run a background process)

Seed a baseline, emit only on a real change, and exit on terminal states. It takes `PR`,
`REPO` and `MARKER` — the marker is the only thing that differs between the two
directions.

```bash
set -euo pipefail
PR="${PR:?PR number required}"
REPO="${REPO:?owner/repo required}"
MARKER="${MARKER:?a regex a CANDIDATE item matches}"
EXCLUDE="${EXCLUDE:-}"                 # optional: a regex that disqualifies an item
case "$PR" in ''|*[!0-9]*) echo "PR must be numeric, got '$PR'" >&2; exit 1 ;; esac

# Count CANDIDATES across ALL THREE streams a response can arrive on — issue comments,
# reviews, and review replies — every page of each.
count() {
  local n=0 ep out c
  for ep in "issues/$PR/comments" "pulls/$PR/reviews" "pulls/$PR/comments"; do
    out=$(gh api --paginate --slurp "repos/$REPO/$ep" 2>/dev/null) || return 1
    c=$(printf '%s' "$out" | jq --arg m "$MARKER" --arg x "$EXCLUDE" '
          [ .[][]
            | (.body // "") as $b
            | select($b | test($m; "i"))
            | select($x == "" or (($b | test($x; "i")) | not)) ] | length') || return 1
    n=$((n + c))
  done
  printf '%s\n' "$n"
}

# `--repo` is not optional: a background runner may inherit no working directory from the
# session that armed it, so `gh pr view` without `--repo` resolves from wherever it starts.
state() { gh pr view "$PR" --repo "$REPO" --json state --jq .state 2>/dev/null; }

seen=$(count) || { echo "PR #$PR: cannot read $REPO — watch not armed"; exit 1; }
st=$(state)   || { echo "PR #$PR: cannot read $REPO's PR state — watch not armed"; exit 1; }
[ "$st" = "OPEN" ] || { echo "PR #$PR is $st, not OPEN — watch not armed"; exit 1; }
echo "PR #$PR: watching from $seen existing candidate(s) for /$MARKER/" >&2
fails=0
while true; do
  sleep 1800         # 30 minutes — see the pacing note above
  if ! n=$(count) || ! st=$(state); then
    fails=$((fails + 1))
    [ "$fails" -ge 3 ] && { echo "PR #$PR: gh failed 3 polls running (~90 min) — this watch is blind"; break; }
    continue
  fi
  fails=0
  if [ "$n" -gt "$seen" ]; then
    echo "PR #$PR: $((n - seen)) new candidate(s) for /$MARKER/ — now $n"
    seen=$n
  fi
  [ "$st" = "OPEN" ] || { echo "PR #$PR is now $st — watch stopping"; break; }
done
```

Arm it with `Monitor`, `persistent: true`, binding `PR`, `REPO`, `MARKER` and (where it
applies) `EXCLUDE` **in the command itself** — a Monitor inherits nothing from the session
that armed it. **Keep the task id the arming call returns**: it's what stops the watch
later, and how you know this PR is already watched instead of arming a second on top.

## OpenClaw: cron `trigger.script` gate

OpenClaw's `cron` tool supports an `every`/`cron` schedule with a `trigger.script` gate: the
script runs headlessly on every tick, and the agent payload only fires when the script
returns `{"fire": true}`. That gives OpenClaw the same "process does the polling, model
only runs on events" property as a Claude Code `Monitor`, but as durable gateway state
that survives the session:

```json5
{
  action: "add",
  job: {
    name: "review-pr watch: OWNER/REPO#NUMBER",
    schedule: { kind: "every", everyMs: 1800000 },
    trigger: { script: "<the count()/state() predicate above, printing {\"fire\":true/false} as JSON>" },
    payload: { kind: "agentTurn", message: "Use review-pr in monitor mode for OWNER/REPO#NUMBER." },
    sessionTarget: "isolated",
  },
}
```

Remove the job (`action:"remove"`) on the same terminal conditions as any other host:
clean review, PR closed/merged, or explicit user stop.

## If no background-watch mechanism is available

Say so and leave watching manual. Do not substitute a raw `sleep` loop outside a
supervised process, a bare cron workaround with no predicate, or a background process you
can't account for later, and do not report the loop as active when it is not. An unarmed
watch that is believed to be armed is worse than none, because nobody goes looking.
