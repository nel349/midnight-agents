# midnight-agents

On Arc we built an app where a person gives an AI agent a spending allowance from a passkey phone
wallet, and the chain enforces the limit, the expiry and the revoke. This repo answers what it would
take to build the same thing on Midnight, and what is missing that we could contribute.

Research, not code. The code lives in:
- [nel349/arc-agent-mandate](https://github.com/nel349/arc-agent-mandate), the phone app, the
  session-key contracts and the MCP connector
- [nel349/arc-maze](https://github.com/nel349/arc-maze), the vendor the agent pays
- [nel349/midnight-wallet-cli](https://github.com/nel349/midnight-wallet-cli), the Midnight CLI and
  MCP wallet
- Kuira, the mobile SDKs:
  [kuira-sdk-android](https://github.com/kuiralabs/kuira-sdk-android),
  [kuira-midnight-ffi](https://github.com/kuiralabs/kuira-midnight-ffi) and
  [midnight-rs](https://github.com/kuiralabs/midnight-rs) (the iOS SDK is private)

| File | Answers |
|---|---|
| `midnight-vs-arc.md` | What exists on Midnight, piece by piece, against what Arc gave us |
| `OPPORTUNITIES.md` | What we could contribute, and where it gets filed |
| `ROADMAP.md` | What order to do it in, and what is blocked |
| `research/` | The long reports: the Arc inventory, a review of MIP-0017, and what agents actually pay for |

Anything not confirmed first-hand is marked unverified.
