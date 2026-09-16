# Opportunities: everything we could contribute to, by where it lands

The **Fit** column says how much of our existing work carries over:
- **Strong:** we have built it, on Arc or on Midnight.
- **Medium:** adjacent experience.
- **Light:** new ground.

## Where each kind of contribution goes

The editors' rulings set the channels, so work in the wrong channel gets closed:

| Kind | Channel | Rule learned the hard way |
|---|---|---|
| Protocol changes, standards, interfaces, data formats | **MPS, then MIP** in `midnight-improvement-proposals` | When no problem statement exists, editors ask for one first, in its own PR (editor on #110). Commits must be signed, the CLA signed, and a human named as author. Never edit `NUMBERS_INDEX.md`. Acceptance runs through an author-filed issue requesting a vote, at least two weeks of commentary, then an editors' poll |
| Compact language features | **CoIP** in `LFDT-Minokawa/compact` | Contract-to-contract calls were redirected to a CoIP (MIP repo PR #56, closed 05-14). ZKIR was closed as an internal implementation detail (MIP repo PR #58, closed 05-13). The live tracker is LFDT-Minokawa/compact issue #345, open since April with no activity since 04-16 |
| Contract building blocks | **OpenZeppelin `compact-contracts`** | Accepts community modules. Only v0.1.0 was audited, and the current line is alpha |
| SDK and core fixes | Issues and PRs in `midnight-wallet`, `midnight-js`, `midnight-ledger`, `midnight-dapp-connector-api`, `midnight-node` | External `yarn install` of midnight-sdk fails (issue #334); fork CI fails in midnight-js (issue #989) |
| CLIs, tools, apps | **`midnight-awesome-dapps`** and the community docs section | "This sort of tooling … does not require formal acceptance through a MIP" (editor, #288) |
| x402 on Midnight | The **x402 spec** repo (outside Midnight), and possibly a MIP for the Midnight-side format | No Midnight scheme exists; [passport#40][] stalled |

## 1. Proposals we can still change

These MIPs are Proposed or Draft, plus [MIP-0002][], which is Accepted but allows new event types as amendments. In practice editors merge as Proposed and authors keep amending,
so feedback goes to the proposal's Discussion and to its authors.

| MIP | State | What it lacks for agents | Contribution | Fit |
|---|---|---|---|---|
| **0013** Multi-key account authorisation (Passport authors) | Proposed; Discussion #244 (last post 09-02). It binds the network id only as a SHOULD, so a signature can replay across networks | Device keys are JubJub only, so a passkey cannot be a device. Every device has full authority, with no limits. Signatures bind to one account-wide counter (`auth_nonce`), so an owner's action cancels an agent's pending call | Define the "scoped delegate" extension the MIP permits: a spending policy bound to a device entry, with an agent's signature bound to its own per-device use counter instead of the account-wide one. Register a P-256 device scheme once Compact has it. Use Discussion #244, where the author already says P-256 is the norm | **Strong**, this is the Arc session key |
| **0012** Contract custody of native assets (Passport authors) | Proposed | Allowances and DUST are explicitly out of scope. Its direct-transfer mode names a disclosed merchant as the motivating case, but there is no payment request or reference | Specify the policy object that §2 already places behind the authorisation seam; that object is the grant. Add a pointer to fee sponsorship for custody accounts | **Strong** |
| **0015** Wallet-derived secrets (`deriveSecret`) | Proposed; Discussion #282 has **zero comments**; test vectors not frozen | Seed-only, so a seedless passkey wallet cannot conform. It discusses WebAuthn PRF without the ecosystem lock | Contribute Kuira's evidence that the same passkey yields different PRF output on iOS and Android. Propose an optional seedless profile with its recovery trade-off stated | **Strong**, Kuira M6 |
| **0004** Fungible token with UTXO conversion | Draft | `approve` deferred until contract-to-contract calls; no expiry in the plan | An optional allowance extension, with expiry and per-period caps and no forgeable caller, ready for when those calls land | Medium |
| **0014** Native unshielded token | Proposed | No payment-request format (amount, recipient, reference, lifetime) | An informative payment-request section, the shape x402 would carry | Medium |
| **0005** Offer files | Proposed | A one-way payment offer is valid but refused by [MIP-0006][] indexers. No attached message, so a payment cannot name the request it pays for | A "directed payment offer" profile, and pushing "attached messages" forward | **Strong**, kuira-offer-links (not yet public) |
| **0002** Contract events | Accepted, but new event types are allowed as amendments | Standard events cover transfers, mints, burns and pause, plus a `Misc` catch-all with no indexing | Add delegation granted, revoked and spent, identity registered, attestation issued, and NFT transfer | Medium |
| **0007** Name service | Proposed; Discussion #190 (quiet since 07-15) | No agent keys, no passkey owner, no reverse lookup | Reserved keys for agent id, agent card and payment endpoint | Light |
| **0008** CAIP-2 network ids | Draft | CAIP-10 account ids out of scope; cross-chain agent registries and x402 key on them | A CAIP-10 profile or companion MIP | Light |
| **0017** Signature-authorised shielded spends | Proposed, merged 09-11 UTC; Discussion #307, no comments | "No new allowances"; fees, contract-paid coins, cancellation and watch-only spend detection are all out of scope (see `research/mip-0017-review.md`) | An informative section on single-use pre-authorised payment evidence (a bounded voucher an owner hands an agent), stating its three limits: it expires within about an hour, it cannot be cancelled, and the owner cannot easily see what they are signing. Its broken links and the MPS-0017 mislabel in Discussion #307 are a cheap first contribution | Medium |
| **0011** Native shielded token | Proposed; an editor session on it is scheduled for 2026-09-16, the last entry on the rolling agenda | Allowances "not representable" | A short note pointing delegated spending at the custody seam | Light |

## 2. Stalled proposals we can help move

| Item | State | What would help | Fit |
|---|---|---|---|
| **PR #251** Private Mandate Tokens | Open, draft; blocked on unverified commit signatures, restated by the editor 2026-09-09; no MPS | A review raising four points with the author: the agent check takes a value the caller supplies, the chain does not enforce the limit, expiry is checked in the mandate proof but not at payment time, and there is no per-period cap or problem statement. Offer the MPS or co-authorship | **Strong** |
| **PR #110** MAIS agent identity, with [MPS-0015][] behind it | Open, unnumbered; author's 08-04 question unanswered | Review with ERC-8004 experience. Make the registries fit Compact; split identity from reputation; implement its reputation, validation and bridge parts (a DID registry by a different author already runs on Preprod: midnight-agent-did-manager, which names MAIS only in a changelog) | **Strong** |
| **PR #214** Credential registry and soulbound profile, with MPS PR #211 | Open (document status Draft); no visible maintainer review | An on-chain verifier profile, and an agent-holder profile tied to [MIP-0013][] keys | Medium |
| **PR #306** Message signing | Open since 09-10 (document status Draft) | Review now; propose a typed, domain-bound follow-on (the EIP-712 equivalent) | Medium |
| **Discussion #218** NFT interface | Zero replies; the author has a reference implementation (midnight-nft), an open MIP PR #214 and co-authorship of MPS PR #211 | Join as co-author, adding badge and soulbound needs | Medium |
| **Discussion #260** Platform gaps for writing standards | Zero replies | Help with hashing layout docs and Merkle non-membership proofs, both prerequisites for revocation lists. Note MPS PR #311 (opened 2026-09-15) now covers the Merkle-path binding half | Light |
| **PR #49** TEE proof servers | Open since February; no activity since 06-30 | Generalise attestation so an enclave can vouch for a result, not only a proof | Light |

## 3. Problem statements and proposals nobody has written

Ordered by how directly the Arc work provides the evidence.

| # | Topic | Type | Hooks already in the repo | Mirrors | Fit |
|---|---|---|---|---|---|
| N1 | **Bounded, expiring, revocable spending authority for agents**, metering every rail and stating that revoke cannot recall released funds | MPS (Standards), then MIP on the [MIP-0012][]/0013 seam | [MIP-0012][] scope note, [MIP-0013][] R6, [MIP-0004][] deferral, PR #251, [MPS-0015][], Passport C10 to C12 | ERC-7715, ERC-7710, session keys on ERC-4337 accounts (ERC-7579 modules) | **Strong** |
| N2 | **Caller identity (`kernel.caller()`)** | MIP recommended by [MPS-0029][], likely routed as a CoIP (unverified) | [MPS-0029][]; requests on [MIP-0007][] and D142. **No longer empty ground:** [MPS-0040][] "Cross-Contract Call Provenance in Compact Circuits" merged 2026-09-16 | Solidity `msg.sender` | Medium |
| N3 | **Pay per call (x402 on Midnight)**, including the metered escrow top-up (the comparison table, row 14). A proven shielded offer is about 16 KB as bech32, too large for an HTTP header, so the payload shape matters | MPS, then a Midnight scheme in the x402 spec, plus a MIP for the payload format if editors want one | [MIP-0014][], [MIP-0005][], [MIP-0008][]; [MPS-0004][] use case 5 (paying provers per proof); [passport#40][] | x402 `exact` scheme with ERC-3009 (formerly EIP-3009) | **Strong** |
| N4 | **Fee sponsorship service** (intent shape, abuse limits, exhaustion, operator model). Passport's C24 names the hard parts: sequential DUST accounting to avoid double-spends, capacity detection, and a latency budget | MPS (or extend [MPS-0031][]), then Standards MIP | [MPS-0018][] open question, [MPS-0006][] goal 7, [passport#30][], [midnight-wallet#376][] (SDK helper declined) | ERC-4337 paymasters | **Strong**, wallet-cli DUST work |
| N5 | **Passkey (P-256 / WebAuthn) signatures** in the ledger, `signData` and contracts | MPS (Core), then Core MIP; the Compact side is a CoIP | Discussion #244; [MPS-0035][]'s unclaimed secure-element signing MIP (still unwritten, and central rather than tangential: the [MIP-0017][] review shows a signer cannot see what it signs, `research/mip-0017-review.md` gap 6); ledger PR #655 merged on `ledger-9`; LFDT-Minokawa/compact PRs #749, #750. Whether Passport reserves a P-256 slot is **unverified**: their tracking issues name a signing primitive (#51) and client-side passkeys (#48) | RIP-7212 / EIP-7951 | **Strong**, Arc passkey work and Kuira |
| N6 | **Wallet permission request** (an agent asks for an allowance, the phone approves) | Standards MIP (connector API) | DApp Connector API. The docs' wallet matrix says Midnight has no native account abstraction, and Lace has neither `signData` nor a proving provider, so the reference desktop wallet cannot approve a grant today. **Passport's planned connection protocol (C23) covers capability grants, so coordinate** | ERC-7715 `wallet_requestExecutionPermissions`, CAIP-25, CIP-30 | **Strong**, Arc's pairing code |
| N7 | **Agent identity registry, ERC-8004 compatible** | Standards MIP under [MPS-0015][] (with or replacing #110) | [MPS-0015][], [MPS-0029][] | ERC-8004 | **Strong** |
| N8 | **Reputation and validation** | Standards MIP | #110, #214, [MPS-0036][] | ERC-8004 reputation and validation, EAS | Medium |
| N9 | **NFT interface with a soulbound profile** | MPS, then MIP (with the D218 author) | D218, #211, #214 | ERC-721, ERC-5192 | Medium |
| N10 | **Checking off-chain results in a contract** (oracle or enclave) | MPS (Core) | PR #49, PR #198, [MPS-0009][]/0010 | On-chain attestation verifiers | Medium, zkSalaria |
| N11 | **Contract interface artifact (an ABI)** for agents and CLIs | One of four unclaimed MIPs under [MPS-0039][] | [MPS-0039][] names agent CLIs; co-design with [MPS-0022][]'s unwritten compiled-contract representation MIP | EVM ABI, Solana IDL | **Strong**, wallet-cli calls contracts |
| N12 | **Mobile support** | MIP recommended by [MPS-0002][], unwritten | [ledger#322][] (open since 04-09), [midnight-js#1213][] | |  **Strong**, Kuira, MidnightMobile |
| N13 | Typed, domain-bound signed data | Standards MIP after #306 | #306 | EIP-712, CIP-8 | Light |
| N14 | Account recovery paths | MIP recommended by [MPS-0018][] and [MIP-0013][], unclaimed | [MIP-0013][] recovery seam | Social-recovery modules on ERC-4337 accounts | Light, Passport owns it |
| N15 | **Hardware and secure-element signing interface for shielded spends** | The second MIP [MPS-0035][] recommends, still unwritten. Parked: it matters for agents only once a policy co-signer is possible | [MPS-0035][]; `research/mip-0017-review.md` gap 6 and its [MPS-0035][] goals table | Hardware wallet signing interfaces | Medium |
| N16 | **Binding an agent to an allowance** (the pairing channel), without contract events | Part of N1, or its own MIP on the connector seam. N6 is the request; N16 is the binding | Arc's pairing code; contract events are ledger 9 only; the comparison table, row 13 | ERC-7715 permission requests | **Strong** |

## 4. Contract standards (OpenZeppelin Compact)

None of these exist as described, confirmed by searching the module sources and open issues. The
closest pieces:
- `ConfidentialFungibleToken` has escrow-based approve and revoke (0.3.0-alpha.1, unaudited).
- `Signer` and `EcdsaSignerManager` manage multisig signers, secp256k1 (0.3.0-alpha.1, unaudited).
- Open issues #548, #647 and #648 propose a JubJub signature verifier.
- A timelock (#145) is still open.
- Revocation primitives are already in progress: a revocable membership tree (#736), delta inboxes, a sharded counter.
- Their 0.3.0-alpha.1 audit filed allowance pitfalls: memo growth from inefficient revocation, and confidential-token allowance semantics diverging from ERC-20.

Build on those rather than beside them:

| Module | What it would be | Fit |
|---|---|---|
| **Scoped allowance** | Cap, period, expiry and revoke epoch over a custody contract, with no forgeable caller. It must meter every rail the agent can reach, or close the ones it does not use, and it cannot recall funds already released | **Strong**, Arc session key, kuira-vault |
| **Signed attestation verifier** | Accept data signed by a key the contract trusts: JubJub, hand-built on today's networks (the stdlib verifier needs toolchain 0.34), or secp256k1 on ledger 9 | Medium |
| **Agent registry** | A reference implementation for MAIS / N7 | **Strong** |
| **Soulbound extension** | Non-transferable NFT and credential badge | Medium |

## 5. SDK and core code with open doors

| Issue | Ask | State | Fit |
|---|---|---|---|
| [midnight-wallet#376][] | A helper for the standalone DUST-spend intent a sponsor adds; the ctime question | Answered 05-11: construction confirmed, SDK helper declined ("no plans"). The reporter's 05-29 feature-request follow-up is unanswered | **Strong**, back the follow-up, or ship the helper in midnight-wallet-cli |
| [midnightntwrk/compact#20][] | `blockTimeLessThan` bricks a deployed contract | Open since 2025-11. On 12-12 a maintainer suggested testing on the latest environment, and no result was posted. A maintainer said the rejection "might be expected" (prover and chain times differ) and later that it is not a Compact bug. A third party reported the same on mainnet on 2026-07-07 | **Strong**, expiry depends on it |
| [midnight-js#1213][] | Declare and verify supported runtimes (Node, browser, React Native) | Untriaged | **Strong** |
| [midnight-js#725][] | Test helper for contract-to-contract calls | Fully designed, unassigned | Medium |
| [midnight-js#982][] / #809 | Configurable transaction lifetime | #982: a volunteer went unanswered. #809 is assigned | Medium |
| [midnight-dapp-connector-api#59][] | `submitTransaction` returns the transaction id | Small, unanswered | Medium |
| [midnight-ledger#653][] | Watch-only decryption | "Purely an API exposure gap", unanswered | Light |
| [midnight-node#1838][] | ECDSA maintenance committees in the toolkit | Task list, zero comments | Light |
| [midnight-docs#1291][] | Outdated cross-contract page | Assigned; waits until Compact 0.33+ reaches Preview | Light |

## 6. Tools and SDKs we own or could own

These are not MIPs, so they go to awesome-dapps and the community docs.

| Tool | What it becomes | Fit |
|---|---|---|
| **midnight-wallet-cli** | The agent wallet that obeys a chain-enforced grant instead of local caps. It could also host a sponsor service and an x402 facilitator | **Strong**, already the only MCP entry on the community wallets page. Its awesome-dapps link points at the archived hub repo and needs fixing |
| **Kuira (Android, iOS SDK, early)** | The phone side: grant, pair and revoke from a passkey wallet with on-device proving | **Strong** |
| **React Native binding over Kuira or midnight-rs** | What the Arc phone app would need; the evidence for N12 | **Strong** |
| **kuira-offer-links** (not yet public) | Payment links for directed offers ([MIP-0005][] profile) | Medium |

## 7. Errors in Midnight's own docs

Small fixes, visible, and a cheap way to start.

| Where | What is wrong | Fix |
|---|---|---|
| [MIP-0017][] | Six internal links are broken: the proposal and its specification point at each other and at the problem statements with paths that do not resolve | A pull request correcting the paths |
| Discussion #307 ([MIP-0017][]) | Labels its problem statement "MPS-0017", which is a different, open proposal about governance observability. [MIP-0017][] answers [MPS-0035][] | A comment, or an edit by the author |
| awesome-dapps | The `midnight-wallet-cli` entry points at the hub repo, archived in August 2026 | A pull request pointing it at the source repo |

## 8. Where privacy is load-bearing

From `research/what-agents-pay-for.md`. Most of what agents pay for today is data through APIs, and
most of those listings are never called twice. The model that works is the inverted one: the agent
pays to do something, and the seller supplies the challenge and the verdict. These are the shapes
where a privacy chain is the product rather than a preference, best first.

| Shape | What privacy buys | Our evidence |
|---|---|---|
| Contests with hidden state (mazes, puzzles, capture-the-flag) | A public instance is trivially solved or front-run; shielding lets a host commit without revealing | The maze, running on Arc |
| Stake to be evaluated | The score is public, the strategy stays private, which is what makes a lab willing to be benchmarked | Numerai's model, plus our verdict flow |
| Blind agent-versus-agent markets | Stops copy-trading and extraction against a known agent's positions | None yet |
| Certification with confidential evidence | Prove you passed a red-team suite without publishing your failures | None, and nobody buys this on-chain today |
| Sealed-bid task and compute markets | Hides reserve prices and bidding strategy | None yet |
| Escrow between agents | Confidential terms, public settlement | None yet |

## 9. Outside Midnight

- **x402 spec:** a `midnight` network and scheme. There is no Midnight scheme today.
- **Existing prototypes to consolidate:** m402, Shade402 (live on Preview; its proof checks limits its backend supplies, which the chain cannot verify),
  x402midnight, NightPay. Talking to their authors first costs little and avoids a fifth fork.

## Who to coordinate with before filing

- **Midnight Passport** (IOG) people wrote [MIP-0012][] and [MIP-0013][]. Their plans include:
  - scoped grants (C10 to C12)
  - the fee model (C24, [passport#30][])
  - x402 ([passport#40][])
  - the dApp↔wallet connection protocol covering capability grants (C23)
  - passkey-derived device keys. A reserved P-256 slot is **unverified**: their tracking issues name a signing primitive (#51) and client-side passkeys (#48)

  C26 is a skills package for integrators, not agent functionality. A grant MIP filed without
  them duplicates work they are reported to have prototyped ([passport#69][], #70). Their plans want an
  external co-author, and our agent evidence is what they lack.
- **MIP editors:**
  - nstanford5 merges most proposals; riusricardo also merges.
  - hbulgarini authored [MIP-0012][] and [MIP-0013][] and has merge rights; codeowner status is **unverified**, since the CODEOWNERS file names only a team.
  - Sessions are listed in `ROLLING_AGENDA.md`: roughly weekly in July, irregular since (1 September,
    then 16 September, the last entry as of 2026-09-15).
- **Authors of paused work:** #251, #110 / [MPS-0015][], D218.
- **Active prototype authors:** Shade402, m402.
- **The token:** VIA Labs and Moneta, for USDM.

<!-- links -->
[MIP-0002]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0002-public-contract-log-emission.md
[MIP-0004]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md
[MIP-0005]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0005-offer-files.md
[MIP-0006]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0006-p2p-atomic-swaps.md
[MIP-0007]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0007-name-service-registry-and-resolver.md
[MIP-0008]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0008-caip-2-network-identifiers.md
[MIP-0012]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0012-native-asset-custody.md
[MIP-0013]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0013-account-authorisation.md
[MIP-0014]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0014-native-unshielded-token.md
[MIP-0017]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0017-shielded-spend-auth/mip-0017-shielded-spend-auth.md
[MPS-0002]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0002-developer-tooling.md
[MPS-0004]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0004-trusted-proof-serving.md
[MPS-0006]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0006-shielded-asset-custody.md
[MPS-0009]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0009-sig-rsa.md
[MPS-0015]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md
[MPS-0018]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md
[MPS-0022]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0022-standard-contract-representation.md
[MPS-0029]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0029-compact-caller-identity.md
[MPS-0031]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0031-dust-grant-dao.md
[MPS-0035]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0035-shielded-spend-key-exposure.md
[MPS-0036]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0036-security-evidence-for-compact.md
[MPS-0039]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0039-lightweight-contract-interaction.md
[MPS-0040]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0040-cross-contract-call-provenance.md
[ledger#322]: https://github.com/midnightntwrk/midnight-ledger/issues/322
[midnight-dapp-connector-api#59]: https://github.com/midnightntwrk/midnight-dapp-connector-api/issues/59
[midnight-docs#1291]: https://github.com/midnightntwrk/midnight-docs/issues/1291
[midnight-js#1213]: https://github.com/midnightntwrk/midnight-js/issues/1213
[midnight-js#725]: https://github.com/midnightntwrk/midnight-js/issues/725
[midnight-js#982]: https://github.com/midnightntwrk/midnight-js/issues/982
[midnight-ledger#653]: https://github.com/midnightntwrk/midnight-ledger/issues/653
[midnight-node#1838]: https://github.com/midnightntwrk/midnight-node/issues/1838
[midnight-wallet#376]: https://github.com/midnightntwrk/midnight-wallet/issues/376
[midnightntwrk/compact#20]: https://github.com/midnightntwrk/compact/issues/20
[passport#30]: https://github.com/midnightntwrk/passport/issues/30
[passport#40]: https://github.com/midnightntwrk/passport/issues/40
[passport#69]: https://github.com/midnightntwrk/passport/issues/69
