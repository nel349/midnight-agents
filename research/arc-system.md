# 01 — The Arc side: what Arc/Circle provide for agents, and what Agent Mandate actually used

Spike input for "what would it take to build an agent-spending system like our Arc one on Midnight".
Read-only inventory, compiled 2026-09-13.

**Sources read**
- Internal planning notes for the Arc build (not public).
- App repo `arc-agent-mandate`: `docs/FINDINGS.md`, `docs/ARCHITECTURE.md`, `docs/CONTRIBUTION.md`, `mcp/README.md`, `package.json`, `mcp/package.json`, and code structure of `contracts/`, `mcp/`, `src/arc/`, `src/passkey/`, `modules/arc-passkey/`, `integration/`.
- Seller repo `arc-maze`: `README.md`, `cre/README.md`, `package.json`, `src/arc/*`, `contracts/src/*`, `cre/verdict/workflow.ts`, `cre/evidence/simulation-25b9f044.txt`.
- Web (fetched 2026-09-13): docs.arc.io (`/build`, `/llms.txt`, agentic-economy, account-abstraction, MCP, opt-in-privacy, register-your-first-ai-agent, create-your-first-erc-8183-job, contract-addresses, gas-and-fees, evm-differences, oracles, data-indexers, monitor-contract-events, sample-applications) and developers.circle.com (modular wallets, how-passkeys-and-modules-work, gas-station, paymaster addresses, gateway/nanopayments, batched-settlement, buyer quickstart, agent-stack/agent-wallets, supported-blockchains, custom-policies, circle-cli, discovery-api), circle.com/agent-stack, docs.chain.link CRE confidential workflows (search result).

Note on fetches: web pages were read through a summarising fetcher. Where a summary said "not mentioned", that is recorded as "not found on that page", not as proof of absence.

**Status legend:** USED = in our shipped code and proven on Arc testnet; USED (sim) = built and run only in a simulator; NOT USED = available, we did not use it; UNVERIFIED = claim I could not confirm from a primary source in this pass.

---

## One-paragraph shape of the Arc system (for orientation)

A person's wallet is a **Circle Modular Wallet** (ERC-4337 v0.7 + ERC-6900 v1 plugin account) owned by a **passkey** on their phone (React Native app with our own Expo passkey module). An allowance is a **session key** installed through **our port of Alchemy's ERC-6900 SessionKeyPlugin** to EntryPoint v0.7 (spend limit on the USDC ERC-20 view, expiry, allowlist with selectors). The agent (Claude Code etc.) runs our **MCP connector**, holds its own **EOA key with no money**, pairs to the phone via QR + on-chain `SessionKeyAdded` tag, and pays **x402** sellers through **Circle Gateway nanopayments** (EIP-3009 authorisations under the `GatewayWalletBatched` domain). Because Gateway only accepts EOA signatures, the account **tops up the agent's Gateway escrow via `depositFor`** in a **Circle-sponsored** user operation per purchase. The seller (**arc-maze / "Toll"**) verifies+settles through Circle's `BatchFacilitatorClient`, checks the payer against the agent's **ERC-8004 identity** (`getAgentWallet`), writes **ERC-8004 reputation** with a run digest, and mints a **CohortZero ERC-721 badge** to the identity owner. A **Chainlink CRE confidential workflow** (declared for AWS Nitro TEE) replays the run and produces a DON-signed verdict for a `MazeVerdict` contract — simulator only, contract not deployed.

---

## 1. Accounts & signing (smart accounts, passkeys/P-256, mobile/React Native)

**What Arc/Circle provides**
- Arc itself provides no account product; `docs.arc.io/build` lists "Account Abstraction — smart wallets, paymasters, and session keys from ecosystem providers" and delegates to a vendor directory: https://docs.arc.io/arc/tools/account-abstraction (Alchemy, Biconomy, Blockradar, Circle Wallets, Crossmint, Dynamic, MetaMask Embedded Wallets, Para, Pimlico, Privy, Thirdweb, Turnkey, ZeroDev — all ERC-4337).
- **Circle Modular Wallets**: passkey (WebAuthn, secp256r1/P-256) owned MSCA smart accounts, gasless via Gas Station, batch + 2D-nonce parallel execution, recovery keys, modules per ERC-6900 (documented modules: Passkey signer and Address Book allowlist). SDKs for **Web, iOS, Android** — React Native not listed. https://developers.circle.com/wallets/modular , https://developers.circle.com/wallets/modular/how-passkeys-and-modules-work
- Arc testnet is first-class in `@circle-fin/modular-wallets-core` (`/arcTestnet` transport path, `ContractAddress.ArcTestnet_USDC`); iOS native SDK already defines Arc mainnet chain id `5042`, Android does not (`arc-sdk/GAP_INVENTORY.md`).
- Circle "Programmable Wallets" legacy line (PIN/social) does have a React Native SDK (`circlefin/w3s-react-native-sdk`) but is not passkey / not ERC-4337 (`GAP_INVENTORY.md`).
- Arc EVM supports EIP-7702 set-code transactions (https://docs.arc.io/arc/references/evm-differences). A P-256 verify precompile (EIP-7951 at `0x100`) is asserted in `arc-sdk/ARCHITECTURE.md`; Arc's baseline is Osaka, which includes EIP-7951, but I did not confirm the precompile on Arc directly — **UNVERIFIED**.

**What we used, and how** — USED
- `@circle-fin/modular-wallets-core ^1.0.15` + `viem ^2.45.3` + `webauthn-p256`: `toPasskeyTransport`, `toWebAuthnCredential`, `toModularTransport`, `toCircleSmartAccount`, viem `toWebAuthnAccount({credential, rpId})`, `createBundlerClient({paymaster: true})` — `arc-agent-mandate/src/arc/account.ts`.
- Our own Expo native passkey module: Swift `ASAuthorizationPlatformPublicKeyCredentialProvider` (`modules/arc-passkey/ios/ArcPasskeyModule.swift`) and Kotlin Credential Manager (`modules/arc-passkey/android/src/main/java/com/arcmandate/passkey/ArcPasskeyModule.kt`).
- TypeScript shim installing `navigator.credentials` so Circle's SDK runs unchanged, CBOR/COSE attestation → SPKI public key parsing, base64url, `crypto.subtle` shim — `src/passkey/shim.ts`, `cose.ts`, `subtle.ts`, `native-json.ts`.
- Account: ERC-6900 v1 MSCA, `circle_6900_v1`, factory `0x0000000DF7E6c9Dc387cAFc5eCBfa6c3a6179AdD`, owner plugin `WeightedWebauthnMultisigPlugin` `0x0000000C984AFf541D6cE86Bb697e68ec57873C8`, EntryPoint v0.7 `0x0000000071727De22E5E9d8BAf0edAc6f37da032` (`docs/ARCHITECTURE.md`, `GAP_INVENTORY.md`).
- Offline counterfactual address derivation pinned by test: `integration/rp-independence.test.ts`, `integration/counterfactual-account.test.ts`.
- The agent side does **not** use a smart account; it is a plain viem `PrivateKeyAccount` stored at `~/.arc-mandate/agent.key` (`mcp/README.md`).

**Rough edges we hit**
- Modular Wallets not supported on React Native; the WebAuthn ceremony is the one OS-deep piece that does not cross (`README.md`, `GAP_INVENTORY.md` §5, `CONTRIBUTION.md` §2).
- Native `ASAuthorization` returns only the CBOR attestation object; Circle's SDK calls browser-only `response.getPublicKey()` → we had to write attestation → COSE → SPKI parsing (`GAP_INVENTORY.md` "Gaps in the chosen path" §1).
- `rpId` must be pinned explicitly: ox's `WebAuthnP256` defaults it to `window.location.hostname` (Metro dev server on RN), so registration succeeds and every later assertion fails (`arc-agent-mandate/docs/FINDINGS.md` §7, `src/arc/account.ts` comment).
- Circle's `X-AppInfo` header derives `uri=<hostname>` and rejects `localhost`/`unknown`; we rewrite that one header on Circle calls to the passkey domain (internal planning notes, `src/passkey/shim.ts:317-356`).
- Circle is the WebAuthn RP for registration; verified the account is still reachable without Circle (address is CREATE2 over published constants + credential pubkey; Circle's API returned a byte-identical address). Obligation: app must persist credential id + public key (`arc-agent-mandate/docs/FINDINGS.md` §7).
- `installPlugin` unreachable except through a user operation (no runtime validation on the passkey multisig plugin) (`arc-agent-mandate/docs/FINDINGS.md` §4).
- Circle native SDKs have distribution problems (Android on GitHub Packages needing a PAT, iOS SPM-only) — reason native-wrapping was rejected (`GAP_INVENTORY.md` §5).
- Expo Go cannot do passkeys; dev builds required; Associated Domains / Digital Asset Links must match; Android release signing key fingerprint must be in assetlinks (`GAP_INVENTORY.md`, `IMPLEMENTATION.md` standing risk 4).
- Passkeys tie to our Apple team/domain, so judges cannot build and create a wallet themselves (#41, `IMPLEMENTATION.md`).
- Modular Wallets is **Arc testnet only** (no Arc mainnet entry yet) (`GAP_INVENTORY.md`).

**Standards:** ERC-4337 (EntryPoint v0.7, `PackedUserOperation`), ERC-6900 (v1 "plugin" generation, superseded by the modules draft), WebAuthn Level 2/3, COSE (RFC 8152/9053), CBOR (RFC 8949), secp256r1 / ES256, CREATE2 (EIP-1014), EIP-7951 P256VERIFY (UNVERIFIED on Arc), EIP-7702 (supported on Arc, not our path).

---

## 2. Delegated, bounded authority (session keys, spending limits, expiry, revocation)

**What Arc/Circle provides**
- `docs.arc.io/build` names session keys as a priority AA primitive "from ecosystem providers": https://docs.arc.io/build . Directory lists session keys for **Alchemy** and **ZeroDev** only: https://docs.arc.io/arc/tools/account-abstraction
- The only ERC-6900 session-key plugin deployed on Arc is Alchemy's at `0x0000003E0000a96de4058e1E02a62FaaeCf23d8d`, built for EntryPoint **v0.6** (`arc-agent-mandate/docs/FINDINGS.md` §1).
- **Circle Modular Wallets SDK has no session keys**: grep of type definitions for `session`, `sessionKey`, `installValidation`, `spendLimit` → zero hits; ownership types are only `{EOA, WebAuthn}`. Documented modules are Passkey + Address Book (`GAP_INVENTORY.md` "Session keys"; https://developers.circle.com/wallets/modular/how-passkeys-and-modules-work).
- **Circle Agent Wallets** (Agent Stack) have spending policies: per-tx ≤ daily ≤ weekly ≤ monthly USDC limits, recipient and contract allow/blocklists; set via `circle wallet limit set …`; a policy change triggers a **second email OTP**; **policies are mainnet-only, testnet unsupported** — https://developers.circle.com/agent-stack/agent-wallets/wallet-operations/custom-policies . Marketing page also claims "time-bounded sessions built in": https://www.circle.com/agent-stack (not found on the policies page — **UNVERIFIED** as a shipped feature).
  - **Discrepancy to flag:** our `PRODUCT.md` (written 2026-09-02) calls the Agent Wallet custodial (email+OTP). Circle's current page describes it as "non-custodial MPC… 2-of-2 MPC key management, key shares are never exposed to the agent", built on user-controlled wallets: https://developers.circle.com/agent-stack/agent-wallets . Either Circle changed positioning or our earlier read was wrong; treat "custodial" in our docs as **UNVERIFIED/possibly stale**.
  - Agent Wallets support Arc Testnet (`ARC-TESTNET`) as a chain (https://developers.circle.com/agent-stack/agent-wallets/supported-blockchains), but since policies are mainnet-only and Arc is testnet-only, an Arc agent wallet today has **no enforceable limits**.

**What we used, and how** — USED
- Ported Alchemy `SessionKeyPlugin` (GPL-3.0) to Circle's ERC-6900 v0.7 interfaces: `arc-agent-mandate/contracts/src/session/SessionKeyPlugin.sol`, `permissions/*.sol`, every change in `contracts/src/session/PORTING.md`. Deployed via CREATE2 at `0x669Dd1eDb85ABD00f74186d88124614EE81E6670` (`docs/ARCHITECTURE.md`; deploy script `contracts/script/DeploySessionKeyPlugin.s.sol`).
- Grant/revoke/inspect from the phone: `src/arc/mandate.ts` — `installPlugin`, `addSessionKey(sessionKey, tag, permissionUpdates)`, `removeSessionKey`, `updateKeyPermissions`, `setERC20SpendLimit`, `setNativeTokenSpendLimit` (set to 0), `setGasSpendLimit` (effectively unlimited, #13), `updateTimeRange(validAfter, validUntil)` for expiry, `setAccessListType`, `updateAccessListAddressEntry(..., checkSelectors)`, `updateAccessListFunctionEntry`.
- The grant is an **allowlist with selectors**: USDC view `transfer`, `approve`; GatewayWallet `depositFor`; ERC-8004 Identity `register()`, `setAgentWallet(...)` (`src/arc/mandate.ts:161-190`, `ESCROW_CALLS`, `IDENTITY_CALLS`).
- "One meter": spend capped on the USDC ERC-20 view only, native limit zero (`contracts/test/ArcOneMeter.t.sol`; `README.md` decision 5).
- The agent signs user operations with the session key; operation built by viem, only the signature is ours: `mcp/session-account.ts`, `mcp/spend.ts`.
- Pairing: one-time code hashed into the `SessionKeyAdded` `tag`, agent scans for its own grant: `mcp/pairing.ts`, `mcp/chain.ts`.
- Tests on Arc fork against real Circle account + EntryPoint: `contracts/test/Arc*.t.sol`, `integration/allowance.test.ts`, `integration/mandate.test.ts`.

**Rough edges we hit**
- Deployed plugin is EntryPoint v0.6, Circle accounts v0.7 → different `userOpValidationFunction` selector and `IPlugin` interface id; the obvious workaround (dependency pointing at the account) makes install **succeed** and then fails on first spend (`arc-agent-mandate/docs/FINDINGS.md` §1, internal planning notes 1). Largest single piece of work.
- `FunctionReference` is a struct `(address,uint8)` in Circle's `installPlugin` (selector `0xf85730f4`), not Alchemy's packed `bytes21` (`arc-agent-mandate/docs/FINDINGS.md` §5).
- Runtime owner dependency had to be removed (passkey multisig has no runtime validation) (`PORTING.md`).
- Empty `ALLOWLIST` (enum 0, the default) means **no one** can be paid, while every getter shows a healthy mandate (`arc-agent-mandate/docs/FINDINGS.md` §8).
- `ALLOW_ALL_ACCESS` is unsafe on Arc because ERC-20 `transfer` carries `value == 0` and escapes a native limit (`arc-agent-mandate/docs/FINDINGS.md` §8).
- A denylist bounds money but not authority: on a fork the agent key transferred the owner's ERC-8004 identity NFT to a stranger with zero spend recorded → switched to allowlist + `checkSelectors`; without `checkSelectors` on USDC, `transferFrom` (unmetered) would open (`arc-agent-mandate/docs/FINDINGS.md` §13).
- Payee scoping and the ERC-20 meter are mutually exclusive: on the ERC-20 rail the access list sees the token contract, never the recipient (`docs/ARCHITECTURE.md` "Why Arc").
- Over-limit on the ERC-20 rail is refused at execution (op included, reverts, sponsored gas) rather than at validation (`docs/ARCHITECTURE.md`).
- `eth_estimateUserOperationGas` always fails (stub signature recovers to a non-session-key → AA23); gas limits are fixed constants (`mcp/spend.ts:120-140`).
- Revocation does not recall money already in the agent's Gateway escrow; mitigated by just-in-time per-purchase top-ups (`docs/ARCHITECTURE.md`).
- An `approve` survives revocation (original reason for native-only, reversed 09-06) (`README.md` decision 5).
- Re-granting to an agent that already has an allowance fails (#50, internal planning notes).
- Watching for the grant is `setInterval` polling (#14).

**Standards:** ERC-6900 v1 plugins (session key plugin, manifest hash), ERC-4337 v0.7, ERC-165 interface ids, EIP-712 (identity link signature). Adjacent but NOT used: ERC-7715 (`wallet_requestExecutionPermissions`), ERC-7710 (delegation manager), ERC-7579 modules (`docs/CONTRIBUTION.md` standards table).

---

## 3. Fees (gas sponsorship / paymaster, who pays)

**What Arc/Circle provides**
- Arc: gas paid in **USDC** (native, 18-decimal accounting), EIP-1559 with EWMA smoothing, **20 gwei base-fee floor** on testnet (below it: dropped/pending), base fee to block beneficiary not burned — https://docs.arc.io/arc/references/gas-and-fees , https://docs.arc.io/arc/concepts/stable-fee-design . Fees convert to ARC at protocol level per the ARC whitepaper (`PRODUCT.md` "What Arc actually is").
- **Circle Gas Station**: ERC-4337 paymaster sponsorship for Circle wallets, policies in Console, billed to developer (docs summary: 5% of gas cost), supports **Arc Testnet** — https://developers.circle.com/wallets/gas-station
- **Circle Paymaster** (user pays gas in USDC, any ERC-4337 wallet, v0.7/v0.8): **Arc not listed** — https://developers.circle.com/paymaster/addresses-and-events (redundant on Arc, where gas already is USDC).
- **Bundler:** Arc has no public bundler; Arc's public RPC answers `eth_supportedEntryPoints` with "method not supported"; Circle's modular RPC returns EntryPoint v0.7 (`mcp/README.md`). Directory lists Pimlico, Biconomy, Alchemy etc. as bundlers but whether each has live Arc testnet bundlers is **UNVERIFIED**.
- Nanopayments (x402 via Gateway) are gas-free for the payer; Gateway settles in batches (see §5).

**What we used, and how** — USED
- Circle Gas Station paymaster for **every** user operation (phone grants/revokes, agent top-ups, identity setup): `createBundlerClient({ paymaster: true })` in `src/arc/account.ts:125-131` and `mcp/spend.ts:110`; bundler config in `mcp/bundler.ts` (`CIRCLE_CLIENT_URL=https://modular-sdk.circle.com/v1/rpc/w3s/buidl`, `CIRCLE_CLIENT_KEY`, `CIRCLE_PASSKEY_DOMAIN`). Probe script `scripts/probe-sponsorship.ts`.
- Result: neither the user's account nor the agent holds gas; the agent EOA never holds a balance (`mcp/README.md` "Why the agent holds nothing").
- Seller side: the maze pays its own gas for reputation writes and badge mints from its server key (`arc-maze/src/arc/reputation.ts`, `badge.ts`).

**Rough edges we hit**
- `estimateFeesPerGas` returns a price Circle's bundler accepts into mempool and never includes; we bid 2x and replace stuck ops at 3x (`mcp/spend.ts:143-160`).
- Gas estimation unusable with session-key signatures (see §2).
- Sponsorship makes gas unbounded in the mandate on purpose (an unset gas limit denies); only matters if Circle stops sponsoring (#13, `IMPLEMENTATION.md`).
- The agent connector must be given the Circle client key (same key already in the mobile bundle, domain-bound, grants no authority) — "bring your own key" for publishing (`mcp/README.md`).
- Without sponsorship, the account would pay unbounded gas the mandate never counts, because spend limits bound `call.value` only (`mcp/README.md`).

**Standards:** ERC-4337 paymaster (`paymasterAndData`, v0.7 `paymasterVerificationGasLimit`/`paymasterPostOpGasLimit`), EIP-1559.

---

## 4. Money (USDC, stablecoin, decimals / rails)

**What Arc/Circle provides**
- USDC is Arc's native gas token; **one balance, two views**: native 18 decimals and ERC-20 at `0x3600000000000000000000000000000000000000` with 6 decimals; ERC-20 view truncates sub-micro amounts — https://docs.arc.io/arc/references/evm-differences , https://docs.arc.io/arc/references/gas-and-fees
- EIP-7708 `Transfer` logs for native value from system addresses; transfers to `0x0` revert; burning prohibited; protocol-level blocklist reverts — evm-differences page.
- Other stablecoins/contracts: EURC `0x89B5…D72a`, USYC, CCTP V2, GatewayWallet `0x0077777d7EBA4688BDeF3E311b846F25870A19B9`, GatewayMinter `0x0022222ABE238Cc2C7Bb1f21003F0a260052475B`, FxEscrow, Memo `0x5294…e505`, Multicall3From `0x522f…47D0`, Permit2 — https://docs.arc.io/arc/references/contract-addresses
- Faucet https://faucet.circle.com ; App Kit (bridge/swap/send/unified balance, pure TS, web-oriented) https://docs.arc.io/app-kit
- Deterministic sub-second finality: https://docs.arc.io/arc/concepts/deterministic-finality

**What we used, and how** — USED
- `Usdc` money type with explicit 6↔18 conversions: `src/arc/usdc.ts` (+ tests).
- Metering on the ERC-20 view only; `USDC_RAILS` list re-derived from the chain in `integration/arc-rails.test.ts`.
- Seller prices in micro-USDC on the wire (`toAtomic`, `arc-maze/src/arc/paywall.ts`).
- Testnet only, chain id `5042002`, viem's `arcTestnet` (`src/arc/chain.ts`).
- NOT USED: EURC, CCTP, App Kit/Bridge Kit, Memo, Multicall3From (Memo/Multicall3From were planned in `API_DESIGN.md`, not built), unified balance.

**Rough edges we hit**
- Two decimal scales over one balance; `balanceOf` can read 0 for a funded account; a limit on one rail is not a limit; limits on both double the bound (`arc-agent-mandate/docs/FINDINGS.md` §2, internal planning notes 2).
- ERC-20 view moves balance through a native precompile at `0x1800…` whose code is one byte (`0x01`); **no fork can execute an ERC-20 transfer** (reverts `StackUnderflow`); `approve` works on a fork (`arc-agent-mandate/docs/FINDINGS.md` §9).
- Standard anvil keys are EIP-7702-delegated to sweepers on Arc testnet; user ops still report `success = true` (`arc-agent-mandate/docs/FINDINGS.md` §3).
- Bridge Kit has no mobile passkey signer adapter (circlefin/modularwallets-ios-sdk issue #35) (`GAP_INVENTORY.md` §4).
- No service in Circle's x402 Discovery catalogue (1,247 resources scanned 09-02) accepted Arc (`PRODUCT.md`).
- Modular Wallets testnet-only puts the mainnet prize track out of reach (internal planning notes).

**Standards:** ERC-20, EIP-7708 (native Transfer logs, shipped ahead of upstream), EIP-3009 (see §5), EIP-2612/Permit2 (available, not used).

---

## 5. Pay-per-call payments (x402, Gateway batching, escrow / depositFor)

**What Arc/Circle provides**
- **Circle Nanopayments (powered by Gateway)**: deposit USDC into `GatewayWallet`, sign **EIP-3009** authorisations off-chain (zero gas), Gateway verifies and deducts immediately, settles on chain in batches; payments down to $0.000001; x402 v2-compatible — https://developers.circle.com/gateway/nanopayments
- **EOA only**: "Nanopayments and x402 batch settlement require EOA signatures and do not support ERC-1271" (same page); buyer quickstart: "Smart contract account (SCA) wallets are not supported" — https://developers.circle.com/gateway/nanopayments/quickstarts/buyer
- SDK `@circle-fin/x402-batching`: buyer `GatewayClient` / `BatchEvmScheme` (`/client`), seller `BatchFacilitatorClient` (`/server`); chain id string `arcTestnet`; testnet API `https://gateway-api-testnet.circle.com`; `client.withdraw()` — buyer quickstart; seller quickstart https://developers.circle.com/gateway/nanopayments/quickstarts/seller
- Batched settlement uses an **AWS Nitro Enclave** to verify every EIP-3009 signature and compute net balance changes — https://developers.circle.com/gateway/nanopayments/concepts/batched-settlement
- Gateway attestation ~0.5 s on Arc vs 13–19 min on Base (`PRODUCT.md`, from Circle Gateway docs, 09-02).
- **Circle Agent Nanopayments / Circle CLI**: agent-wallet path to x402 — https://developers.circle.com/agent-stack/agent-nanopayments , https://developers.circle.com/agent-stack/circle-cli
- **Agent Marketplace / Discovery API** `GET https://api.circle.com/v2/x402/discovery/resources`, no auth; docs describe `supportsCircleGateway`/`supportsVanillax402` flags — https://developers.circle.com/agent-stack/agent-marketplace/discovery-api
- Arc sample app "Arc Nanopayments" (agent pays paywalled API via x402) — https://docs.arc.io/arc/references/sample-applications ; agentic-economy page: https://docs.arc.io/build/agentic-economy
- **ERC-8183 AgenticCommerce** job escrow (create/setBudget/fund/submit/complete) at `0x0747EEf0706327138c69792bF28Cd525089e4583` — https://docs.arc.io/arc/tutorials/create-your-first-erc-8183-job (NOT USED; see §13).

**What we used, and how** — USED
- **Buyer (connector)** hand-rolled x402 client — does **not** use `@circle-fin/x402-batching`: parses `402`, accepts only `extra.name == "GatewayWalletBatched"`, `version "1"`, signs `TransferWithAuthorization` under the `GatewayWalletBatched` EIP-712 domain with the agent EOA, floors validity to 7 days + 100 s, backdates 600 s, retries with the payment header — `arc-agent-mandate/mcp/x402.ts` (esp. L95-135, L204-209).
- **Escrow top-up**: phone account (via session key, sponsored) calls USDC `approve` then `GatewayWallet.depositFor(token, depositor=agentEOA, value)`, just-in-time for the shortfall, `availableBalance(token, depositor)` to check — `mcp/gateway.ts:88-142`; MCP tools `buy`, `top_up`, `pay` in `mcp/server.ts`.
- Tested against Circle's live, unauthenticated `/v1/x402/verify` as an oracle, including a wrong-key negative control (`integration/x402-payment.test.ts`, `arc-agent-mandate/docs/FINDINGS.md` footer).
- **Seller (maze)**: `BatchFacilitatorClient({url: GATEWAY_API})` `verify` then `settle`, payer taken from the settle response; requirements `scheme: "exact"`, `network: arcTestnet`, asset USDC view, `maxTimeoutSeconds` 7 days, `extra {GatewayWalletBatched, 1, GatewayWallet}` — `arc-maze/src/arc/paywall.ts`. Deps: `@circle-fin/x402-batching ^3.4.0`, `@x402/core ^2.25.0`, `@x402/evm ^2.25.0`. Bazaar discovery metadata emitted on every 402 (`src/arc/bazaar.ts`) though nothing catalogues Arc.
- Prices: move $0.001, look $0.002, map $0.01; demonstrated run 12 steps + map = 13 payments, $0.022 (internal planning notes, arc-maze `README.md`).

**Rough edges we hit**
- **Gateway verifies with strict `ecrecover`**: a smart account cannot be the x402 payer however it signs (tested with an ERC-1271 always-magic contract and an on-chain delegate) → forced the agent-EOA + `depositFor` escrow design (`arc-agent-mandate/docs/FINDINGS.md` §10, internal planning notes 5). (Circle docs now state this explicitly.)
- Escrow is not claw-back-able: `withdraw` pays `msg.sender`, only the depositor; revoke does not recall escrowed funds (`arc-agent-mandate/docs/FINDINGS.md` §10).
- Circle's x402 client **defaults to the mainnet Gateway API**; the error is `unsupported_network` (`arc-agent-mandate/docs/FINDINGS.md` §11).
- Gateway requires **7-day** authorisation validity; a seller's shorter `maxTimeoutSeconds` must be ignored by the buyer (`arc-agent-mandate/docs/FINDINGS.md` §11).
- **Paid ≠ settled**: batches land ~15 min later; leaderboards/receipts must distinguish claimed vs settled (`arc-agent-mandate/docs/FINDINGS.md` §11).
- `/v1/x402/verify` checks signature and window but not funds (useful as test oracle) (`arc-agent-mandate/docs/FINDINGS.md` §11).
- Grepping bytecode for `1626ba7e` wrongly suggested USDC lacks ERC-1271; call it instead (`arc-agent-mandate/docs/FINDINGS.md` §10).
- Circle's `BatchFacilitatorClient` declares a narrower `PaymentPayload` than `@x402/core` (types derived from method signature to avoid casts) (`arc-maze/src/arc/paywall.ts` header).
- Discovery API returned neither `supportsCircleGateway` nor `supportsVanillax402` on 09-02, despite the docs; no seller in the catalogue settled Arc; nothing indexes Arc sellers (`PRODUCT.md`, `IMPLEMENTATION.md` #20).
- Every step is its own top-up user op, so a run is slow (internal planning notes).
- Deferred #18: pay x402 directly from the account under the token's own EIP-3009 domain (supports contract signers, no batching) would delete escrow/top-ups/agent key but loses sub-cent economics; every Arc seller asks for the Gateway domain (`IMPLEMENTATION.md`).

**Standards:** x402 (v2 headers `PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE`, scheme `exact`), HTTP 402, EIP-3009 `TransferWithAuthorization`, EIP-712, ERC-1271 (explicitly unsupported by Gateway), Circle Gateway `GatewayWalletBatched` domain (Circle-specific).

---

## 6. Agent identity (ERC-8004 identity registry)

**What Arc/Circle provides**
- ERC-8004 registries deployed on Arc testnet under vanity addresses: Identity `0x8004A818BFB912233c491871b3d84c89A494BD9e`, Reputation `0x8004B663056A597Dffe9eCcC1965A193B7388713`, Validation `0x8004Cb1BF31DAf7788923b405b754f57acEB4272` — https://docs.arc.io/arc/tutorials/register-your-first-ai-agent . (They were not found on the contract-addresses page summary — possibly omitted there.)
- Tutorial flow: owner calls `register(string metadataURI)` (IPFS JSON) → ERC-721 mint → read `Transfer` log for tokenId → `ownerOf`/`tokenURI`. Wallet options: Circle developer-controlled wallets (API key + entity secret, Gas Station sponsored) or self-managed viem keys (same page). Standard: https://eips.ethereum.org/EIPS/eip-8004 (draft).
- Identity contract is an ordinary ERC-721, `name()` "AgentIdentity", symbol "AGENT", ~48k holders (`arc-agent-mandate/docs/FINDINGS.md` §12, `IMPLEMENTATION.md`).

**What we used, and how** — USED
- Connector registers an identity **from the user's smart account** (so the wallet owns it) via the session key allowlist: `register()` (no-arg overload) then `setAgentWallet(agentId, agentEOA, deadline, signature)` with the agent's EIP-712 consent signature (domain `ERC8004IdentityRegistry` v1, type `AgentWalletSet`), 240 s window — `arc-agent-mandate/mcp/erc8004.ts` (L61-100), `mcp/identity.ts`; allowlist `IDENTITY_CALLS` in `src/arc/mandate.ts`. `check_allowance` reports the id; agent passes it to sellers as `?agent=<id>`.
- Seller checks `getAgentWallet(agentId) == payer` before crediting — `arc-maze/src/arc/reputation.ts:54-67`.
- App reads identity: `src/arc/identity.ts`; `integration/agent-identity.test.ts`.
- Proven 09-11: identity #894344 end to end (`IMPLEMENTATION.md`).

**Rough edges we hit**
- No reverse lookup (address → agentId); agent must declare its id; unchecked declarations let anyone write onto a stranger's identity (`arc-agent-mandate/docs/FINDINGS.md` §12).
- `setAgentWallet` `MAX_DEADLINE_DELAY = 5 minutes` → "deadline too far" (`arc-agent-mandate/docs/FINDINGS.md` §12).
- Two operations needed because the agent's signature commits to an id only assigned inside `register()` (`mcp/erc8004.ts` header).
- Identity is a transferable ERC-721 — reputation can be sold; a denylist mandate let the agent key move it (`PRODUCT.md`, `arc-agent-mandate/docs/FINDINGS.md` §13).
- Circle's ERC-8004 walkthroughs assume server-side developer-controlled wallets; passkey-account path was untrodden (`GAP_INVENTORY.md` final note).
- ERC-8004 is a draft; ERC-8004 writes can only be proven against the live chain, not a fork (`arc-agent-mandate/docs/FINDINGS.md` footer; `IMPLEMENTATION.md` standing risk 2).
- Before 09-11 the only working identity was hand-made; agents solved and earned nothing (`mcp/erc8004.ts`).

**Standards:** ERC-8004 (draft, Aug 2025), ERC-721, EIP-712.

---

## 7. Reputation & validation (ERC-8004 reputation / validation)

**What Arc/Circle provides**
- ReputationRegistry `giveFeedback(uint256 agentId, int128 value, uint8 valueDecimals, string tag1, string tag2, string endpoint, string feedbackURI, bytes32 feedbackHash)`; owners/operators cannot give self-feedback — https://docs.arc.io/arc/tutorials/register-your-first-ai-agent
- ValidationRegistry: `validationRequest(address validator, uint256 agentId, string requestURI, bytes32 requestHash)`, `validationResponse(bytes32, uint8 response, string, bytes32, string tag)` (100 pass / 0 fail), `getValidationStatus(bytes32)` — same tutorial.
- Spec caveats (from EIP text, summarised in `PRODUCT.md`): feedback needs no prior transaction, Sybil inflation acknowledged, validation is hooks (staked re-execution, zkML, TEE oracles) not a performed check.
- Circle's framing: "trusted discovery … rank them on demonstrated merit" (quoted in `IMPLEMENTATION.md`; source URL not recorded — **UNVERIFIED** exact source).

**What we used, and how** — USED (reputation), NOT USED (validation)
- Maze writes `giveFeedback` with value 100 `efficiency-pct`, tags `arc-maze`/`efficiency-pct`, feedbackURI = public run page, feedbackHash = sha256 run digest, from the maze server key — `arc-maze/src/arc/reputation.ts:98`; orchestration `src/reward.ts`. On-chain tx `0x226f769f…63f39` (arc-maze `README.md`).
- Phone reads feedback attributed per reviewer: `src/arc/reputation.ts` (`reputationOf`), shown on agent screen (#52 half built).
- `MazeVerdict.sol` hard-codes tags/decimals and calls `giveFeedback` as `msg.sender` on behalf of the Chainlink DON — `arc-maze/contracts/src/MazeVerdict.sol` (tested, **not deployed**).
- Validation registry: address known in `src/arc/chain.ts`, "not used by it yet" (`docs/ARCHITECTURE.md` table).

**Rough edges we hit**
- `NewFeedback` event ABI has non-obvious indexing (`feedbackIndex` unindexed, `indexedTag1` indexed); guessed ABI decoded "score 1 at 100 decimals" — fetch the real ABI from the explorer (`arc-agent-mandate/docs/FINDINGS.md` §12).
- Self-feedback refusal is the only guard; a seller can still flatter its own customers → motivated CRE re-derivation (`IMPLEMENTATION.md` "Why the verdict should not be ours").
- Today the maze writes reputation with its own key (not the network).
- Serverless host stopped after responding, so reward writes were lost until the solve step awaited payout (#48, `JOURNEY.md` "The finish is orphaned").

**Standards:** ERC-8004 Reputation and Validation registries (draft).

---

## 8. Badges / NFTs

**What Arc/Circle provides**
- Nothing badge-specific. Standard EVM + OpenZeppelin; Circle Wallets advertise ERC-20/721/1155 support (https://docs.arc.io/arc/tools/account-abstraction). Circle "Contract Platform" / deploy-contracts tutorial exists: https://docs.arc.io/arc/tutorials/deploy-contracts (NOT USED).

**What we used, and how** — USED
- `CohortZero` ERC-721, fixed supply 100, separate `admitter` (mint-only) role vs `owner`, one badge per holder, minted to the ERC-8004 identity's **owner** (the user's wallet) — `arc-maze/contracts/src/CohortZero.sol`, deployed `0xe5A8fAEf7139d04582C7E17C3F615710343b53a3`; mint code `arc-maze/src/arc/badge.ts`; deploy `scripts/deploy-badge.ts`; art `src/web/badge-art.ts`.
- Phone reads badges without an indexer by iterating minted ids (≤100) — `arc-agent-mandate/src/arc/badges.ts`; `integration/badges.test.ts`. Badge #1 and #2 proven on chain.

**Rough edges we hit**
- No indexer for holders; linear scan bounded by the cap (`src/arc/badges.ts` header).
- A fresh clone showed the badge as a broken image (fixed, `IMPLEMENTATION.md` "Done 09-11").
- Social-card caches outlive hourly rounds (`IMPLEMENTATION.md` open risk).

**Standards:** ERC-721 (+ metadata `tokenURI`), OpenZeppelin `Ownable`.

---

## 9. Off-chain verification / oracles (Chainlink CRE, TEEs)

**What Arc/Circle provides**
- Arc oracles directory: Chainlink (Data Feeds, Data Streams), Chronicle, Pyth, RedStone, Stork — https://docs.arc.io/arc/tools/oracles . **CRE is not listed there**; whether CRE's forwarder is deployed on Arc testnet is **UNVERIFIED**.
- Arc sample app "Arc Prediction Markets" uses UMA Optimistic Oracle V2; "Arc Escrow" is AI-validated via Refund Protocol — https://docs.arc.io/arc/references/sample-applications
- Circle Gateway itself uses AWS Nitro Enclaves for signature verification (§5).
- Chainlink CRE Confidential Workflows: handler declared to run in a TEE, secrets from Vault DON (threshold encryption, DKG), ConfidentialHTTP — https://docs.chain.link/cre/concepts/confidential-workflows , https://docs.chain.link/cre/guides/workflow/using-confidential-http-client

**What we used, and how** — USED (sim)
- `@chainlink/cre-sdk 1.18.0` workflow: `cre.handlerInTee(cron, onCronTrigger, [{tee: 'nitro', regions: ['us-west-2']}])`; `runtime.getSecret({id: ARCHIVE_TOKEN})`; `HTTPClient` GET from Upstash Redis run store; imports the maze's own `verify`/`efficiency`/`digest`; `runtime.usingTheDons().report(...)` ABI payload `(bytes32 runId, uint256 agentId, int128 value, string feedbackURI, bytes32 feedbackHash)` — `arc-maze/cre/verdict/workflow.ts`, `cre/verdict/workflow.yaml`, `cre/secrets.yaml`.
- Receiver `MazeVerdict` (`IReceiver.onReport`, forwarder-gated) → `giveFeedback` — `arc-maze/contracts/src/MazeVerdict.sol` (not deployed).
- Evidence: two simulations matching on-chain score and hash — `arc-maze/cre/evidence/simulation-25b9f044.txt`, `simulation-83031b89.txt`.

**Rough edges we hit** (all from internal planning notes Chainlink feedback and `arc-maze/cre/README.md`)
- Confidential Workflows deploy access (beta) not granted → simulator only; verdict never reached a chain.
- CLI installs to `~/.cre/bin`, not on PATH, undocumented.
- Simulation requires `cre login` (hard in CI).
- Simulator prints "Using default private key for chain write simulation" for a no-chain workflow.
- Unclear whether plain `HTTPClient` inside `handlerInTee` counts as a confidential response or `ConfidentialHTTP` is required.
- Template leftovers (API_TOKEN, Sepolia RPCs, production target without example config).
- Simulator is "not a real TEE"; no simulated attestation.
- Workflow takes the run record's word for payer and identity.

**Standards:** Chainlink CRE report/`IReceiver` interface, ERC-165, AWS Nitro Enclaves attestation, keccak256/ECDSA DON signatures; sha256 digest for the run record.

---

## 10. Events, indexing, watching chain state

**What Arc/Circle provides**
- EIP-7708 native `Transfer` logs from system addresses; system emitter `0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE` (`GAP_INVENTORY.md` §3; evm-differences page).
- Arc tutorial teaches **Circle Contracts API event monitors with webhooks**, and notes USDC needs a separate monitor on the system emitter to avoid duplicate counts; test environment does not replicate EIP-7708 or the USDC precompile — https://docs.arc.io/arc/tutorials/monitor-contract-events ; Circle event monitoring https://developers.circle.com/contracts/scp-event-monitoring
- Indexers: Alchemy (Data APIs, Webhooks), Envio HyperIndex, Goldsky, Pinax, The Graph, Thirdweb Insight, Zerion — https://docs.arc.io/arc/tools/data-indexers
- Node providers: Alchemy, QuickNode, Blockdaemon, dRPC — https://docs.arc.io/build
- Explorer https://testnet.arcscan.app

**What we used, and how** — USED (raw RPC only; no indexer, no webhooks)
- Activity feed built on `UserOperationEvent` (indexed by `sender`), agent attribution from `nonce >> 64 == sessionKey`, plus `SessionKeyAdded`/removal and ERC-721 `Transfer` identity logs, windowed `eth_getLogs` — `arc-agent-mandate/src/arc/activity.ts`; `integration/arc-activity.test.ts`.
- Connector pairing scan for `SessionKeyAdded(account, sessionKey indexed, tag indexed)` in 10k-block windows from plugin deploy block `60_625_268` — `mcp/chain.ts`, `src/arc/mandate.ts:47`.
- Configurable endpoint with fallback to viem's `arcTestnet` RPC — `src/arc/endpoint.ts`, `integration/arc-endpoint.ts`.
- Seller live feed (SSE-style stream of round deltas) is off-chain: `arc-maze/src/live/feed.ts`. Runs stored in Upstash Redis: `arc-maze/src/storage.ts`.

**Rough edges we hit**
- Two emitters double-count every USDC payment (1,707 txs in a 300-block sample) (`arc-agent-mandate/docs/FINDINGS.md` §6).
- `eth_getLogs` capped at 10,000 blocks **and** 20,000 results; ~46 logs/block → unfiltered queries fail rather than truncate (`arc-agent-mandate/docs/FINDINGS.md` §6).
- Anvil forks emit no native-transfer logs → feed on `Transfer` untestable locally (`arc-agent-mandate/docs/FINDINGS.md` §6).
- viem's `getLogs` cannot express "either event at topic position" → raw `eth_getLogs` (`src/arc/activity.ts:341`).
- Grant detection polls with `setInterval` (#14); connector fails against providers refusing wide block ranges (#53).
- Blocklist revert consumes gas with no receipt; don't poll for a receipt forever (`GAP_INVENTORY.md`).

**Standards:** EIP-7708, ERC-4337 `UserOperationEvent`, ERC-721 `Transfer`, JSON-RPC `eth_getLogs`.

---

## 11. Agent tooling (MCP connector, CLI, SDKs, what the agent runs)

**What Arc/Circle provides**
- **Arc MCP Server** — docs-only (search, get page), `https://docs.arc.io/mcp`, no auth; `claude mcp add --transport http arc-docs https://docs.arc.io/mcp` — https://docs.arc.io/ai/mcp . It does not transact.
- **Circle Agent Stack**: Agent Wallets, Agent Marketplace, Circle CLI, Nanopayments, Circle Skills (+ Circle MCP server for SDK/docs access) — https://www.circle.com/agent-stack . Networks listed include Arc testnet.
- **Circle CLI** `npm install -g @circle-fin/cli`: agent wallets with email-OTP auth (non-interactive flow for agents), policies, local self-custodial wallets via Open Wallet Standard, CCTP bridging, Gateway x402 nanopayments, marketplace search — https://developers.circle.com/agent-stack/circle-cli
- **Circle Skills**: https://github.com/circlefin/skills (e.g. `use-arc`, `use-agent-wallet`, `agent-wallet-policy` SKILL.md); setup at `agents.circle.com/skills/setup.md`.
- SDKs: `@circle-fin/modular-wallets-core`, `@circle-fin/x402-batching`, `@circle-fin/bridge-kit`/App Kit, developer-controlled wallets SDK; viem ships `arcTestnet`.

**What we used, and how** — USED
- Our own MCP server (stdio), `@kuiralabs/arc-mandate` (not yet on npm, #42), deps `@modelcontextprotocol/sdk`, `viem`, `zod`, `qrcode-terminal`, `dotenv` — `arc-agent-mandate/mcp/server.ts`, `mcp/package.json`, `mcp/README.md`. Tools: `get_pairing_address`, `check_allowance`, `pay`, `buy`, `top_up`. Installed with `claude mcp add arc-mandate -s user -- node "$PWD/mcp/server.ts"`.
- The agent in the demo is **Claude Code** (internal planning notes).
- NOT USED: Circle CLI, Agent Wallets, Circle Skills, Arc docs MCP (as a runtime dependency), `@circle-fin/x402-batching` on the buyer side.
- The mobile app: Expo 54 / RN 0.81 / expo-router, expo-camera for QR scan — `arc-agent-mandate/package.json`, `app/`.

**Rough edges we hit**
- No Circle path for a non-custodial, phone-present human to bound an agent; Circle's limit-setting skill spends ~40 lines on piping an email OTP into an interactive CLI (`PRODUCT.md` "The gap this fills").
- Circle Agent Wallet policies are mainnet-only; testnet rejected (`PRODUCT.md`; confirmed on custom-policies page).
- `bin` points at `server.ts` with no build step; Node `^22.18 || >=23.6` for type stripping (`mcp/README.md`).
- Env var shadowing (`ARC_TESTNET_RPC_URL` vs `ARC_RPC_URL`), local-scope MCP config shadows others, wrong path shows as `CONNECTION_CLOSED` (`mcp/README.md`).
- Agent learns the journey by failing; refusals must name next action and owner (`JOURNEY.md`).

**Standards:** Model Context Protocol, x402, QR pairing (our own scheme; no WalletConnect — Arc testnet not registered with WalletConnect per `GAP_INVENTORY.md`).

---

## 12. Privacy (Arc Privacy Sector, TEEs)

**What Arc/Circle provides**
- **Arc Privacy Sector (APS)**: confidential Solidity execution parallel to the public EVM, both state roots committed per block; users encrypt txs to an APS network key and submit to a precompile; validators decrypt inside hardware enclaves; no results/return values/event logs exposed publicly; synchronous composability with public contracts; default-deny with function policies Open/Restricted/Locked and trust domains; X-Wing KEM (X25519+ML-KEM-768), AES-256-GCM; master key Shamir-shared across validators. **Status: "Privacy features are on the roadmap and not yet available on Arc."** — https://docs.arc.io/arc/concepts/opt-in-privacy
- From Arc's privacy whitepaper (as recorded in `PRODUCT.md`, not re-verified in this pass): `ethCallAuthorized` with EIP-712 caller auth and response encrypted to a receiver key, `addTrustees`, grants with `validAfter`/`validBefore` block ranges, AWS Nitro enclaves, side-channels excluded from threat model — **UNVERIFIED against current docs** (the docs page summary did not mention these APIs).
- Post-quantum security concept page: https://docs.arc.io/arc/concepts/post-quantum-security (not read in depth).
- Gateway uses Nitro Enclaves for verification (§5) — integrity, not user privacy.
- Arc validators are permissioned (PoA → PoS), permissioning managed off-chain (ARC whitepaper, via `PRODUCT.md`).

**What we used, and how** — NOT USED
- Nothing shipped. Design rule adopted: mandate read surface as discrete selectors (APS grants are per-selector) (`PRODUCT.md` "What to do now"). Plugin loupe is already selector-shaped (`getERC20SpendLimitInfo`, `getKeyTimeRange`, etc. in `src/arc/mandate.ts`).
- The only TEE we touched is Chainlink's (§9), to hide a credential, not user data.

**Rough edges we hit**
- APS not buildable: appears only under Concepts, no precompile address, no SDK/sample; six plausible precompile addresses probed returned zero bytes (`GAP_INVENTORY.md` "Arc Privacy Sector").
- Direct conflict: our supervision feed depends on public EIP-7708/`UserOperationEvent` logs, and APS disables event logs by default → private mandates would need a redesigned feed (`PRODUCT.md`).
- An agent's public x402 history leaks what it buys (motivation, unaddressed) (`PRODUCT.md`).
- APS trust model is TEE + permissioned validators, not ZK — explicitly contrasted with Midnight (`PRODUCT.md`, `GAP_INVENTORY.md`).

**Standards:** TEE attestation (AWS Nitro), X-Wing KEM / ML-KEM-768 (FIPS 203), AES-256-GCM, Shamir secret sharing, EIP-712.

---

## 13. Anything else Arc/Circle markets for agents that we did not use

| Offering | What it is | Link | Why not used / note |
|---|---|---|---|
| **ERC-8183 AgenticCommerce** | Job escrow lifecycle: `createJob`, `setBudget`, `fund`, `submit`, `complete`; client / provider / evaluator roles; USDC | https://docs.arc.io/arc/tutorials/create-your-first-erc-8183-job ; https://eips.ethereum.org/EIPS/eip-8183 ; addr `0x0747EEf0706327138c69792bF28Cd525089e4583` | Explicit v2, out of four-week scope (`README.md` decision 3, `PRODUCT.md` v2) |
| **ERC-8004 ValidationRegistry** | request/response validation hooks | register-your-first-ai-agent tutorial | Known, unused (`docs/ARCHITECTURE.md`) |
| **Circle Agent Wallets** | MPC wallets with per-tx/daily/weekly/monthly limits, allow/blocklists, email OTP | https://developers.circle.com/agent-stack/agent-wallets | Our product is the non-custodial passkey alternative; policies mainnet-only |
| **Circle CLI** | agent-native CLI (wallets, policies, x402, discovery, CCTP) | https://developers.circle.com/agent-stack/circle-cli | Replaced by our MCP connector |
| **Circle Skills / Circle MCP** | agent instruction packs, SDK/docs MCP | https://github.com/circlefin/skills | Not needed at runtime |
| **Agent Marketplace + Discovery API** | curated, sanctions-screened, health-checked catalogue of x402 services | https://developers.circle.com/agent-stack/agent-marketplace/discovery-api | Measured, not integrated; no Arc sellers on 09-02 (#19-#21) |
| **Arc MCP Server** | docs search for coding assistants | https://docs.arc.io/ai/mcp | Dev convenience only |
| **Arc Nanopayments sample** | agent pays paywalled API over x402 | https://docs.arc.io/arc/references/sample-applications (repo `circlefin/arc-nanopayments` per agentic-economy page) | We wrote our own buyer/seller |
| **Arc Escrow sample / Refund Protocol** | AI-validated freelance escrow | same page; `circlefin/arc-escrow` | Not used |
| **Circle developer-controlled wallets** | server-side custodial wallets used in Arc's agent tutorials | https://docs.arc.io/arc/tutorials/register-your-first-ai-agent | Contrary to our non-custodial posture |
| **App Kit / Bridge Kit / Unified Balance / CCTP V2** | cross-chain USDC inflow, swap, send | https://docs.arc.io/app-kit | v2; no passkey signer adapter on mobile (issue #35) |
| **Memo** `0x5294…e505` | memo metadata on calls, sequential `Memo` events (reconciliation) | https://docs.arc.io/arc/references/contract-addresses | Planned in `API_DESIGN.md`, not built |
| **Multicall3From** `0x522f…47D0` | batch preserving original `msg.sender` | same | Not used |
| **FxEscrow, EURC, USYC** | FX escrow, euro stablecoin, yield token | same | Not relevant |
| **Circle Paymaster** | pay gas in USDC on other chains | https://developers.circle.com/paymaster | Not on Arc (redundant) |
| **Compliance vendors** (Elliptic, TRM Labs) | tx monitoring / screening | https://docs.arc.io/arc/tools/compliance-vendors | Not used; Arc blocklist is protocol-level anyway |
| **Indexers / webhooks** (Circle event monitors, Alchemy, Envio, Goldsky, The Graph…) | push events | §10 links | Used raw RPC instead |
| **Ecosystem session-key providers** (Alchemy Smart Wallets, ZeroDev, MetaMask Delegation Toolkit ERC-7715/7710) | alternative bounded-authority stacks | https://docs.arc.io/arc/tools/account-abstraction ; `docs/CONTRIBUTION.md` | Incompatible with Circle Modular Wallets / not chosen |
| **Nanopayments $0.000001 floor, Gateway cross-chain withdrawal** | spend Arc-funded balance on other chains | https://developers.circle.com/gateway/nanopayments | Used same-chain only |
| **Arc mainnet** | announced 2026-09-16 (press); whitepaper "summer 2026" | `IMPLEMENTATION.md` | UNVERIFIED; Modular Wallets have no Arc mainnet entry |

---

## Cross-cutting: what Arc gave us "for free" vs what we had to build

**Took from Arc/Circle unchanged:** Circle Modular Wallets TS SDK (account, bundler, passkey RP), Circle Gas Station sponsorship, Circle bundler (only one on Arc), USDC native gas + ERC-20 view, GatewayWallet + Gateway API + `BatchFacilitatorClient` (seller), ERC-8004 Identity + Reputation registries, EntryPoint v0.7, viem `arcTestnet`.

**Had to build ourselves:** React Native passkey ceremony + COSE/SPKI + WebAuthn shim + `X-AppInfo` rewrite; EntryPoint v0.7 port of the session-key plugin (+ allowlist/one-meter policy design); agent-EOA + `depositFor` escrow top-up pattern; x402 buyer client; pairing over on-chain event tags; activity feed from `UserOperationEvent` with nonce-key attribution; identity-owned-by-wallet registration flow; payer ↔ `getAgentWallet` check; replayable runs + digest; CohortZero badge; CRE replay workflow + `MazeVerdict` receiver.

**The Arc-specific couplings a Midnight port would have to replace one-for-one:** (a) passkey-owned smart account with a pluggable validation hook; (b) an on-chain bounded-delegation module (limit, expiry, call allowlist, revoke); (c) a sponsor for fees so neither owner nor agent holds gas token; (d) a dollar asset and a single metered rail; (e) an off-chain-signed, batch-settled pay-per-call scheme with an escrow a third party can fund; (f) agent identity + third-party-only reputation registry; (g) events queryable per account for a live feed and for pairing; (h) an oracle/TEE path to attest off-chain game results.
