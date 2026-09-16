# Midnight versus Arc: the agent system, piece by piece

Anything not confirmed first-hand is marked **unverified**.

## The question

On Arc we built this: a person grants an AI agent a spending allowance from a passkey phone
wallet. The chain enforces the limit, the contract and function allowlist, the expiry and the
revoke. Fees are sponsored for the person and the agent. The agent pays per API call with x402 in
USDC. It holds an ERC-8004 identity, earns reputation and a badge, and its results can be replayed
by a Chainlink CRE workflow, which so far has run only in Chainlink's simulator.

The question is what it would take to build the same system on Midnight: which pieces exist,
which exist in part, and which nobody is building.

## The answer in one paragraph

Midnight's primitives can express the core promise: money held by a contract, released to an agent
only within a limit, before an expiry, until revoked, with more privacy than Arc. Passport is
reported to have run a capped grant with issue, spend and revoke on a local network ([passport#69][],
#70; we could not confirm the prototype code is public). It is not proven on a public network. Expiry is the one thing standing in the way of proving it on
Preview: it depends on block-time checks with an open bug report ([compact#20][]). Mainnet is a separate
question, because deployment there is permissioned and a pooled vault scores 3 for value on the
rubric, which triggers a block.

Around that promise, most pieces are missing:
- There is **no standard** for the allowance.
- Midnight **cannot verify a passkey (P-256) signature**, in a transaction or in a contract.
- There is **no native stablecoin**, though USDM from Cardano has been live on mainnet since
  2026-08-13.
- There is **no x402 scheme**.
- There is **no agent identity or reputation standard**.
- There is **no public oracle network or standard**. Midnight's own bounty issue #304 says
  "Midnight doesn't have native oracle support". Protofire is reported to run closed-source DIA
  price feeds; we could not verify them (**unverified**).
- Contract-to-contract calls, contract events and secp256k1 checks in contracts are on
  **ledger 9, which is not on mainnet**. Ledger 9 is at release-candidate stage, and a ledger 10
  alpha was published on 2026-09-14.

Almost every gap has a paused proposal, an open design question in someone's plan, or nothing at
all. That is the opening for contribution.

## Two facts that frame everything

**1. Two eras are in flight.** The official support matrix (`docs/relnotes/support-matrix.mdx`) pins
mainnet, Preprod and Preview to Compact toolchain 0.31.1 on ledger 8. The Compact 0.33.0 release
notes say mainnet runs ledger 8; the 0.34.0 notes say ledger 9 "will be, but is not yet, deployed on
Midnight Mainnet". Both say contracts for mainnet should stay on 0.31.x. Ledger 9 brings:
- contract-to-contract calls
- contract events
- in-contract secp256k1 ECDSA and keccak

`jubjubSchnorrVerify`, the cheap in-contract signature check, appears in the compiler changelog at
the unreleased toolchain 0.31.104. There is **no stable 0.33.0**; the first stable release carrying
it is **v0.34.0** (2026-08-25), and it is **not** in v0.31.1. So a contract deployable on mainnet
today authorises an agent by either (our reading, no source enumerates these):
- a secret-hash witness (the bboard pattern), or
- a hand-built JubJub check.

Prefer the signature form. A secret passed as a private input is visible to whoever generates the
proof, which for an agent usually means a hosted prover (`research/mip-0017-review.md`, gap 4).

**2. User money is coin-based; contract money is a balance map.** NIGHT and native tokens are coins
a wallet holds, so a contract cannot pull from a wallet the way ERC-20 `transferFrom` does. A
contract's own holdings, by contrast, are an account-style balance map
(`onchain-state/src/state.rs`, `balance: HashMap<TokenType, u128>`).

The ledger does allow a user-signed payment straight into a contract (`spec/contracts.md`,
`unshielded_inputs`; [MPS-0018][] notes user to contract NIGHT works). What it does not allow is the
agent spending the user's coins while the user is offline: every unshielded spend needs a fresh
signature over the whole intent (`spec/night.md`). That is why an agent allowance points at a
**custody contract that holds the funds** and releases them within bounds. [MIP-0012][] standardises
that custody contract, and it explicitly leaves allowances out of scope.

## Side by side

Status key: **Have**, **Partial**, **Missing**, for Midnight, today.

| # | Piece | Arc (what we used) | Midnight today | Status | The gap |
|---|---|---|---|---|---|
| 1 | Account & signing | Circle Modular Wallet: a smart account owned by a passkey (P-256). No React Native support, so we wrote the native passkey module | Accounts are keys: Schnorr over secp256k1 ([MIP-0003][] adds ECDSA secp256k1). Contracts can hold and move funds. In-contract checks: JubJub Schnorr (stable from toolchain 0.34, not on public networks yet), secp256k1 ECDSA (ledger 9) | Partial | **No P-256 on any network, and no MIP filed.** P-256 operations for the proof backend merged into the `ledger-9` branch (ledger PR #655, 07-31), though issue [ledger#603][] is still open. Compact PRs are open: LFDT-Minokawa/compact #749 and #750 (issues #532, #674). Passport has agreed to use P-256 ([passport#51][], 2026-08-20), with secp256k1 as an interim, a per-scheme registry in a [MIP-0013][] successor, and PR #117 verifying a real platform-authenticator assertion in-circuit. So this is joint work, not a rival statement. Mobile: [ledger#322][] (open since 04-09) says React Native's Hermes engine has no WebAssembly, and every core package depends on it. Today a passkey must go through PRF: Passport derives a JubJub device key that way; Kuira derives a wallet seed and DID |
| 2 | Bounded authority (limit, expiry, revoke) | Session-key plugin ported to EntryPoint v0.7: spending limit, an allowlist of contracts and functions (the native rail is closed by default), expiry, revoke | A custody contract can express all of it: a limit in state, expiry with `blockTimeLt`, a revoke flag or epoch, the agent authorised by secret or signature. OpenZeppelin `FungibleToken.approve` has no expiry | Partial | **No standard.** Compact **cannot read the verified caller**: [MPS-0029][] recommends a `kernel.caller()` proposal (whether that lands as a MIP or a CoIP is **unverified**), and [MPS-0040][], "Cross-Contract Call Provenance in Compact Circuits", merged on 2026-09-16 UTC, so this is now work to review and extend rather than empty ground, and `ownPublicKey()` is forgeable. [MIP-0012][] excludes grants. [MIP-0013][] permits "scoped grants" as a later extension. Passport is reported to have prototyped a capped grant with issue, spend and revoke on a local network ([passport#69][], #70), but its grant format is an open question (C10). PR #251 (mandate tokens) is stalled, and the chain does not enforce its limit. **Expiry risk:** [midnightntwrk/compact#20][], "kernel.blockTimeLessThan bricks deployed contract", open since 2025-11. A Compact maintainer said the rejection "might be expected" and later that it is not a Compact bug; in 2026-07 a mainnet team reported the same problem, and nobody has posted a retest |
| 3 | Fees | Circle Gas Station pays for the person's and the agent's operations; the seller pays its own gas for reputation and badge writes | A user can point their NIGHT's DUST generation at the agent's key. A sponsor can add DUST to someone else's proven transaction (`balanceFinalizedTransaction` with DUST only; official guide and `example-private-party`). 1AM ProofStation runs a sponsor (**unverified**), and SundaeSwap's Capacity Exchange Service sells capacity (what exactly is **unverified**) | Have (mechanism) | **Contracts cannot hold or spend DUST**, so there is no contract paymaster. One DUST key per NIGHT address, publicly mapped, so designation cannot fund more than one agent per address. DUST caps at 5 per NIGHT, takes about a week to fill, does not survive a hard fork, and the protocol reserves the right to reset it. Registering cNIGHT for DUST takes about 12 hours on public networks. No standard sponsor service, API or operator model; Passport's fee model is an "Open decision" ([passport#30][], C24). No SDK helper builds a standalone DUST-spend intent: [midnight-wallet#376][] ("Dust ctime questions", still open) asked for one, a maintainer said on 05-11 there are "no plans", and a 05-29 request to reopen it as a feature request is unanswered. No MIP. [MPS-0031][] (DUST grant DAO) lists its recommended MIPs as "None Identified". **Paying a fee through a hosted prover still exposes the payer's DUST secret** (`research/mip-0017-review.md`, gap 1) |
| 4 | Money | USDC, one balance with two decimal views | NIGHT, plus tokens contracts mint (shielded or unshielded). **USDM** (issuer Moneta, fiat-backed) moves natively between Cardano and Midnight over VIA Labs messaging, issuer-minted rather than wrapped or bridged, live on mainnet and testnet since 2026-08-13, and **unshielded** (VIA's docs: amounts, recipients and balances are public). OpenZeppelin token modules (latest release 0.2.0; newer releases alpha; "highly experimental"). [MIP-0004][] is Draft; [MIP-0011][] and 0014 are Proposed | Partial | **No native Midnight stablecoin.** shieldUSD was announced in January 2026 and is not live; its deployment request (#105) scored 3 on two categories and was closed unmerged on 06-19 (the causal link is our reading). How USDM fits [MIP-0014][] is **unverified** |
| 5 | Pay per call | x402 via Circle Gateway. The agent pays from its own key; the wallet funds its escrow with `depositFor` | Nothing standard. Prototypes: m402; **Shade402**, live on Preview, whose proof checks per-agent limits that its backend supplies (the chain cannot verify them; funds are pooled); x402midnight (mock USDC); NightPay (Masumi bounties). A payment on chain costs a proof plus about 6 to 18 s | Missing | **No x402 scheme for Midnight** in the x402 spec repo (searched; the two hits are unrelated). The only issue about x402 support in the Midnight org is [passport#40][], "x402 Add Midnight network support", stalled since 06-29 as "not trivial". No payment-request format, request binding or receipt standard. Per-call on-chain payment is slow, so batching is worth exploring, but pre-signed payments expire within about an hour and cannot be cancelled (`research/mip-0017-review.md`, gaps 2 and 5) |
| 6 | Agent identity | ERC-8004 identity registry; the agent's id is owned by the user's wallet | Nothing agent-specific in the ledger, Compact, wallet or indexer. Midnight's reference `did:midnight` implementation (midnight-did v0.5.0) plus community DID prototypes, none agent-specific. [MPS-0015][] names the agent gap and points at ERC-8004. MAIS (PR #110) is open, unnumbered, and the author's 08-04 question is unanswered | Missing | No accepted agent registry. MAIS's spec is Solidity-shaped, and its "private registration" does not map onto how Midnight stores state. Binding an identity to an owner hits the caller gap again |
| 7 | Reputation & validation | ERC-8004 reputation registry, written by the seller; self-feedback refused. The score is computed from the run, not a constant | Buildable as contracts, with a validator signing and the contract verifying. Reputation thresholds can be proven without revealing history | Missing | No standard. MAIS reputation is proposal-only. The credential registry (PR #214) is open, with document status Draft and no maintainer review |
| 8 | Badge | Our own 100-place ERC-721 | A unique native token (amount 1) works but is transferable. OpenZeppelin has `NonFungibleToken` (in release 0.2.0; "highly experimental") | Partial | **No NFT standard** (MRC721 was closed as outdated; Discussion #218 floats one). No soulbound standard (MPS PR #211 is open) |
| 9 | Checking results off chain | Chainlink CRE workflow declared to run in an enclave; simulator only, receiver not deployed | Off-chain data enters only as untrusted prover input, so trust must come from a signature checked in the contract | Missing / Partial | No public oracle network or standard: Midnight's bounty issue #304 says there is no native oracle support, and VIA Labs' "Private Oracle" quickstart is EVM and Solidity only. Protofire is reported to run closed-source DIA feeds (**unverified**). No way to verify enclave attestations (P-256/P-384 or RSA). TEE provers (PR #49) quiet since 06-30. Proof verification in Compact (PR #198) is a draft with no comments, and every mechanism in it is `midnight-zk` (that it covers only Midnight's own proofs is our reading) |
| 10 | Events & watching | `eth_getLogs` over session-key grant and revoke events, Gateway deposits, USDC transfers and `UserOperationEvent`, in 10,000-block windows | Indexer GraphQL queries and subscriptions for contract state and transactions (mainnet). [MIP-0002][] contract events are Accepted | Have / Partial | Contract events need **ledger 9 and the indexer 4.4.0 release-candidate line**; the mainnet maintenance line (4.3.x) has no `contractEvents` field. Standard event types have no delegation or identity types, only a `Misc` catch-all ([MIP-0002][] allows new types as amendments). No pre-finality view ([MPS-0028][]). For shielded coins, a watch-only app cannot even tell that a coin was spent without the key holder's help (`research/mip-0017-review.md`, gap 7) |
| 11 | Agent tooling | Our own MCP connector in Claude Code | Headless Node wallet works. Proving needs a proof server or WASM; Kuira is the only mobile SDK documented as proving on the device. **`midnight-wallet-cli` is the only entry in the MCP column of the community wallets page** (`sdks/community/wallets/cli-and-mcp.mdx`) | Have (Node) / Missing (mobile, safety) | That page calls the agent wallet "the highest-risk cell" and suggests manual caps. No general agent wallet reads a chain-enforced grant (Shade402's policy is bespoke and custodian-supplied). Calling a contract needs its full artifacts, with no ABI ([MPS-0039][], which names agent CLIs). Whoever proves a shielded spend sees the spend key ([MPS-0035][]). [MIP-0017][] (Proposed, merged 09-11) would remove that for new shielded notes, but DUST proving still exposes its secret |
| 12 | Privacy | Arc Privacy Sector: enclaves, "not yet available" | Shielded spending, explicit disclosure, viewing keys. A stateless custody contract ([MIP-0012][]) could hide amounts, balances and counterparties | Have, with caveats | Contract state, unshielded flows and the NIGHT to DUST link are public. Nobody has decided what a private allowance should reveal (the amount? the payee? the agent's id?). Under [MIP-0017][] a contract cannot pay the new shielded coin type at all, so contract-released funds stay on the old type (`research/mip-0017-review.md`, gap 3) |
| 13 | Pairing | A one-time code the phone scans, hashed into the grant event so only that agent can use the grant | No contract events before ledger 9, so the pairing channel has to be designed another way: contract state the agent reads, or an off-chain channel bound to a key | Missing | Undesigned. It is the first thing an implementer hits after the allowance itself |
| 14 | Escrow top-up | The wallet funds the agent's x402 escrow with `depositFor`, metered by the same limit | No counterpart. Either the agent holds funds (and the limit stops meaning anything) or every payment goes through the custody contract | Missing | Nobody has designed the metered top-up. It decides whether x402 on Midnight can batch at all, so it belongs inside the pay-per-call work (N3) |

## Constraints a builder hits beyond the table

- **Mainnet contract deployment is permissioned.** The deployment rubric (in the MIP repo,
  `deployments/contract-deployment-rubric.md`) scores privacy, value and state-space risk from 1 to
  3. A 3 in any category triggers a deployment block, though the rubric says that is not permanent.
  Value tier 3 is funds that accumulate in a permanent, long-term vault, pool or treasury; tier 2 is
  funds escrowed for a bounded time, and it is scored, not exempt. A pooled allowance vault reads as
  tier 3. Whether a per-user, expiring custody contract would be accepted at tier 2 is **unverified**,
  and that argument depends on expiry working. Calibration point: Ascend Perps scored value 2 because
  its contracts hold no token balances directly.
- **Contract-to-contract calls are not enabled on mainnet.** [MIP-0004][] defers
  `approve/transferFrom` for that reason. Phase 2 of those calls (callees with witnesses and private
  state) is only a problem statement ([MPS-0021][]).
- **Contract-created outputs to a user carry no coin ciphertext** ([MIP-0011][]), because Compact's send
  takes only a coin public key, so nothing can encrypt the note. A seller paid by a contract needs
  out-of-band delivery or an encrypted inbox ([MIP-0012][]). The mirror rule, that contract-owned outputs
  must not carry ciphertext, is in the ledger spec.
- **A limit on one rail is not a limit.** Midnight has unshielded and shielded rails plus DUST. A
  grant has to meter every rail the agent can reach, or close the ones it does not use. On Arc we
  learned this the hard way: the native rail is closed by default in our plugin.
- **Revoke does not recall funds already released.** Whatever the agent already holds, or already
  moved into an escrow, stays with it. This has to be said in the interface, as it is in our app.
- **Limits** (ledger parameters at HEAD and ledger-8.1.2; these are genesis defaults on a
  governance-adjustable struct, and mainnet's live values are unverified):
  - 1 MiB per transaction
  - 50,000 bytes of state writes per block
  - 1-hour transaction lifetime
  - 6 s blocks; finality measured at 2 blocks on mainnet, and the docs say one or two, within 18 s
- **A fee-sponsored agent has no caller at all.** The ledger derives the caller from unshielded
  inputs, so an agent with no coins of its own produces none. Caller identity and sponsorship pull
  against each other.
- **A coin minted by one call cannot be spent in the same transaction** ([midnight-js#968][],
  maintainer-confirmed), which rules out the obvious vault-then-pay shape.
- **Unshielded NIGHT transfers cap at about 4 inputs** in the guaranteed segment
  ([midnight-ledger#716][]), which bounds payment batching.
- **`unshieldedBalance` requires an exact balance match** when the transaction applies, a footgun in
  any custody contract.
- **Mainnet deployment also needs an API key** to a guarded node (PR #206), on top of the rubric.
- **Churn:** Compact broke source compatibility from 0.31 to 0.34, midnight-js is on v5 beta, and
  the indexer runs two release lines.

## Who is already working on the same problem

| Who | What | Relation to the Arc design | Where to look |
|---|---|---|---|
| **Midnight Passport** (IOG, October 2026 MVP) | Passkey-rooted account, multi-device, recovery, **scoped grants** (C10 to C12), fee model (C24), dApp to wallet connection protocol covering capability grants (C23), integrator skills package (C26). Its people wrote MIP-0012 and MIP-0013 | The same design converging from the wallet side. A capped grant with issue, spend and revoke is reported to run on a local network. x402 support stalled in June. The grant format, fee model and connection protocol are open and **unfiled** as MIPs. Their MVP lands in October, so a co-authoring conversation is time-sensitive | [midnightntwrk/passport](https://github.com/midnightntwrk/passport), issues [#69](https://github.com/midnightntwrk/passport/issues/69) and [#70](https://github.com/midnightntwrk/passport/issues/70) (grant prototype), [#51](https://github.com/midnightntwrk/passport/issues/51) (P-256 agreed), [#117](https://github.com/midnightntwrk/passport/pull/117) (passkey assertion in-circuit), [#40](https://github.com/midnightntwrk/passport/issues/40) (x402), [#30](https://github.com/midnightntwrk/passport/issues/30) (fees) |
| **Shade402** | x402 facilitator on Preview, contract confirmed on chain. Its proof checks balance, per-payment cap and daily limit; those values come from its own backend, which its roadmap calls out as the thing to remove in "Wave 2" | The closest working piece to the allowance plus x402. Built for one product, with pooled funds and limits the chain does not check | [mhizer-fatai/shade402](https://github.com/mhizer-fatai/shade402) |
| **Private Mandate Tokens** | The only agent-allowance proposal | Blocked on unverified commit signatures; the editor restated the block on 2026-09-09. Reading it, we had three questions to raise with the author: the agent check takes a value the caller supplies, expiry is checked in the mandate proof but not at payment time, and funds move outside the contract | [PR #251](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/251) |
| **MAIS** (agent identity), behind MPS-0015 | Agent identity, reputation and validation registries; optional ERC-8004 bridge | Waiting on the editors. Needs a Compact-accurate rework; reputation, validation and the bridge are unbuilt | [PR #110](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/110), [MPS-0015](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md) |
| **midnight-agent-did-manager** (apestchanker) | A DID registry running on Preprod, by a different author; it names MAIS only in a changelog | The closest working identity piece | [apestchanker/midnight-agent-did-manager](https://github.com/apestchanker/midnight-agent-did-manager) |
| **VIA Labs + Moneta** | USDM moves natively between Cardano and Midnight, mainnet since 2026-08-13, issuer-minted and unshielded | The dollar an x402 price could use | [VIA Labs guide](https://developer.vialabs.tech/docs/examples/guides/usdm-cardano-midnight/), [Moneta](https://moneta.global/) |
| **AgenticDID** (bytewizard42i) | Claims scoped-grant circuits with two-cap budgets, attenuation-only delegation and cascade revocation (**unverified**, README only) | A fourth party in the allowance space, worth reading before filing | [bytewizard42i/AgenticDID_io_me](https://github.com/bytewizard42i/AgenticDID_io_me) |
| **1AM ProofStation** (Webisoft) | Hosted prover that also sponsors fees, behind an API key (**unverified**, first-party docs only) | The de facto Gas Station, proprietary | [1am.xyz/developers](https://1am.xyz/developers), [proofstation.1am.xyz](https://proofstation.1am.xyz/) |
| **OpenZeppelin Compact contracts** | Tokens and access control in release 0.2.0; multisig, treasuries and `ConfidentialFungibleToken` only from 0.3.0-alpha.1; current line 0.4.0-alpha.1. **Only v0.1.0 was audited** (May 2026); the README says the library "has never been audited nor thoroughly reviewed for security vulnerabilities" | No expiring allowance, delegation, identity registry or soulbound module. The confidential token has escrow-based approve and revoke. Issue #548 is a worked JubJub verifier spec; #647 and #648 are unfilled templates, and #648 asks for a compiler primitive rather than a library module | [OpenZeppelin/compact-contracts](https://github.com/OpenZeppelin/compact-contracts), [#548](https://github.com/OpenZeppelin/compact-contracts/issues/548) |

## What we already bring (from our own repos)

- **Kuira:**
  - Passkey to wallet seed and DID on mobile (self-reported; the docs stress seedlessness), with
    **on-device proving**.
  - Android is alpha on Maven Central and listed in the official wallet docs; the iOS SDK exists and
    is early (pushed 2026-09-03).
  - Our M6 finding that passkey PRF is ecosystem-locked. Note [MIP-0015][] is about wallet-derived
    deterministic secrets, not passkeys, and it does discuss PRF, rejecting it as credential-bound;
    the ecosystem lock-in point is still unmade there.
- **midnight-wallet-cli:** the only MCP entry on the community wallets page, headless connector, DUST
  tooling (sync cut from about 22 minutes to about 4; our two notes say 4.5 and 3.7, and neither
  names the network). Its awesome-dapps entry points at the hub repo archived in August, so the
  link needs fixing.
- **kuira-vault:** OpenZeppelin treasury composition with passkey signers on phones, which is custody
  contract experience.
- **midnight-rs:** 10 commits ahead of upstream `main` in the kuiralabs fork (more on side
  branches), none contributed upstream.
- **kuira-offer-links** (not yet public): a [MIP-0005][]/0006 codec checked byte for byte.
- **Prior prototypes:**
  - subscriptions and bounded recurring payments (midnight-pay)
  - authorisations between contacts (midnight-bank)
  - auditor attestations and ZKML verification (zkSalaria)
- **The whole Arc design,** which is working prior art for every row above.
- **No MIP or MPS authored yet.**

## Known gaps in our own work

Being fixed. Listed so nobody leans on the Arc evidence further than it holds:

- **The deployed plugin address survives only in prose.** `contracts/broadcast/` is gitignored, so a
  fresh clone has no deployment artifact, and no test pins the address.
- **42 of 48 Foundry tests skip without `ARC_TESTNET_RPC_URL`**, so a CI box without that variable
  reports green while testing almost none of the security claims.
- **Payee-scoped mandates are unreachable from the app.** The library supports them and unit tests
  cover them, but the grant screen always passes an empty payee list. They are a capability, not a
  shipped feature.
- **The first two badges are on a superseded contract.** The current cohort was redeployed to split
  admitter from owner.

## Research behind this

| File | Covers |
|---|---|
| `research/arc-system.md` | What Arc and Circle provide for agents, and what our app actually used |
| `research/mip-0017-review.md` | [MIP-0017][] (shielded spend authorisation), reviewed for what it changes for agents |

Local library checkouts are months behind upstream, so every load-bearing fact above was checked
against upstream. Ledger 8.1.2 is the latest stable and mainnet reports 8.1.2; ledger 9 is at
release-candidate stage and a ledger 10 alpha exists.

<!-- links -->
[MIP-0002]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0002-public-contract-log-emission.md
[MIP-0003]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0003-ecdsa-support.md
[MIP-0004]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md
[MIP-0005]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0005-offer-files.md
[MIP-0011]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md
[MIP-0012]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0012-native-asset-custody.md
[MIP-0013]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0013-account-authorisation.md
[MIP-0014]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0014-native-unshielded-token.md
[MIP-0015]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0015-wallet-derived-deterministic-secrets.md
[MIP-0017]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0017-shielded-spend-auth/mip-0017-shielded-spend-auth.md
[MPS-0015]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md
[MPS-0018]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md
[MPS-0021]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0021-phase2-contract-to-contract.md
[MPS-0028]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0028-pre-finality-state-visibility.md
[MPS-0029]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0029-compact-caller-identity.md
[MPS-0031]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0031-dust-grant-dao.md
[MPS-0035]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0035-shielded-spend-key-exposure.md
[MPS-0039]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0039-lightweight-contract-interaction.md
[MPS-0040]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0040-cross-contract-call-provenance.md
[compact#20]: https://github.com/midnightntwrk/compact/issues/20
[ledger#322]: https://github.com/midnightntwrk/midnight-ledger/issues/322
[ledger#603]: https://github.com/midnightntwrk/midnight-ledger/issues/603
[midnight-js#968]: https://github.com/midnightntwrk/midnight-js/issues/968
[midnight-ledger#716]: https://github.com/midnightntwrk/midnight-ledger/issues/716
[midnight-wallet#376]: https://github.com/midnightntwrk/midnight-wallet/issues/376
[midnightntwrk/compact#20]: https://github.com/midnightntwrk/compact/issues/20
[passport#30]: https://github.com/midnightntwrk/passport/issues/30
[passport#40]: https://github.com/midnightntwrk/passport/issues/40
[passport#51]: https://github.com/midnightntwrk/passport/issues/51
[passport#69]: https://github.com/midnightntwrk/passport/issues/69
