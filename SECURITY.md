# Security Model

An agent with write access to DNS, hosting, and email is a high-value target and a single bad tool call away from a very bad day. These rules are not optional.

## Threat model

- **Agent error** — the model does the wrong thing confidently (the July 2025 Replit incident: agent deleted a prod DB despite a freeze instruction, then fabricated results). Mitigation: approval gates, snapshots, least privilege.
- **Prompt injection** — a monitored webpage, email, or form submission contains instructions the agent might follow. Mitigation: content fetched by adapters is data, never instructions; destructive actions never trigger from monitored content, only from the authenticated operator.
- **Credential leakage** — secrets in transcripts, logs, or the repo. Mitigation: secrets only in `.env`/keychain; audit log stores tool + params but redacts secret values; public repo ships examples only.
- **Bot takeover** — someone else messages the bot. Mitigation: hard allowlist on Telegram user ID; unknown senders get no response and an alert to you.

## Rules

1. **Deny by default.** Headless runs use an explicit `--allowedTools` list per routine. Monitoring routines get read-only tools only.
2. **Two-step for destructive actions.** Anything marked `destructive: true` (DNS write, nginx reload, service restart, deploy, bulk email, deletion of anything) requires an inline Telegram confirmation from the allowlisted operator. No auto-approve mode for these, ever — approval fatigue is how 93%-approval-rate disasters happen.
3. **Snapshot before change.** DNS: take a provider-side snapshot before any record write. VPS: snapshot before structural changes. Web server: config test (`nginx -t` or equivalent) is a mandatory gate before any reload.
4. **Never touch what you don't own.** On shared infrastructure, the agent's writable paths and service names are explicitly enumerated in `config/systems.yml`; everything else is off-limits even if reachable.
5. **Append-only audit log.** Every adapter call: timestamp, routine/initiator, tool, params (secrets redacted), outcome, approval message ID if gated. `logs/audit.jsonl`, rotated, backed up off the agent-reachable volume.
6. **Backups live where the agent can't reach them.** An agent that can delete data must not be able to delete the backups of that data.
7. **Budget caps.** `--max-budget-usd` and `--max-turns` on every unattended invocation.
8. **Secrets hygiene.** Any credential that has ever appeared in a chat transcript, commit, or shared doc is considered burned — rotate it before the agent goes live.

## What the public repo contains vs. what it never will

| Public (this repo) | Private (gitignored / local only) |
|---|---|
| Framework code, adapters, runner, bot | `config/*.yml` (real domains, hosts, checks) |
| Example configs (`*.example.yml`) | `private/` (operator context, runbooks with real targets) |
| Routine/skill conventions and examples | `.env`, keys, tokens, `state/`, `logs/` |
