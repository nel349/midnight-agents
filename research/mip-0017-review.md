# MIP-0017 review: what it means for agents

Prepared for the MIP meeting. Read on 2026-09-14.

---

## Background, for readers new to this

**Shielded coins.** Midnight can hold coins privately. To spend one, the wallet must produce a zero-knowledge proof that it owns the coin.

**Provers.** Making that proof takes heavy computation. A wallet either proves on its own machine, or sends the work to a **hosted prover**, a service that builds the proof for it. Hosted provers matter for phones, light wallets and AI agents, which often cannot prove locally.

**The problem MIP-0017 fixes.** Today the proof needs the coin's **spending key** as an input. So a wallet using a hosted prover must send its spending key to that service. Whoever runs or compromises the prover can then spend the wallet's coins. MPS-0035 describes this problem.

**What MIP-0017 proposes.** A new version of shielded coins ("v2 notes"). The wallet keeps its key and **signs the exact payment** locally. The prover gets the signature, not the key, and cannot change the payment. Old coins ("v1 notes") keep working as before.

**Other terms used below**

- **DUST:** the resource used to pay transaction fees on Midnight.
- **Intent:** a part of a transaction that can carry contract calls, unshielded payments, DUST fee payments and an expiry time.
- **Fee sponsor:** a service that pays someone else's transaction fee by adding its own DUST payment to their transaction.
- **Nullifier:** a public marker published when a coin is spent, so it cannot be spent twice.
- **Viewing key:** a key that lets an app see incoming coins without being able to spend them.
- **Allowance or mandate:** permission for an agent to spend within limits set by a person.

---

## Documents reviewed

- **MIP-0017**, "Signature-Authorized Shielded Spends with VRF Nullifiers", status Proposed, author Ricardo Rius: [mip-0017-shielded-spend-auth.md](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0017-shielded-spend-auth/mip-0017-shielded-spend-auth.md)
- **Supporting specification:** [specification.md](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0017-shielded-spend-auth/specification.md)
- **MPS-0035**, the problem it answers: [mps-0035-shielded-spend-key-exposure.md](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0035-shielded-spend-key-exposure.md)
- **Discussion [#307](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/307):** no comments when read
- **Ledger source** at the revision the specification pins (`67f9f97`): `ledger/src/structure.rs` and `ledger/src/verify.rs`

Line numbers refer to the files on `main` as read on 2026-09-14 (last commit to the MIP folder: `8fb305b`).

**Each gap below has five parts**

- **The problem:** what does not work, in plain words.
- **Why it matters for our agent.**
- **Possible solutions:** ways around it. These are our ideas unless marked as coming from the MIP. Most gaps have no agreed fix yet.
- **Tentative solutions:** two suggestions, one for our agent now, and one for what the MIP could add for everyone.
- **Evidence:** quotes and code that support the problem, with line numbers.

---

## Summary

MIP-0017 solves the key problem for **ordinary shielded payments**. An AI agent spending under a mandate usually also needs to **pay fees**, **call contracts**, and **receive coins from contracts**. MIP-0017 covers none of these, so an agent using a hosted prover would still have to share secrets. It also leaves open how long a signed payment stays valid, how to cancel one, and how a signer checks what it is signing.

| # | Gap | Covered by MIP-0017? |
|---|---|---|
| 1 | Paying fees still needs a secret sent to the prover | No, excluded on purpose |
| 2 | A shorter expiry is hard when a sponsor pays the fee | Partly |
| 3 | Contracts cannot send the new coin type | No, left to a separate change |
| 4 | Secrets used inside contract calls still reach the prover | No, outside its scope |
| 5 | A signed payment cannot be cancelled | No |
| 6 | A signer cannot see what it is approving | Warned about, not solved |
| 7 | Watch-only apps cannot see that a coin was spent | No |
| 8 | Moving old coins needs local proving | No |
| 9 | The prover still sees the payment details | No, stated as a limit |

---

## What MIP-0017 does well

- **The key stays with the wallet.** "The spending key, wallet seed, key shares, and signing nonces MUST NOT enter a v2 proving request." (MIP line 149)
- **The prover cannot change the payment.** The signature covers "the inputs being spent, outputs and change being created, and any contract intents on which the payment depends", and "A prover MUST NOT be able to remove, substitute, or move protected records without invalidating the owner's evidence." (MIP lines 161-164)
- **Old coins keep working.** "Existing v1 notes MUST remain spendable under their original ownership and nullifier relation." (MIP line 327)
- **Shared custody is possible** without rebuilding the key in one place (MIP lines 258-265).
- **Allowances are deliberately out of scope.** "No new allowances, delegate registry, recovery override, or on-chain authorization lifecycle is introduced." (MIP lines 431-432) MPS-0035 says the same (line 100). This is a scope choice, not a gap.

---

## Gap 1. Paying fees still needs a secret sent to the prover

**The problem.** A shielded send is not one proof. It contains several, of three kinds:

| Proof | How many | Needs a secret today? | Changed by MIP-0017? |
|---|---|---|---|
| **Spend proof** for each shielded coin being spent | One per coin spent | Yes, the coin's spending key | **Yes**, the key is replaced by the local signature |
| **Output proof** for each shielded coin being created | One per coin created, usually the recipient's coin and your change | No, it uses the recipient's public key | Updated for the new coin type |
| **DUST spend proof** for the fee | One per DUST output used to pay | Yes, the wallet's DUST secret key | **No** |

MIP-0017 removes the secret from the first kind only. The fee proof still takes the DUST secret key as an input. So a wallet that pays its own fee through a hosted prover still sends that prover a secret. What someone could do with a leaked DUST secret is not described in the MIP.

**Why it matters for our agent.** An agent using a hosted prover is exactly the case MIP-0017 is meant to make safe. If the agent pays its own fees, it still hands a secret to the prover on every transaction.

**Possible solutions**

- **Use a fee sponsor.** The sponsor pays the fee with its own DUST, so the agent's DUST secret is not needed. Sponsors exist today (our `midnight-vs-arc.md` lists 1AM ProofStation and SundaeSwap's Capacity Exchange Service). This creates gap 2.
- **Prove the fee part locally.** Only works where the agent's machine can prove.
- **Apply the same signing approach to DUST.** This would need a new problem statement or proposal. None was found.

**Tentative solutions (our suggestions)**

- **For our agent, now:** Have a fee sponsor pay the agent's fees, so its DUST secret never goes to the prover. Accept the expiry limit in gap 2, and ask whether DUST will get the same signing approach.
- **For the MIP:** The MIP already warns that DUST proving still uses a secret (lines 430-431). Turn that warning into guidance: recommend a follow-up that applies the same signing approach to DUST, and describe the fee sponsor pattern as the interim way to keep a whole transaction key-free.

**Evidence**

- MIP lines 430-431: "Dust is unchanged. Its current proving path still uses its own secret, so v2 shielded key isolation does not make every request key-free."
- Specification lines 674-675: "delegated Dust and v1 proving remain secret-bearing and must not be described as key-free".
- Ledger code: fee payments sit inside an intent, in its `dust_actions` field ([structure.rs lines 873-880](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/structure.rs#L873-L880)).
- Ledger code, the three proof kinds:
  - Each shielded input and each shielded output carries its own `proof` field ([zswap structure.rs lines 214-220 and 305-311](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/structure.rs#L214-L220)).
  - The spend circuit takes `sk: Either<ZswapCoinSecretKey, ContractAddress>`; the output circuit takes only the recipient's public key, the coin and randomness ([zswap.compact lines 34-35 and 82-86](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/zswap.compact#L34-L35)).
  - Each DUST spend carries its own `proof` ([dust.rs lines 469-474](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/dust.rs#L469-L474)), and its circuit takes `sk: DustSecretKey` ([dust.compact lines 73-75](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/dust.compact#L73-L75)).
- Our notes: a sponsor adds its own separate DUST payment intent (`OPPORTUNITIES.md`, midnight-wallet#376).

---

## Gap 2. A shorter expiry is hard when a sponsor pays the fee

**The problem.** When an owner signs a payment, how long can someone use that signature?

- **The upper limit is about one hour.** The signature names a recent state of the coin tree, and the ledger forgets old states after a network setting (`global_ttl`, 3600 seconds by default). After that, the signature no longer works. That limit is a safety net.
- **Expiring sooner needs an intent.** Only intents carry an expiry time. For the owner's signature to respect it, the signature must reference that intent, so the intent must exist before the owner signs.
- **Sponsors add their intent afterwards.** A sponsor's fee intent is added after the owner signs, so the owner's signature cannot reference it. The payment is then limited only by the one-hour window.

**Why it matters for our agent.** An owner may want to hand an agent one pre-signed payment, like a voucher, usable for five minutes. With a sponsor paying the fee, that voucher stays usable for up to an hour, and gap 5 means it cannot be cancelled.

**Possible solutions**

- **The owner adds its own intent with a short expiry before signing.** Not verified: we did not check whether the ledger accepts an intent that carries only an expiry. If that intent also pays the fee, gap 1 returns.
- **The sponsor adds its intent first, then the owner signs.** This reverses how sponsoring works today, where the sponsor pays for an already-built transaction.
- **Give shielded payments their own expiry field.** A protocol change; not proposed anywhere we found.

**Tentative solutions (our suggestions)**

- **For our agent, now:** Do not hand agents pre-signed payments yet. Let the agent spend from the custody contract, where limits are enforced on chain, and treat any signed payment as usable for up to the full window (one hour by default). Expiry inside the contract has its own open issue (`midnight-vs-arc.md`, compact#20).
- **For the MIP:** State the one-hour window as the maximum lifetime of a signed payment. Then either let the signed scope carry its own expiry, or let it reference an intent that carries only an expiry, and explain how this works when a sponsor adds the fee afterwards.

**Evidence**

- Specification lines 364-370 and 625-627: the signed scope names its Merkle root, so "evidence also expires when that root leaves the ledger's retention window. That bound is coarse and network-configured".
- Ledger code: the retention setting `global_ttl` defaults to 3600 seconds ([structure.rs line 1371](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/structure.rs#L1371)).
- Ledger code: expiry (`ttl`) exists only on intents ([structure.rs lines 873-880](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/structure.rs#L873-L880)), and an intent's expiry cannot be more than `global_ttl` after the block time ([verify.rs lines 1731-1750](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/verify.rs#L1731-L1750)).
- Specification lines 621-623: expiry must be "bound through the relevant referenced intent, not enforced only by a wallet timer".
- Specification lines 351-352: referencing another participant's records "presumes those records and their segments are fixed before the digest is computed".

---

## Gap 3. Contracts cannot send the new coin type

**The problem.** The new coins need a new kind of address. Midnight's contract language, Compact, can only create old-type coins when a contract pays a user. A contract cannot pay a new-type address, and trying to do so by hand makes the coin unspendable. The MIP leaves fixing Compact to a separate change, after which contracts must be redeployed.

**Why it matters for our agent.** Our allowance design keeps funds in a custody contract that releases them to the agent. If those funds are shielded, the agent receives old-type coins. Spending old-type coins through a hosted prover still requires sending the key, which is the original problem.

**Possible solutions**

- **Start with unshielded tokens** for the allowance. Whether our first version is shielded or unshielded is still open (`ROADMAP.md`, decision 2).
- **Have the agent prove locally** when it spends coins received from a contract.
- **Ask when Compact will support new-type recipients** and whether it will ship in the same network upgrade. The MIP names this as needed (MIP lines 361-362).

**Tentative solutions (our suggestions)**

- **For our agent, now:** Build the first allowance on unshielded tokens. Move to shielded once Compact can pay new-type addresses.
- **For the MIP:** List the Compact change as a dependency and ship it in the same network upgrade. If that is not possible, document the interim path: contracts pay old-type coins, and the wallet moves them to the new type later.

**Evidence**

- MIP line 358: "Contracts cannot yet pay v2 recipients."
- MIP lines 361-364: "Extending that toolchain is a separate change against which deployed contracts must be redeployed ... Until then a user who receives from contracts keeps a v1 receiving address".
- Specification lines 539-554: the Compact standard library fixes the old recipient format, and putting a new address in that slot makes "the coin unspendable".

---

## Gap 4. Secrets used inside contract calls still reach the prover

**The problem.** MIP-0017 protects only the key used to spend shielded coins. Contracts often check who is calling by asking for a secret as a private input. A hosted prover proving that contract call sees the secret. The MIP does not change contract calls.

**Why it matters for our agent.** One of our allowance options authorises the agent "by secret" (`ROADMAP.md`, Allowance). If the agent uses a hosted prover, that secret goes to the prover, and anyone holding it could act as the agent.

**Possible solutions**

- **Check a signature inside the contract** instead of a secret. Today this means a hand-built JubJub check (`ROADMAP.md`). A cleaner way to know the caller is the subject of MPS-0029.
- **Prove contract calls locally.**
- **Choose the signature option** for our allowance design.

**Tentative solutions (our suggestions)**

- **For our agent, now:** In our custody contract, authorise the agent with a signature check, not a secret.
- **For the MIP:** Add a line under Out of Scope saying contract calls that take secrets are still exposed to a hosted prover, and point to MPS-0029 (caller identity) as the related work.

**Evidence**

- MIP lines 352-353: "Contract execution, including fallible segments and transient coins, retains its current meaning."
- Specification line 83: Midnight.js "forwards serialized preimages to checking/proving endpoints. It does not remove spending secrets from them."
- The connection to contract secrets is our inference; the MIP does not discuss contract call secrets.

---

## Gap 5. A signed payment cannot be cancelled

**The problem.** Once the owner has signed a payment, nothing in the design withdraws that signature. It stops working only when the coin is spent in another transaction, or when it expires (gap 2).

**Why it matters for our agent.** A person who presses "revoke" in the app expects the agent to stop immediately. A payment signed before the revoke could still be used until it expires.

**Possible solutions**

- **Keep standing permission in the custody contract,** which can be revoked on chain, and avoid long-lived pre-signed payments.
- **Cancel by spending the coin** back to the owner first. This costs a fee.
- **Keep pre-signed payments short,** which depends on gap 2.

**Tentative solutions (our suggestions)**

- **For our agent, now:** Make revoke a change to the custody contract, which takes effect on chain, and do not issue pre-signed payments. If one was issued, cancel it by spending the coin.
- **For the MIP:** Add an informative note: issued evidence can only be invalidated by spending the coin or by expiry. This ties to the expiry option in gap 2.

**Evidence**

- Specification line 620: "Issued evidence neither reserves a coin nor creates revocable on-chain permission."
- MIP lines 390-391: "Competing spends still share one nullifier; normal ledger rules determine which, if any, succeeds."

---

## Gap 6. A signer cannot see what it is approving

**The problem.** The owner signs a summary of the payment. That summary is built from hashes of encrypted and hidden data, so it does not show the amount or the recipient. A phone, hardware wallet or policy service asked to sign it cannot tell what it approves unless it is given the underlying details and recomputes the summary. The MIP warns about this but defines no format for passing those details. The protocol for splitting a key between several signers is also not defined.

**Why it matters for our agent.** A natural agent design is a **spending-limit co-signer**: the user holds one share of the key, a service holds another, and the service refuses payments above the limit. That service must see the real amount and recipient, and it needs the shared-key protocol. Neither exists yet.

**Possible solutions**

- **Define a signing package:** the data a signer needs to recompute the summary and see the real payment. MPS-0035 recommends a separate hardware wallet signing MIP (line 118); this would fit there. That MIP has not been written.
- **Until then,** only the party that builds the transaction should sign it.
- **Ask who is writing the shared-key signing protocol.**

**Tentative solutions (our suggestions)**

- **For our agent, now:** The wallet that builds the payment is the one that signs it; no separate co-signer until a signing package and a shared-key protocol exist. Offer to draft the signing package as part of our voucher section.
- **For the MIP:** Define a signing package: the data a signer needs to recompute the signed summary and see the real amount and recipient. It could sit in the hardware wallet signing MIP that MPS-0035 recommends. Name who will write the shared-key signing protocol.

**Evidence**

- MIP lines 374-378: "A device that blindly signs a host-supplied digest can approve a malicious payment without losing its key. Deployments claiming protection against host compromise need trusted user approval or independent policy over the actual recipients, values, network, and conditions."
- Specification lines 335-346: the signed scope is built from digests of commitments, value commitments and ciphertexts.
- Specification lines 664-666: that protection "requires independent reconstruction of the protected records and digest".
- Specification lines 652-654: the shared-key signing protocol remains "unspecified"; MIP line 269: "One-base FROST is not a drop-in protocol for this two-base relation."
- No hardware wallet signing MIP is in the `mips/` folder on `main`.

---

## Gap 7. Watch-only apps cannot see that a coin was spent

**The problem.** To know a coin was spent, an app looks for its nullifier on chain. With new-type coins, only the key holder can compute the nullifier. A viewing key shows incoming coins but not whether they were spent. The key holder would have to give the app a value for every coin received, and no format for sharing that is defined.

**Why it matters for our agent.** The phone app shows what the agent has spent and lets the person revoke. An auditor may also need to see spending. Neither can see spends from a viewing key alone.

**Possible solutions**

- **The key holder shares a per-coin value** with the watching app. This needs a defined format, which the specification leaves out.
- **Track spending through the custody contract's public state** rather than through coins.

**Tentative solutions (our suggestions)**

- **For our agent, now:** The phone app reads the agent's spending from the custody contract, not from coin nullifiers.
- **For the MIP:** Define, or point to a follow-up that defines, how a key holder shares per-coin spend markers with a watch-only app.

**Evidence**

- MIP lines 306-308: "A v2 incoming viewing key can identify receipts without a spending key, but cannot independently derive nullifiers. The key holder can provide per-coin VRF outputs for the wallet to build a nullifier list."
- Specification lines 317-318: "A transport or authenticated batching protocol for this list is outside this specification."

---

## Gap 8. Moving old coins needs local proving

**The problem.** Old coins stay old. To move them to the new format, the wallet spends them once, and that spend uses the old proof, which needs the old key. To keep the key private, that proof must be made on the wallet's own machine.

**Why it matters for our agent.** A wallet that depends on a hosted prover, which is who MIP-0017 is for, can only move its old coins by exposing its old key one last time.

**Possible solutions**

- **Move coins on a device that can prove locally,** such as a desktop, or a mobile SDK that proves on the device (our `midnight-vs-arc.md` lists Kuira).
- **Leave old coins where they are** and spend them only from a device that proves locally.

**Tentative solutions (our suggestions)**

- **For our agent, now:** Tell users to move old coins from a device that proves locally, never through a hosted prover.
- **For the MIP:** Add guidance for wallets that cannot prove locally, for example migrating on a trusted device that can prove, and say what the risk is if the old key is exposed during migration.

**Evidence**

- Specification line 637: "migration uses a locally proven v1 spend and a new v2 output".
- MIP lines 328-330: "A wallet that must keep its old key private produces the v1 input proof locally".
- MPS-0035 line 75 describes phones as unable to run a prover. Our `midnight-vs-arc.md` records at least one mobile SDK that proves on the device, so this affects wallets without local proving rather than all phones.

---

## Gap 9. The prover still sees the payment details

**The problem.** A hosted prover can no longer steal coins, but it still sees the details it needs to build the proof, and it can link requests from the same wallet.

**Why it matters for our agent.** An agent making many small payments through one hosted prover shows that prover its spending pattern.

**Possible solutions**

- **Spread requests across several provers.**
- **Prove sensitive payments locally.**

**Tentative solutions (our suggestions)**

- **For our agent, now:** Say plainly in our docs that a hosted prover sees payment details, and use local proving where the agent's payments are sensitive.
- **For the MIP:** Keep the existing warning, and add guidance: use more than one prover, and measure how much the new public scope data reveals about who contributed what to a merged transaction.

**Evidence**

- MIP line 393: "Delegated proving is non-custodial, not private from the prover."
- MIP line 297: the prover "does see the supplied transaction details".
- MIP line 369 and specification line 662: an observer of proving requests can "link requests".

---

## Goals MPS-0035 set that MIP-0017 does not yet show

| MPS-0035 goal | What is missing | Evidence |
|---|---|---|
| Signing takes only milliseconds on a phone (goal 3, line 89) | No speed measurements. MPS-0035 asked for "a benchmark gate against the v1 circuit" (line 116) | Specification lines 646-648: "no such measurements are claimed here" |
| Signing can happen inside a hardware wallet chip (goal 4, line 90) | The hardware signing MIP is not written; the curve is "not a claim that Jubjub is ... universally supported by secure elements" | MPS-0035 line 118; MIP lines 241-243 |
| No new cryptographic assumptions (goal 6, line 92) | Two building blocks are still undecided: a fixed hash profile and a mapping that gives exactly one curve point per input | Specification lines 686-687; line 69 notes the existing mapping "does not assert a non-identity output" |
| Using the wallet feels the same as today (goal 7, line 93) | A new address type, and users who receive from contracts keep an old address too | Specification lines 258-261; MIP lines 363-364 |

**Privacy note.** New public data in the transaction "may also reveal how participants' contributions were grouped in a merged transaction", and the MIP says "this tradeoff must be explicit" (MIP lines 393-396). How much it reveals is not measured.

---

## Problems in the documents

These links were checked on 2026-09-14 and return "page not found".

| Where | Link as written | Should point to |
|---|---|---|
| MIP lines 55, 409, 437 | `mip-xxxx/specification.md` | `specification.md` in the same folder |
| MIP lines 439-441 | `../mps/...` | `../../mps/...` |
| MIP lines 443-444 | `mip-0005-offer-files.md`, `mip-0006-p2p-atomic-swaps.md` | `../mip-0005-offer-files.md`, `../mip-0006-p2p-atomic-swaps.md` |
| Specification line 21 | `../mip-0017-shielded-spend-auth.md` | `mip-0017-shielded-spend-auth.md` in the same folder |
| Specification line 705 | `../mip-xxxx.md` | `mip-0017-shielded-spend-auth.md` in the same folder |
| Discussion #307 | Specification link with the folder name repeated | The specification file |

Other points:

- **Wrong problem number.** Discussion #307 labels its link "MPS-0017", but the MIP answers **MPS-0035** (MIP line 11). MPS-0017 is a different, open proposal: PR #168, "Governance Observability for Builders".
- **Title mismatch.** The discussion says "Shielded Spend Auth with VRF Nullifiers"; the MIP says "Signature-Authorized Shielded Spends with VRF Nullifiers".
- **Related documents not listed.** The MIP lists MPS-0024, MPS-0016, MIP-0005 and MIP-0006 (lines 12-13). It does not list MPS-0004 ("Trustworthy Delegated Proof Generation for Privacy-Preserving Transactions"), which is about hosted proving itself, or MIP-0012 (native asset custody) and MIP-0013 (account authorisation), where agent spending permissions would plug in.

---

## Questions for the meeting

Short versions to ask in the meeting, or to follow up on afterwards. The gap each comes from is in brackets.

1. **Fees.** If an agent uses a hosted prover, paying its fee in DUST still sends a secret to that prover. Is anyone working on fixing that? (Gap 1)
2. **Expiry.** A signed payment lasts about an hour at most. If a sponsor pays the fee after I sign, how do I make my payment expire sooner? (Gap 2)
3. **Contracts.** When will contracts be able to send coins to the new addresses? (Gap 3)
4. **Contract secrets.** Is there a plan so contract calls don't need to send secrets to a hosted prover? (Gap 4)
5. **Cancelling.** Is there any way to cancel a signed payment other than spending the coin first? (Gap 5)
6. **Checking before signing.** How can a phone, a hardware wallet or a spending-limit service see the real amount and recipient before it signs? (Gap 6)
7. **Shared keys.** Who is writing the protocol for splitting a key between several signers? (Gap 6)
8. **Watching.** How does a watch-only app find out a coin was spent? (Gap 7)
9. **Old coins.** How does a wallet that cannot prove locally move its old coins without exposing its key? (Gap 8)
10. **Speed.** Are there numbers yet for how long signing takes on a phone? (Goals table)
11. **Links.** Several links in the MIP and the discussion are broken, and the discussion says MPS-0017 instead of MPS-0035. Can we send a fix? (Documents)

---

## Where our work fits

`OPPORTUNITIES.md` lists MIP-0017 with this opening: an informative section on a single-use pre-signed payment an owner hands an agent (a voucher). Gaps 2, 5 and 6 are what that section would need to explain: how long a voucher lasts, that it cannot be cancelled, and how the owner checks what they sign.

## Not checked

- The cryptography itself. The MIP says it needs independent analysis (MIP lines 399-403).
- Whether the ledger accepts an intent that carries only an expiry (gap 2).
- What someone can do with a leaked DUST secret (gap 1).
- Anything in the ledger beyond the two files named above.
