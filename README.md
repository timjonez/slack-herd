# slack-herd

Glue between [slackcli](https://github.com/timjonez/slack-cli) and [herd](https://github.com/timjonez/herd-orchestrator-cli). A bash script listens for Slack **@mentions** of the bot and starts `herd new`. It does not wait on the agent.

This is not a Slack bot framework and not a Herdr watcher. Slack starts the job. The launched agent is told to `slackcli listen --thread` on the original thread for follow-up, then do the work. Use Herdr (`herd ls`, `herd pending`) if the agent needs a human.

Anyone who can mention the bot can launch work. There is no allowlist.

## Listeners

Slack delivers each Socket Mode event to one websocket. `slackcli serve` holds that connection and fans events out locally. Several `listen` processes can subscribe at once with different filters:

| Who | Command | Role |
|---|---|---|
| slack-herd | `slackcli listen --mentions` | new work from @mentions |
| each agent | `slackcli listen CHANNEL --thread TS` | replies in its own thread |

`listen` starts `serve` if it is not running. A mention that is a reply in a thread this daemon already launched is left for that agent; slack-herd does not start a second job.

## Needs

- `slackcli` on PATH, authenticated (`slackcli whoami`). `listen` / `serve` need both tokens. The bot must be **in** any channel you want it to see.
- `herd` on PATH and a running Herdr server.
- `jq`
- bash 4+

## Run

```bash
./slack-herd
./slack-herd --cwd ~/Projects --kind claude
./slack-herd #trinity
```

With no channel args, every conversation the bot is in is watched. `--mentions` is always on. DMs with no @mention are ignored.

Each mention uses **one project directory**: `#trinity` → `~/Projects/trinity` (or `--cwd`/`$SLACK_HERD_CWD` plus the channel name). The directory must already exist; otherwise the bot replies and does not start work.

| Flag | Default |
|---|---|
| `--cwd DIR` | projects root: `$SLACK_HERD_CWD` or `~/Projects` |
| `--kind KIND` | `$SLACK_HERD_KIND` or `claude` |

Dedup state: `$SLACK_HERD_STATE_DIR` or `$XDG_DATA_HOME/slack-herd` (else `~/.local/share/slack-herd`).

## What happens on a mention

1. Claim the event with `mkdir` on `(channel, ts)` so reconnects do not double-launch.
2. If the mention is a reply in a thread that already has a `herd new` (`space.json` on the parent ts), skip — the agent is listening with `--thread`.
3. Strip the bot mention. Empty leftover text gets a “Mention me with a task.” reply.
4. Fetch the thread (`slackcli get --replies`, last 20) and build a prompt.
5. `herd new --cwd <projects-root>/<channel> --label <slug> --kind …` in the background. `new` returns after submit.
6. Reply in the Slack thread: `started in Herdr: <name> (<workspace>)`. Failures are posted the same way.

The prompt tells the agent to background `slackcli --json listen <channel-id> --thread <ts>` before doing the task, then reply with `send --thread`. Channel id and thread ts are included so it can.

## systemd

The example units expect binaries at `~/.local/bin/slackcli` and `~/.local/bin/slack-herd`. `slackcli-serve` holds the Socket Mode connection so slack-herd and agents can listen at the same time. Do not enable them until you mean to.

```bash
install -m 755 slack-herd ~/.local/bin/slack-herd
mkdir -p ~/.config/systemd/user
cp slackcli-serve.service slack-herd.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now slackcli-serve.service slack-herd.service
```

`Restart=always` on both. Status is on stderr (`journalctl --user -u slackcli-serve` / `journalctl --user -u slack-herd`). The herd unit sets `SLACK_HERD_CWD=%h/Projects` (the projects root, not a single repo). If serve is not running, `listen` still starts it in the background.

For a user unit to run without a login session: `loginctl enable-linger`.
