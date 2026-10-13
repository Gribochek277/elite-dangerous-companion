> Sanitized mirror of Forgejo `serhii/elite-dangerous-companion`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/elite-dangerous-companion` | Synced: 2026-10-13T04:07:21Z

---

# Elite Dangerous Companion

A project-scoped [Pi](https://github.com/earendil-works/pi) extension that turns
*Elite Dangerous* journal files into a live, remembered companion: current game
state, classified event history, analytics, a crew with an ongoing chronicle, a
grounded lore historian, and EDCoPilot chatter export.

It is a TypeScript port of
[`GWLlosa/elite-dangerous-local-ai-tie-in-mcp`](https://github.com/GWLlosa/elite-dangerous-local-ai-tie-in-mcp)
(a Python MCP server), rebuilt around what Pi adds on top of MCP: push
notifications, memory between sessions, a crew that lives outside the game, and
a narrative chronicle.

- Design document: [`docs/plan.md`](docs/plan.md)
- Coding standards: [`docs/code-standards.md`](docs/code-standards.md)
- Ticket store: [`docs/tasks/`](docs/tasks/)

---

## What it does

- **Live state** — `elite_status` reports the commander's current system,
  station, ship, fuel, cargo and credits, updating as the journal grows.
- **Classified history** — all 213 journal event types map to 17 categories with
  human-readable summaries; `elite_events` and `elite_search` query the live
  buffer, `elite_history` scans the raw archive by date range — one `range`
  phrase (including `2026-01-14 to 2026-01-17`) or independent `from`/`to`
  phrases — and reports which machines contributed.
- **Push notifications** — critical events (`Died`, `Interdicted`, low fuel,
  low hull) raise a turn; lower-priority events batch into a digest.
- **Crew and chronicle** — a seven-member crew (`Helm`, `Operations`,
  `Engineering`, `Comms`, `Science`, `Security`, `Historian`) whose dialogue
  (`elite_crew_say`) and narrative beats (`elite_chronicle`) accumulate in a
  JSONL chronicle, injected into the agent prompt as the `elite_crew` section.
- **Lore historian** — `elite_historian` answers questions grounded in a
  vendored Fandom snapshot (CC-BY-SA) with source URLs, preferring pages about
  the current system; web search is a fallback.
- **EDCoPilot export** — four custom chatter file types generated under a formal
  grammar with validation and backups (`elite_edcopilot_*`).
- **Themes** — crew themes and per-ship crew configurations (`elite_theme_*`).
- **A status widget** — a compact, live line above the editor showing the data
  source, current system and pending notification beats.

## Architecture

This repository holds the **server** side: a project-scoped Pi extension under
`.pi/extensions/elite/`. Pi loads it only when launched from this project root.

The machine that runs the game is a **client**. `client/elite-stream.mjs` is a
small Node program with **no Pi dependency**: it watches the Elite Dangerous
journal directory, tracks a byte offset per file, and streams raw journal lines
to the server over HTTP with a bearer token. It never writes anything on the
game machine.

The server owns everything: parsing, classification, the event store, the raw
journal archive, crew and chronicle state, the vendored lore snapshot,
analytics, and EDCoPilot export.

A local journal path is still supported and is used automatically when no
client is connected, so the whole system can be exercised on one machine with
fixtures.

```
.pi/extensions/elite/     the extension (server)
  core/                   pure logic: no UI, no sockets, no ctx
  net/                    the only place a socket exists
  analytics/ resources/ prompts/ notify/ edcopilot/ crew/ lore/ ui/
  test/                   node --test suites and fixtures
client/elite-stream.mjs   the game-machine streamer
scripts/                  snapshot-lore.mjs, elite-play.sh, elite-pi.sh,
                          elite-wait-for-game.sh, check-lean-launch.sh,
                          check-parity.mjs, check-game-machine-quickstart.sh
```

## Requirements

- Node.js 22+ (the extension itself has no runtime npm dependencies).
- Pi launched **from the project root** (project `.pi` configuration is read
  from the working directory only); `scripts/elite-pi.sh` starts it there for you.
- For the lore historian: the vendored snapshot under `.pi/extensions/elite/lore/`
  is committed; regenerate it with `node scripts/snapshot-lore.mjs`.
- For a remote game machine: network reach to the server's ingest port.

## Quick start

```bash
git clone <this-repo> elite_dangerouse_componion
cd elite_dangerouse_componion

cd .pi/extensions/elite && npm install --no-audit --no-fund && cd ../..

scripts/elite-pi.sh
```

Then run `/elite` to see the one-screen status summary, or ask *"where am I?"*.
On first launch Pi may ask you to trust the project. A fresh clone reaches a
working state within ten minutes: `npm install` pulls only `typescript` and
`@types/node` (dev-time), and the extension loads with no further steps.

`scripts/elite-pi.sh` is the lean session: it loads the elite extension and the
agent's own tools and leaves every global extension, package, MCP server, skill
and prompt template out of the prompt. Bare `pi` from the project root still
works and loads everything; see [Lean session](#lean-session-context-budget).

### Verify it end to end

```bash
cd .pi/extensions/elite
node --test test/*.test.mjs      # unit + integration tests
npx tsc -p tsconfig.json         # typecheck
```

## Quick start: the game machine

The machine that runs the game needs Node 22+ and the repository (or one file
from it) — no Pi and no `npm install`. It runs a single program that watches
the journal directory and streams it to the Pi session on the server. Five
steps, in order.

### 1. Collect the two values from the server

Everything below needs the server's address and the bearer token its listener
checks. On the **server**, print the token:

```bash
node ~/.pi/agent/bin/pi-config.mts get elite.server.token
```

and find the address the listener actually accepts. It defaults to
`127.0.0.1`, which only the server itself can reach, so the game machine needs
the server's LAN address (`hostname -I` on Linux) or its tailnet address:

```bash
hostname -I            # e.g. 192.168.1.20 — the address both machines share
tailscale ip -4        # when both are on your tailnet
```

Keep the two values: every step below uses them as `SERVER_URL`
(`http://<address>:7345`) and `TOKEN`.

**If this is not the first game machine, admit it now.** With an empty
`elite.server.machineIds` the server is single-tenant: it auto-registers the
*first* machine id it sees, so one game machine needs no configuration and a
second one is rejected until you list it. On the server, add every id you
stream from — a machine's id defaults to its hostname:

```bash
node ~/.pi/agent/bin/pi-config.mts set elite.server.machineIds '["desktop","laptop"]'
```

Then restart the Pi session on the server: the `elite` config is read once, at
session start. Until the id is admitted, every batch from the client is answered
with

```json
{"ok":false,"reason":"unknown-machine"}
```

and the server logs one rate-limited warning naming the id and the fix:

```
[elite] ingest: rejected machine id "laptop" (unknown-machine): the server is single-tenant and already registered "desktop"; add it to elite.server.machineIds / ELITE_SERVER_MACHINE_IDS to admit it
```

### 2. Get the client onto the game machine

Clone the repository (recommended — `client/elite-stream.mjs` and
`scripts/elite-play.sh` come along, with their `--help`):

```bash
git clone <this-repo> elite_dangerouse_componion
```

or copy just the one file, from the server, with a single `scp` line:

```bash
scp <this-repo>/client/elite-stream.mjs gamerig:~/elite-stream.mjs
```

The streamer is plain Node ESM with no dependencies and no Pi: nothing else has
to be installed on the game machine.

### 3. Find the journal directory

Elite writes `Journal.<timestamp>.<part>.log` files and a `Status.json` into one
per-user directory. Where that is depends on how the game is installed:

| Install | Journal directory |
|---|---|
| Steam/Proton on Linux (app 359320) | `~/.steam/steam/steamapps/compatdata/359320/pfx/drive_c/users/steamuser/Saved Games/Frontier Developments/Elite Dangerous` — on another library, `<library>/steamapps/compatdata/359320/pfx/drive_c/users/<user>/Saved Games/...`; a running Steam game also exports the prefix as `$STEAM_COMPAT_DATA_PATH` |
| Heroic | `<defaultWinePrefix>/<game>/drive_c/users/<user>/Saved Games/Frontier Developments/Elite Dangerous`, e.g. `~/Games/Heroic/Prefixes/default/EliteDangerous/drive_c/users/steamuser/Saved Games/Frontier Developments/Elite Dangerous`; `defaultWinePrefix` is set in Heroic's own `config.json` |
| Lutris | `<prefix>/drive_c/users/<user>/Saved Games/Frontier Developments/Elite Dangerous`, where `<prefix>` is the `prefix:` of the game's YAML in `~/.config/lutris/games/` |
| Any other prefix (self-managed Wine/Proton; Elite has no native Linux build) | `<prefix>/drive_c/users/<user>/Saved Games/Frontier Developments/Elite Dangerous` |
| Windows | `%USERPROFILE%\Saved Games\Frontier Developments\Elite Dangerous` |
| macOS | `~/Library/Application Support/Frontier Developments/Elite Dangerous` |

One line proves the directory is the right one — it must hold at least one
journal and `Status.json`:

```bash
ls "<journal dir>"/Journal.*.log "<journal dir>"/Status.json
```

If nothing matches, start the game once (Elite only creates the directory once
it has run) and let the machine find it:

```bash
find ~ -name 'Status.json' -path '*Elite Dangerous*' 2>/dev/null
```

### 4. Start the streamer

With the two values from step 1 and the directory from step 3, on the **game
machine**:

```bash
export SERVER_URL="http://192.168.1.20:7345"
export TOKEN="<the token from step 1>"
export JOURNAL="/home/you/.steam/steam/steamapps/compatdata/359320/pfx/drive_c/users/steamuser/Saved Games/Frontier Developments/Elite Dangerous"

ELITE_JOURNAL_PATH="$JOURNAL" \
ELITE_CLIENT_SERVER_URL="$SERVER_URL" \
ELITE_CLIENT_TOKEN="$TOKEN" \
ELITE_CLIENT_MACHINE_ID="$(hostname)" \
  node client/elite-stream.mjs          # or ~/elite-stream.mjs from the scp copy
```

`ELITE_CLIENT_MACHINE_ID` is the id step 1 admitted and defaults to
`os.hostname()`; set it explicitly so the id does not change when the machine
is renamed. (`--journal-dir`, `--server-url`, `--token`, `--machine-id` are the
same values as flags.) On first connect the streamer backfills the last
`backfillDays` (default 30) of journals and then streams new lines as the game
writes them.

Or let `scripts/elite-play.sh` manage the lifecycle — it starts the streamer,
waits until it is connected, runs the game in the foreground and stops the
streamer when the game exits. It needs `ELITE_GAME_CMD` to **block until the
game exits**, and Steam does not give you that: with Steam already running,
`steam -applaunch 359320` (like the `steam steam://rungameid/359320` URL form)
forwards the command line to the running client and returns immediately, so
`elite-play.sh` would stop the streamer the moment it launched the game.

**For Steam, run the streamer independently of the launcher** — start it in the
background, play, and stop it when you are done:

```bash
mkdir -p ~/.cache/elite-stream

ELITE_JOURNAL_PATH="$JOURNAL" \
ELITE_CLIENT_SERVER_URL="$SERVER_URL" \
ELITE_CLIENT_TOKEN="$TOKEN" \
ELITE_CLIENT_MACHINE_ID="$(hostname)" \
  node client/elite-stream.mjs >~/.cache/elite-stream/streamer.log 2>&1 &
echo $! >~/.cache/elite-stream/streamer.pid    # stop it: kill "$(cat ~/.cache/elite-stream/streamer.pid)"

steam -applaunch 359320
```

`elite-play.sh` still fits when the launch command really does block on the
game: launching the Proton binary directly, or wrapping a launcher that returns
early with `scripts/elite-wait-for-game.sh`. That wrapper runs the launcher,
waits for the game process to appear (`ELITE_GAME_WAIT_APPEAR_S`, default 60 s),
then waits for it to exit:

```bash
ELITE_JOURNAL_PATH="$JOURNAL" \
ELITE_CLIENT_SERVER_URL="$SERVER_URL" \
ELITE_CLIENT_TOKEN="$TOKEN" \
ELITE_CLIENT_MACHINE_ID="$(hostname)" \
ELITE_GAME_CMD="scripts/elite-wait-for-game.sh EliteDangerous64 steam -applaunch 359320" \
  scripts/elite-play.sh
```

`EliteDangerous64` there is a `pgrep -f` match on the running game's command
line, not a fixed name — check yours with `pgrep -af Elite` while the game runs
and pass what it prints. `elite-play.sh` runs `ELITE_GAME_CMD` as one command
line, so run it from the clone root. The log is
`~/.cache/elite-stream/elite-play.log`.

### 5. Verify it is streaming

The streamer prints its target on startup and writes a ready marker once the
server has accepted its first batch:

```
[elite-stream] streaming /home/you/.steam/.../Elite Dangerous -> http://192.168.1.20:7345 as desktop
```

```bash
ls ~/.cache/elite-stream/ready     # exists while the streamer is connected
```

Ask the server with the token (from the game machine or any host on the LAN):

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.1.20:7345/health?machineId=desktop"
# {"ok":true,"version":"1","acceptingUpTo":{"lastSeq":6,"files":{"1768551300000:2":835,"...":18422}},"backfillRequest":null}
```

`acceptingUpTo` is null until the first batch is accepted; a cursor with files
and a `lastSeq` means the server has consumed this machine's journals and will
resume from exactly there after either side restarts.

Finally, on the **server**, in the Pi session: `/elite`, or ask *"where am
I?"*. `elite_status` switches to the connected machine:

```
Elite Dangerous Companion — source: remote
Journal:   remote: desktop
Machines:  allow-list (desktop, laptop); connected: desktop (active)

System:    Sol
Station:   Abraham Lincoln (docked)
Ship:      FerDeLance "Good Intentions" (EXP-01)
Fuel:      28.8 / 32.0
Cargo:     0
Credits:   3,141,592
```

`source: remote` and `remote: desktop` mean a connected game machine is the
live source, and `Machines:` shows the gate, who is connected and which client
is active. While a client is connected the local journal path (if one is
configured) is muted; when the last client stops, the server falls back to it.

### When it does not work

- **The server is unreachable.** The streamer logs
  `cannot reach server (<reason>); retrying in ...`, keeps retrying (backoff up
  to 30 s) and loses nothing: un-acknowledged lines are spooled to
  `~/.cache/elite-stream/spool.jsonl` (`ELITE_CLIENT_SPOOL_DIR`) and replayed on
  reconnect, while the byte offsets in the `state.json` next to it mean a reboot
  on either side resumes where it stopped, without duplicates. Only the last
  `backfillDays` (30) of journals stream on a first connect — from Pi, ask for
  older history with `elite_backfill` (a `range` phrase like `"last year"`, or
  `from`/`to`).
- **`403 {"ok":false,"reason":"unknown-machine"}`.** Another machine id is
  registered and this one is not in `elite.server.machineIds`. Add it (step 1
  again), restart the Pi session on the server, and the already-retrying client
  starts streaming on its own.
- **Connected but no data.** `ready` only means the server accepted the
  connection — an empty or wrong journal directory still connects and streams
  nothing. The streamer's log line names the directory it watches; check it with
  the `ls` from step 3.
- **The game is running, Pi shows no new events, and the streamer stopped.**
  `ELITE_GAME_CMD` returned as soon as it launched the game — Steam forwarding
  `-applaunch` to the running client does exactly that — so `elite-play.sh` took
  the streamer down with it. Use the background streamer above, or wrap the
  launcher with `scripts/elite-wait-for-game.sh` and give it the process name
  from `pgrep -af Elite`.
- **`401 unauthorized`.** The two tokens differ, and the streamer's log shows
  `cannot reach server (unauthorized)`. Compare
  `node ~/.pi/agent/bin/pi-config.mts get elite.server.token` on the server with
  the `ELITE_CLIENT_TOKEN` you exported on the game machine. A token that is not
  present at all is cleared at session start rather than used as a literal
  `${...}` string, and shows up in the server's config warnings.
- **Nothing reaches the server at all.** The listener binds `127.0.0.1` by
  default, which a remote machine cannot reach. Set `elite.server.host` to
  `0.0.0.0` (or the LAN/tailnet address) on the server, restart the Pi session,
  and confirm the listener with `ss -ltnp | grep 7345` on the server. If it is
  bound correctly, a firewall is the next suspect.

The whole flow above is exercised end to end by a script, with the real server
listener and the real streamer over the committed journal fixtures:

```bash
bash scripts/check-game-machine-quickstart.sh
```

## Lean session (context budget)

A bare `pi` from the project root loads every global extension, package, MCP
server, skill and prompt template installed on the machine, and their
descriptions share the context window with the game. `scripts/elite-pi.sh`
starts the same project session with only what the companion needs:

```bash
scripts/elite-pi.sh                   # lean session, from any working directory
scripts/elite-pi.sh --continue        # every argument is passed to pi unchanged
scripts/elite-pi.sh --help            # the flags, what they drop, the measured numbers
ELITE_PI_FULL=1 scripts/elite-pi.sh   # opt back in: plain pi, everything loaded
```

The launcher resolves the repository root from its own path and changes into it
(Pi reads project `.pi` configuration from the working directory only), then
runs:

```bash
pi --no-extensions \
   -e builtin:codemode \
   -e .pi/extensions/elite \
   --no-mcp --no-skills --no-prompt-templates "$@"
```

| Flag | Drops | Get it back |
|---|---|---|
| `--no-extensions` | global extensions and npm/git packages: their tools, commands and prompt sections | `ELITE_PI_FULL=1`, or bare `pi` |
| `--no-mcp` | MCP servers and their tools (`~/.pi/agent/mcp.json`, project `.pi/mcp.json`) | `ELITE_PI_FULL=1`, or bare `pi` |
| `--no-skills` | the `skills` system-prompt section and `/skill:*` commands | `ELITE_PI_FULL=1`, or bare `pi` |
| `--no-prompt-templates` | prompt templates, exposed as `/name` commands | `ELITE_PI_FULL=1`, or bare `pi` |

What stays: the built-in agent tools (`read`, `bash`, `edit`, `write`),
`codemode` (`-e builtin:codemode`), and the complete `elite` extension — its six
direct tools, the `elite_*` tools reachable from codemode, the `/elite*`
commands, the `elite_crew` and `elite_historian` prompt sections and the status
widget.

Measured on Pi 1.1.0 against this repository (both prompts embed the project
`AGENTS.md`, so editing it moves both numbers together):

| | Bare `pi` | `scripts/elite-pi.sh` |
|---|---|---|
| system prompt | 24,244 chars | 13,489 chars (**44 % smaller**) |
| `skills` section | 8,323 chars | dropped |
| tool declarations (`tools` section) | 1,331 chars, 16 tools | 891 chars, 11 tools |

Both numbers are reproducible: `bash scripts/check-lean-launch.sh` runs each
launch through `pi --mode rpc` (one trivial prompt each), reads the system
message the session actually rendered, and asserts the lean tool set, the
missing `skills` section, the retained `elite_crew`/`elite_historian` sections
and the ≥ 25 % reduction. On a machine whose full session has no skills section
to drop, the ratio check is reported as not applicable instead of failing — as
is the whole baseline comparison when plain `pi` cannot start because a
user-scope extension of the machine is broken.

```bash
bash scripts/check-lean-launch.sh
```

### Why not a project settings file?

A project `.pi/settings.json` cannot gate user-scope extensions, so it cannot be
the mechanism. Project override entries (`-path`) are applied only while Pi
collects *project* resources; user extensions are filtered by the user settings
list alone, and user-scope packages are resolved separately from project ones,
so a project `-npm:...` entry does not disable them either. Verified on this
machine: a project settings file containing

```jsonc
{ "extensions": ["-~/.pi/agent/extensions/plan-mode"],
  "packages":   ["-npm:@juicesharp/rpiv-ask-user-question"] }
```

left `plan_enter` and `ask_user_question` declared, while the same syntax aimed
at a project extension (`-extensions/beta/index.ts`) did disable it. Built-in
extensions are the documented exception — a project `+`/`-`/`!` entry overrides
the user setting — but that covers only `builtin:*`. A launcher with
`--no-extensions` and explicit `-e` loads is the mechanism.

## Configuration

Machine-local values live in the pi-config `elite` section (never in this
repository). Resolution order, low → high: built-in defaults → system
pi-config → user pi-config → `ELITE_*` environment variables → autodetection.

```jsonc
{
  "elite": {
    "role": "server",                          // "server" | "client"
    "journalPath": null,                       // local fallback; null → autodetect
    "dataDir": ".pi/elite",                    // runtime data (state, crew, archive), project-relative
    "exportDir": "export",                     // EDCoPilot output root, project-relative
    "server": {
      "host": "127.0.0.1",
      "port": 7345,
      "token": "${eliteToken}",                // bearer token shared with the client; set secrets.eliteToken in pi-config
      "machineIds": [],                        // allow-list; empty = single-tenant auto-register (see "Several game machines")
      "clientTimeoutMs": 60000
    },
    "client": {
      "serverUrl": "",                         // client side: e.g. http://<server>:7345
      "token": "",
      "spoolDir": "~/.cache/elite-stream",
      "backfillDays": 30,
      "machineId": ""                          // default: os.hostname()
    },
    "maxRecentEvents": 10000,
    "notifications": { "enabled": true, "mode": "digest", "digestMinutes": 10, "quietHours": null },
    "crew": { "enabled": true, "autoBeats": true, "activeWindowMinutes": 5, "briefingOnStart": true },
    "historian": { "enabled": true, "webSearch": true, "maxPassages": 5, "nestedWhenHeadless": true }
  }
}
```

`${...}` tokens interpolate against the pi-config `vars` and `secrets`
leaves and the process environment — the same substitution pi-config applies
to the rest of its config file. Reference a leaf **by name alone**: a token
stored as `secrets.eliteToken` is written `${eliteToken}`. The dotted
`${secrets.eliteToken}` / `${vars.name}` spellings are accepted too, but the
flat form is the convention (and the only one pi-config documents). A string
that is exactly one placeholder yields the raw value (number, boolean or
object); embedded tokens are stringified. A resolved value may name further
placeholders — the chain is followed until it resolves or a depth limit of 10
(the same limit as pi-config) is reached.

A placeholder that does not resolve is never silently accepted: it is left in
the config as literal text, collected into `configWarnings`, and logged at
session start and shown by `elite_status` and `/elite`. A token is the one
exception — an unresolved `server.token`/`client.token` is cleared rather than
kept, so a config copied without its secret fails loudly instead of making the
literal string `${eliteToken}` the shared secret.

Any leaf can also be set via `ELITE_*` (the dotted path upper-cased, camelCase
humps split by `_`): `server.port` → `ELITE_SERVER_PORT`,
`client.backfillDays` → `ELITE_CLIENT_BACKFILL_DAYS`,
`notifications.digestMinutes` → `ELITE_NOTIFICATIONS_DIGEST_MINUTES`.

Autodetection covers Steam (`libraryfolders.vdf`), the Proton prefix for app
359320, Heroic, Lutris, Windows `%USERPROFILE%\Saved Games\Frontier Developments\Elite Dangerous`,
and the macOS equivalent. Set `elite.journalPath` (or `ELITE_JOURNAL_PATH`) when
autodetection does not find your install.

## Client setup (game machine)

The game machine runs only `client/elite-stream.mjs`. It needs Node and no Pi.
The step-by-step version is [Quick start: the game machine](#quick-start-the-game-machine);
what follows is the reference.

```bash
ELITE_JOURNAL_PATH="/path/to/journal" \
ELITE_CLIENT_SERVER_URL="http://<server-host>:7345" \
ELITE_CLIENT_TOKEN="<same token as server.server.token>" \
node client/elite-stream.mjs
```

Or use `scripts/elite-play.sh`, which starts the streamer, waits until it is
connected, launches the game in the foreground, and stops the streamer when the
game exits. It reads the same `ELITE_*` variables plus `ELITE_GAME_CMD` (a
command that blocks until the game exits — Steam's `-applaunch` returns
immediately, so wrap it as the quick start shows). The streamer, `elite-play.sh`
and `elite-wait-for-game.sh` all answer `--help`.

On first connect the client backfills the last `backfillDays` (default 30) of
journals, then streams deltas. Use `elite_backfill` with a `range` phrase (or
`from`/`to`) to request older history, delivered on the client's next
heartbeat. `scripts/elite-play.sh` (with its optional `elite-wait-for-game.sh`
wrapper) and the streamer are the only things that run on the game machine;
everything else lives on the server.

## Several game machines

One machine needs no extra configuration: with an empty
`elite.server.machineIds` the server is single-tenant and auto-registers the
first machine id it sees. For two or three machines set the allow-list
explicitly on the server — without it the second machine is rejected as
`unknown-machine`:

```jsonc
{ "elite": { "server": { "machineIds": ["desktop", "laptop"] } } }
```

or, without editing pi-config, `ELITE_SERVER_MACHINE_IDS='["desktop","laptop"]'`.
Each game machine presents its own id: it defaults to `os.hostname()` and is
overridden with `ELITE_CLIENT_MACHINE_ID` (or `--machine-id`):

```bash
ELITE_JOURNAL_PATH="/path/to/journal" \
ELITE_CLIENT_SERVER_URL="http://<server-host>:7345" \
ELITE_CLIENT_TOKEN="<same token as server.server.token>" \
ELITE_CLIENT_MACHINE_ID="desktop" \
node client/elite-stream.mjs
```

Every machine is tracked independently. The de-duplication cursor
(`ingest-cursors.json` under `dataDir`) and the raw journal archive
(`archive/<machineId>/<filename>`) are per machine, and `elite_history` scans
all of them as one merged history. Switching from the desktop to the laptop
loses nothing: the laptop streams from its own cursors, and the history of
both machines stays queryable.

Several clients may be connected at once, and the live buffer is shared: the
event store is a single stream because it is the same commander in the same
galaxy either way. Play one machine at a time only because the game account
itself is single, not because the server cannot hold two connections. When the
last client disconnects, the local journal path (if configured) resumes.

### Verify a connection end to end

From the game machine (or any host on the LAN), with the server's token:

```bash
curl -H "Authorization: Bearer <token>" \
  "http://<server-host>:7345/health?machineId=desktop"
# {"ok":true,"version":"1","acceptingUpTo":null,"backfillRequest":null}
```

`acceptingUpTo` is null on a first connect (the client then backfills the last
`backfillDays`); afterwards it carries that machine's per-file cursor. A machine
id that is not admitted gets HTTP 403 with the exact reason:

```bash
curl -i -H "Authorization: Bearer <token>" -H "content-type: application/json" \
  -d '{"machineId":"intruder","clientSeq":1,"mode":"delta","files":[]}' \
  http://<server-host>:7345/ingest
# HTTP/1.1 403 Forbidden
# {"ok":false,"reason":"unknown-machine"}
```

The server writes a warning to the console of the Pi session that runs the
extension, naming the rejected id and the fix. It is rate-limited per machine
id, so a retrying client cannot spam the log:

```
[elite] ingest: rejected machine id "intruder" (unknown-machine): the server is single-tenant and already registered "desktop"; add it to elite.server.machineIds / ELITE_SERVER_MACHINE_IDS to admit it
```

`elite_status` (and `/elite`) shows the same picture live: the gate mode, the
configured ids, and who is connected — with the active source marked:

```
Machines:  allow-list (desktop, laptop); connected: desktop (active), laptop
```

## Tools and commands

Tools are grouped under the `elite` namespace. Six are **hot** (`direct`,
declared to the model): `elite_status`, `elite_events`, `elite_history`,
`elite_resource`, `elite_historian`, `elite_crew_say`. Everything else —
`elite_search`, `elite_presence`, `elite_chronicle`, `elite_backfill`, the
`elite_edcopilot_*` and `elite_theme_*` tools — is reachable through codemode
under the same namespace.

Commands:

- `/elite` — one-screen status summary.
- `/elite-explore`, `/elite-trade`, `/elite-combat`, `/elite-mine`,
  `/elite-mission`, `/elite-engineer`, `/elite-journey`,
  `/elite-performance`, `/elite-strategy` — analytics prompt commands built
  from live state.

## Development

```bash
cd .pi/extensions/elite
node --test test/*.test.mjs      # unit + integration tests (hermetic, offline)
npx tsc -p tsconfig.json         # typecheck
```

`core/` is pure logic with no UI, sockets or `ExtensionContext`; `net/` is the
only place a socket exists. Tests import TypeScript sources directly (Node
strips types), write scratch data only under `test/tmp/`, and never touch the
real journal or pi-config.

Golden parity (`scripts/check-parity.mjs`) regenerates the committed fixtures
under `test/fixtures/golden/` from the Python original when that checkout and a
Python venv are present; the unit suite itself stays hermetic.

`scripts/check-lean-launch.sh` exercises both launch modes of
`scripts/elite-pi.sh` through `pi --mode rpc` and checks the resulting context
budget (see [Lean session](#lean-session-context-budget)); it asserts on the
rendered system message, not on the model's answer.

`scripts/check-game-machine-quickstart.sh` replays the game-machine quick start
end to end: the real ingest listener with a temp dataDir and an ephemeral port,
the real `client/elite-stream.mjs` over the committed journal fixtures, then the
archive, `/health` cursor, `elite_status` (`source: remote`, plus a live delta),
the allow-list rejection and the `elite-wait-for-game.sh` blocking guarantee. It
starts and kills its own processes and removes its temp tree.

## Known limitations

- The lore snapshot is static: it is refreshed only by running
  `node scripts/snapshot-lore.mjs`; fresh GalNet/patches require web search.
- EDCoPilot chatter files are written to the server's export directory only;
  delivery into the running game is out of scope.
- Push notifications are gated on the agent being idle; non-critical events
  batch into a digest rather than interrupting an active turn.
- The raw journal archive is never pruned automatically — cleanup is manual.
- Linux `fs.watch` can drop events, so both client and server use a poll
  fallback; brief gaps are re-covered by the byte-offset cursor.
- Pi must be launched from the project root, or through `scripts/elite-pi.sh`,
  which starts there: a subdirectory launch does not load the project extension.
