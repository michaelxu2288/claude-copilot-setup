# claude-copilot-setup — bare-metal hand-off runbook

Turn a **completely fresh Mac into a working Claude Code running Opus 4.8**, where the
model runs through a local **LiteLLM proxy → GitHub Copilot**.

**This assumes the machine has NOTHING** — no Homebrew, no Python packages, no Docker,
no Node, no LiteLLM, and **no Claude Code**. Every step below installs what it needs
from scratch. Model auth is done by **seeding one token** (no GitHub OAuth/2FA login).

> **The only manual human step:** paste one token into `.env` (or one file). Everything
> else the agent runs, one command at a time.

---

## Two ways to run this

- **Manual (recommended, debuggable):** follow the numbered steps below. Each is one
  command with a **verify** and a **if it fails** note, so an agent can run them one at
  a time and fix problems as they arise. This is the source of truth.
- **Fast (optional):** `./install.sh` runs every step below automatically. Use it only
  if you want a one-shot; if anything breaks, fall back to the manual steps.

---

## The one manual step (do this first)

Get your Copilot access-token from a machine that already works:
```bash
cat ~/.config/litellm/github_copilot/access-token   # a string starting with ghu_ (or gho_)
```
On the new Mac, put it in `.env` (raw — no quotes, no trailing spaces):
```bash
cd claude-copilot-setup
cp .env.example .env
# edit .env: COPILOT_ACCESS_TOKEN=ghu_xxxxxxxx...
```
Then run the steps. (If you can't edit files easily yet, Step 8 shows how to paste the
token directly instead of using `.env`.)

---

# AGENT RUNBOOK — fully explicit, from a bare Mac

You are an AI agent on a fresh Mac. Run each step in order **in the same terminal**
(so PATH changes persist). After each step, run its **Verify**; do not continue until it
passes. If it fails, do the **If it fails** note, then retry.

---

## Phase A — system prerequisites

### A1. Xcode Command Line Tools (gives you `git`, compilers)
```bash
xcode-select -p >/dev/null 2>&1 || xcode-select --install
```
**Verify:** `xcode-select -p` prints a path.
**If it fails:** a GUI dialog may open — the human clicks "Install" and waits. Homebrew
(next) also triggers this; you can proceed to A2 and let it handle CLT.

### A2. Homebrew (the package manager everything else uses)
```bash
command -v brew >/dev/null || NONINTERACTIVE=1 /bin/bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/opt/homebrew/bin/brew shellenv)"
grep -qs 'brew shellenv' ~/.zprofile || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```
**Verify:** `brew --version` prints a version.
**If it fails:** if `brew: command not found` after install, you skipped the `eval` line —
run `eval "$(/opt/homebrew/bin/brew shellenv)"` and retry. On Apple Silicon brew lives at
`/opt/homebrew`; on old Intel Macs it's `/usr/local` (use `eval "$(/usr/local/bin/brew shellenv)"`).

---

## Phase B — runtime + tools

### B1. Python 3 (runs the proxy and the config steps)
```bash
python3 -c 'import sys' >/dev/null 2>&1 || brew install python
```
**Verify:** `python3 --version` prints 3.x.
**If it fails:** the stock `/usr/bin/python3` can be a stub that pops a CLT prompt —
`brew install python` gives a real one. Ensure `/opt/homebrew/bin` is on PATH (A2).

### B2. pipx + LiteLLM (the proxy itself)
```bash
command -v pipx >/dev/null || brew install pipx
pipx ensurepath
export PATH="$HOME/.local/bin:$PATH"
pipx install "litellm[proxy]"
```
**Verify:** `~/.local/bin/litellm --version` prints a version.
**If it fails:** if `litellm: command not found`, `~/.local/bin` isn't on PATH — run the
`export PATH=...` line. If pipx errors mid-install, `pipx uninstall litellm` then retry.

### B3. jq (used by the status line)
```bash
command -v jq >/dev/null || brew install jq
```
**Verify:** `jq --version` prints a version. **If it fails:** not fatal — the status line
degrades gracefully without jq, but install it if you can.

---

## Phase C — install Claude Code

### C1. Node.js (Claude Code ships via npm)
```bash
command -v node >/dev/null || brew install node
```
**Verify:** `node --version` and `npm --version` both print versions.

### C2. Claude Code
```bash
command -v claude >/dev/null || npm install -g @anthropic-ai/claude-code
```
**Verify:** `claude --version` prints a version.
**If it fails:** if `npm i -g` hits EACCES/permission errors, it's trying to write to a
non-writable prefix — with Homebrew Node the prefix is writable, so confirm you're using
brew's node (`which node` → `/opt/homebrew/...`). Alternative installer:
`curl -fsSL https://claude.ai/install.sh | bash` (puts `claude` in `~/.local/bin`).

---

## Phase D — the LiteLLM gateway (Claude via Copilot)

You are inside the cloned `claude-copilot-setup` repo for these `cp` commands.

### D1. Write the gateway config
```bash
mkdir -p ~/.config/litellm
cp litellm/config.yaml   ~/.config/litellm/config.yaml
cp litellm/gateway.sh    ~/.config/litellm/gateway.sh && chmod +x ~/.config/litellm/gateway.sh
```
**Verify:** `grep disable_copilot_system_to_assistant ~/.config/litellm/config.yaml`
prints the line (this is the critical flag).

### D2. Generate the local gateway key (guards only `127.0.0.1:4000`; not a real secret)
```bash
[ -s ~/.config/litellm/.master_key ] || ( umask 077; printf 'sk-%s' "$(openssl rand -hex 24)" > ~/.config/litellm/.master_key )
KEY="$(cat ~/.config/litellm/.master_key)"
```
**Verify:** `echo "${KEY:0:3}"` prints `sk-`.

### D3. Seed your Copilot token (this is what skips the GitHub login)
From `.env`:
```bash
set -a; . ./.env; set +a
TOKEN="$COPILOT_ACCESS_TOKEN"
TOKEN="${TOKEN//$'\r'/}"; TOKEN="${TOKEN//\"/}"; TOKEN="${TOKEN//\'/}"; TOKEN="${TOKEN// /}"  # strip CR/quotes/spaces
mkdir -p ~/.config/litellm/github_copilot
printf '%s' "$TOKEN" > ~/.config/litellm/github_copilot/access-token
chmod 600 ~/.config/litellm/github_copilot/access-token
```
(No `.env`? Paste directly instead: replace the `TOKEN=...` line with
`TOKEN="ghu_your_token_here"`.)
**Verify:** `head -c4 ~/.config/litellm/github_copilot/access-token` prints `ghu_` (or `gho_`).
**If it fails:** wrong prefix means you pasted a quote/space or the wrong string — re-copy
the raw token. LiteLLM regenerates the short-lived `api-key.json` from this automatically.

### D4. Record runtime = python and start the proxy
```bash
printf 'python' > ~/.config/litellm/.runtime
~/.config/litellm/gateway.sh start
```
**Verify:** `~/.config/litellm/gateway.sh status` shows `health=200`.
**If it fails:** check `tail ~/.config/litellm/proxy.log`. A 401/403 there = the seeded
token is stale/revoked (get a fresh one, redo D3). Port already in use = a proxy is
already running (`~/.config/litellm/gateway.sh restart`).

### D5. Confirm the model actually answers
```bash
curl -s -m 90 http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"model":"claude-opus-4-8[1m]","messages":[{"role":"user","content":"reply with the single word: ready"}],"max_tokens":16}' | grep -qi ready && echo "MODEL OK" || echo "MODEL FAILED"
```
**Verify:** `MODEL OK`.
**If it fails:** `MODEL FAILED` with an auth error = stale token (D3). With a
model-access error = your Copilot org hasn't enabled Claude models for your seat.

---

## Phase E — configure Claude Code

### E1. Install the status-line script (proxy-aware context bar)
```bash
mkdir -p ~/.claude
cp claude/statusline-command.sh ~/.claude/statusline-command.sh && chmod +x ~/.claude/statusline-command.sh
```
Why: the built-in context bar can't show a % through the proxy (the proxy reports
`max_input_tokens=null`); this script reads real usage from the transcript ÷ the true 1M
window, so `/context` and the bar show the correct 1,000,000 window.

### E2. Write `~/.claude/settings.json` (Opus 4.8 1M + your top-level settings)
```bash
python3 - <<'PY'
import json, os, time
tmpl = json.load(open("claude/settings.template.json"))
key  = open(os.path.expanduser("~/.config/litellm/.master_key")).read().strip()
dst  = os.path.expanduser("~/.claude/settings.json"); os.makedirs(os.path.dirname(dst), exist_ok=True)
cur = {}
if os.path.exists(dst):
    raw = open(dst).read(); open(dst + ".bak." + str(int(time.time())), "w").write(raw)  # back up RAW
    try: cur = json.loads(raw)
    except Exception: cur = {}
if not isinstance(cur, dict): cur = {}                       # existing file could be a list/scalar
env = {k: (key if v == "__MASTER_KEY__" else v) for k, v in tmpl.get("env", {}).items()}
if not isinstance(cur.get("env"), dict): cur["env"] = {}
cur["env"].update(env)                                        # env: ours win, keep user's others
for k, v in tmpl.items():                                   # top-level keys (model, statusLine, ...)
    if k != "env": cur[k] = v
json.dump(cur, open(dst, "w"), indent=2); print("wrote", dst)
PY
```
**Verify:**
```bash
python3 -c "import json;d=json.load(open('$HOME/.claude/settings.json'));print(d['env']['ANTHROPIC_BASE_URL'], d['env']['CLAUDE_CODE_MAX_CONTEXT_TOKENS'], d.get('model'))"
```
prints `http://localhost:4000 1000000 claude-opus-4-8[1m]`.
**If it fails:** if it errors on an existing settings.json, the file was invalid JSON —
the script backed it up as `settings.json.bak.*` and started fresh; that's fine.

### E3. Install the `/artifact` skill (optional but part of the setup)
```bash
mkdir -p ~/.claude/skills && rm -rf ~/.claude/skills/artifact
cp -R skills/artifact ~/.claude/skills/artifact
```
**Verify:** `ls ~/.claude/skills/artifact/SKILL.md`.

---

## Phase F — persistence (proxy restarts after reboot)

```bash
PLIST="$HOME/Library/LaunchAgents/com.local.litellm-gateway.plist"
mkdir -p "$HOME/Library/LaunchAgents"
cat > "$PLIST" <<PLISTEOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.local.litellm-gateway</string>
  <key>ProgramArguments</key><array><string>/bin/bash</string><string>$HOME/.config/litellm/gateway.sh</string><string>start</string></array>
  <key>EnvironmentVariables</key><dict>
    <key>PATH</key><string>$HOME/.local/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    <key>HOME</key><string>$HOME</string></dict>
  <key>RunAtLoad</key><true/>
  <key>StandardOutPath</key><string>$HOME/.config/litellm/launchagent.log</string>
  <key>StandardErrorPath</key><string>$HOME/.config/litellm/launchagent.log</string>
</dict></plist>
PLISTEOF
launchctl unload "$PLIST" 2>/dev/null; launchctl load -w "$PLIST"
```
**Verify:** `launchctl list | grep litellm-gateway` shows the label.
**If it fails:** not fatal now (the proxy is already running from D4); after a reboot just
run `~/.config/litellm/gateway.sh start`.

---

## Phase G — final check

```bash
claude --version
claude -p "reply with the single word: ready" 2>/dev/null | grep -qi ready && echo "CLAUDE OK" || echo "CLAUDE FAILED"
```
**Verify:** `CLAUDE OK`.
**Optional human check:** run `claude`, then `/status` → base URL `http://localhost:4000`,
model **Opus 4.8 1M**; the status line shows `Ctx …/1000000`.

**Done.** Report: what got installed, `gateway.sh status`, and that `claude -p` answered.

---

## What each file is
```
litellm/config.yaml            Claude via github_copilot; the critical disable_copilot_system_to_assistant flag
litellm/gateway.sh             start/stop/status the proxy (python or docker)
claude/settings.template.json  Opus 4.8 1M recognition + your top-level settings (model, statusLine, effort, theme, permissions)
claude/statusline-command.sh   proxy-aware context bar (installed to ~/.claude/)
skills/artifact/               the /artifact phone-viewer skill
.env.example                   -> copy to .env, paste your token (the one manual step)
install.sh                     optional one-shot that runs every step above
uninstall.sh                   stop proxy + LaunchAgent + restore settings backup
remote/                        optional SSH/Tailscale/tmux helpers
```

## Troubleshooting quick table
| Symptom | Fix |
| --- | --- |
| `brew: command not found` | `eval "$(/opt/homebrew/bin/brew shellenv)"`, retry |
| `litellm`/`claude` not found | `export PATH="$HOME/.local/bin:$PATH"` (pipx/native installer target) |
| health `000` / proxy not up | `tail ~/.config/litellm/proxy.log`; `~/.config/litellm/gateway.sh restart` |
| auth/401 in proxy.log | seeded token stale/revoked → redo D3 with a fresh token |
| model denied (not 401) | org must enable Claude models for your Copilot seat |
| seeded token alone won't auth | copy the whole `~/.config/litellm/github_copilot/` folder from a working Mac, then `gateway.sh restart` |
| Claude weakly follows instructions | confirm `disable_copilot_system_to_assistant: true` in the config, `gateway.sh restart` |

## Notes
- `.env` is git-ignored — your token never gets committed and never leaves the machine.
- The proxy binds `127.0.0.1` only. This is your own token on your own Mac.
- Prefer the native-Python runtime (above). Docker (via Colima) is optional: set
  `printf docker > ~/.config/litellm/.runtime` and let `gateway.sh` manage the container.
