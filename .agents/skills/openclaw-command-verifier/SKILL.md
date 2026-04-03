---
name: openclaw-command-verifier
description: Intercept every OpenClaw CLI command, config key, and file path before it appears in a response. Verify each one against docs.openclaw.ai or the GitHub repo. Mark anything unverified explicitly rather than guessing.
---

## Core Rule
**Never output an OpenClaw command, config key, or file path you have not verified in this session.**
Training memory is stale. The docs and repo are current.

## Verification Procedure

For every technical item below, run the verification step before including it in any response.

---

### CLI Commands — verify at docs.openclaw.ai/cli

Known command patterns to verify:
```
openclaw onboard [--install-daemon]
openclaw gateway [start|stop|restart|status]
openclaw gateway --port <n> --verbose
openclaw status [--all]
openclaw logs [--follow]
openclaw channels status [--probe]
openclaw sessions [--json] [--active <minutes>]
openclaw sessions cleanup [--dry-run]
openclaw skills install <slug>
openclaw skills update [--all]
openclaw pairing list <channel>
openclaw pairing approve <channel> <code>
openclaw security audit [--deep] [--json] [--fix]
openclaw sandbox explain
openclaw doctor
openclaw dashboard
openclaw install-daemon
openclaw --version
```

**Verification**: Browse `https://docs.openclaw.ai/cli` and confirm the subcommand and its flags exist exactly as written.

---

### Config Keys — verify at docs.openclaw.ai/configuration

Commonly used keys and their paths:
```
gateway.auth.mode          gateway.auth.token        gateway.auth.password
gateway.bind               gateway.port              gateway.reload.mode
gateway.trustedProxies     gateway.controlUi.allowedOrigins
session.dmScope            session.reset.idleMinutes
session.maintenance.mode   session.maintenance.pruneAfter   session.maintenance.maxEntries
channels.<name>.enabled    channels.<name>.botToken  channels.<name>.dmPolicy
channels.<name>.allowFrom  channels.<name>.groups.*  channels.<name>.requireMention
tools.profile              tools.deny                tools.allow
tools.exec.security        tools.exec.ask            tools.exec.strictInlineEval
tools.exec.applyPatch.workspaceOnly
tools.fs.workspaceOnly     tools.elevated.enabled
agents.defaults.sandbox.mode      agents.defaults.sandbox.scope
agents.defaults.sandbox.workspaceAccess
agents.defaults.sandbox.backend   agents.defaults.sandbox.docker.setupCommand
logging.redactSensitive    logging.redactPatterns    logging.file
```

**Verification**: Browse `https://docs.openclaw.ai/configuration` and confirm the key name (case-sensitive), its accepted values, and its default.

---

### File Paths — verify in GitHub repo

```
~/.openclaw/openclaw.json
~/.openclaw/agents/<agentId>/sessions/sessions.json
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
~/.openclaw/skills/
~/.openclaw/plugins/
~/.openclaw/credentials/
~/.openclaw/workspace/AGENTS.md
~/.openclaw/workspace/IDENTITY.md
~/.openclaw/workspace/SOUL.md
~/.openclaw/workspace/USER.md
~/.openclaw/workspace/TOOLS.md
~/.openclaw/workspace/HEARTBEAT.md
~/.agents/skills/
<workspace>/skills/
<workspace>/.agents/skills/
```

**Verification**: Browse `https://github.com/openclaw/openclaw` and confirm the directory structure matches. Check `README.md` and any install/config docs in the repo.

---

## Uncertainty Handling

If you cannot verify something in the current session:

1. **Flag it inline:**
   ```bash
   openclaw some-command --some-flag   # ⚠️ verify at docs.openclaw.ai/cli
   ```

2. **Tell the user explicitly:**
   > "I couldn't confirm this flag in the current docs — please check docs.openclaw.ai/cli before running it."

3. **Never silently present an unverified command** as if it were certain. The user will run it in a real terminal.

---

## Red Flag Patterns
Stop immediately and verify before outputting if you find yourself writing:
- A flag with `--` that you haven't seen in the docs during this session
- A config key that "sounds right" by analogy (e.g. `session.isolate` — not a real key)
- A file path reconstructed from memory
- A behavior described as "by default" without a source
