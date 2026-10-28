# Shadow Arena: Flux Finance (Ondo)

**Difficulty**: Beginner (Compound V2 graduates)
**Time Limit**: 2 days
**Scope**: the lending contracts and adjacent CASH/KYC subsystem described below
**Historical contest scope**: 4,365 SLOC; the lending-only estimate was ~2,610. These are inherited counts, not a new combined source measurement.

## What Is Flux Finance?

Flux Finance is a **Compound V2 fork** built by Ondo Finance that adds KYC and sanctions compliance to a lending market. It allows permissioned assets (like OUSG, a tokenized US Treasury product) and permissionless stablecoins (DAI, USDC, USDT, FRAX, LUSD) to coexist in the same lending protocol.

The core idea: Ondo needed a lending market where KYC'd users could borrow against their real-world asset tokens, while permissionless stablecoin suppliers could earn yield without KYC. Flux achieves this by forking Compound V2's cToken contracts and injecting KYC/sanctions checks at specific entry points.

## Exercise scope

This exercise follows the Academy target across both the lending contracts and the adjacent CASH/KYC subsystem in the January 2023 Ondo contest snapshot. Start with the familiar lending fork, then review `contracts/cash/CashManager.sol`, `contracts/cash/kyc/`, and the CASH tokens in `contracts/cash/token/`. Include the functions, imports and interfaces those paths depend on. The historical contest total is 4,365 SLOC; the per-file counts below describe the existing lending inventory, not a fresh combined recount.

The contracts under `contracts/lending/compound/` provide a comparison baseline and dependencies; they are not separate targets to re-audit wholesale. Read inherited assumptions where needed rather than assuming unchanged code is safe in every initialization.

## Architecture

The lending directory contains two parallel cToken implementations plus shared infrastructure:

### Modified Contracts (Primary Audit Targets)

| Contract | Path | SLOC | Description |
|----------|------|------|-------------|
| CTokenModified | `tokens/cToken/CTokenModified.sol` | 884 | cToken with sanctions checks on transfer/mint/redeem/seize, KYC checks on borrow/repay |
| CTokenCash | `tokens/cCash/CTokenCash.sol` | 881 | cToken with KYC checks on ALL operations (permissioned asset markets) |
| CTokenInterfacesModified | `tokens/cToken/CTokenInterfacesModified.sol` | 286 | Storage layout, sets `protocolSeizeShareMantissa = 1.75%` |
| CTokenInterfacesModifiedCash | `tokens/cCash/CTokenInterfacesModifiedCash.sol` | 286 | Storage layout, sets `protocolSeizeShareMantissa = 0%` |
| CErc20 (cToken) | `tokens/cToken/CErc20.sol` | 164 | ERC20 wrapper for CTokenModified |
| CCash | `tokens/cCash/CCash.sol` | 164 | ERC20 wrapper for CTokenCash |
| CTokenDelegate | `tokens/cToken/CTokenDelegate.sol` | 31 | Delegate implementation for fDAI-type tokens |
| CCashDelegate | `tokens/cCash/CCashDelegate.sol` | 31 | Delegate implementation for cCASH tokens |
| cErc20ModifiedDelegator | `tokens/cErc20ModifiedDelegator.sol` | 828 | Proxy with KYC storage for both token types |
| OndoPriceOracle | `OndoPriceOracle.sol` | 74 | Oracle: manual prices for CASH, Compound oracle for stables |
| OndoPriceOracleV2 | `OndoPriceOracleV2.sol` | 207 | V2 oracle: adds Chainlink support and price caps |
| JumpRateModelV2 | `JumpRateModelV2.sol` | 123 | Interest rate model, modified `blocksPerYear` only |

### Reference and dependency directories

| Directory | Reason |
|-----------|--------|
| `contracts/lending/compound/` | Unchanged Compound V2 Comptroller and libraries |
| `contracts/lending/tokens/cErc20Delegate/` | Unchanged Compound V2 base contracts |
| `contracts/lending/chainlink/` | Chainlink interface |
| `contracts/lending/ondo/` | Ondo-specific interfaces |

## Key Concepts

### Permissioned vs. Permissionless Assets

Flux has two types of markets:

- **Permissioned (cCASH)**: OUSG and other RWA tokens. ALL operations (mint, redeem, transfer, borrow, repay, seize) require KYC verification. Uses `CTokenCash` with `protocolSeizeShareMantissa = 0%`.
- **Permissionless (fToken)**: DAI, USDC, etc. Supply and redeem only require sanctions checks. Borrowing requires KYC. Uses `CTokenModified` with `protocolSeizeShareMantissa = 1.75%`.

### KYC and Sanctions Architecture

The KYC system lives in `contracts/cash/` (included in the extended exercise scope), but the lending contracts consume it:

- Each cToken stores a `kycRegistry` address and a `kycRequirementGroup`
- KYC checks call `kycRegistry.getKYCStatus(kycRequirementGroup, account)`
- Sanctions checks call `sanctionsList.isSanctioned(account)`
- Different functions gate on different check types (see tables above)

### OndoPriceOracle

The oracle supports multiple price sources:
- **Manual prices**: Set by admin for CASH tokens (based on off-chain NAV calculation)
- **Compound oracle**: Proxies to Compound's UniswapAnchoredView for listed assets
- **Chainlink feeds** (V2 only): Direct Chainlink price feeds with configurable price caps

### Modified protocolSeizeShareMantissa

Compound V2 uses `protocolSeizeShareMantissa = 2.8%` (the protocol's cut of seized collateral during liquidation). Flux changes this:
- fTokens (permissionless): `1.75%`
- cCASH (permissioned): `0%`

This affects the economics of liquidation in both market types.

## Suggested Reading Order

1. **CTokenInterfacesModified.sol** and **CTokenInterfacesModifiedCash.sol**: Start here. See what storage was added (KYC fields) and what constants changed (protocolSeizeShareMantissa). Compare the two.
2. **CTokenModified.sol**: The big one. Read the KYC/sanctions checks in `transferTokens`, `mintFresh`, `redeemFresh`, `borrowFresh`, `repayBorrowFresh`, `seizeInternal`. Compare each to vanilla Compound V2.
3. **CTokenCash.sol**: Same structure as CTokenModified but with KYC checks everywhere instead of sanctions checks. Spot the differences.
4. **OndoPriceOracle.sol** then **OndoPriceOracleV2.sol**: Understand how asset prices are determined. Look at the oracle type system in V2.
5. **cErc20ModifiedDelegator.sol**: The proxy. Check that storage layout matches the delegate implementations.
6. **JumpRateModelV2.sol**: Quick read. Only `blocksPerYear` changed.

## CASH subsystem reading route

After the lending pass, map the CASH request, settlement, refund and administration paths. Follow epoch state and identity status across transactions; read the complete called functions and dependency contracts, rather than only their names. Record which source path and snapshot each submission concerns.

## What to Look For

These prompts guide your analysis without spoiling specific findings. Think about each one carefully.

### Fork Differential Analysis
- What did Ondo change from vanilla Compound V2? For each change, ask: did they change it completely and correctly, or did they miss an edge case?
- When a fork modifies a constant, trace every code path that uses that constant. Do all paths still work correctly with the new value?

### Access Control and Gating
- Map every function that has a KYC or sanctions check. Now map every function that does NOT. Are there paths that should be gated but are not?
- What happens when a user's KYC status changes between two related operations (e.g., between supplying collateral and getting liquidated)?

### Exchange Rate and First-Depositor Dynamics
- How is the exchange rate calculated when a market is brand new (zero total supply)?
- What is the relationship between `totalSupply`, `totalCash`, and `exchangeRate`? Can any of these be manipulated independently?

### Oracle Trust Model
- Who can set prices? What validations exist on price values?
- What happens if the oracle returns zero or an extremely large value?
- How does the price cap mechanism in V2 work? Can it be bypassed?

### Liquidation Economics
- Trace a complete liquidation flow. What share goes to the liquidator, what goes to the protocol?
- How does `protocolSeizeShareMantissa = 0%` for cCASH affect liquidation incentives?

### Storage Layout
- When a proxy delegates to an implementation, storage slots must align. Does the modified delegator match the modified implementations?

## Scoring

| Severity | Points (True Positive) | Points (False Positive) |
|----------|----------------------|------------------------|
| High | +75 | -35 |
| Medium | +50 | -25 |

Findings must include: severity, affected file(s), root cause description, impact assessment, and a recommended fix. Vague observations without specific code references do not count.

## Important Notes

- **Scope includes lending and CASH/KYC as described above.** This matches the Academy exercise. Review both subsystems before submitting; the debrief belongs after your attempt.
- The `contracts/lending/compound/` directory contains unmodified Compound V2 code and is not in scope.
- Compare Ondo changes with the baseline, and check inherited assumptions against market initialization and the new integrations. Prior review of a parent does not establish every fork’s safety.
