# Roadmap: becoming the person who brings agents to Midnight

Written from our side: contributors getting into the Midnight ecosystem, using what we already
proved on Arc. Concepts and goals only; details belong in the proposals and code.

## The strategy in four sentences

Midnight's own team (Passport) is converging on the same design as the Arc app from the wallet side:
grants, fees, passkeys, wallet connection. On the **agent** side the work so far is paused or purpose-built for one product (PR #251 is blocked on commit signatures, PR #110 is waiting on the editors, Shade402 enforces its limits in its own backend), so no chain-enforced standard exists yet: spending policy an
agent obeys, pay per call, sponsorship for an agent that holds nothing, identity. We have the rarest
evidence for that side: a working agent allowance on Arc, the only MCP entry on the community wallets page,
and the only mobile SDK documented as proving on the device. So the play is not a rival product: it
is to write the problem statements only we can evidence, co-author the solutions with the people who
own the neighbouring proposals, and prove each one with a reference built from our existing tools,
leaving something public with our name on it at every phase.

## Principles

- **Problem before proposal.** When no problem statement exists, editors ask for one first, in its
  own PR (#110). It is also where the Arc evidence lands most naturally.
- **Coordinate, do not compete.** Passport's plans cover grants, fees, x402, P-256 and wallet
  connection. Talk to them before filing on any of those. Talk to the authors of #251, #110 and D218
  before writing alternatives.
- **Prove on Midnight, not on paper.** Each proposal ships with a reference that runs on Preview. No
  public testnet is confirmed to run ledger 9, so assume ledger 8 forms.
  The live networks pin toolchain 0.31.1 on ledger 8; releases have moved on (0.34.0, ledger 9 release candidates, a ledger 10 alpha) but are not deployable there yet.
- **Right channel.** Standards go to MIPs, language features to CoIPs, modules to OpenZeppelin, and
  tools to awesome-dapps and the docs.
- **Mechanics.** Sign commits, sign the CLA, and name a human as author.

## Phase 0, Show up (September 2026)

Cheap, visible and useful on its own. It also builds the relationships the later phases need.

- **Review work with nobody on it yet:**
  - **[MIP-0015][]** (Proposed; its Discussion #282 has zero comments). Bring Kuira's evidence that
    passkey PRF output is locked to one platform.
  - **PR #306**, message signing (opened 09-10).
- **Attend the 16 September editor session** on [MIP-0011][]. It is the last entry on `ROLLING_AGENDA.md`, so ask there when the next one is.
- **Retest the block-time bug** ([midnightntwrk/compact#20][]) on the current toolchain and post the
  result. Nobody has posted one. Separate an expected rejection caused by the prover's clock
  differing from the chain's (test with a margin) from the lost-state failure. A third party reported
  the same failure on mainnet on 2026-07-07. Allowance expiry depends on the answer.
- **The DUST-only sponsor helper** ([midnight-wallet#376][]): a maintainer declined to add it to the SDK.
  Back the reporter's feature-request follow-up with a concrete use, or ship the helper in
  midnight-wallet-cli.
- **Open a conversation with Passport** on:
  - C10, the grant format
  - C24, the fee model
  - C23, wallet connection
  - whether they have a P-256 slot at all (their tracking issues name a signing primitive and client-side passkeys, not P-256)
  - [passport#40][], x402

  Offer the Arc agent evidence and ask where a co-author helps.
- **Review PR #251 and PR #110** in their threads, as review rather than rival proposals.

**Done when:** our name appears in the MIP repo, a core SDK repo and the Passport conversation.

**Timing risk:** Passport's MVP lands in October 2026, so the co-authoring conversation is worth having before their design freezes.

## Phase 1, Name the problems (October 2026)

The problem statements the Arc work evidences, each in its own PR:
1. **Agents need bounded, expiring, revocable spending authority** that the chain enforces (N1).
   This is the anchor; everything else refers back to it.
2. **Agents need to pay per call**, the x402 problem (N3). It can include [MPS-0004][]'s case of paying a
   prover per proof.
3. **An agent that holds nothing needs a sponsor.** Either a new statement or an extension of
   [MPS-0031][] (N4).
4. **Passkey signatures cannot be verified on Midnight** (N5), with Discussion #244 and Kuira's PRF
   finding as evidence. If Passport has a P-256 slot (unverified), make this a joint statement or
   evidence for theirs, not a rival.

**Done when:** at least the first is merged as Proposed, and Passport has reviewed it. Note that
editors merge problem statements as Proposed, so merging is filing, not acceptance. The stronger bar
is a second reader outside our own circle taking it up, which is what Phase 3 needs anyway.

## Phase 2, Prove it on Midnight (October to November 2026)

A spike on Preview that turns the problem statements into evidence, not a product.

- **Allowance:** a custody contract holds funds; an agent spends within a cap and before an expiry;
  the owner revokes. Use forms available on today's networks. Prefer the hand-built JubJub signature
  check over a secret witness: a secret passed as a private input is visible to a hosted prover
  (`research/mip-0017-review.md`, gap 4). Expiry goes in once the block-time retest shows how to use it safely.
- **Agent side:** midnight-wallet-cli reads the grant from chain instead of enforcing local caps.
- **Fees:** the agent holds no NIGHT; its fees come from DUST designation or a sponsor. The official
  flow has a working reference: the DUST sponsorship guide and `scripts/sponsor-service.ts` in
  `midnightntwrk/example-private-party`. Designation
  still means the agent's own DUST secret goes to whoever proves the transaction, so a sponsor is the
  only fully key-free route today (`research/mip-0017-review.md`, gap 1).
- **Pay per call:** one seller, one paid request, settled through the allowance, including how the
  agent's escrow is topped up under the same limit (N3, the comparison table, row 14). Price it in USDM,
  which is live on Preview and mainnet through VIA Labs, if its token form fits the custody
  contract; otherwise in NIGHT. USDM on Midnight is unshielded. Explore whether signed vouchers
  settled in batches beat one on-chain payment per call, with the caveats from
  `research/mip-0017-review.md`: a pre-signed payment lasts about an hour at most and cannot be cancelled.
- **Pairing:** bind the agent to the allowance without contract events (N16, the comparison table, row 13). It is
  the first thing an implementer hits after the allowance itself.
- **Rails:** decide which rails the allowance meters (unshielded, shielded, DUST) and close the rest.
  Note that a contract cannot pay the new shielded coin type at all (`research/mip-0017-review.md`, gap 3),
  so contract-released funds stay on the old type.
- **Phone:** grant and revoke from Kuira, if the Kuira side is ready. Otherwise from the CLI, with
  the phone as a stated follow-up.

**Done when** the chain itself refuses:
- a spend over the limit
- a spend after expiry
- a spend after revoke
- a spend by anyone but the agent

And the agent never held NIGHT. State plainly what revoke does not do: it cannot recall funds the
agent already holds or already moved into an escrow. Record every failure mode found, as the Arc run did; those notes
become the proposals' security sections.

## Phase 3, Propose the solutions (November 2026 to Q1 2027)

- **Scoped grants:** co-authored with Passport as the [MIP-0013][] extension. Or, if they prefer, as a
  separate Standards MIP on the [MIP-0012][] seam.
- **x402 on Midnight:** a scheme in the x402 spec, plus a Midnight-side MIP for the payment format if
  editors want one. Consolidate what m402, Shade402 and x402midnight learned.
- **Sponsorship interface:** a Standards MIP, with the Phase 2 sponsor as reference.
- **[MIP-0002][] amendment:** delegation and identity event types, for ledger 9. Not cheap: an event-type
  amendment needs a ledger release and a serialization tag bump, so coordinate with the ledger owners.
- **Caller identity:** [MPS-0040][] (merged 2026-09-16 UTC) now covers cross-contract call provenance, so
  review and extend it for the agent case rather than writing the `kernel.caller()` proposal from
  scratch.
- **Wallet pairing and permission** (N6): folded into Passport's connection protocol (C23) if they
  agree, rather than a separate MIP.

**Done when:** the grant and x402 proposals are merged as Proposed, each with a public reference
implementation on a public network, and each reviewed by someone outside our own circle. The strong proposals (0012, 0013, 0015, 0003) set a
higher bar worth aiming at: a second independent implementation, published test vectors, a public
testnet deployment, and cryptographer review.

## Phase 4, Make it reusable (Q1 2027)

- An **OpenZeppelin scoped-allowance module**, upstreamed. Build on their JubJub verifier work
  (issues #548, #647, #648) rather than a parallel one.
- A **grant-aware agent wallet** in midnight-wallet-cli, listed in the docs as the safe agent path.
- **Kuira** support for grant, pair and revoke. Then the **mobile support** MIP (N12), with Kuira and
  a React Native binding as evidence.
- **midnight-js runtime declaration** (#1213) to include React Native honestly.

**Done when:** the module is released by OpenZeppelin, the grant-aware wallet is published and
listed, and the mobile proposal is filed.

## Phase 5, Identity, reputation, badges, results (2027)

Once spending works, the rest of the Arc system follows, with less competition:
- Help **MAIS** (#110) become a Compact-accurate ERC-8004-compatible registry. Its identity registry
  already has an aligned implementation (midnight-agent-did-manager). Build what is missing:
  reputation, validation and the bridge. Relate it to Midnight's reference `did:midnight`
  implementation.
- **Reputation** where the evidence is a paid receipt, so feedback cannot be self-issued.
- **NFT and soulbound badge** standard with the D218 author.
- **Checking off-chain results** in a contract, starting with signed attestations and moving to
  enclaves once P-256 or RSA verification exists.

**Done when:** an agent identity registry and a reputation record exist on a public network, written
by a seller and readable by a buyer.

## What is blocked on Midnight itself

These are not ours to fix, but the roadmap has to route around them:

| Blocker | Affects | Route around |
|---|---|---|
| Ledger 9 not on mainnet (contract calls, events, secp256k1 checks). It is at release-candidate stage, with a ledger 10 alpha published 2026-09-14 | Composable allowances, event-driven feeds, agent pairing | Build on ledger 8 forms first; write proposals for ledger 9 |
| No P-256 on any network. Proof-backend support merged on `ledger-9` (ledger PR #655); Compact PRs open (LFDT-Minokawa/compact #749, #750) | Passkeys signing directly | Passkey → PRF → derived key (Passport: JubJub device key; Kuira: wallet seed) |
| No native stablecoin; shieldUSD not live | Dollar-priced x402 | Use USDM from Cardano (live since 2026-08-13, issuer-minted by Moneta over VIA Labs messaging, unshielded) |
| Mainnet deployment permissioned; a score of 3 in any category triggers a block, and value tier 3 is long-term vaults, pools and treasuries | Shipping to mainnet | Per-user custody with bounded-time funds may argue value tier 2, which is scored but does not trigger the block, and it needs expiry to work. Stay on Preview until then |
| Block-time bug report ([midnightntwrk/compact#20][]) | Expiry | Retest first. If it fails, check expiry with a signed time quote, which on ledger 8 needs a hand-built JubJub verifier |

## Open decisions for us

1. **Lead or co-author on grants?** Filing N1 ourselves is faster. Co-authoring with Passport is more
   likely to be accepted and more visible to IOG. They intend to adopt upstream formats (C22), so a
   community standard could become the thing they adopt (our reading).
2. **Shielded or unshielded first?** Unshielded matches Arc, is simpler to verify, and is what a
   custody contract can actually pay out today: under [MIP-0017][] a contract cannot pay the new shielded
   coin type (`research/mip-0017-review.md`, gap 3). Shielded remains the stronger story, later.
3. **Where the spike lives:** a new Midnight repo, or inside midnight-wallet-cli and Kuira.
4. **How much of Phase 2 waits for Kuira iOS,** versus doing the phone side on Android only. The iOS
   SDK exists but is early; Android is far ahead.
5. **Lead or co-author, decided before Passport's October MVP** freezes their design.

## Checklist

- [ ] Phase 0: [MIP-0015][] review, PR #306 review, [compact#20][] retest, DUST sponsor helper, Passport conversation, next editor session
- [ ] Phase 1: agent spending authority MPS merged
- [ ] Phase 2: the chain refuses over-limit, expired, revoked and wrong-agent spends on Preview
- [ ] Phase 3: grant and x402 proposals Proposed, with references
- [ ] Phase 4: OpenZeppelin module and grant-aware agent wallet published
- [ ] Phase 5: identity, reputation and badges standardised

<!-- links -->
[MIP-0002]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0002-public-contract-log-emission.md
[MIP-0011]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md
[MIP-0012]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0012-native-asset-custody.md
[MIP-0013]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0013-account-authorisation.md
[MIP-0015]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0015-wallet-derived-deterministic-secrets.md
[MIP-0017]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0017-shielded-spend-auth/mip-0017-shielded-spend-auth.md
[MPS-0004]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0004-trusted-proof-serving.md
[MPS-0031]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0031-dust-grant-dao.md
[MPS-0040]: https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0040-cross-contract-call-provenance.md
[compact#20]: https://github.com/midnightntwrk/compact/issues/20
[midnight-wallet#376]: https://github.com/midnightntwrk/midnight-wallet/issues/376
[midnightntwrk/compact#20]: https://github.com/midnightntwrk/compact/issues/20
[passport#40]: https://github.com/midnightntwrk/passport/issues/40
