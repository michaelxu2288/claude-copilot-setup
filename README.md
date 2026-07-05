# claude-copilot-setup — hand-off runbook

Turn a **fresh Mac** into a working **Claude Code + Opus 4.8** setup. Model calls run
through a local **LiteLLM proxy → GitHub Copilot**. On a machine that can't do the
GitHub OAuth/2FA login, you seed one token and skip the login entirely.

> **The only manual human step:** paste one token into a file (`.env`). Everything
> else is done by the agent following this README.

---

## TL;DR for the human (10 seconds of work)

```bash
cd claude-copilot-setup
cp .env.example .env
# open .env, paste your Copilot access-token after COPILOT_ACCESS_TOKEN=
# PASTE IT RAW — no quotes, no trailing spaces.
# get it from a working machine:  cat ~/.config/litellm/github_copilot/access-token
```
Then hand this repo to the agent and say: *"follow README.md."*

---

## AGENT RUNBOOK — execute top to bottom on this new Mac

You are an AI agent. Set this machine up autonomously. Run each step, then its **verify**
line; don't continue until it passes. On failure, see **Troubleshooting** and retry.

> Run the whole runbook in ONE shell so PATH changes persist. If you run steps in
> separate shells, re-run the Step 2 `brew shellenv` line at the top of each.

### Step 0 — you're in the repo
```bash
test -f install.sh && test -f litellm/config.yaml && echo "IN REPO" || echo "cd into claude-copilot-setup first"
```

### Step 1 — the token is present (the human's one step)
```bash
test -f .env && grep -qE '^COPILOT_ACCESS_TOKEN=("?)(ghu_|gho_)' .env && echo "TOKEN OK" || echo "TOKEN MISSING"
```
- `TOKEN MISSING` → STOP; tell the human to do the TL;DR step (paste their `ghu_...`
  token into `.env`, raw, no quotes). It's the one thing only they can do.
- `TOKEN OK` → continue.

### Step 2 — Homebrew (install if missing, and put it on PATH persistently)
```bash
command -v brew >/dev/null || NONINTERACTIVE=1 /bin/bash -c \
  "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# put brew on PATH now AND for future shells:
eval "$(/opt/homebrew/bin/brew shellenv)"
grep -qs 'brew shellenv' ~/.zprofile || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```
**Verify:** `brew --version` prints a version. (install.sh also self-heals brew PATH,
but do this so later steps in new shells have it too.)

### Step 3 — run the installer (does everything else)
```bash
chmod +x install.sh
./install.sh            # add --remote to also install SSH/Tailscale/tmux
echo "exit=$?"
```
It installs the runtime (native Python by default; `--docker` for Colima), writes
`~/.config/litellm/config.yaml`, generates a local gateway key, **seeds your Copilot
token from `.env` (no GitHub login)**, starts the proxy on `127.0.0.1:4000`, installs
Claude Code, writes `~/.claude/settings.json` with your **Opus 4.8 1M** config + your
statusline/effort/theme/permissions settings, installs the `/artifact` skill, and adds
a boot LaunchAgent.

**Verify:** last line is `Opus 4.8 responded — setup complete` **and** `exit=0`.
(The installer exits non-zero if the model didn't actually answer — don't treat mere
completion as success.) Also:
```bash
~/.config/litellm/gateway.sh status                                        # runtime=... health=200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:4000/health/liveliness   # 200
```

### Step 4 — confirm Claude Code answers on Opus 4.8 (non-interactive)
```bash
command -v claude && claude --version
claude -p "reply with the single word: ready" 2>/dev/null | grep -qi ready && echo "CLAUDE OK" || echo "CLAUDE FAILED"
```
**Verify:** `CLAUDE OK`. (Optional human check: run `claude`, then `/status` should show
base URL `http://localhost:4000` and model **Opus 4.8 1M**, and the status line shows a
`Ctx …/1000000` context bar.)

### Step 5 — done
Report: runtime used, `gateway.sh status`, and that `claude -p` answered. Setup complete.

---

## Why no GitHub login is needed (token seeding)
GitHub Copilot's login caches an access-token at
`~/.config/litellm/github_copilot/access-token` (a `ghu_...` string). LiteLLM uses it to
mint short-lived Copilot API keys automatically. Pasting that token into `.env` writes it
straight there, so the proxy authenticates without the device/2FA flow. Same account,
same seat, new machine. (An OAuth device-login path exists in the installer as a
fallback, but it's skipped whenever a token is seeded — which is the point here, since
2FA is blocking the interactive login.)

## What gets configured (your exact setup)
- `litellm/config.yaml` — `claude-opus-4-8[1m]` → `github_copilot/claude-opus-4.8`
  (1M in / 32k out) + `claude-sonnet-5`; `Editor-Version: vscode/1.100.0`; and the
  **critical** `disable_copilot_system_to_assistant: true`.
- `claude/settings.template.json` — the env that makes Claude Code **recognize/display
  Opus 4.8 1M** (`ANTHROPIC_CUSTOM_MODEL_OPTION*`, `CLAUDE_CODE_MAX_CONTEXT_TOKENS=1000000`,
  etc.) PLUS your top-level settings (model, fallbackModel, permissions, effortLevel,
  theme, statusLine, …).
- `claude/statusline-command.sh` — the context-bar that works **through the proxy** (the
  built-in bar can't compute a % because the proxy reports `max_input_tokens=null`; this
  reads real usage from the transcript ÷ the true 1M window).

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `brew: command not found` when running install.sh | Run `eval "$(/opt/homebrew/bin/brew shellenv)"` (or open a new terminal), then re-run. install.sh also self-heals this. |
| `TOKEN MISSING` | Human pastes the `ghu_...` token into `.env` (Step 1), raw, no quotes. |
| `TOKEN OK` but auth fails / not "ready" | Check `.env` for quotes or a Windows CRLF; re-save as LF, no quotes. Or the seeded token is stale/revoked — get a fresh `cat ~/.config/litellm/github_copilot/access-token` and re-run. |
| Auth works but model denied | Your Copilot org must have Claude models enabled for your seat. |
| Seeded token alone won't authenticate | Fallback: copy the whole `~/.config/litellm/github_copilot/` folder (both files) from the working machine to the same path here, then `~/.config/litellm/gateway.sh restart`. |
| Health stays `000` on first Docker run | The `litellm` image is still pulling — wait, then `~/.config/litellm/gateway.sh status`. Or use the default Python runtime (`./install.sh --python`). |
| Claude Code weakly follows instructions | Confirm `disable_copilot_system_to_assistant: true` in `~/.config/litellm/config.yaml`, then `~/.config/litellm/gateway.sh restart`. |
| Gateway dead after reboot | `~/.config/litellm/gateway.sh start` (the LaunchAgent should do this automatically). |

## Repo layout
```
install.sh                     Step 3 — does everything (idempotent, exits non-zero on real failure)
uninstall.sh                   stop proxy + LaunchAgent + restore settings backup (--purge for full removal)
.env.example                   -> copy to .env, paste your Copilot token (the one manual step)
litellm/config.yaml            proxy config: Claude via github_copilot (no secrets)
litellm/gateway.sh             start/stop/status the proxy (docker or python)
claude/settings.template.json  Claude Code env + top-level settings: Opus 4.8 1M
claude/statusline-command.sh   proxy-aware context bar (installed to ~/.claude/)
skills/artifact/               the /artifact phone-viewer skill
remote/                        optional SSH/Tailscale/tmux helpers (install.sh --remote)
```

## Notes
- `.env` is git-ignored — the token never gets committed.
- The gateway binds `127.0.0.1` only. This is your own token on your own machine.
