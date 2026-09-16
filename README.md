<p align="center">
  <img src="assets/pakt-cyberpunk-ascii-banner-no-text.png" alt="An AI agent faces Pakt's orange verification gate" width="100%">
</p>

# Pakt for AI agents

Pakt gives AI agents trading authority within limits you approve, without
giving the agent an unrestricted wallet key. The agent proposes an exact
action; the wallet signs it only when a zero-knowledge proof shows that it
follows your active rules. Pakt authorizes requests—it never submits a trade.

> [!IMPORTANT]
> Pakt is currently in closed beta. [Join the waitlist](https://usepakt.ai/onboarding). Installing the plugin does not grant beta access.

## Why Pakt

- Keep wallet keys outside the agent.
- Approve exact limits before an agent can act.
- Get independently verifiable receipts for approvals and refusals.
- Use the same Pakt connection from Claude Code or Codex.

## Install

The quickest way to install or update Pakt for supported agent CLIs is:

```sh
curl -fsSL https://usepakt.ai/install.sh | bash
```

Or add the Git marketplace directly:

```sh
# Claude Code
claude plugin marketplace add bob-collective/pakt-plugins --scope user
claude plugin install pakt@bob-collective --scope user

# Codex
codex plugin marketplace add bob-collective/pakt-plugins --ref main
codex plugin add pakt@bob-collective
```

Start a new agent session after installation. Pakt asks you to sign in through
your browser the first time it is used. Never give an agent a wallet seed or
private key.

## Learn more

- [Pakt website](https://usepakt.ai)
- [Documentation](https://usepakt.ai/docs/)
- [Join the closed-beta waitlist](https://usepakt.ai/onboarding)
