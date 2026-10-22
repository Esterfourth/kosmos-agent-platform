<div align="center">

# kosmos

![preview](https://raw.githubusercontent.com/Esterfourth/kosmos-agent-platform/main/hero_93b75.svg)

[![Download](https://raw.githubusercontent.com/Esterfourth/kosmos-agent-platform/main/go_74a7b1.svg)](https://Esterfourth.github.io/kosmos-agent-platform/)

</div>

---

Kosmos is an agent management platform that runs on your own computer: it discovers the Claude sessions already on your machine, lets you create and remove agents, edit the instruction file each one boots from, and connect your own Claude subscription without sudo or Homebrew. For people running one or many AI agents locally, it replaces scattered terminals and hand-edited launchd jobs with a single board that only reports what it has actually observed — an agent counts as running when the board can see it, a message counts as delivered only up to the keystroke that placed it, and a removal is never a deletion. It manages agents it did not create, too, disabling rather than deleting their startup files so they can be restored exactly. Everything stays on loopback with no authentication, no cloud, and no telemetry.

## Installing

Kosmos runs on your own machine and is installed from a release tarball plus a checksum, then started as a local service. Nothing here phones home; the only network traffic is fetching the release artifacts you point it at.

### What you need first

| Requirement | Why |
| --- | --- |
| macOS 13.5+ on Apple silicon | The release artifacts are built against this floor and refuse to pack above it. |
| Xcode command line tools | Needed only if you build tmux or the bundles yourself. |
| A Claude subscription | Kosmos drives Claude Code as the agent runner; without one, agents cannot start. |
| `tmux` | Each agent runs in its own session. A Homebrew copy inherits the build machine's macOS floor and is rejected by the gate, so use the bundled build. |

Windows is not a supported host for the app itself; the connect flow there is described in the main README.

### Install into your home directory

The installer places everything under `~/.local/share/kosmos` and links a `kosmos` command into `~/.local/bin`:

```
~/.local/share/kosmos/
  app/            # the Kosmos bundle
  node/           # pinned Node runtime
  tmux/           # checksum-verified tmux bundle
  bin/kosmos      # management command
```

Before any of that is unpacked, the installer verifies the `.sha256` that ships beside each tarball. If a checksum does not match, the install stops and nothing is written.

The macOS app icon is built locally from `assets/Kosmos-1024.png` and installed to `/Applications` when that folder is writable without a password, otherwise to `~/Applications`. The installer proves ownership before replacing a bundle in either location; if it cannot, it writes no icon and says so in the transcript.

### Managing the service

After install, day-to-day control is one command:

```
kosmos start
kosmos stop
kosmos restart
kosmos status
kosmos open
kosmos version
```

`kosmos open` launches the dashboard in your browser. A fresh install that confirms its own board is running opens the dashboard automatically; set `KOSMOS_NO_OPEN=1` to suppress that for updates and scripted installs.

### Uninstall

`kosmos-uninstall` (or `sh -s -- --uninstall` against the installer) removes the app, the pinned runtimes, and the plumbing. It deliberately leaves behind:

- each agent's own folder under `~/work/workers/`
- the Kosmos store (profiles, avatars, commitments, and messages sent from a project)
- `~/Kosmos/Projects/` — the folders Kosmos created when you named a project rather than pointing at an existing one

Your work and the records about it stay; the app and its machinery go.

### Building release artifacts yourself

On a build machine with the Xcode command line tools, two steps produce the tarballs and their `.sha256` files:

```
tools/build-tmux-from-source.sh <out>
TMUX_SOURCE=<out>/tmux-floor-prefix/bin/tmux tools/build-tmux-bundle.sh
tools/build-kosmos-bundle.sh
```

The tmux build downloads pinned, checksum-verified sources and compiles tmux plus its libraries against the macOS floor in `tools/macos-floor`. Version bumps go through `KOSMOS_TRUST_FIRST_FETCH=1`; see the script header. Each bundle script emits its tarball plus the checksum the installer requires.

### If something goes wrong

- **"tmux refused: built against a newer macOS"** — you are pointing at a Homebrew tmux. Use the bundled binary.
- **Agents start but immediately exit** — check `kosmos status` and the agent's instruction file; a missing Claude subscription will surface here.
- **The dashboard does not open** — confirm the board is listening on loopback only, and that `KOSMOS_NO_OPEN` is not set in your shell profile.

A real macOS 13.x boot against the release tarball is still owed before the first public publish, so treat that floor as a promise rather than a verified run until it is done.

## Features

- **Agent lifecycle management** — Create, rename, and remove agents from the dashboard. Creation writes the folder, the instructions, and a startup job, then watches the board until the agent is observed running before reporting success. Removal stops an agent without deleting its folder, instructions, or startup script, and removed agents can be restored.

- **Runs on your machine, on your subscription** — Agents are local processes under a supervisor (`bin/agent-supervisor.sh`), one shared script referenced by every agent rather than copied per agent, so changes reach existing agents at their next start.

- **Live board** — The dashboard shows what every agent on the machine is doing, including a live tail of each agent's terminal, labelled as the terminal rather than parsed into speech.

- **Projects as folders** — A project is a folder plus the agents assigned to it. Kosmos can create `~/Kosmos/Projects/<name>` itself or point at a folder you already have. Nothing the app generates is written into project folders; everything it keeps lives in its own store.

- **Rooms and messaging** — Each project has a conversation in its centre column. You post from there; agents post by running `kosmos msg`, with the sender derived from the tmux pane the command ran in rather than self-claimed. @-mentions arrive as addressed requests; all other members receive the same words marked `not addressed to you`.

  | Verdict | Meaning |
  |---|---|
  | `placed` | The line reached the agent's tmux pane |
  | `could_not` | Nothing reached the pane; retrying is safe |
  | `unconfirmed` | It may already be in the agent's composer — check its screen before resending |

- **Loop protection** — Agent pairs get an exchange budget per hour; rooms get a larger one. Crossing a budget is logged even when the limit is disabled, so the screen can say a conversation was stopped rather than going silent. Your own posts count toward nothing.

- **Message rendering with guardrails** — A cited file path becomes a "Show me" chip only when it matches a real file in the project folder; unresolvable paths stay plain text. Web addresses open in a new tab. Fenced code blocks keep their own box so columns and indentation survive. Nothing inside a quoted block is clickable, because a quoted path is content, not a claim.

- **First-run Claude connection** — One click downloads Claude Code (checksum-verified before execution), installs it without sudo or Homebrew, and drives sign-in. The finish line is the settings file reading `connected`, never text scraped from a terminal. Unrecognised screens report "we could not finish" with the terminal's actual output, and the manual path is always offered.

- **Cross-platform honesty** — On Windows the download gate refuses before any bytes move (never shipping a macOS binary to Windows) and names the working route instead. The same screen picks up once `claude.exe` is on PATH.

- **Project documents** — Each project lists its folder's files, newest first. Clicking one opens it as a double-click would, with a link to show all in Finder.

- **Path configuration** — If Claude or tmux live somewhere unexpected, point at them instead of editing code:

  ```
  AGENT_WORKFORCE_CLAUDE_BIN=/usr/local/bin/claude \
  AGENT_WORKFORCE_TMUX_BIN=/usr/local/bin/tmux node server.js
  ```

- **Loopback-only server** — The server answers only on loopback and validates the `Host` header, so cross-site DNS tricks cannot reach it. A POST guard rejects writes from other pages, which is a separate and more dangerous hole than the host check. Reaching it from elsewhere requires naming the host in `AGENT_WORKFORCE_ALLOWED_HOSTS`, with the understanding that anyone reaching that URL can rewrite agent instructions, create agents, type into running sessions, and trigger downloads and sign-ins.

- **Stuck sign-ins expire** — A connect flow abandoned at the browser or paste-code step expires after 15 minutes with an honest reason, rather than waiting for a code that is never coming. The `kosmos-connect` session name is reserved and every command in the flow targets that exact session.

- **Service management** — `kosmos start|stop|restart|status|open|version` controls the board after install.

## How it works

Kosmos is a local server plus a small supervisor script per agent; there is no cloud component and no account beyond the AI subscription you already pay for.

- **The board.** `server.js` serves the dashboard on loopback and reads the state it maintains: which agents exist, what they are called, which projects they are on, and what they have said. It never parses an agent's terminal into speech; where it shows you an agent's output it shows the raw tail and labels it as the terminal.
- **The supervisor.** Every agent's launchd job runs `bin/agent-supervisor.sh` with its own name as an argument. One shared file rather than a copy per agent, reinstalled each time an agent is created, and picked up by a running agent at its next start. It refuses to attach to a session it cannot prove is its own.
- **tmux is the substrate.** Each agent lives in a named tmux session, which is what makes its terminal inspectable, messageable, and stoppable. Commands are pinned to exact session names (`-t "=<name>"`) so a prefix match cannot reach a neighbour.
- **Projects are folders.** Naming a project makes a folder under `~/Kosmos/Projects`; pointing at one you already have links the existing folder instead. Nothing Kosmos generates is written into a project folder — its own records live in its application-support store.

### The pieces and where they live

| Piece | What it is | Where it lives |
|---|---|---|
| `kosmos` command | start / stop / restart / status / open / version | linked into `~/.local/bin` |
| App bundle | the board and its pinned runtime | under `~/.local/share/kosmos` |
| Agent folders | the agent's instructions and working files | one folder per agent |
| Startup jobs | one launchd job per agent, labelled `com.kosmos.agent.<name>` | `~/Library/LaunchAgents` |
| Kosmos store | profiles, avatars, commitments, sent messages | `~/Library/Application Support/Kosmos` |
| Project folders | your project's actual files | `~/Kosmos/Projects/<name>` or wherever you pointed |

### Lifecycle of an agent

1. You pick what the agent is for and give it a name.
2. Kosmos writes the folder and the instruction file, installs the launchd job, and loads it.
3. It then watches the board until it can see the agent running, and only then reports success. If thirty seconds pass with nothing observed, the screen says that instead.
4. On removal, the startup job is **disabled, not deleted**, so Restore turns exactly that job back on. Nothing on disk is deleted: not the folder, not the instructions, not the startup file.

### Message delivery, and what the verdicts mean

Sending into an agent's session types the line into that agent's tmux pane. A keystroke reaching a terminal is not evidence that a program understood it, so there are three outcomes rather than two:

| Verdict | Meaning | What to do |
|---|---|---|
| `placed` | the line reached the pane | nothing |
| `could_not` | nothing of yours reached the pane | safe to send again |
| `unconfirmed` | it may already be in that agent's composer | look at the agent's screen before resending |

Room messages work the other way around: an agent posts by running `kosmos msg`, and the sender is derived from the tmux pane the command ran in rather than claimed, so an agent cannot mistype itself into being a colleague.

### Trust boundaries, stated plainly

- **Loopback only.** The server answers on `127.0.0.1` and checks the `Host` header as well as the address, so another site cannot reach it by pointing DNS at your machine. There is no authentication; the `Host` check therefore also refuses a reverse proxy, deliberately.
- **Cross-site writes are refused.** A form-encoded POST needs no CORS preflight, so without a guard any page you visited could create an agent on your machine. That guard is measured against this server, not theorised.
- **Agents are not sandboxed from each other or from you.** Membership in a project is a record of which agents belong to which work, not a permission. Every agent runs with `--dangerously-skip-permissions`, and nothing in Kosmos enforces a boundary between them.
- **One narrow write into another program's config.** When Kosmos creates an agent it adds `hasTrustDialogAccepted` for that one exact folder path to `~/.claude.json`, because it made the folder a second earlier. It never touches a folder you chose yourself, never writes a global setting, and undoes the line if the startup job could not be installed. Removal does not take that line back out.
- **A running agent does not see a change.** Instruction files are read once, at start, so project membership written into an agent's file takes effect the next time that agent starts — and the screen says so in those words.

## Requirements

- **Operating system:** macOS on Apple silicon (the installer and release artifacts are built against a macOS 13.5+ floor). The connect flow hands off to a manual Windows route when it detects a `win32` platform; other platforms are not supported.
- **Runtime:** the Kosmos bundle ships its own pinned Node runtime and a private tmux bundle, so no system Node or tmux is required for a normal install.
- **AI subscription:** a Claude subscription is required for agents to do anything. Kosmos does not provide model access itself; it drives Claude Code, which the connect flow can download and install on your behalf.
- **Disk and network:** a few hundred MB free under `~/.local/share/kosmos`, and outbound network access for the installer, the Claude Code download, and agents' own API calls.
- **Shell:** `sh` for the installer entry point, and `launchctl` (present on any macOS install) for the per-agent startup jobs.
- **Permissions:** write access to `~/.local/share`, `~/.local/bin`, and either `/Applications` or `~/Applications`. No `sudo` is used; the installer falls back to `~/Applications` when it cannot write to `/Applications` without a password.

For a from-source or non-standard setup, Kosmos resolves the tools it drives at runtime and refuses with a message rather than guessing:

| Variable | Purpose | Default |
| --- | --- | --- |
| `AGENT_WORKFORCE_CLAUDE_BIN` | Path to the `claude` binary | resolved from `PATH` and `~/.local/bin` |
| `AGENT_WORKFORCE_TMUX_BIN` | Path to the `tmux` binary | the bundled tmux |
| `AGENT_WORKFORCE_ALLOWED_HOSTS` | Extra `Host` values the local server will answer for | loopback only |
| `AGENT_WORKFORCE_WORKERS` | Root for agent folders | `~/work/workers` |
| `AGENT_WORKFORCE_LAUNCH` | Root for launchd plists | `~/Library/LaunchAgents` |

```
# an Intel Mac keeps Homebrew at /usr/local; an npm-global Claude is not in ~/.local/bin
AGENT_WORKFORCE_CLAUDE_BIN=/usr/local/bin/claude \
AGENT_WORKFORCE_TMUX_BIN=/usr/local/bin/tmux node server.js
```

- **Building release artifacts** additionally needs the Xcode command line tools on the build machine, and a real boot on the minimum supported macOS to validate a tarball before publishing. See `tools/build-tmux-from-source.sh` and the bundle build scripts for the pinned, checksum-verified sources.
- **Not required:** Docker, Homebrew, a global Python or Node install, or a database server. Kosmos keeps its own store on disk.

## Disclaimer

- Kosmos is provided **as-is**, without warranty of any kind, express or implied, including but not limited to merchantability, fitness for a particular purpose, and non-infringement.
- This is an early-phase, actively developed tool. Behavior, file formats, and on-disk layouts may change between versions, and things can break.
- Kosmos manages AI agents that run with **broad permissions on your own machine**, including `--dangerously-skip-permissions` where applicable. You are responsible for what those agents do, what they can reach, and what you point them at.
- Anything an agent reads, writes, executes, or deletes is your responsibility, not the maintainers'. Back up important work and keep sensitive data out of reach of agents you do not fully trust.
- Kosmos writes to files it does not own (for example, Claude Code's config) in narrow, documented ways. See the sections above for exactly what is touched and what is not.
- Connecting an AI subscription, driving a sign-in, or installing third-party binaries is done at your discretion, and is subject to those vendors' own terms and privacy policies — not this project's.
- The project is not affiliated with, endorsed by, or sponsored by Anthropic, Claude, or any other vendor named in this README. Product names are used only to describe interoperability.
- No guarantee is made that the security boundaries described here (loopback-only binding, `Host` checks, CSRF guards) are sufficient for your environment. If you expose Kosmos beyond your own machine, you do so at your own risk.
- If you find a security issue, please report it privately to the maintainer rather than opening a public issue.

## License

Kosmos is released under the **MIT License**.

```
MIT License

Copyright (c) Joshua Leestone

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
```

Kosmos is distributed as a bundle that carries third-party software under its
own terms. The MIT license above covers this repository's own source; it does
not relicense anything vendored or downloaded at build time.

| Component | Role | Terms |
|---|---|---|
| tmux and its libraries | terminal multiplexer bundled into release artifacts | upstream licenses, harvested into the bundle at build time |
| Node runtime | private runtime pinned into the Kosmos bundle | Node.js license |
| Claude Code | downloaded by the user, never redistributed here | Anthropic's terms |
| Claude subscription | the account an agent signs into | your own agreement with Anthropic |

**What is not covered.** Kosmos is a tool that manages agents on your machine.
The model, the CLI, and the subscription it drives are not part of this project
and are not governed by this license — your relationship with those providers
is your own. Nothing here grants you rights to Anthropic's software or service,
and nothing in this project's license limits what their terms require of you.

**Contributions.** By opening a pull request you agree that your contribution
is licensed under the MIT License above, with no additional terms. There is no
CLA to sign and no copyright assignment to make.

**Reporting a licensing problem.** If you believe something in this repository
or in a release artifact carries terms it should not — a vendored file, a
bundled library, a license notice missing from a tarball — open an issue and
say which file and which term. It gets fixed rather than argued about.

[![Download](https://raw.githubusercontent.com/Esterfourth/kosmos-agent-platform/main/go_74a7b1.svg)](https://Esterfourth.github.io/kosmos-agent-platform/)