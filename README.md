# claude-duet-bridge

Let a Claude running in the cloud (a Claude Code web session, a scheduled agent, the Claude app) hand jobs to
Claude Code on **your own computer**, Windows or Mac, and let your local agents hand questions back.

The channel is GitHub issues on a **private** repo you own. No server, no open port, no webhook: your machine
polls, runs only what you allowed, and comments the result.

```
cloud Claude ──opens issue "job"──▶  private jobs repo  ◀──polls every 10 min── bridge.py on your PC / Mac
             ◀──reads the comment──                     ──comments, labels done──
```

## What it does

- **Jobs.** The cloud side opens an issue titled with a job name (`git status`) and the argument in the body.
  Your machine runs the command you mapped to that name, comments the output, labels it `done` and closes it.
- **ask agent.** Title `ask agent`, body `<agent name>` on the first line and the question below. Your machine
  starts headless Claude Code (`claude -p`) in that agent's folder with read only tools, and comments the answer.
- **bridge status.** Built in: what this machine can run and its last few jobs.
- **Asks for you.** Local agents can open an issue labeled `for-owner`; your reply comment is picked up on the next
  poll, labeled `answered`, and appended to a local answers file.

## Safety, by construction

- Only issues labeled `job` **and** opened by `owner` are read. Anyone else's issues are never looked at.
- A job runs only if its name is in **both** the allowlist file (`jobs.md`) and `jobs` in your local config.
  The command lives only in the local config, so editing the issue or the repo cannot change what runs.
- Arguments must match the job's regex and are passed as argv, never through a shell.
- `needs_yes` jobs and agents are labeled `waiting` until you type `python bridge.py approve N` on the machine.
  A comment saying "approved" does nothing.
- `ask agent` runs with `--tools Read,Grep,Glob,WebSearch`, `--strict-mcp-config` (no MCP servers), a time limit,
  and `Read(...)` deny rules for the paths in `deny_read`.
- Output is scrubbed of token shaped strings before posting. Keep the jobs repo private anyway.

## Setup (Windows or Mac)

Needs Python 3.9+, the [GitHub CLI](https://cli.github.com) signed in as the owner (`gh auth login`), and
Claude Code (`claude`) on PATH if you use `ask agent`.

```bash
git clone https://github.com/skelly1313/claude-duet-bridge
cd claude-duet-bridge
cp bridge.config.example.json bridge.config.json     # Windows PowerShell: copy bridge.config.example.json bridge.config.json
gh repo create <you>/claude-bridge --private
```

Fill in `bridge.config.json`:

| key | what |
|---|---|
| `owner` | your GitHub login; the only author whose issues count |
| `jobs_repo` | the private repo for issues, `owner/name` |
| `platform` | `auto` (default), or force `windows` / `mac` / `linux` |
| `root` | the folder jobs run in; absolute, `~/...`, or relative to the config |
| `allowlist_file` | a markdown file listing allowed names as bullets: ``- `git status`: what it does`` |
| `jobs` | name → `{what, arg (regex, named groups; {EMAIL} is a ready pattern), defaults, run (list of argv lists), cwd, needs_yes, timeout_seconds}` |
| `agents` | name → `{cwd, who, needs_yes}` for `ask agent` |
| `ask` | `model`, `tools`, `timeout_seconds`, `add_dirs`, `deny_read`, `prefix` (`{who}` is filled in) |

In `run`, `{python}` is the interpreter running the bridge, `{npm}` / `{npx}` become `npm.cmd` / `npx.cmd` on Windows,
and `{group}` is any named group from `arg`.

Then:

```bash
python bridge.py setup     # creates the labels on the jobs repo
python bridge.py config    # check paths and jobs
python bridge.py poll      # one pass
```

Keep it polling, any of:

- **Claude Code:** `/loop 10m run python bridge.py poll and tell me only if a job ran, failed or waits on my yes`
- **Mac:** `crontab -e` → `*/10 * * * * cd /path/to/claude-duet-bridge && /usr/bin/python3 bridge.py poll`
- **Windows:** Task Scheduler, every 10 minutes, `python C:\path\to\claude-duet-bridge\bridge.py poll`

## Using it from the cloud side

Tell the cloud Claude: "To run something on my computer, open an issue on `<you>/claude-bridge` with label `job`,
title = the job name from `jobs.md`, body = the argument. Read the result in the comments 10 minutes later."
A session with GitHub access can do that with `gh issue create -R <you>/claude-bridge -l job -t "git status" -b "my-app"`.

## License

MIT
