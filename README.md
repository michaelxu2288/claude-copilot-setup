# claude-copilot-setup — bare-metal hand-off runbook

Turn a **completely fresh Mac** into **Claude Code running Opus 4.8**, via a local
**LiteLLM proxy → GitHub Copilot**. Assumes the machine has **nothing** — no Homebrew,
Python, Docker, Node, LiteLLM, or Claude Code.

Allows **any machine** to have access to Opus 4.8/GPT 5.5 max effort thinking with **unlimited** tokens/usage, through Claude Code (Anthropic models only in CC)

The runbook is in two parts, easiest-first:

- **PART 1 — get Opus 4.8 working in the CLI (do this first).** The fastest route to a
  green end-to-end test, using the native-Python proxy (no Docker to install yet). Stop
  here once `claude` answers on Opus 4.8.
- **PART 2 — make it permanent + add the extras (only after Part 1 works).** Switch the
  proxy to **Docker with auto-restart** (mirrors the main machine — always-on, nothing to
  run each time), then add the context-bar status line, the `/artifact` skill, and
  optional remote/phone access.

> **The only manual human step:** paste one token into `.env`.

---

## The one manual step (do this first)

Get your Copilot access-token from a machine that already works:
```bash
cat ~/.config/litellm/github_copilot/access-token   # a ghu_... string (40 chars)
```
On the new Mac, in the repo, put it in `.env` (raw — no quotes, no trailing spaces):
```bash
cd claude-copilot-setup
cp .env.example .env
# edit .env:  COPILOT_ACCESS_TOKEN=ghu_xxxxxxxx...
```

---

# PART 1 — get Opus 4.8 running in the CLI (easiest)

**Goal:** `claude` answers on Opus 4.8, end to end. This is the milestone — everything in
Part 2 is optional polish you do afterward.

> **Fast path:** `./install.sh` does all of Part 1 automatically (native-Python runtime).
> The manual steps below are the same thing, one command at a time, so an agent can run
> and debug them individually. Run them **in one terminal** so PATH changes persist.

### 1. Homebrew (the package manager)
```bash
command -v brew >/dev/null || NONINTERACTIVE=1 /bin/bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/opt/homebrew/bin/brew shellenv)"
grep -qs 'brew shellenv' ~/.zprofile || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```
**Verify:** `brew --version`. **If it fails:** run the `eval` line, retry.

### 2. The proxy (Python + LiteLLM — easiest, no Docker)
```bash
python3 -c 'import sys' >/dev/null 2>&1 || brew install python
command -v pipx >/dev/null || brew install pipx
pipx ensurepath; export PATH="$HOME/.local/bin:$PATH"
pipx install "litellm[proxy]"
command -v jq >/dev/null || brew install jq
```
**Verify:** `~/.local/bin/litellm --version`. **If it fails:** run `export PATH="$HOME/.local/bin:$PATH"`.

### 3. Claude Code
```bash
command -v node >/dev/null || brew install node
command -v claude >/dev/null || npm install -g @anthropic-ai/claude-code
```
**Verify:** `claude --version`. **If it fails:** confirm `which node` is under `/opt/homebrew`; or use `curl -fsSL https://claude.ai/install.sh | bash`.

### 4. Gateway config + local key
```bash
mkdir -p ~/.config/litellm
cp litellm/config.yaml ~/.config/litellm/config.yaml
cp litellm/gateway.sh  ~/.config/litellm/gateway.sh && chmod +x ~/.config/litellm/gateway.sh
[ -s ~/.config/litellm/.master_key ] || ( umask 077; printf 'sk-%s' "$(openssl rand -hex 24)" > ~/.config/litellm/.master_key )
KEY="$(cat ~/.config/litellm/.master_key)"
```
**Verify:** `grep disable_copilot_system_to_assistant ~/.config/litellm/config.yaml` prints the line.

### 5. Seed your Copilot token (this is what skips the GitHub login)
```bash
set -a; . ./.env; set +a
TOKEN="$COPILOT_ACCESS_TOKEN"
TOKEN="${TOKEN//$'\r'/}"; TOKEN="${TOKEN//\"/}"; TOKEN="${TOKEN//\'/}"; TOKEN="${TOKEN// /}"
mkdir -p ~/.config/litellm/github_copilot
printf '%s' "$TOKEN" > ~/.config/litellm/github_copilot/access-token
chmod 600 ~/.config/litellm/github_copilot/access-token
```
**Verify:** `head -c4 ~/.config/litellm/github_copilot/access-token` prints `ghu_` (or `gho_`).

### 6. Start the proxy
```bash
printf python > ~/.config/litellm/.runtime
~/.config/litellm/gateway.sh start
```
**Verify:** `~/.config/litellm/gateway.sh status` shows `health=200`.
**If it fails:** `tail ~/.config/litellm/proxy.log` — a 401/403 means the seeded token is stale (redo Step 5).

### 7. Minimal Claude Code settings (Opus 4.8 recognition + 1M context)
```bash
mkdir -p ~/.claude
python3 - <<'PY'
import json, os, time
tmpl = json.load(open("claude/settings.template.json"))
key  = open(os.path.expanduser("~/.config/litellm/.master_key")).read().strip()
dst  = os.path.expanduser("~/.claude/settings.json")
cur = {}
if os.path.exists(dst):
    raw = open(dst).read(); open(dst + ".bak." + str(int(time.time())), "w").write(raw)
    try: cur = json.loads(raw)
    except Exception: cur = {}
if not isinstance(cur, dict): cur = {}
env = {k: (key if v == "__MASTER_KEY__" else v) for k, v in tmpl.get("env", {}).items()}
if not isinstance(cur.get("env"), dict): cur["env"] = {}
cur["env"].update(env)                       # PART 1: env only — enough for Opus 4.8 in the CLI
json.dump(cur, open(dst, "w"), indent=2); print("wrote env (Opus 4.8 recognition + 1M context)")
PY
```
(Top-level settings — status line, theme, effort — come in Part 2B.)

### 8. ✅ END-TO-END TEST
```bash
curl -s -m 90 http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-4-8[1m]","messages":[{"role":"user","content":"reply with the single word: ready"}],"max_tokens":16}' | grep -qi ready && echo "PROXY OK" || echo "PROXY FAILED"
claude -p "reply with the single word: ready" 2>/dev/null | grep -qi ready && echo "CLAUDE OK" || echo "CLAUDE FAILED"
```
**Success = `PROXY OK` and `CLAUDE OK`.** Optional human check: run `claude`, then `/status`
→ base URL `http://localhost:4000`, model **Opus 4.8 1M**.

**🎉 Opus 4.8 works in the CLI. If you only wanted it working, you're done.** Part 2 makes
it permanent and adds the extras.

---

# PART 2 — make it permanent + add the extras (after Part 1 works)

## A. Always-on proxy via Docker (mirrors the main machine)

Part 1's Python proxy stops if you log out. The main machine runs the proxy as a **Docker
container with `--restart unless-stopped`**, so it auto-restarts on crash and boot — you
never run anything by hand. Switch to that (it reuses the same config + token from Part 1):

```bash
~/.config/litellm/gateway.sh stop                 # stop the Python proxy
brew install --cask docker && open -ga Docker      # Docker Desktop (or headless: brew install colima docker && colima start)
until docker info >/dev/null 2>&1; do sleep 2; done
printf docker > ~/.config/litellm/.runtime         # tell gateway.sh to use Docker
~/.config/litellm/gateway.sh start                 # runs the litellm container, --restart unless-stopped, bound 127.0.0.1:4000
```
Then set **Docker Desktop → Settings → General → "Start Docker Desktop when you sign in"**
so the engine (and the auto-restart container) come back after a reboot.
**Verify:** `docker ps` shows `litellm` `Up`; `~/.config/litellm/gateway.sh status` → `health=200`.

**Lighter alternative (no Docker):** keep the Python proxy and add a login LaunchAgent so it
restarts at boot — run `./install.sh` (which installs the LaunchAgent) or see `install.sh`
Phase 10 for the plist.

## B. The context-bar status line + full settings

The built-in context bar can't show a % through the proxy (the proxy reports
`max_input_tokens=null`); this script reads real usage from the transcript ÷ the true 1M
window, so `/context` and the bar show the correct `…/1000000`.
```bash
cp claude/statusline-command.sh ~/.claude/statusline-command.sh && chmod +x ~/.claude/statusline-command.sh
# now write the FULL settings (adds statusLine, theme, effortLevel, permissions, ...):
python3 - <<'PY'
import json, os, time
tmpl = json.load(open("claude/settings.template.json"))
key  = open(os.path.expanduser("~/.config/litellm/.master_key")).read().strip()
dst  = os.path.expanduser("~/.claude/settings.json")
cur = {}
if os.path.exists(dst):
    raw = open(dst).read(); open(dst + ".bak." + str(int(time.time())), "w").write(raw)
    try: cur = json.loads(raw)
    except Exception: cur = {}
if not isinstance(cur, dict): cur = {}
env = {k: (key if v == "__MASTER_KEY__" else v) for k, v in tmpl.get("env", {}).items()}
if not isinstance(cur.get("env"), dict): cur["env"] = {}
cur["env"].update(env)
for k, v in tmpl.items():
    if k != "env": cur[k] = v                # top-level: model, statusLine, effortLevel, theme, permissions, ...
json.dump(cur, open(dst, "w"), indent=2); print("wrote full settings")
PY
```
**Verify:** run `claude`; the status line shows a `Ctx …/1000000` bar.

## C. The `/artifact` skill
Local HTML viewer (rendered diffs/code/docs) served to a phone over Tailscale — the
vscode.dev replacement.
```bash
mkdir -p ~/.claude/skills && rm -rf ~/.claude/skills/artifact
cp -R skills/artifact ~/.claude/skills/artifact
```
**Verify:** `ls ~/.claude/skills/artifact/SKILL.md`. Usage is in `skills/artifact/SKILL.md`.

## D. Remote / phone access (Tailscale + tmux) — optional
Drive Claude Code from your phone over Tailscale.
```bash
cp remote/claude-remote.zsh ~/.claude-remote.zsh
grep -qs 'claude-remote.zsh' ~/.zshrc || echo '[ -f "$HOME/.claude-remote.zsh" ] && source "$HOME/.claude-remote.zsh"' >> ~/.zshrc
mkdir -p ~/bin && cp remote/setup-remote.sh ~/bin/setup-remote.sh && chmod +x ~/bin/setup-remote.sh
~/bin/setup-remote.sh            # installs SSH + tmux + mosh + Tailscale + always-on power (prompts for sudo + a one-time Tailscale login)
```
Then from the phone: `mosh <mac> -- tmux attach`. Helpers: `llm` (restart gateway),
`ccnew <name>` (new agent), `wake` (keep awake).

## E. Docs-wiki harness (per-project LLM knowledge base) — optional

An opt-in, per-project knowledge base an LLM builds and maintains (Karpathy model:
raw sources → compiled, cross-linked `.md` articles). Two subagents + one hook + three
slash commands. The switch is a `docs-wiki/` folder in a project root — no folder = the
agents and hook stay 100% silent.

```bash
mkdir -p ~/.claude/agents ~/.claude/commands ~/.claude/hooks
cp claude/agents/docs-scout.md claude/agents/docs-writer.md ~/.claude/agents/
cp claude/commands/docs-init.md claude/commands/docs-sync.md claude/commands/docs-ask.md ~/.claude/commands/
cp claude/hooks/docs-scout-nudge.sh ~/.claude/hooks/ && chmod +x ~/.claude/hooks/docs-scout-nudge.sh
# register the UserPromptSubmit nudge hook (idempotent)
python3 - <<'PY'
import json, os
p = os.path.expanduser("~/.claude/settings.json"); d = json.load(open(p)) if os.path.exists(p) else {}
arr = d.setdefault("hooks", {}).setdefault("UserPromptSubmit", [])
cmd = "~/.claude/hooks/docs-scout-nudge.sh"
if not any(cmd in json.dumps(x) for x in arr):
    arr.append({"hooks": [{"type": "command", "command": cmd, "timeout": 15}]}); print("registered docs-scout-nudge hook")
else: print("hook already registered")
json.dump(d, open(p, "w"), indent=2)
PY
```
**Verify:** `ls ~/.claude/agents/docs-scout.md ~/.claude/commands/docs-init.md`.

**How to use it:**
- **`docs-scout`** — read-only librarian (Read/Grep/Glob only); deep-reads `docs-wiki/` in
  its own context and returns a compact, cited synthesis.
- **`docs-writer`** — scribe; compiles durable session knowledge into `docs-wiki/` only
  (never commits, never deletes).
- **`docs-scout-nudge.sh`** — on a substantial prompt *in an opted-in project*, nudges the
  main agent to consult docs-scout first. Silent everywhere else.
- **Lifecycle:** in a project, `/docs-init` scaffolds `docs-wiki/{raw,wiki,images,.backups}`
  + a README index and appends usage lines to that project's `./CLAUDE.md`. Drop sources in
  `docs-wiki/raw/`; do work; run **`/docs-sync`** to fork several `docs-writer` subagents in
  parallel that compile `raw/`→`wiki/` with `[[wikilinks]]`; query anytime with
  **`/docs-ask "question"`**. The folder is a valid Obsidian vault.

## F. Optional extra hooks
```bash
cp claude/hooks/format-python.sh ~/.claude/hooks/ && chmod +x ~/.claude/hooks/format-python.sh
cp claude/hooks/artifact-on-stop.py ~/.claude/hooks/
python3 - <<'PY'
import json, os
p = os.path.expanduser("~/.claude/settings.json"); d = json.load(open(p)) if os.path.exists(p) else {}
h = d.setdefault("hooks", {})
pt = h.setdefault("PostToolUse", [])
if not any("format-python.sh" in json.dumps(x) for x in pt):
    pt.append({"matcher": "Edit|Write|MultiEdit", "hooks": [{"type": "command", "command": "~/.claude/hooks/format-python.sh", "timeout": 60}]})
st = h.setdefault("Stop", [])
if not any("artifact-on-stop.py" in json.dumps(x) for x in st):
    st.append({"hooks": [{"type": "command", "command": "python3 ~/.claude/hooks/artifact-on-stop.py", "timeout": 15}]})
json.dump(d, open(p, "w"), indent=2); print("registered optional hooks")
PY
```
- **format-python.sh** — auto-formats Python after edits. Dormant until `brew install ruff`.
- **artifact-on-stop.py** — auto-publishes an artifact on Stop (best-effort; pairs with the
  `/artifact` skill from C; only for tmux-run agents).

---

## Reference

**What each file is**
```
litellm/config.yaml            Claude via github_copilot; the critical disable_copilot_system_to_assistant flag
litellm/gateway.sh             start/stop/status the proxy (python or docker, per ~/.config/litellm/.runtime)
claude/settings.template.json  Opus 4.8 1M recognition (env) + top-level settings (model, statusLine, effort, theme, permissions)
claude/statusline-command.sh   proxy-aware context bar (Part 2B)
claude/agents/                 docs-scout + docs-writer subagents (Part 2E)
claude/commands/               /docs-init /docs-sync /docs-ask (Part 2E)
claude/hooks/                  docs-scout-nudge (2E) + optional format-python / artifact-on-stop (2F)
skills/artifact/               the /artifact phone-viewer skill (Part 2C)
.env.example                   -> copy to .env, paste your token (the one manual step)
install.sh                     optional one-shot for PART 1 (native-Python runtime)
uninstall.sh                   stop proxy + LaunchAgent + restore settings backup (--purge for full removal)
remote/                        SSH/Tailscale/tmux helpers (Part 2D)
```

**Troubleshooting**
| Symptom | Fix |
| --- | --- |
| `brew: command not found` | `eval "$(/opt/homebrew/bin/brew shellenv)"`, retry |
| `litellm`/`claude` not found | `export PATH="$HOME/.local/bin:$PATH"` |
| proxy health `000` | `tail ~/.config/litellm/proxy.log`; `~/.config/litellm/gateway.sh restart` |
| auth/401 in the log | seeded token stale/revoked → redo Part 1 Step 5 with a fresh token |
| model denied (not 401) | your Copilot org must have Claude models enabled for your seat |
| token alone won't auth | copy the whole `~/.config/litellm/github_copilot/` folder from a working Mac, then `gateway.sh restart` |
| Claude weakly follows instructions | confirm `disable_copilot_system_to_assistant: true` in the config, `gateway.sh restart` |
| Docker `litellm` not up after reboot | enable Docker Desktop "start at login"; the container is `--restart unless-stopped` |

**Notes**
- `.env` is git-ignored — your token never gets committed and never leaves the machine.
- The proxy binds `127.0.0.1` only (both runtimes). Your own token, your own Mac.
