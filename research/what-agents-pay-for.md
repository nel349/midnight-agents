# What agents actually pay for, and why we inverted the model

Researched 2026-09-15, then fact-checked against primary sources. Numbers that could not be
confirmed are left out, and self-reported figures say so.

## Why this matters to us

On Arc, the default shape is: a seller exposes data through an API, and an agent pays per call.
We scanned Circle's Discovery catalogue on 2026-09-02, all 1,247 resources, and found it was data
and financial analysis, three providers holding 45% of listings, prices around a cent, and nothing
accepting Arc at all.

So we inverted it. Instead of buying data, our agent pays to **do something**: solve a maze. The
seller supplies the challenge, the verdict, reputation and a badge. This note is the evidence that
the inversion was the right call, and where else it applies.

## The listings are mostly dormant

- **An independent index of the Coinbase Bazaar registry** (15,147 listings, 1,604 hosts,
  [x402-bazaar-explorer](https://github.com/nunojsferreira/x402-bazaar-explorer), 2026-08-25) found
  that in 30 days only **240 listings, 1.6%, had 10 or more distinct payers**. 8.7% saw 10 or more
  calls. 0.2% saw a thousand. Total traffic across everything: 304,282 calls from 42,986 payers.
  Self-published, and it covers listed services, not the protocol as a whole.
- **Visa and Artemis cleaned the volume** in
  [Agentic Payments from the Ground Up](https://www.visa.com/en-us/thought-leadership/innovation/agentic-payments-from-the-ground-up)
  (2026-07-14, data as of 2026-04-21): raw activity of $135.7M across 178.3M transactions filters
  down to **$15.0M across 109.6M transactions**, discarding 89% of the dollars and 39% of the
  transactions as wash, test or internal transfers. Lifetime: about 422,000 buyers and **5,300
  sellers**. The filtering method is proprietary and not reproducible.
- **The headline counter is frozen.** [x402.org](https://www.x402.org/) has shown the same four
  figures (75.41M transactions, $24.24M, 94.06K buyers, 22K sellers) since at least early January
  2026, under a heading that reads "Last 30 Days".
- **Volume has fallen hard.** x402scan's own dashboard read about $1.06M a day in mid-November
  2025; the current 30-day average is about $45K a day.

**What this does not say:** the protocol works, settlement is real, and a handful of sellers do
well. The top earner is around $9,400 a day. The point is the shape of the distribution, not that
nobody is paid.

## What people do pay for

Ranked by the evidence available. No clean taxonomy exists, since most listings carry no useful
category.

1. **Data, search and enrichment APIs.** Most calls by volume.
2. **Inference and model routing.** Largest payments.
3. **Crypto and DeFi analytics.** The one segment with evidence of high-value repeat buyers.
4. **Web automation and unblocking.**

Chainalysis noted the composition shifted through 2026: payments over $1 went from about half of
volume to nearly all of it. The sub-cent micropayment thesis is not what is being paid.

## The inverted model, where it already exists

Three shapes are live. **None of them use x402 or ERC-8004**; all are plain contract calls.

**Stake to be evaluated, and lose the stake if you are wrong**
- **Numerai.** Stake at least 0.1 NMR a round; negative scores are burned, not redistributed.
  About 520,000 NMR staked, by 677 of 2,782 active accounts. On-chain staking since round 1343,
  2026-08-28 ([docs](https://docs.numer.ai/numerai-tournament/staking), figures from Numerai's own
  API). This is the closest precedent to what we built.
- **Bittensor.** Registering a miner burns a floating price set at execution time, currently around
  847 TAO on the main network. Creating a subnet locks a minimum that starts at 1 TAO and doubles
  with each registration, then decays.

**Pay per attempt into a prize pool**
- **Freysa.** A jailbreak game where each message costs, starting at $10 and rising to a cap.
  Parameters changed per round, so no single set of numbers describes it. The organisation is
  alive, but we found no evidence of a live paid round in 2026.

**The agent's own wallet pays, continuously**
- **Olas.** The only case where the agent itself pays: a live 15% protocol fee on agent-to-agent
  payments, confirmed on chain. Be careful with their headline counts: "transactions" means every
  on-chain transaction by an agent, not payments. Lifetime marketplace turnover is about $109,500
  and lifetime fees about $790, which is roughly three quarters of a cent per agent-to-agent
  transaction.

**Entry-fee arenas** exist (Recall, Prizefight) but publish no fee amounts.
[x402 Arena](https://x402arena.gg) publishes its numbers and they are small: 617 agents, 81 buyers,
$109.38 of revenue, with no listed entry fee.

**Watch the direction of payment.** Most "AI arenas" run the other way: the house pays the
competitor. Alpha Arena funded each model itself; jailbreak bounties, bug bounties and red-team
programmes all pay the attacker. Those are not precedents for an agent paying to enter.

## Markets for a judgement about an agent

Thinner than it looks.

- **ERC-8004 is still Draft**, and its spec says plainly that "payments are orthogonal to this
  protocol". Identity and reputation registries are deployed across about 20 chains;
  **there is no canonical deployment of the validation registry**, which is the part that would
  carry a paid verdict. One reference implementation exists on a testnet.
- **The money in judgement is insurance and certification, and it is human-mediated.** AIUC has
  raised $55M in total for auditing and insuring agents; Armilla sells cover for "AI agent
  mistakes" as a Lloyd's coverholder; Testudo launched in January 2026 with generative-AI
  liability cover. Crypto-native insurance has nothing: neither Nexus Mutual nor OpenCover offers
  agent cover.
- **Paid red-teaming exists but sells to labs, not to agents.**

## Adjacent areas, and who the counterparty is

| Area | Why an agent pays | Counterparty |
|---|---|---|
| Data mining and labelling | It does not. The agent **earns**; it pays only a stake or bond to be trusted | Task market, in the Bittensor shape |
| Crawling gated content | Access it cannot otherwise get | Publishers, through Cloudflare's pay-per-crawl (closed beta) and its newer monetisation gateway, which settles over x402 |
| Synthetic data | Cheaper than licensing real data | Generator, or another agent |
| Evaluation sets | Needs problems it has not memorised | Benchmark owner |
| Prediction markets | Pays a fee to express a belief and be scored by reality | Venue |
| Games and contests | Entry to a scarce, contested outcome | The host. **This is our maze** |
| Matchmaking | To be introduced to a counterparty it can trust | Registry or broker |
| Compute auctions | To win a slot under a deadline | Compute market |
| Escrow between agents | Settlement it cannot enforce itself | Escrow contract |

## Why the inversion is structurally sounder

**Proving delivery is the hard part of the data model.** An API can return plausible garbage and
the buyer often cannot tell. That is consistent with what the index shows: most listings are never
called twice, and nobody notices when they stop answering.

In the inverted model, **the agent knows whether it won**. Verification is intrinsic to the
outcome rather than a claim about content. The risk moves to the other side: a cheating player,
replayed solutions, sybil entrants farming a pool, a host draining its own prize. Those are
handled with commit-reveal, per-attempt nonces and rate-limited identity, not with an oracle.

Prize pools are self-proving. Reputation scores are not, which is why every free reputation
registry has no skin in the game.

## Where privacy is load-bearing

Ranked for a privacy chain specifically. The question in each case is whether privacy is the
product or a nice-to-have.

1. **Contests with hidden state:** mazes, puzzles, capture-the-flag. If the instance is public, the
   contest is trivially solved or front-run. A shielded chain lets a host commit to an instance
   without revealing it, then settle a verified win. **The only category where privacy is the
   product.** This is what we built.
2. **Stake to be evaluated, privately.** Numerai's model with confidential submissions: the score
   is public, the strategy is not. That is what would make a lab willing to be benchmarked.
3. **Blind agent-versus-agent markets:** prediction, trading, auctions. Privacy stops copy-trading
   and extraction against a known agent's positions. Olas shows agents will pay to participate.
4. **Certification with confidential evidence.** Prove you passed a red-team suite without
   publishing your failures. Nobody buys this on-chain today, so it is a bet, not a market.
5. **Sealed-bid task and compute markets.** Privacy hides reserve prices and bidding strategy.
   Real, but the benefit is ordinary auction theory rather than anything agent-specific.
6. **Escrow between agents.** Confidential terms, public settlement. Small today, but a primitive
   the ones above need.

## Left out on purpose

- A widely repeated claim that a probe of 78,000 endpoints found only 21% answering. The numbers
  in it are contradicted by the live registry, which lists about 16,000 resources in total.
- A claim that the main agent-reputation company wound down in mid-2026. Both its sites are live
  and its repositories are unarchived, so we could not support it.
- Freysa's per-round economics, which differ per round and could not be confirmed.
