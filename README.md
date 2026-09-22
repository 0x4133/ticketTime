# tt — SOC ticket queue timer

Tracks how long you've known about every ticket, time-to-acknowledge, time-to-resolve,
and MTTA/MTTR — with points and a leaderboard so clearing the queue is a game.
One Rust binary: full CLI plus an embedded web GUI.

Data lives in `~/.local/share/tt/tickets.db` (SQLite). Override with `TT_DB=/path/to.db`.
`tt backup --keep 14` snapshots it safely (cron-able); `tt verify` audits integrity,
event order, and recomputes every analyst's points from scratch.

## GUI

```
tt serve              # http://127.0.0.1:7777  (or --port N)
./start.sh            # same thing daemonized: tt.pid + tt.log in the repo dir
./stop.sh
./restart.sh          # stop + start — picks up a freshly built binary
```

Queue / Board / Stats / Templates / Store tabs (+ Admin for admins). Live-ticking ages,
⚠ SLA breach flags, one-click ack, modals for close/note/assign, points toasts (🏆 on a
new category record), and "pts on the table" so you can see what the open queue is worth.
The queue has a search box (`/` to focus, Esc clears — matches title, tags, IOC values,
assignee), severity facet chips, sortable columns, a ▦ **lanes view** (drag a card
open → acked to ack it, drag to closed to pop the close modal), and a 📌 **focus mode**
(pin one ticket to a strip under the header — visible from every tab, `f` on the hotkey
cursor). Daily-driver polish: **Enter submits any modal** (Ctrl+Enter from a note box),
filters/ranges survive reload, table headers stick, ages carry absolute-time tooltips,
`#id` cross-references are clickable everywhere (links, radar, records, feed), IOC
values click-to-copy (📋 copies `#id title`), ⚡ IOC recall stays pinned in the row,
assignee fields autocomplete from known analysts, bulk mode gets a select-all box, and
tickets that appear in the background (hook scripts, another shell) toast 🆕.
The Stats tab has a range picker, a closes-per-day chart with MTTA/MTTR sparklines
(inline SVG, no libraries), the queue-aging histogram, and the 🖨 shift-handoff report.
The Board tab has the leaderboard (with 🔥 streaks and ⭐ prestige), season champions,
⚔ duels, and a cross-ticket activity feed.

The frontend is a single HTML file embedded in the binary at compile time
(`src/ui/index.html`) — no node, no build step, nothing to deploy but the binary.
JSON API under `/api/*` (`tickets`, `stats`, `board`, `me`, `store`, `suggest`, `next`,
`bulk/*`, `templates/*`, `tasks/*`, `ai/*`, `tags`, `iocs`, `links`, `use`, `duel`,
`handoff`, `activity`, `hook`, plus `tickets/:id/sev|wait|resume` and
`admin/bounty`) if you want to script against it — or use `--json` on
`tt ls / show / stats / board / me / next` and skip the HTTP entirely.

## Lifecycle

```
tt                                    # bare tt = the queue (same as tt ls)
tt add "Phishing email reported by finance" -s high -c phishing --source email -a aaron
tt add "Mimikatz on WKSTN-042" -s crit -c malware --at 50m   # backdate: known for 50 min
tt add "IR call in progress" -s h --ack   # work already underway — starts acked, TTA 0
tt ack 2 -n "isolating host"          # stops the time-to-ack clock (alias: tt a)
tt a 3 4 5                            # multi-id ack — sweep the overnight batch
tt note 2 host isolated, pulling memory dump    # no quotes needed (alias: tt n)
tt assign 2                           # no name = assign to yourself
tt sev 2 critical -n "lateral movement confirmed"   # escalate — a timeline event
tt wait 2 -n "waiting on user reply"  # ⏸ display-only: clocks and SLA keep running
tt resume 2
tt close 2 -r tp -n "reimaged, creds rotated"       # alias: tt c · -r tp/fp/dup
tt c 8 9 10 -r fp                     # multi-id noise close
tt close 7 --dup-of 2                 # infers -r duplicate + links to the original
tt reopen 2 -n "it came back"         # clock resumes from original detection
tt undo                               # ↩ reverse the last ack/close/note (audited)
```

- **Severities**: `critical high medium low info` (shorthand `crit h m l i`, any case)
- **Categories**: `phishing malware intrusion exfil vuln policy insider ddos access recon other` (`phish` works)
- **Resolutions**: `true-positive false-positive benign duplicate` (shorthand `tp fp dup`)
- `--at` accepts `"2026-08-19 09:30"`, `"2026-08-19"`, bare `"09:30"` (today — rolls to
  yesterday if that's in the future), `"yesterday"`, `"yesterday 22:15"`, or relative-ago
  `30m` / `2h` / `3d` / `1w` on `add`, `ack`, and `close` — log work after the fact
  without wrecking the metrics.

**Severity escalations are scored honestly**: `tt sev` records old → new with a required
note, and points + SLA windows always use the **highest severity the ticket ever held** —
escalating late can't shrink a blown SLA window, and downgrading before close can't dodge
one (or farm a bump-at-close).

**⏸ Waiting** is honest too: MTTA/MTTR stay wall-clock truth, points and SLA keep
decaying — stats just additionally report **active TTR** (waits excluded), so "3 days
because the user was on PTO" is visible without falsifying the headline number.

**↩ Undo** reverses the most recent ack, close, or note (`tt undo [id]`). History is
corrected in the open: every undo appends an `undo` event naming what was reversed, a
close-undo returns any consumed ☕/🧊 boost to stock, an ack-undo returns a 💊 focus shot.

## Tags, IOCs & links

```
tt tag 9 apt-x ransomware             # freeform labels (searchable, point-neutral)
tt tag 9 apt-x --rm
tt ioc add 9 ip 203.0.113.7           # kinds: ip domain hash email url host
tt ioc find 203.0.113.7               # every ticket carrying it
tt link 14 --dup-of 9                 # or --related / --parent (child of an incident)
tt unlink 14 9
```

Adding an IOC that already exists on past tickets prints **⚡ seen on #4 (closed
true-positive, 3w ago)** — the "haven't I seen this IP before?" recall a one-person SOC
otherwise loses to memory. All of it is point-neutral metadata recorded as timeline
events; the GUI shows tag chips on queue rows and full editors in the expanded row.

## Search & filters

```
tt ls --grep mimikatz --tag apt-x --source edr --assignee aaron --sev high --cat malware --since 7d
tt stale                              # 🧟 open tickets past 2× their SLA target
```

`--grep` matches title, notes, and IOC values; everything composes with `--json`.
The GUI equivalent is the search box + severity chips + sortable headers.

## Templates, sets & subtasks

Templates are reusable ticket blueprints; templates sharing a `--set` form a **set**
you can launch as one batch (a morning routine, an incident-type playbook). Each
template carries a **subtask checklist** onto every ticket it spawns.

```
tt tpl seed                       # install the starter sets (see below)
tt tpl ls                         # templates grouped by set
tt tpl launch morning -a aaron    # one ticket per template in the set
tt tpl use pb-malware --title "Malware on WKSTN-042" -a aaron
tt tpl add pb-ransomware --title "Ransomware containment" -s critical -c malware \
    --set malware --task "Isolate host" --task "Capture memory" --task "Engage IR retainer"
tt tpl edit pb-ransomware --title "Ransomware response" -s crit --set ir \
    --task "Isolate host" --task "Capture memory"    # any flag replaces that field
tt tpl edit pb-ransomware --rename pb-ransom         # rename the key
tt tpl rename-set morning am-routine                 # move a whole set (--clear ungroups)
tt tpl rm pb-ransomware           # template only — existing tickets keep their checklists
```

In the GUI, every template row has a ✎ edit button (name, set, severity, category,
source, title, and the whole checklist in one modal) and every set header has ✎ to
rename or ungroup the set.

Starter sets from `tt tpl seed`: **morning** (overnight alert review, phishing inbox,
threat intel brief, sensor & backup health) plus **phishing** / **malware** / **access**
playbooks with full response checklists.

Subtasks work on *any* ticket, template-born or not. In the GUI: a ☑ 2/8 chip on the
queue row, and the checklist lives in the expanded row — check items off, add more
inline, ✕ to remove. The close modal warns if items are still unchecked (closing is
still allowed). Subtasks are **point-neutral** — points come from the timing events
only, so a checklist can't be farmed.

```
tt task add 5 "Notify team lead"
tt task done 5 2                  # numbers as shown in `tt show 5`
tt task undo 5 2
tt task rm 5 9
tt show 5                         # checklist + timeline + tags/IOCs/links + SLA runway
```

### AI assist (opt-in)

AI assist is a store unlock: buy the **🤖 AI Copilot** (1200c) — or engage 🚀 turbo mode
to test it. With the unlock *and* an Anthropic API key set, tt can **✨ generate a subtask
checklist** for any ticket (like the seeded playbooks, but tailored to the ticket's
title/severity/category) and **🧭 recommend next steps** from the ticket's current
checklist progress, notes, and SLA position. GUI: both buttons live in the expanded queue
row next to "add subtask"; generated tasks are appended to the checklist (with a timeline
note), recommendations are shown inline and not stored.

The copilot also does **✨ intake**: paste any block of text — a checklist, an overnight
handover, an alert dump, an email — and it structures it for you. Tickets mode splits
unrelated incidents into separate tickets (severity/category guessed, checklist lines
attached as subtasks); template mode turns pasted playbooks into reusable templates —
and a doc with several sections becomes a whole **set** (up to 8 templates, one per
routine, ready for `tt tpl launch <set>`).
It only ever *proposes*: the CLI shows the parse and asks before creating, and the GUI's
✨ Paste button (next to "+ New ticket") opens an **editable preview** — fix titles,
severities, and tasks, untick what you don't want, then create. Created tickets carry a
"✨ AI intake" timeline note; detection clocks start at creation, so the metrics stay honest.

```
tt config set anthropic-key sk-ant-…     # or Admin tab → 🤖 AI assist (admin-gated)
tt config set ai-model claude-sonnet-5   # optional; this is the default
tt config ls                             # key shown redacted (***last4)
tt ai tasks 9                            # generate + append the checklist
tt ai advise 9                           # next-step recommendation
tt ai intake                             # paste, Ctrl-D, review, y — tickets on the queue
tt ai intake --mode template --file playbook.txt   # paste a playbook → reusable template
```

This is the one exception to "no outbound connections", and it's strictly opt-in and
on-demand: **no key → no calls**, nothing fires automatically, and each click sends only
that one ticket's title, checklist, and notes to `api.anthropic.com`. The key lives in
the settings table of the local DB and is never shown unredacted (GUI shows `***last4`;
`ANTHROPIC_API_KEY` in the server's env works as a fallback). AI settings in the GUI are
admin-gated; the ✨/🧭 buttons appear for analysts who own the 🤖 unlock once a key is set.

## Automation, import & export

```
tt hook token                     # token for POST /api/hook (--rotate to invalidate)
curl -s -X POST -H "X-TT-Token: $(tt hook token)" -H content-type:application/json \
  -d '{"title":"OpenSCAP high finding","severity":"medium","category":"vuln","source":"openscap"}' \
  http://127.0.0.1:7777/api/hook
```

The hook lets local automation (Splunk alert scripts, rtl_433 anomalies, scan wrappers)
file tickets — and only file them: no ack or close via hook, `detected_at` bounded to the
last 10 minutes, optional `dedupe_minutes` suppresses identical-title repeats from a
chatty sensor. 403 without the token. Loopback only, like everything else.

```
tt import tickets.csv --dry-run   # validate; then run without --dry-run
tt import tickets.json            # or "-" for stdin
```

Import maps rows straight onto the real event model — `detected_at` required,
`acked_at`/`closed_at`/`resolution` optional, `detected ≤ acked ≤ closed`, nothing in the
future, a missing ack means *no ack event*, never a synthesized one. Every imported
ticket gets an `imported from <file>` timeline note, and points fall out of the normal
recompute. CSV header:
`title,severity,category,source,assignee,detected_at,acked_at,closed_at,resolution,closed_by`.

```
tt export 42                      # one ticket: full record + verbatim timeline (md)
tt export 42 --format json
tt export --days 30 --format csv -o aug.csv    # range report over closed tickets
tt export --days 30 -o aug-report.md           # md: MTTA/MTTR summary + table
```

```
tt watch                          # live terminal dashboard: 2s refresh, SLA countdowns,
                                  # ⚠ BREACH flags, points on the table; Ctrl-C to exit
tt handoff                        # shift handoff since the last marker, then mark
tt handoff --preview --since 8h   # print only, custom window
tt completions fish > ~/.config/fish/completions/tt.fish
```

`tt handoff` renders new tickets, closes (+points), notable events (escalations, waits,
bounties, reopens), and the open queue — computed from the same event log as the metrics,
so it can never disagree with them. The GUI version (Stats → 🖨 Handoff report) is
print-ready via `@media print`.

## Queue & metrics

```
tt ls            # open queue: severity order, age (KNOWN), time in current state, point value
tt ls --all      # include closed
tt ls --closed
tt show 2        # one ticket: full timeline with gaps between events
tt stats         # MTTA / MTTR (mean + median), active TTR, per-severity, FP rate,
                 # queue aging histogram, oldest open
tt stats --days 7
tt ls --json | jq '.[] | select(.severity=="critical")'   # pipeline citizen
```

MTTA = ack − detection. MTTR = close − detection (from when you *knew*, not when you acked).

## Game

Closing tickets earns points; `tt board` is the leaderboard (`--season` for the
calendar-month race, `--days 7` for a weekly one).

| severity | base | SLA target |
|----------|-----:|-----------:|
| critical | 100  | 4h  |
| high     | 60   | 8h  |
| medium   | 30   | 24h |
| low      | 15   | 72h |
| info     | 5    | 7d  |

- Close **under the SLA target**: +50% base. Under **half** the target: +100% (double points).
- Ack within **15 minutes** of detection: +10 (💊 focus shot doubles it to +20).
- `false-positive` / `duplicate` closes: flat 10 (triage credit, no farming).
- Scoring severity = the **highest the ticket ever held** (see escalations above).
- 💰 **bounties** (admin-placed, reason required) pay flat on a true-positive close only.
- Fastest true-positive close per category is a **record** — beat it and the close announces 🏆.
- The `WORTH` column in `tt ls` is what a ticket pays if you close it right now; it decays
  as the SLA slips, so the hot tickets are literally worth more.
- Points go to `--by`, else the assignee, else `$USER`. Recomputed from timing data, so
  they're consistent and retroactive.

### 📜 Contracts, 🔥 streaks & seasons

- **Daily contracts** (`tt contracts`, or the 📜 panel on the queue): two challenges a
  day, picked deterministically from the date — "close 2 under half-SLA", "close a
  critical/high TP", … Completing one pays a fixed bonus (+35…+75). Only true-positive
  closes count, and completion is recomputed from the events like everything else.
- **Close streaks**: consecutive days with ≥1 true-positive close show as 🔥 N on the
  board; milestone days pay bonuses (7d +50, 30d +250, 100d +1000), once per run. FP
  closes don't keep a streak alive, and a missed day honestly resets it.
- **Seasons**: every calendar month is a race — `tt board --season` (GUI: "this season").
  Past months' champions are computed on the fly and shown with 🏆; nothing is persisted,
  so retro-edits honestly re-crown.
- **⚔ Duels**: `tt duel start bob 100 --hours 24` — most true-positive close points
  inside the window wins the stake (≤500c, one active duel per pair). Stakes are
  **escrowed**: both sides need the balance up front and staked credits can't be spent
  while the duel runs. Settlement is lazy, atomic, and audited: two ledger adjustments
  by the `duel` actor. Ties return the stakes. `tt duel ls` or the Board tab for the
  live score. (Identity is self-asserted on this loopback tool — like `--as`
  everywhere else, duels are gentleman's bets, not a security boundary.)

### RPG: characters, credits, and the store

You're playing a SOC analyst. Every point you earn is **XP** (lifetime, sets your level
and rank) *and* a **credit** you can spend in the store. Ranks: Intern → Triage Rookie
(100) → Alert Wrangler (300) → Shift Lead (700) → Threat Hunter (1500) → IR Commander
(3000) → SOC Legend (6000) → Threat Oracle (10000) → SOC Immortal (16000). The
leaderboard shows everyone's level, rank, badges, 🔥 streak, and ⭐ prestige.

**Rank perks**: Shift Lead+ gets 5% off in the store, IR Commander+ 10%, SOC Legend+
15% — applied server-side, computed from the same XP as the leaderboard.

```
tt me                  # character sheet: level, rank, XP, credits, streak, contracts,
                       # inventory, trophies
tt store               # browse the catalog (your discounted price shown)
tt buy espresso        # spend credits (--as <who> to act as someone else; default $USER)
tt use grace 9         # apply a 🧾 SLA Grace Voucher to ticket 9
tt next                # queue radar (needs the 🎯 unlock): what to work first
```

| item | cost | kind | what it does |
|------|-----:|------|--------------|
| ⚙ Autofill Module | 250c | feature | Fuzzy search across every field of past tickets. GUI: type 3+ chars in a new ticket's title, click a match to fill the form. CLI: `tt add "mimikatz again" --auto`. |
| ⌨ Hotkey Deck | 350c | feature | Keyboard-drive the queue: `j`/`k` move, `enter` expand, `a` ack, `c` close, `n` note, `x` select, `f` 📌 focus, `?` for the map. |
| 🛰 SLA Overwatch | 500c | feature | Early warning at **80% of SLA**: the age cell pulses with a 🛰 flag, the tab title counts down to the next breach, and (opt-in 🔔) browser notifications fire at 80% and at breach. |
| 🎯 Queue Radar | 600c | feature | "Next up" panel + `tt next` + `/api/next`: what to work first, what each ticket drops to and when, ack bonuses about to expire, points decaying in the next hour. |
| ⚡ Quick-close Macro | 750c | feature | `✓ TP` button on each queue row — one-click true-positive close. |
| 🧹 Bulk Triage | 900c | feature | Checkboxes on queue rows — ack a sweep at once, or noise-close as FP/duplicate (bulk close is **noise only**). Server-gated. |
| 🤖 AI Copilot | 1200c | feature | ✨ AI-generated subtask checklists + 🧭 next-step recommendations (see **AI assist**). Server-gated; also needs the API key. |
| 🎨 Ops Themes | 400c | feature | Accent color themes, picked on the Store tab. |
| 🎉 Party Cannon | 300c | feature | Confetti + a synth fanfare on every close; a new category record gets the full barrage. |
| ☕ Double-shot Espresso | 150c | consumable | Your **next close earns 2× points**. Stackable; consumed automatically at close. |
| 🧊 Nitro Cold Brew | 400c | consumable | Your **next close earns 3× points**. Drunk before any espresso in stock. |
| 🧾 SLA Grace Voucher | 300c | consumable | `tt use grace <id>`: that ticket's speed bonus is scored as if the SLA clock started 25% later. One per ticket, TP closes only. |
| 💊 Focus Shot | 200c | consumable | Auto-consumed when you ack inside the 15-minute window: that ticket's fast-ack bonus doubles (+20). Never wasted on a slow ack. |
| 🛡 ⚔ 🧙 👑 Badges | 200–1000c | badge | Flair next to your name on the leaderboard. |
| ⭐ Prestige Star | 5000c | — | Pure endgame flex: permanent stars next to your rank, buy as many as you can afford. |

Purchases are **per analyst** — the GUI's 👤 chip sets who's on shift (click it), the CLI
uses `$USER` or `--as`. Feature gating is enforced server-side. Boost multipliers,
grace/focus flags, and bounties are stored on the ticket, so recomputed totals stay honest.

### Admin

An **Admin** tab appears in the GUI for admins (and matching CLI commands exist). Until the
first admin is created, *everyone* has admin powers — adding the first admin closes the
gate (it's a loopback-only tool, so this is the bootstrap, not a security boundary).
Admin actions are enforced server-side (`/api/admin*` 403 for non-admins).

```
tt user add aaron --admin      # add to the roster / grant admin (first one closes bootstrap)
tt user ls                     # roster with XP + credits
tt adjust bob 100 --reason "covered the weekend shift"   # negative to dock
tt bounty 17 50 --reason "week-old exfil lead"           # 💰 paid on a TP close only; 0 clears
tt rm 17                       # permanently delete a ticket + timeline (asks; --force skips)
```

- **Roster** — add analysts, grant/revoke admin (the last admin can't demote themselves).
- **Points ledger** — points can't be *edited*; admins grant/dock via an **audited
  adjustment ledger** (delta, required reason, who, when). Duel settlements land there too.
- **💰 Bounties** — turn the shameful bottom of the queue into treasure: admin-placed
  with a required reason (a timeline event is the audit trail), paid flat on a
  true-positive close, forfeited on FP/duplicate/benign.
- **Delete tickets** — 🗑 on any queue row (confirm modal), or `tt rm`. Points recompute
  without it. For test junk — real incidents should be closed, not deleted.
- **🚀 Turbo mode** — a per-analyst test switch (Admin tab, or `tt turbo on|off [--as X]`):
  unlocks **every store feature** so you can try things without buying them. It grants
  **no** points, credits, badges, or consumables; real purchases survive turning it off.

### Trophy case

Achievements are **earned, not bought** — recomputed from the timing events like points,
so they can't drift. Shown on the Store tab and in `tt me`:

| | trophy | how |
|--|--------|-----|
| 🩸 | First Blood | close your first true-positive |
| ⚡ | Speed Demon | true-positive close within 10 minutes of detection |
| 🌙 | Night Watch | close a ticket between midnight and 6am |
| 🎩 | Hat Trick | three true-positive closes inside one hour |
| 🧹 | Clean Sweep | one of your closes left the queue at zero |
| ☢ | Critical Mass | close 10 critical true-positives |
| 🛡 | SLA Guardian | 25 true-positive closes inside the SLA target |
| 💯 | Century | close 100 tickets |
| 🃏 | Full House | a true-positive close in every category |
| 🔁 | Comeback Kid | close a reopened ticket as true-positive |
| 🐢 | Dragon Slayer | true-positive close of a ticket open 7+ days |
| 📅 | Perfect Week | 7 consecutive days with a close, every one inside its SLA |
| 🌅 | Early Bird | close a ticket between 5 and 9 am |

## Build

```
cargo build --release   # binary at target/release/tt
```
