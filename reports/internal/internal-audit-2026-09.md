---
title: "Internal audit, September 2026"
description: "Public report of the BTR internal audit campaign, 2026-09-02 to 2026-09-16: scope, method, funnel, and every finding on the open-source components as severity-ordered bundles."
audience: both
type: reference
status: live
lang: en
updated: "2026-09-16"
alias: "/docs/reports/internal/2026-09-16"
publish: true
---
# Internal audit, September 2026

Internal audit of the BTR DEX, run from 2026-09-02 to 2026-09-16 and closed before any
mainnet deployment. This page publishes the whole campaign for the open-source components.
Previously published as `2026-09-16` under the title "Security review, September 2026"; that
address redirects here.

Findings are published under the [disclosure policy](/docs/3-4-overview): a finding appears
once its fix is deployed to every chain running the affected code. Nothing below has run on
mainnet.

Code is named as it stands today, not as it stood when a row was filed: the pricing and config
libraries carry their `*Lib` names throughout, and the per-chain ceremony scripts quoted below
(`Deploy.s.sol`, `PoolDeploy.s.sol`, the `Arc*` and `OracleV*Deploy` scripts) were consolidated
after the campaign into `script/Protocol.s.sol`, `script/Pools.s.sol` and `script/Assets.s.sol`.
Locations are given as file plus symbol; line numbers are omitted because they move.

## 1. Funnel

Every row ever filed in the campaign, and what reached this page.

| Stage | Rows | Note |
|---|---|---|
| Filed | 829 | every candidate that survived refutation and was given an id |
| Not a real finding | 179 | 102 duplicate, 28 subsumed, 27 moot, 22 refuted |
| Real findings | 650 | the campaign's actual defect and design population |
| In open-source scope | 437 | located in `dex-evm`, `shared`, `sdk`, `front` or `core` |
| Published here | 433 | 284 of them above informational; 4 rows are held until their residual closes |

The remaining 213 real findings are located in the services, the keepers and the operational
environment. They are out of the public scope and are disclosed to auditors under
non-disclosure. Four price-feed rows are tracked with the upstream feed provider and are not counted above.

Published rows by severity and disposition:

| Severity | Fixed | Accepted | Closed | Total |
|---|---|---|---|---|
| Critical | 0 | 0 | 0 | 0 |
| High | 33 | 0 | 0 | 33 |
| Medium | 82 | 2 | 1 | 85 |
| Low | 156 | 3 | 7 | 166 |
| Informational | 53 | 13 | 83 | 149 |
| **Total** | **324** | **18** | **91** | **433** |

"Fixed" means code or documentation changed and the change is an ancestor of the published
component head. "Accepted" means the behaviour is the intended design and carries an operational
control instead of a code change; the reasoning is stated in the block. "Closed" means no code
change was warranted: a record correction, a testnet-only property, or a step of the launch
ceremony that retires the row.

The findings below are root-cause bundles, not raw rows: rows that share one mechanism and one
fix are presented as one finding, and each block lists the row ids it absorbs. They are ordered
Critical to Informational.

## 2. Scope

Audited heads, frozen 2026-09-15:

| Component | Repository | Audited head |
|---|---|---|
| AIMM pools, Admin, oracles, periphery, deploy scripts | `dex-evm` | `149fb16e3c` |
| Shared access control, quorum, upgrade gate, timelock | `shared` | `183ee41264` |
| TypeScript SDK: ABIs, router, transport | `sdk` | `99bc688fb9` |
| Web application | `front` | `ea7150fb` |
| Rust pricing mirror | `core` | `0d1990f7dd` |

Remediation heads, merged 2026-09-16: `dex-evm aca89e4e5a`, `shared 957d3b0939`, `sdk a95aabe`,
`front 8557e6c7`, `core 8ef3ef66b9`. Every fix commit cited below is an ancestor of its component
head, asserted mechanically by the workbook gate.

## 3. Scope and disclosure

This page covers the open-source components only. The back-end services, the keepers, the price
feed producer and the operational environment were reviewed in the same campaign under the same
method; those findings are disclosed to auditors under non-disclosure rather than published,
because their write-ups name infrastructure, key custody and operational procedure. Rows that are
still open are held until the residual closes, then published on the same rule as every other
finding. Proof-of-concept exploits are never committed in any repository, and
implementation-specific checklists are held: they are a map, not a method.

This audit is internal. It is not an independent opinion and does not substitute for one;
third-party audit is pending and reports will be linked from the
[Security Overview](/docs/3-overview) when they land.

## 4. Methodology

The audit is a multi-model adversarial process, not a single pass. Several frontier model families
are rotated across finding, refutation, debate, cross-validation and test generation, because
different pretraining produces different blind spots. It is refute-first: every candidate goes to
two independent reviewers instructed to refute it and defaulting to refuted, the party that
disagrees carries the burden, and a split goes to a third reviewer who has seen neither refutation.
Severity is graded at the shipping configuration, never at a hypothetical one, with exploit
difficulty and blast radius stated beside the grade. Surviving findings are re-derived against the
frozen tree and pinned as regression tests or property harnesses. The named lead engineer holds the
final call. The full method, including the stop rule, the severity doctrine, the coverage floor and
the anti-gaming register, is published at
[METHODOLOGY.md](https://github.com/btr-protocol/audits/blob/main/METHODOLOGY.md).

## 5. Findings

### F-01  A rejected price push could wedge a feed permanently, and the release lever shipped incomplete

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV4.sol:497`, `dex-evm/src/oracles/ExternalOracleV4.sol:842-857`, `dex-evm/src/oracles/ExternalOracleV4.sol:874-893` |

**Severity rationale.** A single ordinary gap at live sigma and cadence could make a feed unusable for the life of the deployed, non-upgradeable oracle, which halts every pool leg quoting that asset.

#### Description

The V4 deviation band gates each incoming lane against the previously stored mark. When the move exceeded the band, the lane was skipped and the stored mark stayed where it was, so the next push was measured against the same stale mark and was refused for the same reason. The gap never shrank on its own. The wedge threshold in practice was far below the nominal ten times `maxDevBps` figure the design assumed, because the band is also a function of sigma and of the elapsed time since the last accepted observation.

```solidity
// dex-evm/src/oracles/ExternalOracleV4.sol:497
        if (pm != 0) {
          uint256 nm = _decode(nl, int8(uint8(cfg >> 16)));
          uint256 diff = nm > pm ? nm - pm : pm - nm;
          uint256 movePbps = (diff * 1e6) / pm;
          if (movePbps / 100 > uint16(cfg)) {
            uint256 lsig = ((sPrev >> (lane * 24)) & SIG_MASK) << 4;
            if (!FeedMathLib.withinBand(pm, nm, dt, uint16(cfg), uint32(lsig))) {
              flags |= uint256(1) << (8 + lane);
              continue;
            }
```

The first remediation added a governed widen lever, `requestFeedWiden` / `executeFeedWiden`, and that lever carried its own defects across several rounds: it could not release a wedge beyond the `MAX_DEV_THRESHOLD_BPS` ceiling of 2000 bps; it wrote an absolute band value over whatever a guardian had tightened in the meantime; it dropped the stale-payload snapshot guard its V1 twin carried; it refused to overwrite an expired pending operation; it cleared the lane and the band anchor without stamping the slot clock; and the event it emitted was not reachable by any consumer. The lever also sat at the BASE timelock tier, which is two days on mainnet, even though a pure release cannot set a price.

#### Impact

A wedged lane quotes a disowned mark or reverts, and on a non-upgradeable oracle the only remedies were a governed widen at a two-day delay or a full redeploy with a repoint of every consuming leg. A paused and wedged feed needed two separate ceremonies with interleavings that could re-wedge it.

#### Exploit scenario

1. A market move larger than the lane's adaptive band arrives at the oracle.
2. The lane is skipped and the stored mark is left at its pre-move value.
3. Every subsequent push is measured against that same stale mark and is refused for the same reason, so the gap never closes.
4. The feed stays dark until governance executes a widen at the BASE delay.

#### Remediation

Rule R7 self-heal was added: on the first band refusal a lane quarantines itself by writing its own observation second and clearing the lane, so `gate` reverts rather than quoting a disowned mark, and the band then widens against the lane's own gap. No new storage, no read-path change and no happy-path gas cost. The widen lever was hardened in successive rounds, and in V5 the release and the band widen are split: the release is `attestReentry`, quorum-attested and anchor-bounded, while the band widen is config-only at the LISTING tier with a guardian cancel. `executeFeedWiden` in V5 touches configuration only.

[`502feb1`](https://github.com/btr-protocol/dex-evm/commit/502feb1df5277235521f6af1ee3098691dd884ec), [`e03d016`](https://github.com/btr-protocol/dex-evm/commit/e03d016754d46b67f8a5070cf5a2a7a07f1fe552), [`f142169`](https://github.com/btr-protocol/dex-evm/commit/f1421692ecdd5ee8fdcf16bd05bb2c42ed257d07)

#### Status

Fixed. The lever hardening is verified on the fix branch. The R7 self-heal is fixed on `main` and is not deployed: the oracle is not upgradeable, so it lands with the redeploy and the 37-leg repoint. The tier split ships in V5.

**Rows.** 19 rows.

### F-02  Role bootstrap and the quorum check were fail-open before the signer set was seeded

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Closed | Audit | `dex-evm/src/oracles/ExternalOracleV5.sol:219-238`, `shared/evm/src/access/AccessControl.sol:535-545` |

**Severity rationale.** An uninitialized or zero-threshold oracle accepted unsigned marks, and the same bootstrap path let an owner install a new treasury owner instantly, so both the price surface and the treasury role were reachable without the intended quorum.

#### Description

`_quorumCheck` compared the recovered signature count `n` against the threshold `k` with `n < k`. At `k == 0` an empty signature set satisfies the comparison and the loop body never runs, so the check passes. Nothing forced `initialize` to run before `registerFeed` or `push`, so an oracle that had not been seeded, or one whose threshold was zero, accepted marks with no signatures at all.

```solidity
// dex-evm/src/oracles/ExternalOracleV5.sol:225 (_quorumCheck)
    uint256 n = sigs.length / 65;
    if (sigs.length % 65 != 0 || n < k) revert Err.NotAuth();
    address prev;
    for (uint256 i; i < n;) {
      uint256 off;
      unchecked {
        off = i * 65;
      }
      address rec = ECDSA.recoverCalldata(digest, sigs[off:off + 65]);
      if (rec <= prev || !set[rec]) revert Err.NotAuth();
      prev = rec;
    }
```

On the shared access-control side, `bootstrapRole(TREASURY_OWNER)` had no post-arm guard. If `treasuryOwnerBootstrapped` was never spent, the owner could install a new treasury owner immediately, bypassing the seven-day rotation and the incumbent veto. `bootstrapRole(FACTORY)` was likewise ungated, and `armQuorumPolicy` did not require a non-zero factory, so a quorum policy could be armed around an unset or burnable factory role.

#### Impact

Unsigned marks on an unseeded oracle mean the price surface has no authority behind it. The bootstrap gap is an instant, veto-free replacement of a privileged role that the rotation path exists to make slow and contestable.

#### Exploit scenario

1. An oracle is deployed but `initialize` has not run, or its threshold is zero.
2. A caller submits a push with an empty signature blob.
3. `_quorumCheck` returns without reverting and the marks are stored.

#### Remediation

`_quorumCheck` now rejects `k == 0` and empty signature sets, and `_register` refuses to register against an unseeded signer set. `bootstrapRole` reverts post-arm for both `TREASURY_OWNER` and `FACTORY`, and `armQuorumPolicy` requires a non-zero factory. Covered by the quorum and oracle test pins.

[`d04a781`](https://github.com/btr-protocol/shared/commit/d04a78111ab4eab2929e0e883e09784f3bc5ed08), [`08df008`](https://github.com/btr-protocol/shared/commit/08df008a7dd2bb80eb495360cfaa6dca05a420c3), [`d35ca5f`](https://github.com/btr-protocol/dex-evm/commit/d35ca5fe02790be886f32c3b5674ecc270550df5), [`456b10a`](https://github.com/btr-protocol/dex-evm/commit/456b10a6ed5d35de4d79818708326ca4f40cbd69)

#### Status

Closed.

**Rows.** 3 rows.

### F-03  Pools created through the permissionless factory path were born with no authority and no fee sink

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | QA | `dex-evm/src/Pool.sol:116-128`, `dex-evm/src/PoolFactory.sol:113`, `dex-evm/src/PoolFactory.sol:167-174` |

**Severity rationale.** The permissionless creation path was reachable by anyone at the shipping configuration and every pool it produced was either unusable or governed by the wrong key, so likelihood was certain and the impact reached protocol fee routing and pool control.

#### Description

`createPool` was permissionless, but `Pool.initialize` wrote only `baseToken`, `wnative`, `flowCooldownSecs`, `factory` and `initialized` while still accepting `protoSharePct`. A pool created that way accrued a protocol share to an unset treasury address.

```solidity
// dex-evm/src/Pool.sol:117
  function initialize(address baseToken_, address wnative_, IPool.FeeParams calldata feeParams)
    external
  {
    if ($.initialized) revert Err.InvalidState();
    // ONE definition of the fee-param bounds, shared with `adminSetFeeParams`.
    PoolConfig.setFeeParams($, feeParams);
    $.baseToken = baseToken_;
    $.wnative = wnative_;
    $.flowCooldownSecs = C.DEFAULT_FLOW_COOLDOWN_SECS;
    $.factory = msg.sender;
    $.initialized = true;
```

The same path produced a pool with zero listed assets whose twenty configuration entrypoints were all gated on `AccessControl(AC).owner()`, so the deployer who paid for the proxy could never list an asset: the deployer address was never persisted. Where the deployer could name itself through `_requireAuthoritySelfNamed`, `_registerPool` still branded the pool official because official status keyed on the creator alone, producing a pool inside `officialPools` on which the owner multisig held no write at all. Adjacent defects on the same surface: `initialize` did not reject `baseToken == 0`, `donate` skipped the seal and allowlist gates that `deposit` carried, the donate-back sentinel pinned a static key instead of resolving `AccessControl.owner()`, `setProtocolDeployer` was instant while its factory sibling was timelocked, `receive()` accepted stray native value with no ledger, `adminSetDeadSeedPow10` allowed a bounded pre-seed grief, and the bootstrap seal was read from a new pool bit while live pools had been sealed through the Admin mapping.

#### Impact

A stranger-created pool was a brick or, in the self-naming case, an official-branded pool outside the owner's write surface that could list assets off its own oracle, seal bootstrap and take third-party deposits. Pools created before the fix also routed a protocol share to an unset address.

#### Remediation

`initialize` now pins `$.treasury` from `AccessControl.treasury()` and reverts on zero, and rejects `baseToken == 0`. The creator is recorded as `poolAdmin`, so a third-party pool is configurable by the party that deployed it. Official status is no longer derived from the creator: `_requireAuthoritySelfNamed` does not exist and only the owner's `requestOfficial` / `executeOfficial` lane grants the brand, with `_assertOfficialShape` pinning the protocol treasury. `donate` shares `deposit`'s seal, allowlist and cap gates, the donate-back sentinel resolves through Admin, `protocolDeployer` and `setProtocolDeployer` are deleted, `receive()` is gated to `wnative` with a sweep arm, and one `Admin.bootstrapSealed` mapping is the single seal latch.

Commits: [`f5c281f7`](https://github.com/btr-protocol/dex-evm/commit/f5c281f7cdc161bb244d80d70e2cf3db1d64ccd7), [`e13745ae`](https://github.com/btr-protocol/dex-evm/commit/e13745ae095e491de72cc6a23a550e7cf5d44244).

#### Status

Fixed. Verified 2026-09-11; the residual informational rows were closed on the 2026-09-14 review.

**Rows.** 11 rows.

### F-04  Hook ledger writers booked reserves, liabilities and the liquidity index without proving the underlying balance moved

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | QA | `dex-evm/src/Pool.sol:765-820`, `dex-evm/src/Pool.sol:709-753`, `dex-evm/src/Pool.sol:682-685` |

**Severity rationale.** `hookWriteDown` wrote the absorbing index state on the exact total-loss case that its own specification claimed to exclude, and the surrounding writers booked value on unproven balances, so a single venue loss could wipe a leg's claim permanently.

#### Description

`hookWriteDown` applied its `minLiab` floor under `if (liabAfter > 0 && idx > 0)`. At the total-loss case `liabAfter` is zero, so the floor was skipped and the function wrote `newIdx = 0`, the absorbing state.

```solidity
// dex-evm/src/Pool.sol:651
    uint256 liabAfter = liabBefore - cutLiab;
    if (liabAfter > 0 && idx > 0) {
      uint256 minLiab = (liabBefore + idx - 1) / idx;
      if (liabAfter < minLiab) {
        liabAfter = minLiab; // <= liabBefore for idx >= 1, so cutLiab only ever shrinks
        cutLiab = liabBefore - liabAfter;
      }
    }
```

The other writers on the same ledger shared the shape. `hookCreditYield` booked reserves, liabilities, invested and the index with no `balanceOf` proof, so an honest-but-buggy or donation-inflated venue NAV was sufficient to inflate the index, and it carried no `HALT_MASK` gate while `hookDeploy` did. `hookDeploy` booked `invested` with no delta proof, so a fee-on-transfer leg overstated the invested balance and blocked withdrawals. `hookRecall` proved its balance against the raw token, so the native sentinel spelling always read zero and reverted. `collectProtocolFees` lacked the `requireNoFlash` guard its four sibling hook writers carried.

#### Impact

A total venue loss set the liquidity index to zero and bricked the leg's claim irrecoverably. Index inflation from an unproven credit was socialized across LPs at up to the daily rate cap. The `hookDeploy` and `hookRecall` defects were liveness only: blocked withdrawals and a reverting native recall path.

#### Remediation

The `minLiab` floor now applies unconditionally whenever `idx > 0`, so a total loss no longer writes a zero index; flooring `liabAfter` rather than the index preserves `S*idx/WAD <= L`. `hookCreditYield` proves `bal >= R_liq + protocolFees + amount` before any book move, no longer touches `invested`, and refuses new credit under `HALT_MASK` while recall and write-down stay open for exit. A hook cannot be installed on a `TOKEN_EXOTIC_BIT` leg, so `hookDeploy` never books a taxed push. `hookRecall` balances the wrapped leg token. `collectProtocolFees` calls `requireNoFlash`.

Commits: [`e36dbb86`](https://github.com/btr-protocol/dex-evm/commit/e36dbb86c4e0d36a0b57893a168f9875bfb9aba1).

#### Status

Fixed. Verified 2026-09-10 and 2026-09-11.

**Rows.** 6 rows.

### F-05  Genesis deploy scripts could brick an immutable mainnet deployment

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | QA | `dex-evm/script/OracleV5Deploy.s.sol:119`, `dex-evm/script/PoolDeploy.s.sol:223`, `dex-evm/script/ArcRiskRestore.s.sol` |

**Severity rationale.** The governance-delay omission ran against an immutable `AccessControl`, a beacon and a burned CREATE3 salt, so one mainnet run at the wrong schedule was unrecoverable.

#### Description

`OracleV5Deploy._deployTier` never called `_govDelays()`, so a `class=mainnet` run accepted the retired seven-tier schedule. The V4 script called it; the V5 branch dropped the call between the chain assertion and the first broadcast.

```solidity
// dex-evm/script/OracleV5Deploy.s.sol:86
  function _deployTier() internal returns (address ac, address oracle) {
    _assertChain();
    string memory outPath = _outPath();
    require(
      !_oracleLive(outPath) || vm.envOr("REDEPLOY", false),
      string.concat("already deployed: ", outPath)
    );
    uint256 pk = vm.envUint("DEPLOYER_PK");
    address deployer = vm.addr(pk);
    address guardian = vm.envAddress("GUARDIAN");
    require(guardian != address(0) && guardian != deployer, "GUARDIAN must be independent");
```

Four further ceremony defects sat on the same scripts. `PoolDeploy` bound `REF_ORACLE` with no provenance check, so a wrong but valid oracle address was accepted as the reference tier. `deployCore` validated `.depositors` mid-broadcast, after the core singletons were already live, so a missing list left a half-deployed genesis. `ArcRiskRestore` gated the risk op on `kappa` alone in both `request()` and `execute()`, so a cap-only backfill could never arm. The deploy record for the target chain was a twin of another chain's record, and DEPLOY.md named retired scripts, the retired governance-delay schedule and stale keeper configuration files.

#### Impact

A mainnet genesis run could have provisioned the wrong immutable governance schedule, bound the wrong reference oracle, or stopped half way with the core singletons already live and no path back.

#### Remediation

`_govDelays` now runs before the first broadcast. The reference tier is bound off its own record with class-aware lanes and the V4 reference record lands where `PoolDeploy` reads it. `.depositors` is validated pre-broadcast. `ArcRiskRestore` gates `UPDATE_RISK` on the whole payload. The chain deploy record is an explicit zero-address scaffold that the ceremony overwrites, with `PoolDeploy` refusing a zero `.ac` or `.oracle`, and DEPLOY.md was swept for the retired scripts, the three-tier delay table and the live generator names.

Commits: [`921bbdbf`](https://github.com/btr-protocol/dex-evm/commit/921bbdbf8681b15e5a6d70d92d6df97f66d8ed30), [`719da508`](https://github.com/btr-protocol/dex-evm/commit/719da508c74773d3b09a276666682460041cbfbe), [`60adb83a`](https://github.com/btr-protocol/dex-evm/commit/60adb83ae74e89d080e0fd378b0a09364b2dcdc8), [`99c6b56b`](https://github.com/btr-protocol/dex-evm/commit/99c6b56be6b91a15c16998cb1f2c1a0442e71f29), [`3e761867`](https://github.com/btr-protocol/dex-evm/commit/3e7618672bd4aa1bc09ab8e668c1087738d10365), [`10a8b1e9`](https://github.com/btr-protocol/dex-evm/commit/10a8b1e97ff2a5ae0ad601445bcb79579544c0fd), [`acdc119b`](https://github.com/btr-protocol/dex-evm/commit/acdc119b5dbbbe3ade8f4d4efd15cdc09f13c5c0), [`efa9d800`](https://github.com/btr-protocol/dex-evm/commit/efa9d8005e6f5564cf8b1d1dcc90c59f47b3d544), [`1f721f0c`](https://github.com/btr-protocol/dex-evm/commit/1f721f0c8d1c1f53dd4d2ed27be94e5ecdccccf8), [`91e73452`](https://github.com/btr-protocol/dex-evm/commit/91e73452f5423f91815666bf2aab0d87400c1b70).

#### Status

Closed. Script fixes landed 2026-09-15; the deploy-record scaffold was verified on the 2026-09-14 review.

**Rows.** 8 rows.

### F-06  Oracle signer and reference governance had no tests, and revoke could leave the quorum permanently unsatisfiable

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | QA | `dex-evm/src/oracles/ExternalOracleV5.sol:823-966`, `dex-evm/src/oracles/ExternalOracleV5.sol:758-772`, `dex-evm/test/unit/ExternalOracleV5.t.sol:333-340` |

**Severity rationale.** An untested governance lifecycle on an immutable oracle plus a revoke path that can raise the threshold above the signer count makes a permanent, unrecoverable authorization brick reachable through a routine key rotation.

#### Description

Every V5 signer and reference-signer governance function other than `revokeSigner` was uncovered: the grant batch execute and cancel paths on both tiers, `requestRefSignerGrantBatch`, `revokeRefSigner`, both threshold-decrease lanes and `clearQuarantine`. A guard could be deleted and CI would stay green.

`_revoke` removed the signer from the set and the enumeration list and never lowered the threshold, so revoking from a k-of-k set left `threshold > signerCount` and every quorum check reverted `NotAuth` forever. The existing `test_revoke_and_enumerate` asserted the shrink green with no quorum floor.

```solidity
// dex-evm/src/oracles/ExternalOracleV5.sol:757
  function _revoke(mapping(address => bool) storage set, address[] storage list, address a, bool ref)
    private
  {
    if (!set[a]) return;
    set[a] = false;
    uint256 n = list.length;
    for (uint256 i; i < n; ++i) {
      if (list[i] == a) {
        list[i] = list[n - 1];
        list.pop();
        break;
      }
    }
```

#### Impact

A revoke taken from a k-of-k roster permanently disables mark pushes on that oracle instance. The absent tests meant the same class of regression could land unnoticed on any of the ten untested lifecycle entrypoints.

#### Remediation

Revoke below the threshold is now refused on both tiers, so a k-of-k roster must grant before it revokes. Nine lifecycle tests cover the grant, cancel, revoke and threshold paths, and `clearQuarantine` was deleted in favour of a latching quarantine bit.

Commits: [`44b8858c`](https://github.com/btr-protocol/dex-evm/commit/44b8858cf9493b476f20505a7e0bd8fbb0d75bf4), [`e20dc680`](https://github.com/btr-protocol/dex-evm/commit/e20dc68007b9590858df6e8844d13db8e2987bde), [`761a7f07`](https://github.com/btr-protocol/dex-evm/commit/761a7f07f938ccba6f2970ea4fdf150649fc68ca).

#### Status

Closed.

**Rows.** 2 rows.

### F-07  The release shipped an unratified governance ladder and a beacon upgrade delay that contradicted it

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | QA | `dex-evm/src/oracles/OracleBeacon.sol:39`, `dex-evm/src/Admin.sol:246-284`, `shared/evm/src/ConstantsLib.sol:28-53` |

**Severity rationale.** The delay a beacon reads at construction is immutable for the life of that beacon, so shipping it before the mechanism was ratified would have frozen the wrong upgrade lane into the deployment.

#### Description

The release tip carried a paused three-tier governance-delay ladder together with a V5 beacon whose constructor pinned the upgrade delay to the `GOVERNANCE` tier, while the mechanism decision itself was still open. `DELAY_UPGRADE` is set once in the constructor.

```solidity
// dex-evm/src/oracles/OracleBeacon.sol:34
  constructor(address ac_, address impl_) {
    if (ac_ == address(0) || impl_ == address(0)) revert Err.ZeroAddr();
    if (ac_.code.length == 0 || impl_.code.length == 0) revert Err.NotCode();
    AC = ac_;
    implementation = impl_;
    DELAY_UPGRADE = SC.delayOf(AccessControl(ac_).GOV_DELAYS(), SC.Tier.GOVERNANCE);
  }
```

#### Impact

Deploying against an unratified ladder would have written a governance schedule and an oracle upgrade lane that the eventual decision did not match, on contracts where neither is changeable after deployment.

#### Remediation

The owner ratified an upgrade-only beacon with the oracle implementation at the `LISTING` tier, the guardian veto kept, the `UPDATE_ORACLE` repoint lane deleted and the three-tier ladder unpaused. `DELAY_UPGRADE` now reads the `LISTING` tier off the timelock op word, the `UPDATE_ORACLE` lane and its enum slot are gone with every consumer renumbered, the shared constants natspec states the ratified ladder, and the delegatecall overhead was measured in a test.

Commits: [`e4c3cb19`](https://github.com/btr-protocol/dex-evm/commit/e4c3cb19ba895c4b3119c1fae483d4996f9062ec), [`1730979d`](https://github.com/btr-protocol/dex-evm/commit/1730979d7defdb900bc0f458dcd7a8c47611923c), [`21fea2d0`](https://github.com/btr-protocol/dex-evm/commit/21fea2d08713926dbf7c610d2992af279c7e170e), [`6325fe91`](https://github.com/btr-protocol/dex-evm/commit/6325fe91b129057f7091593488156b7c234070f6), [`a860d45f`](https://github.com/btr-protocol/dex-evm/commit/a860d45fee660b182ceeee2246cf66f299523c15), [`cafa28d`](https://github.com/btr-protocol/shared/commit/cafa28d09d0725b23d9ae277f1d730c07edd77ba).

#### Status

Closed. Ratified 2026-09-14, verified on the same review.

**Rows.** 1 row.

### F-08  Guardian and foreign-pool authority lanes reached pools the protocol does not administer, allowing a permanent brick and an LP-facing rug

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/Admin.sol:309`, `dex-evm/src/Admin.sol:467`, `dex-evm/src/libraries/PoolConfigLib.sol:282-286` |

**Severity rationale.** A single guardian key could, without a timelock and without cooperation from the pool that owns the assets, put a foreign pool's leg into a state that only a CRITICAL-tier operation could restore, and then veto every restore attempt.

#### Description

`Admin` exposes several levers whose authority check was protocol-scoped rather than pool-scoped. `collapseAnchor` was reachable by the protocol guardian on any pool, including pools with their own `poolAdmin`, and its mandatory companion halt makes the leg's mark meaningless until an anchor is re-established. The only un-collapse path is `executeAnchorUpdate` at CRITICAL tier, and `cancelTimelock` took the same guardian-or-admin check and deletes a pending operation in both directions, so the same principal could cancel every re-request. Re-request is permitted, so the loop was unbounded.

```solidity
// dex-evm/src/Admin.sol:309
  function cancelTimelock(address pool, uint8 opType, bytes32 subject) external {
    _onlyGuardianOrAdmin();
    bytes32 key = _keyOf(pool, opType, subject);
    if (pendingOps[key] == 0) revert Err.NoPending();
    delete pendingOps[key];
    delete pendingData[key];
    emit TimelockCancelled(pool, key, opType);
  }

// dex-evm/src/Admin.sol:467
  function collapseAnchor(address pool, address token, address newAnchor) external {
    _onlyGuardianOrAdmin();
    IPool(pool).adminCollapseAnchor(token, newAnchor);
```

A second defect sat on the release side. `unhaltAsset` took a caller-supplied `src` mask and `PoolConfigLib.setHalt` cleared whatever bits the mask named with no record of which principal set them, so a foreign pool admin calling `unhaltAsset(pool, token, HALT_MASK)` cleared the guardian's halt alongside their own. On a pool with a non-zero `poolAdmin`, `collapseAnchor` (guardian-or-pool-admin) and `unhaltAsset` (pool-admin) were reachable by one principal, which is the collapse-then-unhalt rug against that pool's own LPs. Separately, the guardian held an un-halt edge on foreign pools at all, contradicting the documented HALT / TIGHTEN / CANCEL one-direction invariant in `shared/evm/src/AccessControl.sol`.

`setRiskFences` was opt-in with no opt-out: it reverted on `maxDeltaBps == 0` and no clearing function existed, so once a foreign pool admin armed fences on a leg, the protocol risk steward held permanent write access to that leg's `minLiquidity`, `minFee`, `vega` and `haircut`. The bootstrap instant listing lanes also never expired and the guardian could not seal them.

#### Impact

On any pool with an independent administrator, a protocol guardian key could halt a leg indefinitely, and the pool's own administrator could clear a guardian halt that existed for a reason. Neither direction required a timelock, so the normal governance delay offered no window to react. Withdrawals on a collapsed leg are gated, so the brick is a fund-availability event, not only a liveness one.

#### Exploit scenario

1. A pool with a non-zero `poolAdmin` lists a leg and takes third-party liquidity.
2. The guardian calls `collapseAnchor` on that leg. The mandatory halt lands with it and the mark is no longer meaningful.
3. The pool's administrator requests `executeAnchorUpdate` at CRITICAL tier to restore the anchor.
4. The guardian calls `cancelTimelock` on that key. The operation is deleted before it can mature.
5. Steps 3 and 4 repeat without bound. The leg stays halted and withdrawals stay gated.

#### Remediation

Cancel is now seat-routed: on a foreign pool only the pool's own seat can cancel, and `collapseAnchor` refuses foreign pools outright, so the collapse lever exists only where the protocol is the administrator. The guardian no longer holds an un-halt edge on foreign pools. Halts are refcounted by source, so clearing one source's bit cannot clear another's. `clearRiskFences` was added so an armed fence can be revoked. `sealBootstrap` is guardian-or-owner, a one-way tightening of the listing lane, so the instant bootstrap lanes can be closed before a pool opens to public liquidity. The `haltAsset` natspec was rewritten to state the single and combined `HALT_MASK` behaviour for governed pools and `NotAuth` for foreign pools; the governed path already refcounted by source, so no code change was needed there.

Commits: [`e13745ae`](https://github.com/btr-protocol/dex-evm/commit/e13745ae095e491de72cc6a23a550e7cf5d44244), [`f5c281f7`](https://github.com/btr-protocol/dex-evm/commit/f5c281f7cdc161bb244d80d70e2cf3db1d64ccd7), [`78153012`](https://github.com/btr-protocol/dex-evm/commit/78153012cd344440b98cddd164aeae43b9939e9e), [`fb975b2b`](https://github.com/btr-protocol/dex-evm/commit/fb975b2bc306c0d04ce66258ef31b5f18bbc85cf).

#### Status

Fixed. Foreign-pool cancel and collapse verified 2026-09-10; guardian seal verified 2026-09-11; natspec verified 2026-09-11. Three rows are closed rather than fixed: the partial batch sweep (`try`/`catch` plus `BatchLegSkipped`, no retry queue) is specified behaviour and was closed as a design note on 2026-09-14; the sweep-completeness caveat and the missing `AssetHalted` emission on collapse and batch legs were closed as observability-only with no fund path. Pools are sealed in the listing broadcast.

**Rows.** 11 rows.

### F-09  The Rust pricing mirror settled a sell leg in spoke scale against a base-scale hub book

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `core: src/route.rs`, `core: src/pricing.rs`, `dex-evm/src/libraries/PricingLib.sol:562-579` |

**Severity rationale.** The mirror is the quote surface's reference implementation, and a mixed-decimal sell produced a settled amount wrong by the decimal shift between the two legs on every route through a hub whose book is held in a different scale.

#### Description

`core` quotes a leg in the spoke's scale and then settles against the hub book. For a sell the gross quantity was passed to `Endpoint::settle` without being re-denominated into the hub's raw units, while the Solidity path applies `_legScaleOut` before `_settleQuote`. The test fixture that would have caught it carried a hub with zero liabilities, which turns the coverage wall off, so the parity wall was effectively disabled on exactly the path that diverged.

```rust
// core: src/route.rs
        // Both quantities leave the pricer in the SPOKE's scale; a sell then shifts to the base's.
        let (net, gross) = if selling {
            (
                dec_shift(q.amount_out, s.decimals, base_dec),
                dec_shift(q.gross_out, s.decimals, base_dec),
            )
        } else {
            (q.amount_out, q.gross_out)
        };
```

Two adjacent mirror defects sit on the same surface. The `cov_toll` `kappa == 0` arm returned `gross_out` directly while the Solidity V5 gross-cap path takes a different branch, so the two implementations disagreed on the `kappa == 0` edge. Separately, both sigma terms of the spread were documented on the BPS scale while the code prices one percent of sigma on the PBPS scale, a hundredfold difference between the stated and the implemented semantics, and the research simulator ran `STALE_Z = 100` against a chain value of 472 with its parity gate pointing at a deleted path.

#### Impact

A sell through a hub whose book is in a different decimal scale settled against the wrong quantity, which moves the coverage wall and the settled output. The documentation divergence on vega meant an operator reading the parameter tables would have chosen a vega a hundred times away from the intended one.

#### Remediation

`Pricing::walk` and `Quote::settled` were split, and a sell's gross is rescaled into the hub's raw units before `Endpoint::settle`, mirroring `_legScaleOut`. The hub fixture carries liabilities and `kappa = 600` again, so the coverage wall is on in the parity tests, and directed sell, buy and round-trip vectors were added. The `kappa == 0` short-circuits were dropped so the Rust arm follows the same branch as `PricingLib.sol`. Vega is documented once as PBPS-scaled across contract, SDK, core and the public parameter docs; the owner decision is to keep one percent of sigma on the PBPS scale for the phase-one stable set and to rescale only alongside the volatile-core fit, so there is no pricing change. Research constants are pinned to `PricingLib.sol` and the parity gate re-points at the sibling checkout.

Commits: `core@6440d93`, `core@4485e8d`, `core@b75f647`, `core@3f71c59`, `core@35c8e7ad`, `core@0d1990f`, [`83d87923`](https://github.com/btr-protocol/dex-evm/commit/83d87923904b563c12de51969a52b38a4d138110), [`8c21532`](https://github.com/btr-protocol/sdk/commit/8c2153214febdf4f0e97b538caced9b9dd528c03), [`fac1138`](https://github.com/btr-protocol/sdk/commit/fac1138679e9607785dd4bfd321251dce526758c), `research@524cec4`, `research@52f999f`, [`e16806e`](https://github.com/btr-protocol/content/commit/e16806edebb6a3975ebf20c68ac90bd3bab1bc7b).

#### Status

Fixed, verified 2026-09-14. The `kappa == 0` parity row was conditional on the matching contract change landing.

**Rows.** 3 rows.

### F-10  The front end transaction builder debited the full typed amount on the first leg and pinned token decimals by symbol

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `front/src/components/features/swap/SwapForm.tsx:1121`, `front/src/config/testnet-tokens.ts:205`, `front/scripts/lib/livePools.ts:168` |

**Severity rationale.** Both defects produce a correctly signed transaction carrying the wrong amount, and the decimal one is off by a factor of 1e12 on a chain where the affected token is a primary leg.

#### Description

On the market-first LP deposit route, leg zero was given the full typed input amount while later legs were sized from float values carried on the route steps rather than from the previous leg's floored output. The multi-leg deposit therefore did not compose: the first hop spent the whole budget and the later hops were sized from numbers that had already lost precision.

```ts
// front/src/components/features/swap/SwapForm.tsx:1121
            .map((st, i) => ({
              pool: (st.poolAddr ?? poolAddr) as Address,
              tokenIn: tokenAddr(st.tokenIn),
              tokenOut: tokenAddr(st.tokenOut),
              amountIn:
                i === 0 && exactAmountIn !== undefined
                  ? exactAmountIn
                  : stepBig(st.amountIn, st.tokenIn),
              quotedOut: stepBig(st.amountOut, st.tokenOut),
              minOut: stepBig(st.minOut, st.tokenOut),
            })),
```

Token metadata was keyed by symbol, and the table pinned USDC and EURC at six decimals for every chain. BSC USDC has eighteen. Every encoded amount and every LP floor derived from that table was out by 1e12 on that chain. Separately, the router harnesses hand-rolled `curveToWire`, including a median of 5000 and a written sentinel boundary, instead of calling the SDK, so the harness and the shipped encoder could disagree.

#### Impact

A user submitting a market-first LP deposit could send a transaction whose first leg consumed the entire typed amount. On a chain where the symbol-keyed decimals are wrong, an amount intended as one unit encodes as 1e12 units or the reverse, depending on direction, and the LP floors derived from the same metadata are wrong by the same factor.

#### Remediation

Encoded amounts now equal the typed amount in the token's decimals for that chain, and every hop after the first is sized from the previous hop's floored output rather than from a float. Token decimals resolve per chain instead of per symbol. The router harnesses call the SDK encoder, removing the hand-rolled duplicate.

Commits: [`c24b397`](https://github.com/btr-protocol/front/commit/c24b3977d525ed46f4ab7e9ef6037bb97632200f), [`828dc37`](https://github.com/btr-protocol/front/commit/828dc37605bbc68bc31a2739354f1a9f38621e32), [`a2d8219`](https://github.com/btr-protocol/front/commit/a2d8219897e2f52d10ea6f61605739be50f1726e), [`c2a7783`](https://github.com/btr-protocol/front/commit/c2a77838a6756fc2eed96dad4d2e0a0a4ca71ec5).

#### Status

Fixed 2026-09-16.

**Rows.** 3 rows.

### F-11  The client-side quote read chain risk parameters that did not match the chain, first failing open on the coverage wall and then failing closed on every pool

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `front/src/hooks/useAllPools.ts:92`, `front/src/hooks/usePoolData.ts:87`, `front/src/config/aimm-profiles.ts:69` |

**Severity rationale.** The front end quotes off an in-browser replica of the on-chain pricing law, so any divergence between the replica's parameters and the chain's is either a systematically wrong price shown to every user or a fleet-wide loss of quoting, both reachable with no attacker and no special state.

#### Description

The pool state builder defaulted `kappaCovBps` to `0` whenever the parameter was absent from the multicall result. Zero is the value that disables the coverage wall, so the replica quoted a zero coverage toll on legs where the chain was charging 600 to 2500 basis points. The default was fail-open in the one direction that matters: the client understated the cost of the trade it was about to send.

```ts
// front/src/hooks/useAllPools.ts:92
      const hub: PoolState['hub'] = baseRow
        ? {
            res: baseRow.amount,
            liab: baseRow.liabilityAmount,
            vegaBps: liveProfile(cfg.tag, baseRow.params)?.vega ?? 0,
            kappaCovBps: baseRow.params?.kappaCovBps ?? 0,
          }
        : undefined;
```

The first remediation removed the fail-open default but routed the value through a numeric guard that rejects `bigint`. The SDK ABI decoder returns `uint` as `bigint`, so the guard rejected every well-formed read: the wall resolved as unknown, every pool degraded to the illustrative (non-quotable) path, and the fleet had zero quotable pools. That regression was tracked separately and fixed by giving the decoder a typed asset conversion and a `bigint`-aware resolver.

A third divergence sat in the static profile mirror: `aimm-profiles.ts` still carried `vega` 10000 and minimum fees of 50, 1032 and 1000 basis points after the chain had moved to 3000 to 4500 and 90, 1500 and 1200.

#### Impact

With the fail-open default in place the displayed quote understated the toll the chain would charge, so the user saw a better price than the one the transaction would settle at. With the regression in place the product could not quote at all. The stale profile mirror moved the replica's spread and floor away from the chain's on every leg it covered.

#### Remediation

The fail-open default is gone, the coverage wall is a required input on every leg, and the resolver accepts the `bigint` the decoder actually produces and floors it. The profile mirror is re-derived per class from chain values. Regression tests pin the decoder conversion and the wall resolution.

Commits: [`c9b0b44a`](https://github.com/btr-protocol/front/commit/c9b0b44a45e0a770a36697f6ccfba1b6c268efce), [`f40a8490`](https://github.com/btr-protocol/front/commit/f40a849081c45b3a97f4cab3c613f8a2403bd591), [`39bb177`](https://github.com/btr-protocol/sdk/commit/39bb17754144967e90928c6d190bebcba5974e15), [`edddb2c2`](https://github.com/btr-protocol/front/commit/edddb2c244dbe61e057ad4ce94a9154481fc5fbc).

#### Status

Fixed. The fail-open default and the profile drift were closed on 2026-09-10 and 2026-09-11; the `bigint` regression was found by the 2026-09-11 validation pass and fixed in the same wave.

**Rows.** 3 rows.

### F-12  The safety console served a stale oracle ABI, leaving the guardian pause selector empty during a live incident

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `front/src/pages/safety/SafetyLevers.tsx:107`, `front/src/pages/SafetyPage.tsx:567`, `front/src/pages/safety/safetyModel.tsx:300` |

**Severity rationale.** The emergency console is the operational path to the fail-closed levers; an empty feed selector removes the intended way to use them at exactly the moment they are needed, and the workaround is a hand-built raw transaction.

#### Description

The console resolved the oracle ABI by name and was served the V1 interface for a V4 address. `getFeedIds()` does not exist on any oracle generation and reverts on V4, so the "Pause feed" selector enumerated nothing and the guardian's own emergency lever could not be driven from the emergency UI. During a live incident the affected feed had to be paused by a raw contract call instead.

```ts
// front/src/pages/safety/SafetyLevers.tsx:107
  // the roster is a BUILD-TIME fact (SDK lane map x venue feedIds), never an on-chain
  // enumeration - no oracle generation exposes `getFeedIds()`, and asking a V4 for it reverts, which
  // is what left this selector (the emergency pause lever) permanently empty. `laneFeedMeta` is the
  // same roster the transparency page reads; taking it whole (via `useOracleData`) would also drag
  // this mount into an explorer tx-list fetch, a push decode and an indexer roster poll it has no
  // use for, so only the roster + ONE getFeed multicall are borrowed.
  const feedMeta = useMemo(() => laneFeedMeta(chainId), [chainId]);
```

Four smaller console defects sit on the same surface. The pause button stayed re-fireable once a feed was already paused, producing an inert success and a duplicate event at gas cost; the admin side gated correctly. The unhalt copy claimed to clear every halt source, while an anchor halt clears only on re-attestation. The veto card offered cancel actions that always revert outside the veto window, with no getter to show a spent one-shot. The confirmation dialog rendered a human-readable summary rather than the signable bytes, with no hex payload, chain id or value; the wallet remains the signer, so this misleads rather than misauthorises.

#### Impact

The guardian could not pause a feed from the console. The remaining rows cost gas on inert repeats and misinform the operator about what a lever will do, without changing what the chain enforces.

#### Remediation

The console no longer enumerates feeds on chain. The roster is the build-time lane map plus one `getFeed` multicall issued against a V4 write ABI; 26 feeds were verified enumerated with the expected single paused feed. The pause controls now gate on the read-back paused state.

Commits: [`48587b54`](https://github.com/btr-protocol/front/commit/48587b54b24820dde04608759c4048ad237fc82a).

#### Status

Fixed. Verified by the 2026-09-11 validation pass. The copy, veto-card and confirmation-preview rows were closed as informational on 2026-09-09 and 2026-09-10 under the low-minimum bar.

**Rows.** 6 rows.

### F-13  Coverage-sensitive LP paths settled at per-slice rates, letting an exit outrun the pool's own haircut

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/libraries/PoolLiquidityLib.sol:404`, `dex-evm/src/libraries/PoolLiquidityLib.sol:326-333`, `dex-evm/src/libraries/PoolIOLib.sol:159-190` |

**Severity rationale.** Under-covered pools are the state in which the haircut exists at all, and the escape was measured at the live coverage setting on both LP cross paths, so an ordinary LP could extract value from the remaining LPs with no privileged access.

#### Description

The pool applies a haircut when coverage is below par. The settlement arithmetic on the cross-asset LP paths took the source leg's own coverage rather than the pool-level rate, so an exit split into slices converged on a better rate than a single exit of the same size. Measured at the live setting the escape was +61.6% on both LP cross paths.

```solidity
// dex-evm/src/libraries/PoolLiquidityLib.sol:404
    IPool.Asset storage assetFrom = $.assets[ctx.fromTk];
    // withdrawValue ≤ liabilities is enforced at the quote (no clamp): the full face is always burned.
    if (ctx.fromTk == ctx.toTk) {
      assetFrom.reserves -= uint128(ctx.amt);
      assetFrom.liabilities -= uint128(ctx.withdrawValue);
    } else {
      IPool.Asset storage assetTo = $.assets[ctx.toTk];
      assetFrom.liabilities -= uint128(ctx.withdrawValue);
      if (ctx.protoFee > 0) $.protocolFees[ctx.toTk] += ctx.protoFee;
      assetTo.reserves -= uint128(ctx.amt + ctx.protoFee);
      accrueLpFee(assetTo, ctx.toTk, ctx.lpFee);
    }
```

Three further defects sat on the same ledger. The mark cap that keeps an LP conversion at or below the fair oracle rate was applied at the cross-withdraw and liability-swap entrypoints only, not inside the swap pricing core, so a same-asset withdraw followed by a swap reproduced the same end state with the cap absent. That path was reproduced 25 out of 25 runs at the live curve and widened with volatility, from +9 basis points at zero sigma to +177 basis points at 5% sigma. The cap also capped the output while keeping the pre-cap fee, overcharging at dust scale. And `donate` booked its face at par where the specification requires face scaled by the pool coverage rate, moving surplus from every other leg's LPs to the donor leg.

Structural observations were recorded rather than fixed. Reference bands are enforced per node, so on a spoke-hub-spoke cross both guarded nodes can sit at the same edge of their own band and the worst-case composed error is about twice the band. No independent reference exists for a composed cross rate, so per-node guarding is the strongest available control. The remaining rows on this surface are display and documentation gaps below the reporting bar: `previewWithdraw` ignoring the halt and liquidity floor, a natspec claim that fee accrual never reverts, an incomplete file-header event list, a dust-scale fee skip, and a transient-cache hit that skips a re-gate on a path with no reachable oracle write.

#### Impact

An LP in an under-covered pool could recover more than the haircut allowed by slicing the exit or by routing the same economic move through a swap, in both cases at the expense of the LPs who stayed. The `donate` mispricing transferred surplus between legs. The accepted reference-band bound is a known ceiling on composed cross accuracy, not a leak.

#### Exploit scenario

1. The pool sits under-covered, so exits are haircut.
2. An LP splits a withdrawal into slices rather than exiting once, and each slice settles at the source leg's coverage instead of the pool rate.
3. Alternatively the LP performs a same-asset withdraw, which is exactly coverage-preserving and reads no oracle, and then sells the withdrawn asset back through `PricingLib.swap`, which prices at the skew-anchored mid with no mark cap.
4. Either route lands the same end state as the capped cross exit at a better rate, bounded by the actor's own position and requiring an under-covered source leg.

#### Remediation

Cross and liability settlement now use the pool coverage rate and never the source leg's. The sell arm of the swap pricing core clamps execution to the mark, with the off-chain integer mirror pinned to the same behaviour. Capped outputs pro-rate their fees. `donate` books face at the pool coverage rate. Pool-level coverage replaced the per-leg haircut in the mint rate, so surplus enters the pool rate instead of being stranded. The withdraw liquidity gate was aligned with the settlement gate and the vault hook now measures the realised withdrawal delta rather than trusting the requested amount. Coverage proofs pin the residual escape below 1%.

Commits: [`b563405f`](https://github.com/btr-protocol/dex-evm/commit/b563405f08dfc6ed9d9728537211a0e985a4c55b), [`9a9deda0`](https://github.com/btr-protocol/dex-evm/commit/9a9deda055e055ac5e694face8f4e1c06ab8c440), [`150a94e8`](https://github.com/btr-protocol/dex-evm/commit/150a94e82aa6a9757cbb4798e3da95fafbded636), [`e36dbb86`](https://github.com/btr-protocol/dex-evm/commit/e36dbb86c4e0d36a0b57893a168f9875bfb9aba1), [`84fdfa9b`](https://github.com/btr-protocol/dex-evm/commit/84fdfa9b21eccce242fde560f7e81542d92e3c6d), [`68077778`](https://github.com/btr-protocol/dex-evm/commit/68077778d3ba503ae9559987d3dd36b722a9d3d4), `core@04245d1`.

#### Status

Fixed, verified 2026-09-10 and 2026-09-11. The per-node reference-band bound is accepted and recorded, with composed drift priced by a coverage wall at least as large as the larger of the reference band and the per-push deviation band, re-verified 2026-09-14. The remaining informational rows were closed on 2026-09-10 under the low-minimum bar.

**Rows.** 13 rows.

### F-14  An off-factory beacon clone was a fully attacker-governed pool running the real implementation

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/Pool.sol:152`, `dex-evm/src/PoolFactory.sol:300-307`, `dex-evm/src/Admin.sol:155` |

**Severity rationale.** The clone answered the real access-control, admin and flash singletons, inherited beacon upgrades and emitted the protocol's full event stream while remaining invisible to the official pool registry, so a third party could present a pool they governed as one of ours.

#### Description

`Pool.initialize` was unauthenticated and the factory is itself the beacon, so anyone could deploy an ERC-1967 beacon proxy against it and initialize the result. Before the authority change such a clone was inert because ownership resolved unconditionally to the access-control owner; adding a per-pool admin made it operable.

The first fix bound initialization to the beacon by comparing `msg.sender` to the beacon slot. That bind is circular: the attacker supplies their own beacon, which returns the real implementation, and the check passes.

```solidity
// dex-evm/src/Pool.sol:152
    if ($.initialized) revert Err.InvalidState();
    address beacon;
    assembly ("memory-safe") {
      beacon := sload(_BEACON_SLOT)
    }
    if (msg.sender != beacon) revert Err.NotAuth();
    $.poolAdmin = poolAdmin_;
    if (baseToken_ == address(0)) revert Err.ZeroAddr();
```

Three authority defects accompany it. Permissionless `syncOfficial` de-branded every official pool after a treasury rotation, which made the router revert with `UnknownPool` and silently emptied the operational halt roster. The soft per-token pool cap began binding official pools, so 128 squatter clones could keep protocol pools out of token enumeration and out of the asset-halt script's roster. And `SWEEP(NATIVE)` reverted on pools with no wrapped-native token configured, while on pools that had one the native and wrapped sweeps shared a single queue key.

#### Impact

A third-party-governed contract could pass as a protocol pool to any integrator that trusted the implementation and the singletons rather than the official registry. The treasury-rotation and enumeration defects degraded routing and the emergency asset-halt roster without any attacker. The sweep key defect blocked a native sweep on part of the fleet.

#### Exploit scenario

1. The attacker deploys their own beacon whose implementation getter returns the protocol's real pool implementation.
2. They deploy an ERC-1967 beacon proxy pointing at that beacon and call `initialize` from it, passing the `msg.sender == beacon` bind.
3. They are now the pool admin of a contract running the real implementation, answering the real access-control, admin and flash singletons, while absent from both the all-pools and official-pools registries.

#### Remediation

`initialize` is anchored to both the beacon and the factory recorded in access control, which closes the circular bind. A `previousTreasury()` shield at least as long as the pool governance delay plus grace keeps official branding across a treasury rotation. Asset-halt enumeration is restricted to official pools and a squatter leg is skipped with an event rather than reverting the batch. `SWEEP` is keyed on the raw token.

Commits: [`2077627d`](https://github.com/btr-protocol/dex-evm/commit/2077627dfb37e6b299d8a3fc0dd48b2a5f464fc2), [`49b2451`](https://github.com/btr-protocol/shared/commit/49b2451b6dea61e9cf0baafa13a3ef50fd2509d9), [`cb95b49`](https://github.com/btr-protocol/shared/commit/cb95b49ee3884a220ce42a56fe2976b08cee5f90), [`f296fe97`](https://github.com/btr-protocol/dex-evm/commit/f296fe97bec8675149c9a4587dcf0fc7e9843e8a), [`149fb16e`](https://github.com/btr-protocol/dex-evm/commit/149fb16e3c11e5e4b527dcc9ff2f5fccd3168b2d), [`04aa4c09`](https://github.com/btr-protocol/dex-evm/commit/04aa4c09cb7c5205e768ece98c1b8c96ee8dd27b), [`cf5bfc69`](https://github.com/btr-protocol/dex-evm/commit/cf5bfc69d273d35adf88e635426be2133e2276fd), [`9e024ca7`](https://github.com/btr-protocol/dex-evm/commit/9e024ca752d6685927481748a26b85419aeb18bc).

#### Status

Fixed. The authority anchor and the clone reopen are closed by the same commit and pinned by authority tests; the enumeration and sweep rows landed in the 2026-09-15 remediation wave.

**Rows.** 5 rows.

### F-15  Liability re-denomination applied the pool coverage rate twice and minted outside the deposit gates

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/libraries/PoolLiquidityLib.sol:546`, `sdk/src/pool/liability.ts:160` |

**Severity rationale.** The over-mint is deterministic on every call whenever pool coverage exceeds par, and the path was armed at genesis, so it dilutes existing claim holders with no attacker and no unusual state.

#### Description

`swapLiability` converts a claim on one leg into a claim on another. It scaled the incoming face by the pool coverage rate to obtain the fair input, then settled the outgoing claim at that rate a second time and credited the result as face. For a coverage rate above par the outgoing claim was over-minted by one factor of the rate, while the natspec stated the operation was coverage-neutral.

```solidity
// dex-evm/src/libraries/PoolLiquidityLib.sol:546
    // Re-denomination is a CROSS EXIT that stops short of paying out, so it settles on the same
    // rate: face in, face out, both at C. It is C-NEUTRAL by construction - no reserves move and the
    // spread makes B fall - so it can only raise the rate for everyone left.
    uint256 fairIn = (liabIn * PoolSolvency.mintRate($)) / SC.WAD;
    IPool.SwapQuote memory q = Pricing.anchorPathQuoteLp($, inTk, outTk, fairIn);
    uint256 markCap = _markCap($, inTk, outTk, fairIn, q.markPrice);
    if (q.amountOut > markCap) q.amountOut = markCap;
```

The same path minted its outgoing claim without the depositor allowlist and seal checks that `deposit` enforces, so it was a second mint entrypoint with a weaker gate. The off-chain quote mirror also credited the raw conversion rather than the face at the coverage rate, so the client and the chain disagreed on the resulting claim.

#### Impact

Every liability re-denomination at a coverage rate above par minted more claim than the ledger backed, diluting the remaining holders. The missing allowlist and seal checks let a party outside the intended depositor set obtain claims. The mirror divergence meant the displayed result did not match settlement.

#### Remediation

The outgoing liability is re-denominated to face at the pool coverage rate after the incoming side settles at that rate, with coverage proofs pinning the result. The outgoing mint clears the depositor allowlist and the seal gate. The off-chain mirror divides the conversion by the pool rate to match, pinned by unit tests.

Commits: [`97638d2e`](https://github.com/btr-protocol/dex-evm/commit/97638d2e5e59a2de3cbc5db6a2643232d91a26a9), [`037e8a0a`](https://github.com/btr-protocol/dex-evm/commit/037e8a0adb7a81a6e29cdc74503a09ef3768066e), [`095a741`](https://github.com/btr-protocol/sdk/commit/095a741c9aff268f559f4b46e0e84d1b8a77decd), [`e19bac5`](https://github.com/btr-protocol/sdk/commit/e19bac535a5d6a64a97589af50447790098c10d3).

#### Status

Closed.

**Rows.** 3 rows.

### F-16  Failed or stale chain sub-reads were rendered as permissive values instead of as unknown

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `front/src/hooks/useSafetyControl.ts:135`, `front/src/components/features/admin/useSafetyHistory.ts:249`, `front/src/components/features/oracle/oracle.tsx:244` |

**Severity rationale.** A partial multicall failure silently shrank the safety roster rather than reporting it, so the operator could act on a console that looked complete and was not.

#### Description

The safety console builds its roster from a multicall fan-out. Sub-read results were filtered for truthiness, so a failed or stale sub-read was indistinguishable from a pool that does not exist: it vanished from the roster, and downstream state fell back to permissive defaults rather than blocking the action.

```ts
// front/src/hooks/useSafetyControl.ts:135
  const { data: poolRes, loading: poolsLoading } = useReadContracts({
    contracts: poolCalls,
    chainId,
    query: { enabled: poolCalls.length > 0 },
  });
  const pools = useMemo(
    () => poolRes.map((r) => r.result as Address | undefined).filter(Boolean) as Address[],
    [poolRes],
  );
```

Two console-fidelity rows sit alongside it. The halt labels and hints were not updated after the anchor bit was split out, so the console described chain semantics that had changed. The oracle push decoder dropped any lane absent from the build-time lane map instead of surfacing it, so a newly listed lane was invisible rather than flagged.

#### Impact

The operator could see a roster smaller than the fleet with no indication that reads had failed, and act on it. Incorrect labels misdescribe what a lever does. Dropped lanes hide push activity from the transparency view.

#### Remediation

A failed or stale read now renders as unknown and disables the associated action rather than defaulting permissive. Console labels and operation keys mirror the post-split chain semantics. Unknown lanes are surfaced rather than dropped.

Commits: [`2c8892a`](https://github.com/btr-protocol/front/commit/2c8892a0d2fe6f3508718434a9267f87329b8305), [`b8b6405`](https://github.com/btr-protocol/front/commit/b8b6405034ff4a7d56d8e57e4ac5d8619e08b140), [`4a6041d`](https://github.com/btr-protocol/front/commit/4a6041d8781ab19e633ee082d95b38b085265a58), [`1b384ff`](https://github.com/btr-protocol/front/commit/1b384ff105bc81b70a3fc88c1e256c5b785e3892).

#### Status

Fixed 2026-09-16.

**Rows.** 3 rows.

### F-17  Exact-in swaps were non-monotone above the output argmax, so a larger input returned a smaller output

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/libraries/PricingLib.sol:919-937`, `dex-evm/src/libraries/PricingLib.sol:1001`, `core: src/pricing.rs` |

**Severity rationale.** Reachable by any taker on a live leg with no privilege and no setup, but the loss is taker-only, pool-favourable and requires the victim to sign the oversized order, which caps it below critical.

#### Description

The coverage wall charges a toll on the output leg that grows with the fraction of that leg's reserves the fill consumes. Above a size threshold the toll grew faster than the gross output, so the net output fell as the input rose. The exact-in path had no cap on that region and no revert above the argmax: it reverted only where output reached exactly zero, leaving a continuous band in which paying more returned less, down to one wei.

Chain measurement on the live testnet fleet showed USDC.b to USDT returning 49,266 for a $56.5k input and 45,269 for a $58.5k input, and XAUT peaking at 11.7428 for a $65k input and returning 0 at $80k.

```rust
// core: src/pricing.rs
        } else {
            // base→token (buy): size the child-token volume off the mid, then traverse.
            let est_out = amount_in.mul_div(WAD, mid).expect("estOut in range");
            let exec = self.traverse_curve(mark, disp, start, est_out, depth, false, mid);
            amount_in.mul_div(WAD, exec).expect("buy grossOut in range")
        };

        // `PricingLib._settleQuote` (PricingLib.sol:521): `quote.covToll = _covToll(cOut, …)` - selling
        // delivers the counterparty, buying delivers this leg.
        let out = self.settle_out(reserves, liabilities, counterparty, selling);
        let cov = out.toll(gross_out);
        let post_toll = gross_out.wrapping_sub(cov);
```

Two further properties of the same toll were examined in the same pass. First, the recovery ratio of the coverage toll is `rho(c) = c(-ln c - 1 + c) / (1 - c)^2`, which is independent of `kappaCovBps` and tends to 0.5 as coverage approaches the peg: the toll can never recover more than half of the loss-versus-rebalancing it prices, at any kappa. Second, the hub leg carried `kappaCovBps = 300` while the spoke ladder ran 400 and 600, so every spoke down-move drained the weaker wall first. The over-peg region is deliberately toll-free through the `min(c, 1)` clamp, and the toll, skew and haircut accrue as unclaimable reserve surplus rather than as a claimable index rise.

```solidity
// dex-evm/src/libraries/PricingLib.sol:1122-1136
  function _covToll(EndpointCache memory cOut, uint256 grossOut) internal pure returns (uint256) {
    if (cOut.liabilities == 0 || grossOut == 0) return 0;
    uint256 r0 = uint256(cOut.reserves);
    uint256 l = uint256(cOut.liabilities);
    if (grossOut >= r0) return grossOut; // fully drains the leg → wall blocks the whole fill
    uint256 c0 = (r0 * SC.WAD) / l;
    uint256 c1 = ((r0 - grossOut) * SC.WAD) / l;
    if (c0 > SC.WAD) c0 = SC.WAD;
    if (c1 > SC.WAD) c1 = SC.WAD;
    int256 dQ = _covQ(c0) - _covQ(c1);
    if (dQ <= 0) return 0; // draining toward/at peg: no charge (charge-only)
```

#### Impact

A taker who signed an order above the argmax received less than a smaller order would have returned, with the difference retained by the pool. Because the region was continuous down to one wei, an interface that sized an order from a stale or optimistic quote could route a user into it without any revert. The bounded-recovery property means the coverage toll cannot be relied on to make the pool whole against loss-versus-rebalancing at any parameterisation, and the hub-below-spoke kappa ordering concentrated the residual on hub liquidity providers.

#### Remediation

The chain now caps gross output and flags coverage overshoot, so a fill past the argmax is refused rather than filled at a worse price. The Rust mirror carries the same cap and its parity suite pins it. The SDK and the front end refuse a saturated quote instead of encoding it. Hub kappa dominance is enforced on-chain at the three configuration writers plus a deployment-script gate. Kappa sizing is now documented against residual loss rather than against recovery, the over-peg free drain is recorded as designed and net asset value neutral at mark for spreads of at least two theta, and pool-level coverage replaced the per-leg haircut so the surplus question is moot.

Commits: `core@43483cf`, [`577fd8d`](https://github.com/btr-protocol/sdk/commit/577fd8d9669cb2f61cb00d026d51f3c00ffd7d08), [`ea7a16ce`](https://github.com/btr-protocol/front/commit/ea7a16ce1766cf58a53f0269baa13d232f08af46), [`fb975b2b`](https://github.com/btr-protocol/dex-evm/commit/fb975b2bc306c0d04ce66258ef31b5f18bbc85cf).

#### Status

Fixed, with the recovery bound accepted as designed on 2026-09-11 and the informational design notes closed on 2026-09-09 and 2026-09-10.

**Rows.** 11 rows.

### F-18  Pool configuration writers lacked bounds, un-stage levers and roster invariants

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/libraries/PoolConfigLib.sol:650-673`, `dex-evm/src/libraries/PoolConfigLib.sol:93-105`, `shared/evm/src/ConstantsLib.sol:60` |

**Severity rationale.** The reported perpetual admin option would have been unconditional and would have won the correction race, but the staged-admin path turned out not to exist in code, so no live configuration surface carried it.

#### Description

The pool administration handover was reported as holding a `pendingPoolAdmin` slot with no un-stage call and no expiry, which would give a staged recipient a perpetual unconditional option and let them win the seven-day correction race, contradicting the natspec at `Admin.sol:961-966`. Re-examination showed `pendingPoolAdmin` exists only as two reserved storage words in `IPool.sol:252-259` and is not wired to any code path.

The surrounding configuration surface carried real defects. `kappaCovBps` had no upper bound in any writer while the mainnet set raised it four to eight times, putting a fat-finger inside reach. `deregisterPool` deleted `poolToTokens` without repopulating it, so `setBaseToken`'s completeness scan read an empty roster and failed open, and `isInteriorCapable` returned false for every leg, silently handing out `MAX_DISPERSION_PBPS` instead of the interior dispersion cap. `setAssetHook` never checked `hook.token() == t`, so a sentinel-constructed hook could block recall and deadlock a leg with non-zero inventory. `collapseAnchor` never wrapped `newAnchor`, so a guardian alias mistake reverted with `InvalidAnchor` and the halt never landed. On the constants side, the production LOW timelock tier was cut from one day to one hour during the delta with `ADD_ASSET` riding it, and the `MIN_ARMED_DELAY_SECS` natspec still described the old one-day floor.

```solidity
// dex-evm/src/libraries/PoolConfigLib.sol:651-663
    if (curveId == 0 || dispRefPbps == 0) revert Err.InvalidInput(); // 0 = the no-shape sentinel
    // The flag byte is an off-chain curve key, immutable across refits: a change needs a fresh id.
    // Nothing on chain reads it; the coverage wall is unconditional and carries no flag gate
    // (an unwalled asset left on a now-wall-required needle, or vice versa). Changing the wall
    uint256 existing = $.curves[curveId].header;
    int256 oldSpan;
    if (existing != 0) {
      if (uint8(existing >> 248) != flags) revert Err.InvalidInput();
      (, oldSpan) = NUQuarticLib.rangeQ($.curves[curveId], existing);
    }
```

#### Impact

Without an upper bound on `kappaCovBps`, a single mistyped write could raise the coverage wall far enough to make a leg effectively untradable. The `deregisterPool` roster gap removed the interior displacement bound from every leg and turned a safety scan into a no-op. The hook binding gap could deadlock recall on a funded leg. The timelock tier cut placed asset listing, and the oracle configuration that rides it, behind a one-hour delay on production.

#### Remediation

`poolToTokens` now survives deregistration, so both consumers keep a populated roster. `setAssetHook` requires `IPoolHooksToken(hook).token() == leg`, making a sentinel-constructed hook uninstallable. `collapseAnchor` wraps `newAnchor`. `ADD_ASSET` rides the one-day LISTING tier on production and the `Constants.sol` natspec states the TUNING tier as one hour. `kappaCovBps` is bounded at the writers. The staged-admin report is recorded as refuted: the reserved words are documented as reserved.

Commits: [`f5c281f7`](https://github.com/btr-protocol/dex-evm/commit/f5c281f7cdc161bb244d80d70e2cf3db1d64ccd7), [`e36dbb86`](https://github.com/btr-protocol/dex-evm/commit/e36dbb86c4e0d36a0b57893a168f9875bfb9aba1), [`3f5b4e42`](https://github.com/btr-protocol/dex-evm/commit/3f5b4e423ae10dae47ae1f9a77e1a921045bd28c), [`06ee592`](https://github.com/btr-protocol/shared/commit/06ee592d6ba6c959519b0a2aca42642032e2b06e), [`600c6d3`](https://github.com/btr-protocol/shared/commit/600c6d3f93840f1c83d281fd54ddbc44fc4f5367).

#### Status

Fixed. Coverage is pinned by `PoolLifecycle.t.sol:1096`, `TokenContainment.t.sol`, `PoolAnchorTree.t.sol:547` and `DeployBaseSchedule.t.sol:46-50`.

**Rows.** 8 rows.

### F-19  A single unusable leg mark froze every credit and cross-exit path pool-wide

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/src/libraries/PoolSolvency.sol:53-63` |

**Severity rationale.** One dead or out-of-band feed on any roster leg is an ordinary operational event, and it took every deposit, cross exit and hook path in the pool with it.

#### Description

`PoolSolvency.solvency` read a mark for every roster leg, including legs with zero reserves and zero liabilities, and returned `(0, false, 0, 0)` as soon as one read was unusable. A leg that had been funded and then emptied, or whose feed had simply gone dark, therefore froze `deposit`, `donate`, `swapLiability`, `hookCreditYield`, `hookWriteDown` and every cross exit across the whole pool, with no leg-removal or force-skip lever.

Separately, the sum ran every leg's mark through `markToBaseWad` under only the halt gate and the freshness gate. The reference-band depeg breaker was not applied, so a fresh but out-of-band mark moved pool coverage for every liquidity-provider entrypoint, with cross-leg extraction bounded only by `refBandBps`.

```solidity
// dex-evm/src/libraries/PoolSolvency.sol:57-66
    for (uint256 i; i < n; ++i) {
      address leg = $.legs[i];
      IPool.Asset storage a = $.assets[leg];
      // a 0/0 leg contributes exactly 0 to BOTH sums, so skip it BEFORE the oracle read. A
      // dead feed on a funded-then-emptied leg otherwise returned (0,false,..) and froze deposits,
      // cross exits and harvest pool-wide. The skip is exact: r == l == 0 adds nothing either way.
      if (a.reserves == 0 && a.liabilities == 0) continue;
      (uint256 px, bool okk, bool bandOk) = Pricing.markToBaseWad($, leg);
      uint256 d = a.decimals;
      uint256 r = _backedReserves($, leg, a.reserves);
```

#### Impact

A zero-exposure leg with a dead feed was enough to halt all liquidity operations on a pool that was otherwise fully healthy. An out-of-band but fresh mark on any leg moved the pool coverage rate that funds cross exits, so a depeg inside the reference band translated directly into value extracted from the other legs.

#### Remediation

Zero-exposure legs are skipped before the oracle read. An out-of-band leg's mark is bounded at the reference-band edge. The intermediate par-degrade was reverted to a fail-closed semantic: an unusable mark yields `ok = false`, `mintRate` and the weight and cap gates revert `FeedUnavailable`, and the exit cap pays `min(WAD, lastGoodC)` so the same-asset hatch stays open. Freezing credit paths on a funded dead leg is now the deliberate design: there is no leg-removal lever and the heal is on the feed side. The `bandOk` plumbing was removed from `PricingLib.markToBaseWad` and the natspec matches the code.

Commits: [`d9a6b556`](https://github.com/btr-protocol/dex-evm/commit/d9a6b556b0f3292e4ef4aa0e98fd36741404cc59), [`1058318d`](https://github.com/btr-protocol/dex-evm/commit/1058318d936cc8408ed41b257c17db95976c9a71), [`64d80d65`](https://github.com/btr-protocol/dex-evm/commit/64d80d65adcbf188300a745b25f69986aef63ab9), [`88e3d3f8`](https://github.com/btr-protocol/dex-evm/commit/88e3d3f851c5fbc586290e394780666f36a22eda).

#### Status

Closed on 2026-09-15. Pinned by `PoolSolvency.t.sol:615`, `PoolWriteDown.t.sol:183` and `PoolSolvencyDegraded.t.sol:96,:119,:161`.

**Rows.** 3 rows.

### F-20  Testnet-only deploy ceremony paths ran unguarded on a mainnet-class chain

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/script/PoolDeploy.s.sol:341`, `dex-evm/deployments/bnb-risk-params.json:84`, `dex-evm/script/lib/ChainParams.sol:168` |

**Severity rationale.** A single ceremony run on a production chain would have funded a public faucet with real tokens and minted a pool whose fee floor breached the H-2 design gate, and every gate that would have caught it was inert before listing.

#### Description

The pool deploy ceremony branched on roster content rather than on chain class. `_fundFaucet` ran on the same broadcast as `_createPool`, so a mainnet run reached the testnet faucet path with real tokens, and the V4 greenfield path could claim a production CREATE3 row. A swap-gate comment in the same file described a check that no longer existed.

```solidity
// dex-evm/script/PoolDeploy.s.sol:341
    Deploy.Addrs memory core = _loadCore(cfg, outPath);
    _requireSeedBudget(cfg, syms);
    vm.startBroadcast(pk);
    pool = _createPool(core, cfg, syms, true);
    _fundFaucet(cfg, TestnetFaucet(vm.parseJsonAddress(vm.readFile(outPath), ".faucet")), syms);
    vm.stopBroadcast();
```

The same ceremony carried three further classes of defect. The BNB scaffold shipped `minFee` below two theta on USDT and WBNB and a stable TTL of 7200 s against private-relay-only pushes, both of which breach the H-2 and PAR-2 gates. `ChainParams` trusted operator input: the wrong-RPC gate compared a value derived from the same source it was meant to validate, `uint16` casts truncated before the ceiling checks ran, and signer environment overrides applied on mainnet. CI checked out the `shared` dependency at its default branch rather than a pinned commit, and the Arc operator scripts proved swing caps against the target `minDispersion` with a hardcoded cap while restore and unwedge batches aborted on the first non-ready entry.

#### Impact

A production ceremony could have broadcast a pool with a public faucet holding real tokens, with fee and TTL parameters outside the shipped risk design, against a `shared` build that no commit pinned. The parameter defects are economic: a fee floor below two theta and a 7200 s stable TTL both widen the window in which a stale lane can be traded against.

#### Remediation

The ceremony now refuses testnet lanes when the manifest declares `class=mainnet`, validates operator input and runs its preflight before `startBroadcast` rather than mid-broadcast. Scaffold parameters were corrected against H-2 and PAR-2, the BNB manifest notes were reconciled with the shipped design, `ChainParams` casts and gates were made non-tautological, and CI pins the `shared` checkout.

[`633621d`](https://github.com/btr-protocol/dex-evm/commit/633621dbe1a3cfb1b5e81623791f298baa59f9c5), [`8920758`](https://github.com/btr-protocol/dex-evm/commit/892075887adb6a446681b8eff47f334c2b191198), [`3513580`](https://github.com/btr-protocol/dex-evm/commit/3513580855bc9374201856bb25a9c3582e7ee59d), [`a8897e7`](https://github.com/btr-protocol/dex-evm/commit/a8897e7c42449b8e4c140d3654375d12765f6951), [`7398ab1`](https://github.com/btr-protocol/dex-evm/commit/7398ab172f2f486aef1e92a0e3624c6bb9ddf8dd), [`8b67c89`](https://github.com/btr-protocol/dex-evm/commit/8b67c8948ec678d184b5df00f877bb767ba37775), [`bf1caca`](https://github.com/btr-protocol/dex-evm/commit/bf1caca684f53ae43de06392c805af3f740dfa45), [`279ec8e`](https://github.com/btr-protocol/dex-evm/commit/279ec8ee34ae2ec6665e95be39b4b270fdacad93), [`5539d9a`](https://github.com/btr-protocol/shared/commit/5539d9a18599285017072ecdbfdfbfcefbe10b76)

#### Status

Fixed 2026-09-16. Two informational rows are accepted rather than fixed: the published `events.json` carries V5 `FeedRegistered` and `FeedWiden*` shapes while the live Arc deployment still emits V4 shapes, which is handled as a step in the Arc upgrade runbook and not in code; and pool-level solvency stays unarmed after a beacon swap until a governance `BACKFILL_LEGS` call, with no reinitializer.

**Rows.** 8 rows.

### F-21  Optimizer settings and unpinned artifacts broke cross-repo bytecode parity and left upgrade gates without a machine check

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `dex-evm/foundry.toml:30`, `dex-evm/script/UpgradePoolImpl.s.sol:48-54`, `dex-evm/src/interfaces/IAdmin.sol:54-72` |

**Severity rationale.** The parity gate was red at tip, so the bytecode a deployment produced for `dex-evm` no longer matched the `shared` build it linked against, and every downstream storage and library pin that was supposed to catch a mismatch was tautological or unenforced.

#### Description

`dex-evm` was built at `optimizer_runs = 5000` to fit the Pool implementation under EIP-170 while `shared` stayed at 10000. That split the two repositories' bytecode and turned the CI parity gate red at tip. Admin overflowed at either setting and was built separately into an un-pinned `out-admin` directory, which the upgrade path then loaded from, so an upgrade could install a locally stale Admin implementation.

```toml
# dex-evm/foundry.toml:30
solc = "0.8.36"
optimizer = true
# 5000, not 10000: with the C124-econ-1 band check the Pool impl is 24,907 B at 10000 (331 over
# EIP-170) and 24,316 B here. The size decision is what buys the band check, at ~+387 gas on a
# depth-1 swap (~0.2%). Admin overflows either way and is built on its own at runs 200
# (`forge build src/Admin.sol --optimizer-runs 200 --out out-admin`), where it is 24,419 B.
optimizer_runs = 5000
via_ir = true
```

The checks that would have caught a layout or link mismatch did not check anything. `IAdmin.RiskFences` member layout was comment-enforced only, so a same-width reorder passed silently through `setAssetParamsBounded`. `ExternalOracleV5` storage had no machine pin, so a same-width slot swap passed the beacon upgrade gate. `PoolStorageLayout.t.sol` hashed the sentinels it had just written and compared them to a hash of those same sentinels, and asserted local-constant arithmetic, so neither test pinned the compiled artifact. The four library bytecode pins lived only inside `_assertLibPins` in a deploy script that CI never ran, and the sole test harness overrode that function with an empty body.

#### Impact

A stale or foreign library link, a reordered `RiskFences` struct, or a swapped `ExternalOracleV5` slot could all have reached a live upgrade without any gate refusing them. The parity break meant the two repositories no longer agreed on the bytecode of the shared code they both compile.

#### Remediation

`optimizer_runs` was restored to 10000 to match `shared`, with the CI parity gate green at tip; the `bandOk` plumbing was deleted so the Pool implementation fits at 24,508 B, and Admin now fits at the default profile without a side build. The `RiskFences` and `OracleData` frames are pinned by `ArtifactGuards.t.sol`, the tautological layout asserts were deleted, and the CREATE2 library pins were re-derived and are now enforced by `LibraryPins.t.sol` against the tree's own `out/`.

[`d9a6b556`](https://github.com/btr-protocol/dex-evm/commit/d9a6b556b0f3292e4ef4aa0e98fd36741404cc59), [`235d6b2d`](https://github.com/btr-protocol/dex-evm/commit/235d6b2d626a7cba945b8cb32a2ba009adbc8031), [`46bac105`](https://github.com/btr-protocol/dex-evm/commit/46bac10502a661d98a14876849fead086cdef3b7), [`d38f6935`](https://github.com/btr-protocol/dex-evm/commit/d38f69351c90bc3a058c7a20a2b3ca8bfa9c0ece), [`66207fe5`](https://github.com/btr-protocol/dex-evm/commit/66207fe5ee8f9b9eea900abb3fdb6683298481ad), [`05dc7b04`](https://github.com/btr-protocol/dex-evm/commit/05dc7b046882a458fe40feba18ad8f5f2260599e), [`88e3d3f8`](https://github.com/btr-protocol/dex-evm/commit/88e3d3f851c5fbc586290e394780666f36a22eda), [`230ebc23`](https://github.com/btr-protocol/dex-evm/commit/230ebc23b5fe26bbf6b9af2c21b147e6db35e905), [`f039e1e7`](https://github.com/btr-protocol/dex-evm/commit/f039e1e73318021111ab3b26f3444ea7eda9a164), [`c95621e5`](https://github.com/btr-protocol/dex-evm/commit/c95621e511b879b98780184a24f2a5e6510369f6), [`6c6a65a6`](https://github.com/btr-protocol/dex-evm/commit/6c6a65a611021c279047bfde3d5f110deb688e37)

#### Status

Fixed. Closed at the rev2 signoff.

**Rows.** 6 rows.

### F-22  The v2 swap send path was unshippable: the client floor check disagreed with the server formula and cross-core routes were read as an outage

| Severity | Status | Class | Component |
|---|---|---|---|
| HIGH | Fixed | Audit | `sdk/src/router/index.ts:126-136`, `front/src/lib/quoteV2.ts:45`, `sdk/src/amm/aimm.ts:694` |

**Severity rationale.** Every real spread quote failed the client-side floor assertion and every cross-core pair returned no quote at all, so the v2 send path could not complete a swap; all three failure modes were fail-closed, so no funds were at risk.

#### Description

`assertServerFloor` is the trust boundary for a server-authored slippage floor: the client recomputes the floor rather than trusting the served `min_out`. The two sides computed it differently. The server derived `min_out` from a percentage of the spread on a `1e8` scale while the SDK asserted `amount_out * (1e6 - tol_pbps) / 1e6`, so the two differed by the residue of the spread term modulo 100 and the assertion threw on every quote carrying a real spread.

```ts
// sdk/src/router/index.ts:126
export function assertServerFloor(amountOut: bigint, tolPbps: number, minOut: bigint): void {
  if (minOut > amountOut) {
    throw new Error(`server floor ${minOut} exceeds amount_out ${amountOut}`);
  }
  const expected = (amountOut * (1_000_000n - BigInt(tolPbps))) / 1_000_000n;
  const diff = expected > minOut ? expected - minOut : minOut - expected;
  if (diff > 1n) {
    throw new Error(
      `server floor ${minOut} != amount_out*(1e6-${tolPbps})/1e6 (=${expected})`,
    );
  }
}
```

A second defect fed the same assertion the wrong input: the call sites passed the f64-reconstructed plan `amountOut`, built as `toUnits(Number(formatUnits(...)))` by the front plan builder, rather than the served `amount_out`. Any non-round 18-decimal output differed from the served value by more than one wei, so the check threw and the send aborted silently. Separately, the front took the direct-pool `/v2/quote` endpoint for every pair; a cross-core pair has no single pool holding both tokens, so the endpoint answered 422 and the front rendered it as a quote outage rather than as a routing question. Finally, every quote re-uploaded the whole 37-leg fleet, 44.5 KB per POST, with two byte-identical route requests per keystroke.

#### Impact

On the v2 path, spread quotes and non-round outputs both aborted at the client floor check, and cross-core pairs showed a false outage banner instead of a route. The fleet re-upload put 44.5 KB on the wire for every quote.

#### Remediation

The floor is now one formula on both sides, derived from the returned `tol_pbps` and checked against the raw served `amount_out` rather than a reconstructed plan value. The front prices the form on `/v2` chain quotes, takes `/v2/quote` only when a single pool holds both tokens and otherwise posts `/v2/route`, flooring off `best.floors[]` per output token; a `no_route` error reads as "No route" rather than as an outage, and a null best reads as no fill rather than as a stale quote. LP rows now carry pool C.

[`d58f3414`](https://github.com/btr-protocol/front/commit/d58f34143f966e6b764a2d1f318fb721c513b4bb), [`308ed84f`](https://github.com/btr-protocol/front/commit/308ed84feef821884db00290bf72086c06acb569), [`45d9412e`](https://github.com/btr-protocol/front/commit/45d9412eb0d97f36d85c66ac9764a0cd296c6e9c), [`e28d5db6`](https://github.com/btr-protocol/front/commit/e28d5db684373b41105d4e694ea53695a1eceec9), [`6afe57a6`](https://github.com/btr-protocol/front/commit/6afe57a6bf2aae8ac034b524e5e69ff02e907833), [`cbfba89f`](https://github.com/btr-protocol/front/commit/cbfba89fed48d9c25d69c2a869be95ded009d4a4), [`20e4af4b`](https://github.com/btr-protocol/front/commit/20e4af4b3ba03ab3837a89ed6de89116f3a851bb), [`fab5907a`](https://github.com/btr-protocol/front/commit/fab5907af99865f80b05ea1be2ca817ac33bf50a), [`2ce27a94`](https://github.com/btr-protocol/front/commit/2ce27a94963a70d52aab4c8c764be1193e82ced1), [`eb5c289c`](https://github.com/btr-protocol/front/commit/eb5c289c63aff300ed198389727669f0108bf6cc), [`8c1d216`](https://github.com/btr-protocol/sdk/commit/8c1d2166123781cfc2bfc8ae8ca831ba68509560), [`19f0122`](https://github.com/btr-protocol/sdk/commit/19f01226b24982d212aaefaa1a348a98a01b165d), [`a46208e`](https://github.com/btr-protocol/sdk/commit/a46208e91deef00b743e7c8771e869a91d4fee26), [`e22c2e8`](https://github.com/btr-protocol/sdk/commit/e22c2e84769607dfe7c61f4a744badbd4bb9e2a9), [`6915e3d`](https://github.com/btr-protocol/sdk/commit/6915e3ded50b659e0dd9ce6ebc48e6e72ce48c45)

#### Status

Fixed. Deletion of the superseded v1 quote routes remains as ceremony step C-8.

**Rows.** 4 rows.

### F-23  A rebias left the next push unbanded, and the band anchor added to close that gap was itself conditional

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV4.sol:739`, `dex-evm/src/oracles/ExternalOracleV4.sol:484`, `dex-evm/src/oracles/ExternalOracleV4.sol:769-771` |

**Severity rationale.** Every rebias, exponent-bias change or guardian break-glass ceremony opened a window in which one push landed with no deviation band, and the deployment concerned had that ceremony as its only wedge-relief path.

#### Description

`_rebias` zeroes the lane. The band gate reads the lane for the previous mark, so a zeroed lane carried no previous mark and the next push was compared against nothing. The first remediation stored the pre-rebias decoded mark in `_bandAnchor1e18` and made the band fall back to it. That anchor is written only when the lane is live and in ttl, but read unconditionally, so a ceremony on a dark or out-of-ttl lane wrote no anchor and the unbanded push returned. A guardian ttl tighten reopened the same window from a different direction.

```solidity
// dex-evm/src/oracles/ExternalOracleV4.sol:739 (_rebias)
    uint256 word = uint256(priceSlot[slotId]);
    uint256 pl = (word >> shift) & LANE_MASK;
    if (pl & MANT_MSB != 0) _bandAnchor1e18[gi] = _decode(pl, int8(uint8(cfg >> CFG_BIAS_SHIFT)));
    uint256 lanes = (word & ~(LANE_MASK << shift)) & ((uint256(1) << TS_SHIFT) - 1);
    priceSlot[slotId] =
      bytes32((_dayMod(block.timestamp) << DAY_MOD_SHIFT) | (nowDs() << TS_SHIFT) | lanes);
    _zeroSigmaConfLane(slotId, uint32(gi % LANES_PER_SLOT));
```

Three further consequences followed from the same construction. The anchor was never invalidated by the push that resolved it, so a later rebias could band against a mark that was no longer current. Sigma is zeroed alongside the lane, so the post-rebias re-entry band was `maxDevBps` only, with no adaptive term. And because the clock is re-stamped to now, the elapsed-time term of the anchored band collapsed from the real gap to zero, which tightens the band precisely when the lane has been dark longest. A later fix anchored the mark to the slot clock rather than the lane's own observation second, which re-collapsed the same term when a second ceremony ran on the same slot. `registerFeed` also took no sigma or confidence seed, so a cold-start band was the bare floor.

#### Impact

An unbanded push accepts an arbitrary mark for one block on the affected lane, which is the price input every pool leg on that asset quotes from. The collapsed-time variants tighten the re-entry band instead, which turns a recovery ceremony into a fresh refusal.

#### Remediation

The anchor carries its own observation timestamp and the band's elapsed-time term is computed from it, the out-of-ttl exemption is removed, and the anchor is invalidated by the push that resolves it. V5 `registerFeed` takes sigma, confidence and mark seeds with `sigmaSeed` at or above a non-zero floor.

[`502feb1`](https://github.com/btr-protocol/dex-evm/commit/502feb1df5277235521f6af1ee3098691dd884ec), [`231ef08`](https://github.com/btr-protocol/dex-evm/commit/231ef0801076c9b6074e407017b249e904eb7ef4), [`1f38034`](https://github.com/btr-protocol/dex-evm/commit/1f38034510c68dcd827ab6ab2dbd4caad7263cb4)

#### Status

Fixed, verified in the following round. One row, the slot-clock re-stamp on a market-closed slot, is Closed with a proof and no code change. The V5 mark seed registers a live lane, and the stale-scaffold risk that creates is tracked separately.

**Rows.** 14 rows.

### F-24  Lane state was shared across a slot or optional on the wire where it had to be per-lane and mandatory

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV4.sol:514`, `dex-evm/src/oracles/ExternalOracleV4.sol:691-692`, `dex-evm/src/libraries/FeedMathLib.sol:57-61` |

**Severity rationale.** Staleness is enforced off a clock eight lanes share, so a lane could hold an old mark while its slot mates kept its ttl alive, with the staleness premium at zero throughout.

#### Description

The V4 price slot carries one timestamp for all eight lanes. A lane omitted from a routine partial blob, whether through a selective re-fetch or a sigma-only heartbeat, kept its old mark while its mates advanced the shared clock. The ttl therefore never fired, the gate passed and the staleness premium stayed at zero. A paused lane had the same shape from the other side: its entries counted as accepted, so the pause advanced the clock it was supposed to freeze.

```solidity
// dex-evm/src/oracles/ExternalOracleV4.sol:514
        if (uint16(cfg) == 0) {
          flags |= uint256(1) << (8 + lane); // unregistered: fail-soft skip
          continue;
        }
        if (cfg & SCFG_PAUSED != 0) {
          flags += uint256(1) << 64; // paused counts accepted, mark frozen
          continue;
        }
```

Three related defects sit on the same surface. The confidence section of the wire was optional, so a mark could advance while the previous confidence was preserved, and a genesis value of zero reads as maximally confident; the gate halts only above 1000, so a stale-low confidence undercharges the premium. `registerFeed` admitted a ttl above `MAX_RECON_AGE`, so an out-of-window or aliased lane passed registration. A lane at sigma zero could stall its own heal, because a refused push skips the sigma slot while sigma-only blobs still persist sigma. Separately, six on-chain thresholds are policy expressed as constants rather than per-class or adaptive values.

#### Impact

A stale mark that the ttl never rejects is quoted at full confidence with no staleness premium, which is the exact input the adaptive fee is meant to charge for. The registration and confidence gaps widen the same window.

#### Remediation

V5 gives every lane its own clock, pinned by a clock-isolation test, and mandates confidence and price entries in lockstep. `haltFeed` is fail-closed on release. The named heal stall is fixed. The oracle half of the threshold row is closed by per-feed `sigmaFloor` and `maxDevBps` in V5.

[`df7a2c3`](https://github.com/btr-protocol/dex-evm/commit/df7a2c3583346b639ab84afb3e5b05719dbfb7be), [`1f38034`](https://github.com/btr-protocol/dex-evm/commit/1f38034510c68dcd827ab6ab2dbd4caad7263cb4)

#### Status

Fixed in V5. The deployed V4 retains the per-lane clock residual until the repoint. The threshold row is Accepted as residual informational: the base depeg halt is an owner risk decision, the staleness z-score is physics, and the remaining global confidence halt constant is accepted. The peg-leg confidence row is Closed and accepted with zero internal legs listed, to be reopened on the first internal listing. Verified 2026-09-14.

**Rows.** 7 rows.

### F-25  A solvency pin asserted on the wrong path, and two test files were order-dependent or formatter-dirty

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/test/unit/PoolSolvency.t.sol:420`, `dex-evm/test/unit/OracleDeployScripts.t.sol`, `dex-evm/test/unit/ExternalOracleSigned.t.sol:190-468` |

**Severity rationale.** The design's headline solvency claim rested on an assertion that could neither confirm nor refute it, so a real regression in that property would have passed the suite.

#### Description

The pin for the third-party round-trip property asserted on `previewWithdraw`, the same-asset path, whose rate is `min(c_leg, C)`. At the probe case in question the hub sits at `c = 0.999973` with `C > 1`, so the minimum picks `c_leg`, the assertion returns identical wei whatever pool-level `C` does, and the test is blind to the property it was written for. The claim holds in claim value, not in in-kind delivery.

```solidity
// dex-evm/test/unit/PoolSolvency.t.sol:344
  /// RESTATED PIN. The design's headline - "pool-level C closes econ-probe case 3" - is FALSE
  /// as the original was written: `testFuzz_third_party_after_swap_roundtrip` asserted on
  /// `previewWithdraw`, the SAME-ASSET path, whose rate is `min(c_leg, C)`. Case 3's hub sits at
  /// c = 0.999973 with C > 1, so the minimum picks `c_leg`, the assertion returns the identical wei
  /// whatever C does, and the pin can neither confirm nor refute pooling.
  function testFuzz_third_party_claim_value_survives_a_round_trip(
```

Two hygiene defects sit beside it. The oracle deploy-script test was order-dependent, failing roughly two runs in five of the whole suite with a different message each time while passing in isolation. And a signed-oracle test file was formatter-dirty over roughly 280 lines, so a blanket format sweep would rewrite unrelated code, the known formatter-sweep hazard.

#### Impact

A pin that cannot fail does not protect the property it names. A flaky test erodes the signal from the suite, and a formatter-dirty file turns any routine format run into a large unrelated diff.

#### Remediation

The pin is restated against claim value, the sum of face times `C`, which is what a cross exit actually delivers, and asserts that a stranger's swap round trip cannot lower a third party's claim value on any leg pair at any coverage. The deploy-script test is made order-independent, and the signed-oracle test file is formatted after a de-brace scan that found no hazard sites.

[`67c62cb`](https://github.com/btr-protocol/dex-evm/commit/67c62cb615512b3585be575bfb5c0f0f8a7f0ac6)

#### Status

Fixed.

**Rows.** 3 rows.

### F-26  The LP exit could settle below the minimum it displayed, and the pair-persist write was unguarded

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Closed | Audit | `front/src/hooks/usePoolData.ts:169-174`, `front/src/components/features/swap/LpTab.tsx:335-360`, `front/src/pages/swap/pairState.ts:71` |

**Severity rationale.** The LP tab printed a minimum received under a tooltip promising the batch would revert below it, while the same-asset withdraw path could be sent with a floor lower than that figure.

#### Description

The same-asset LP withdraw originally sent `minAmountOut = 0n` while the recap rendered a minimum received derived from the quote. The chain enforces nothing at zero. The first fix threaded a required floor through the hook; the floor helper then took the lower of the promised figure and a slipped fresh preview, which reintroduces a send below the number on screen whenever the fresh preview is lower.

```ts
// front/src/hooks/usePoolData.ts:169 (lpExitFloor, before the fix)
export const lpExitFloor = (promised: bigint, previewOut: bigint, slipFrac: number): bigint => {
  if (previewOut <= 0n) return promised;
  if (previewOut < promised) throw new Error(LP_EXIT_STALE_ERROR);
  const fresh = applySlip(previewOut, slipFrac);
  return fresh < promised ? fresh : promised;
};
```

On the same surface, stale poll data was rendered as live, and the swap-pair persist effect called `localStorage.setItem` without a guard, which throws a `SecurityError` on browsers with storage blocked and has no error boundary above it.

#### Impact

A user could sign an exit that settles below the minimum the interface guaranteed. The storage write is an uncaught exception that takes down the swap surface for any viewer with site data blocked.

#### Remediation

`lpExitFloor` throws `LP_EXIT_STALE_ERROR` when the chain pays below the promise and otherwise returns the promised figure; the minimum is deleted and the behaviour is documented in the helper's own natspec. Staleness is threaded through the liability and LP rows so stale values are shown as stale, and the pair persist goes through the guarded `storageSet`, with a test pinning the blocked-storage log.

[`45d9412`](https://github.com/btr-protocol/front/commit/45d9412eb0d97f36d85c66ac9764a0cd296c6e9c), [`93dc791`](https://github.com/btr-protocol/front/commit/93dc7918358c61c6b8d65250d336096495b01038), [`80bc4fe`](https://github.com/btr-protocol/front/commit/80bc4fedc4666a4e261d7597c995b4f4c738edd6), [`a371e56`](https://github.com/btr-protocol/front/commit/a371e56797a542e7c1f0023326834263867ea53c), [`e92b52e`](https://github.com/btr-protocol/front/commit/e92b52e55df1ee72ac6f34381048302cdf3b78fb)

#### Status

Closed. The LP floor change is signed off on the second revision, 2026-09-14; the storage guard landed 2026-09-15.

**Rows.** 2 rows.

### F-27  The factory upgrade lane had no storage-version gate and no execute-time revalidation

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `dex-evm/src/PoolFactory.sol:396-409`, `dex-evm/src/PoolFactory.sol:432`, `dex-evm/src/PoolFactory.sol:175` |

**Severity rationale.** `PoolFactory` is not upgradeable, so landing the authority change without the storage-version gate would have spent the single irreversible redeploy window; the residual paths need a compromised owner and are covered by notice, guardian cancel and grace expiry.

#### Description

No `STORAGE_VERSION` existed anywhere in the sources or tests, and neither `_validateImplementation` nor `executeReferenceUpgrade` carried a version check. The execute path also skipped request-time revalidation: it checked timing and pending state, then swapped the implementation for the entire live fleet without re-pinning the candidate against live wiring.

```solidity
// dex-evm/src/PoolFactory.sol:327
  function executeReferenceUpgrade() external onlyAdmin {
    if (block.timestamp < upgradeTimelock) revert Err.NotReady();
    // A matured pending upgrade expires SC.GRACE_PERIOD_SECS after its eta so a forgotten,
    // stale-vetted impl cannot be executed months later; must be re-requested past the window.
    if (block.timestamp > upgradeTimelock + SC.GRACE_PERIOD_SECS) revert Err.Expired();
    if (pendingReferencePool == address(0)) revert Err.NoPending();
    address oldImpl = implementation;
    address newImpl = pendingReferencePool;
    delete pendingReferencePool;
    delete upgradeTimelock;
    // Atomic fleet upgrade: every live beacon proxy reads this slot, so they all move together.
    implementation = newImpl;
    emit ReferencePoolUpgraded(oldImpl, newImpl);
  }
```

The registry lifecycle on the same contract had three further defects. `deregisterPool` deleted `isPool`, which is written only inside `_registerPool` and has no re-register path, so de-listing also and permanently revoked the pool's factory write credential: `registerTokens` and `setPoolBaseToken` both reverted, and `PoolConfigLib.initAsset` calls `registerTokens` unconditionally. A guardian could not evict a pool at all, only the owner could. A related fix for a pending-reference guard touched only one of the two cited sites. Registry fill by cheap proxies was capped at 128 entries, and storage-layout safety was build-time only.

#### Impact

Without the version gate an implementation with an incompatible storage layout could be accepted into a fleet-atomic beacon swap. De-listing a pool silently removed its ability to list any further asset or migrate its base token, with no lift path.

#### Remediation

`Pool` exposes `storageVersion()`, `PoolFactory` pins a `STORAGE_VERSION` and applies a forward-only check at both request and execute, and execute re-pins the candidate against live wiring and the recorded layout. An `isClone` flag gates `registerTokens` and `setPoolBaseToken` and survives `deregisterPool`, so de-listing is no longer a write revocation. A guardian can call `requestOfficial(pool, false)` while `deregisterPool` stays owner-only. The forward-version gate plus the layout pins are the accepted control for the build-time-only layout check.

Commits: [`f5c281f7`](https://github.com/btr-protocol/dex-evm/commit/f5c281f7cdc161bb244d80d70e2cf3db1d64ccd7).

#### Status

Fixed. Verified 2026-09-11; the advisory and informational rows were closed on the 2026-09-14 review.

**Rows.** 8 rows.

### F-28  The published ABI surface was ambiguous and its generator gate was red

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `dex-evm/abi/gen.py:74-90`, `dex-evm/src/interfaces/IPool.sol:464`, `dex-evm/abi/events.json` |

**Severity rationale.** Integrators decode events off the published artifacts, so a name carrying two signatures produced silent mis-decodes; the checker failure meant the artifact could not be regenerated to correct it.

#### Description

`abi/gen.py --check` failed on event-name collisions between generations: the generator collects every event signature per name and exits non-zero when any name carries more than one, and three names did. With the check red, `events.json` could not be regenerated, so it kept publishing the first generation's `topic0` for both `FeedWiden` events and lacked `SolvencyUpdated` and `UntrackedSynced` entirely. `LegsBackfilled` was declared with two signatures across the interfaces.

```solidity
// dex-evm/src/interfaces/IAdmin.sol:206
  // SWEEP and BACKFILL_LEGS executes emit nothing here: the pool logs `IPool.Swept` and
  // `IPool.LegsBackfilled` itself. A second declaration of either name on this interface put two
  // signatures (or two indexed layouts) behind one event name, which no off-chain decoder resolves.
```

Five further rows recorded interface-to-implementation gaps that affected documentation only, since all known consumers use the full artifacts: `IPoolFactory` declared none of the creation, upgrade or discovery functions, `IAdmin` declared the batch risk events and enum without the functions and `setRiskFences` without a reader, `IOracle` declared revert types the first generation never throws, and a natspec block grouped an owner-direct function under the admin singleton.

#### Impact

An off-chain decoder resolving an ambiguous event name picked one of two layouts, so a consumer of the published artifacts could mis-decode the wedge-release events or miss two events completely.

#### Remediation

`IPool` now declares one `LegsBackfilled`, `events.json` carries `SolvencyUpdated` and `UntrackedSynced`, the later-generation events are deferred through an explicit pending list in the generator, and `gen.py --check` runs as a CI gate.

#### Status

Fixed. The interface-gap rows were closed as informational on 2026-09-10 under the minimum-severity bar.

**Rows.** 8 rows.

### F-29  The documentation described retired levers and omitted shipped bounds and gates

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `dex-evm/src/Admin.sol:549`, `shared/evm/src/ConstantsLib.sol:60`, `dex-evm/src/libraries/PoolLiquidityLib.sol:266-277` |

**Severity rationale.** Operators act on the runbook, and the drifted pages told them a defensive tighten was instant when the code queues it, so an incident response would have been mistimed; no funds are at risk directly.

#### Description

The parameter documentation still presented a `vegaBps` raise as an instant defensive tighten. The bounded path is deliberately not `setAssetParams` behind a flag and the absolute path queues, so the documented behaviour did not match either lane.

```solidity
// dex-evm/src/Admin.sol:527
  /// @dev Deliberately NOT `setAssetParams` behind a lane flag: fenced, clamped, never queues.
  function setAssetParamsBounded(
    address pool,
    address token,
    uint128 minLiquidity,
    uint16 minFeePbps,
    uint16 vegaBps
  ) external {
    _onlySteward(pool, false);
```

Four further drifts sat on the same pages. The developer guide advertised SDK quote APIs that had been deleted or now throw. Three documentation sites gave the production `LOW` tier as one day where the shared constants set one hour. The upper bound `kappaCovBps` gained was recorded nowhere. And the trade feature bits were described as gating trading through a leg when they gate their own entrypoints: with the swap bits cleared, deposit plus cooldown plus a cross `withdrawTo` reproduces a swap fee for fee at parity. That last one is not a value leak, because the mark cap on the liquidity path keeps it no better than a swap and collapses it off parity, and the halt-only gating is deliberate so a trading pause cannot trap an LP's only exit; the defect is that an operator clearing the swap bit for a wind-down still sees fee-identical flow.

#### Impact

An operator following the documentation would have queued a tighten expecting it to be instant, provisioned the wrong governance delay, or cleared a feature bit believing it stopped flow through the leg.

#### Remediation

One sweep per surface shipped alongside the contract changes: three tiers and the production delay table, the current push path, the `[50, BPS]` kappa bound, the deposit-gated bit and the deposit cap code, the face-at-C table, the exotic-token bit row with a note that feature bits gate entrypoints, the unpause runbook, and an SDK page stating that chained parts are floored by the server or not at all.

Commits: [`c7df8ad`](https://github.com/btr-protocol/content/commit/c7df8ad2ab052fb228cc4645465a3c79345b3da3), [`94189ab`](https://github.com/btr-protocol/content/commit/94189abbeeca0ab3c38c6daa1e373b4dc646491a), [`ded071f`](https://github.com/btr-protocol/content/commit/ded071f2a406317fe55c86eb8c844fa86dab93db), [`cd38999`](https://github.com/btr-protocol/content/commit/cd38999158aa8c2e1690ac8fa3bf4c39828d84de), [`387d7db`](https://github.com/btr-protocol/content/commit/387d7dbf80e2441209890e9f9d7496f3e4fd1df1), [`843ed3b`](https://github.com/btr-protocol/content/commit/843ed3bc479814d014b46affe2080cb2340ef36e), [`19d5de5`](https://github.com/btr-protocol/content/commit/19d5de5f0c9f194d09669a91d6274199959ab955), [`db2a395`](https://github.com/btr-protocol/content/commit/db2a395fdd8548e57062a7dd6600879d04099826), [`6a12938`](https://github.com/btr-protocol/content/commit/6a1293806c5fb34d24ea788bcc8c6c39a0adb8ba), [`5840abb`](https://github.com/btr-protocol/content/commit/5840abb4658dc982f9c77d08940b4af7ad65b5bc).

#### Status

Closed. Documentation sweeps landed 2026-09-15; the residual wording was verified on the 2026-09-14 review.

**Rows.** 5 rows.

### F-30  The client authored its own swap floors and mis-allocated the server floor across split and chained parts

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `sdk/src/router/index.ts:325`, `sdk/src/router/index.ts:84`, `dex-evm/src/libraries/PoolIOLib.sol:93-101` |

**Severity rationale.** A wrong floor either reverts the batch after the first hop has mined or leaves roughly twice the intended tolerance extractable on the legacy two-hop path where no router contract exists.

#### Description

`planToLegs` wrote the end-to-end server floor onto each split part and onto hop 2 of a chained part. Hop 2 is funded by hop 1's floor, not hop 1's quote, so flooring it on the unscaled end-to-end quote left zero margin and ordinary noise reverted the batch after hop 1 had already mined.

```ts
// sdk/src/router/index.ts:324
      const server = opts.serverFloors?.[t2out.address.toLowerCase()];
      // The server's own tolerance scales the intermediate hop when it is present; the legacy
      // caller-supplied fraction only stands in for a plan with no server floor.
      const leg1MinOut = server
        ? applyTolPbps(leg1Quoted, server.tolPbps)
        : applySlip(leg1Quoted, slip);
      const leg2Quoted = toUnits(part.quote.amountOut, t2out.decimals);
```

Alongside it, the SDK kept authoring floors of its own through `refloorLeg`, `refloorRouterPlan` and a `DEFAULT_SLIP` constant, and a fork had lost the deletions and the split tests from an earlier integration change. The legacy two-hop path floored hop 2 at the squared tolerance while the interface promised the single tolerance. On the contract side, the fee-on-transfer output leg measured the pull but pushed face, so `minAmountOut` was checked against face while the recipient received less.

#### Impact

Users on the legacy two-hop path faced roughly double the tolerance they set, and split or chained plans reverted with a threshold violation after paying for the first hop.

#### Remediation

The SDK takes server floors only: `refloorLeg`, `refloorRouterPlan` and `lpRoutes` `DEFAULT_SLIP` are gone, `slippageFrac` is required, and a chained part without a server floor is refused. `planToLegs` builds a per-part map keyed by the floor itself, allocates each end-to-end floor pro rata to the quoted output across the parts landing that token with the residual on the largest, and returns null on a zero quoted total. The chained hop 2 takes its own slice. The pool re-checks the liquidity floor after an exotic-leg push, the front end prices the form on chain quotes and hands the chain floor to the approval preview, and the integration documentation states that `minAmountOut` is checked on face.

Commits: [`aa10f3b3`](https://github.com/btr-protocol/dex-evm/commit/aa10f3b3eca8202476ece9216d91bfb2c17a388c), [`e284f43`](https://github.com/btr-protocol/sdk/commit/e284f43e2e866ae93e2b2d258ba10ceeb995676f), [`d902a2d`](https://github.com/btr-protocol/sdk/commit/d902a2d0e5cb25f4d9b432999f6f9f02c55de02e), [`ccc60d0`](https://github.com/btr-protocol/sdk/commit/ccc60d00e1255aae624bf1011504eb78e77cc529), [`1fc7854`](https://github.com/btr-protocol/sdk/commit/1fc785439a0cc3ac0407a59151ce50b361acf2d3), [`d89c83a`](https://github.com/btr-protocol/sdk/commit/d89c83aba661af4731842f96e9757ba43f623c73), [`d58f3414`](https://github.com/btr-protocol/front/commit/d58f34143f966e6b764a2d1f318fb721c513b4bb), [`d183c660`](https://github.com/btr-protocol/front/commit/d183c6602b5909f086e43eaad3bfcf0a337001be), [`8f260a6b`](https://github.com/btr-protocol/front/commit/8f260a6b9a6ba3daa09c98c12dd00c107e36739c), [`a1eec1ec`](https://github.com/btr-protocol/front/commit/a1eec1ecd9c3f611de18e846227a91895dcf156e), [`f9f29d4`](https://github.com/btr-protocol/content/commit/f9f29d4958717df4db2bd1e7801096e09b925423), [`843ed3b`](https://github.com/btr-protocol/content/commit/843ed3bc479814d014b46affe2080cb2340ef36e).

#### Status

Closed. Landed 2026-09-15 and verified on the 2026-09-14 review.

**Rows.** 4 rows.

### F-31  A server-authored output floor was accepted at any tolerance and was never bounded by the user's slippage

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `sdk/src/router/index.ts:104`, `sdk/src/utils/format.ts:88`, `sdk/src/router/index.ts:210` |

**Severity rationale.** The quote service was the sole author of the floor and the client applied it unchecked up to 999000 pbps, so a compromised or misconfigured quote service could set an effectively zero floor on every swap the SDK builds.

#### Description

The SDK treated the quote service as the sole floor author. A returned `tol_pbps` was applied as given, with no cross-check against the quoted output and no ceiling from the slippage the user had chosen, so the user's setting never bounded the floor actually encoded into the transaction.

```ts
// sdk/src/router/index.ts:104
export function assertServerFloor(amountOut: bigint, tolPbps: number, minOut: bigint): void {
  if (minOut > amountOut) {
    throw new Error(`server floor ${minOut} exceeds amount_out ${amountOut}`);
  }
  const expected = applyTolPbps(amountOut, tolPbps);
  const diff = expected > minOut ? expected - minOut : minOut - expected;
  if (diff > 1n) {
    throw new Error(`server floor ${minOut} != amount_out*(1e6-${tolPbps})/1e6 (=${expected})`);
  }
}
```

Two helper defects were found on the same review. `formatUnits` returned a wrong value for `decimals = 0` and garbled negative bigints rather than throwing, and `approve()` had no zero-first reset for tokens that require one.

#### Impact

A user's slippage setting did not bound the floor written into their swap, so the transaction could execute far below the price the interface showed.

#### Remediation

A server floor is now bounded by the user's slippage and cross-checked against the quoted output using the same formula the service uses, throwing rather than quietly lowering the floor. The formatting helpers throw on out-of-domain input.

Commits: [`58eb51c`](https://github.com/btr-protocol/sdk/commit/58eb51cdeaf916910254597d6d1fd523db1241ca), [`8633f8e`](https://github.com/btr-protocol/sdk/commit/8633f8ed2f42ecb3cd1045fac4109834ae461a7d), [`f4ef785`](https://github.com/btr-protocol/sdk/commit/f4ef7852d90260eef96f8e5ec189ac7fb272f2c8), [`58d301f`](https://github.com/btr-protocol/sdk/commit/58d301ff2c3ac0ebe54fa5cf2543f4e33e165551).

#### Status

Fixed 2026-09-16. The approval reset row is Accepted: no listed token requires the zero-first pattern at the shipping configuration.

**Rows.** 3 rows.

### F-32  A queued absolute risk update silently overwrote a defensive tighten that landed during its delay

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `dex-evm/src/Admin.sol:932-937`, `dex-evm/src/libraries/PoolConfigLib.sol:617-661` |

**Severity rationale.** The tighten is the incident-response lever, so its silent reversal at the end of a tuning delay reintroduces the exact exposure the operator had just closed.

#### Description

A queued `UPDATE_RISK` op carried an absolute `RiskConfig` with no request-time snapshot and no per-field compare-and-swap. The execute decoded the stored payload and wrote it wholesale, so an instant `setRiskConfigTighten` landing during the tuning delay was overwritten without a revert: the cap re-raised, the gate re-cleared, kappa re-lowered.

```solidity
// dex-evm/src/Admin.sol:932
  function executeUpdateRiskConfig(address pool, address token) external {
    _onlyPoolAdmin(pool);
    IPool.RiskConfig memory cfg =
      abi.decode(_consume(_keyToken(pool, OP_UPDATE_RISK, token)), (IPool.RiskConfig));
    IPool(pool).adminSetRiskConfig(token, cfg, false);
    emit RiskConfigUpdated(pool, token, cfg.flags);
  }
```

The first fix refused an instant tighten while a risk op was live, which made the de-risking lane depend on a privileged cancel: a tighten queued behind a cancelable op could be delayed for the delay plus the grace window by whoever held the cancel authority.

#### Impact

A defensive parameter tighten applied during an incident could be silently undone when the pending tuning op executed, or blocked for the length of the delay plus grace by the cancel authority.

#### Remediation

`setRiskConfigTighten` is refused while an `UPDATE_RISK` op is live, a kappa raise voids the stale queued key so the operator re-queues against current state, and the tighten caller can cancel the timelock itself rather than waiting on another authority. `adminSetDeadSeedPow10` stays instant by governance decision.

Commits: [`917040a4`](https://github.com/btr-protocol/dex-evm/commit/917040a416e5e6290a1c42b8602edf1b4b2d304a), [`2c11264f`](https://github.com/btr-protocol/dex-evm/commit/2c11264f1f73821091da35a695ef2f4c4276c8e6).

#### Status

Closed.

**Rows.** 2 rows.

### F-33  A queued asset-parameter operation carried an absolute payload and no version tag, so execution undid an instant defensive tighten

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/Admin.sol:396-413`, `dex-evm/src/Admin.sol:423-435`, `dex-evm/src/Admin.sol:78` |

**Severity rationale.** The queue delay is the exposure window: any tighten applied inside it was silently reverted at execute time, and the operation that did so looked routine.

#### Description

`UPDATE_ASSET_PARAMS` queued an absolute payload. Nothing in the queued blob recorded the values it was authored against, so executing a matured operation wrote its payload over whatever the leg held at that moment, including a defensive tighten landed through the instant steward lane during the delay. The first remediation refused at execute time on leg state rather than on what the payload actually moves, which merely changed the failure mode: an operation that did not conflict still reverted, and the refusal did not distinguish who had tightened, handing the risk steward a repeatable veto over the owner's lane. The execute-time guard also covered two of the three fenced fields, leaving a steward `haircutSuppressor` write unguarded.

The blob shape then changed without a version tag and without an on-chain reader, so an operation queued under one `Admin` and executed under another decoded into the wrong fields.

```solidity
// dex-evm/src/Admin.sol:528
    if (blob.length == ASSET_PARAMS_BLOB_LEGACY_BYTES) {
      p = abi.decode(blob, (AssetParamsPayload));
      snap.minLiquidity = cur.minLiquidity;
      snap.minFeePbps = cur.minFeePbps;
      snap.vegaBps = cur.vegaBps;
      snap.haircutSuppressorBps = cur.haircutSuppressorBps;
      return (p, snap);
    }
    uint8 ver;
    (ver, p, snap.minLiquidity, snap.minFeePbps, snap.vegaBps, snap.haircutSuppressorBps) =
      abi.decode(blob, (uint8, AssetParamsPayload, uint128, uint16, uint16, uint16));
    if (ver != ASSET_PARAMS_BLOB_V1) revert Err.InvalidInput();
```

The legacy arm was first keyed to a 192-byte shape that no deployed `Admin` ever queued, so the only shape that had actually been written to the queue had no arm at all. The compatibility arm was then one-directional: a newer blob executed under an older `Admin` decoded the version byte into `minLiquidity`. Two further rows sit on the same queue: an expired operation blocked a re-queue because it required a cancel plus a re-request per key with no on-chain read of pending state, and the instant lane wrote parameters directly with no compare-and-set, so a steward write between an owner's request and its execute made the owner's operation revert.

#### Impact

A risk tighten applied during a queued window was undone by an operation that had been authored before the tighten existed. The decode mismatch could write parameters nobody authored, or leave a key occupied after a reverting consume, which is the wedge the arm existed to prevent.

#### Remediation

The queued payload carries a per-field snapshot and execution compares each field against the live value, so an operation that does not conflict executes and one that does reverts naming the field. The guard covers all fenced fields. The blob is version-tagged with a length-keyed legacy arm matching the shape that was actually queued, and the rollback hazard is documented in the contract: pending `UPDATE_ASSET_PARAMS` keys must be cancelled before rolling `Admin` back. The steward may not write a field a live owner operation holds, and the instant lane is per-field compare-and-set at execute.

Commits: [`06bdd179`](https://github.com/btr-protocol/dex-evm/commit/06bdd179148c1064d5b26dc22707adb7a432f6fb), [`30b8c235`](https://github.com/btr-protocol/dex-evm/commit/30b8c235e82e0655b9e6082d16032128ec75dd0c).

#### Status

Fixed. Successive remediation rounds each re-verified in the following round; the final blob-compatibility arm was verified in round five against the round-one fix branch. One residual is recorded and accepted: the held-field guard also blocks the steward's tighten on that field for the delay plus the grace period.

**Rows.** 12 rows.

### F-34  Instant risk-write lanes had no cumulative limit, no dispersion re-check, and resolved the native sentinel under a second key

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed; deployment pending | Audit | `dex-evm/src/Admin.sol:394`, `dex-evm/src/Admin.sol:571`, `dex-evm/src/libraries/PoolIOLib.sol:43` |

**Severity rationale.** These are untimelocked writes on live legs; the worst realized case is a leg whose routes all revert, reachable in one transaction by a single key.

#### Description

The steward lane had no on-chain rate limit. The hard fences were the entire 24 hour envelope, and the envelope could be reached in one call and then reached again the next day, so a lane intended for small defensive moves carried the full fence magnitude at every step.

`Admin` treated any `vegaBps` raise as an instant defensive tighten with no `dispersionCap` re-check, so one untimelocked write could push a leg's live dispersion past the interior swing cap, after which every route through that leg reverts. `setFlowCooldown` was untimelocked in both directions and absent from the tier table, the only risk write in that position; the locks read the cooldown live, so setting it to zero retroactively unlocked.

```solidity
// dex-evm/src/Admin.sol:394
  function setFlowCooldown(address pool, uint16 cooldownSecs) external {
    _onlyAdmin();
    IPool(pool).adminSetFlowCooldown(cooldownSecs);
    emit FlowCooldownUpdated(pool, cooldownSecs);
  }
```

Separately, the native sentinel is a keying pattern rather than a single bug. Every token-keyed store outside `PoolIOLib.wrap` keyed on the raw argument while the pool resolves the sentinel to the wrapped native token for its own storage, so the same leg had two names: risk fences armed under one name read as zero under the other, `_keyToken` admitted two pending operations for one leg, `PoolFactory._addTokens` registered the raw token so enumeration by one name missed pools, and `Donated` emitted the raw token so the indexer orphaned the row. Finally, one listed leg held `kappa = 0` with `haircutSuppressor = 10000`, outside the coverage regime every other leg sits in.

#### Impact

A single instant write could make a leg unroutable. The sentinel aliasing meant a fence could be bypassed through the second name on the owner lane, and off-chain enumeration and indexing disagreed about which pools hold a given token.

#### Remediation

An on-chain cumulative 24 hour steward window bounds the lane, and `raiseKappa` was added as a raise-only steward lever so the parameter governing undercoverage persistence is reachable defensively without leaving the fast lane's bounds. A `vegaBps` raise re-checks the dispersion cap. `setFlowCooldown` raises instantly and lowers through the TUNING tier with a one second floor. The sentinel and the wrapped token resolve through one key at all five sites: fence and queue keys resolve through `Admin._rt`, `PoolFactory._addTokens` resolves through `IPool.resolve`, and `Donated` emits the wrapped token. The orphan leg was corrected on chain on 2026-09-04 to a suppressor of zero and `kappa` of 600, and stays halted.

Commits: [`06bdd179`](https://github.com/btr-protocol/dex-evm/commit/06bdd179148c1064d5b26dc22707adb7a432f6fb), [`30b8c235`](https://github.com/btr-protocol/dex-evm/commit/30b8c235e82e0655b9e6082d16032128ec75dd0c), [`f5c281f7`](https://github.com/btr-protocol/dex-evm/commit/f5c281f7cdc161bb244d80d70e2cf3db1d64ccd7).

#### Status

Fixed; deployment pending. Sentinel resolution verified 2026-09-11. One row in this group was refuted rather than fixed: the reported release-clock alias existed only on a parked per-pool-authority branch and never on the audited head, where halts are refcounted mask bits. A build-blocking row, where the contract tree depended on an unpushed shared commit, was closed once the shared repository was in sync with its main branch at [`eba0496`](https://github.com/btr-protocol/shared/commit/eba0496693bfc31dd3b2a1d8a3f21d5e872a8a1f). Two residual notes on the steward window did not survive refutation and are recorded as informational: a raise does not ratchet the window anchor, so a mid-window defensive raise is one-call reversible.

**Rows.** 8 rows.

### F-35  Oracle push semantics admitted an aliased replay and let a quarantined lane heal past its own band

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV5.sol:363-372`, `dex-evm/src/oracles/ExternalOracleV4.sol:243-258`, `sdk/src/oracle/wire.ts:28` |

**Severity rationale.** Exploiting the replay needs a stalled slot plus a drift inside a specific window and costs only gas, and the heal bound governs how far a dark lane can jump when it comes back.

#### Description

The signed-blob push carried no expiry and no nonce, and reconstruction aliased an old timestamp to the present across a 24 hour header and a 96 hour stored cycle. A slot stalled beyond six hours skipped the monotonic check with a maximum delta, the band admitted up to ten times `maxDev`, and the clock was rewritten as fresh, so a permissionless replay could rejuvenate a stalled slot. A future-dated observation inside the accepted bound made reads fail closed as stale until the chain caught up.

The heal bound was the second half. `attestReentry` used an allowance whose sigma term grew with the gap rather than being capped at the intended multiple of `maxDev`, so a long-gap, high-sigma lane could heal past the band, and the quarantine cap was self-healable by the same quorum.

```solidity
// dex-evm/src/oracles/ExternalOracleV5.sol:362
  function _bandPass(OracleStorageV5.OracleData storage $, BandCtx memory c)
    private
    returns (bool)
  {
    uint256 obs = (c.pl >> OBS_SHIFT) & OBS_MASK;
    uint256 dt = c.srcSecs > obs ? c.srcSecs - obs : 0;
    uint256 rw = uint256($.riskSlot[c.slotId]);
    uint256 sigmaStored = (rw >> (c.lane * RISK_STRIDE)) & RISK_SIGMA_MASK;
    uint256 sigmaFloor = uint32(c.scfg >> SCFG_SIGMA_FLOOR_SHIFT);
    uint256 sigma = sigmaStored > sigmaFloor ? sigmaStored : sigmaFloor;
```

V5 had also dropped V4's realized-move sigma floor: `_bandPass` read only the producer-stored sigma against the static per-lane floor, so a sub-`maxDev` realized move never raised sigma and the band was producer-determined. The first fix for that was inert for a price-only submission, because the move array was allocated only when sigma entries were present. The reentry nonce had no getter and incremented under `unchecked`, so a consumer could not sequence attestations, and after the heal fix the quarantine bit was write-only, making the natspec claim that clearing it re-arms the lane false. On the read side the published SDK codec was pinned to wire version five while the contract shipped version six, so a version-six envelope failed the SDK's version byte.

#### Impact

A replayed alias restored a stalled slot to apparent freshness, which is the state consumers gate on. An uncapped heal let a lane that had been dark return with a jump larger than the configured band permits, which is exactly the move the band exists to price. The sigma floor gap left the band producer-determined, affecting both liveness and adverse-selection cost.

#### Remediation

One decoder now treats a lane past the `u128` boundary as stale, closing the alias. A bounded future observation reads as fresh with confidence floored at the realized move. Sigma is floored at the realized move on every blob shape, including price-only submissions, where the price path itself writes the maximum of the stored sigma and the move, and the sigma term is capped at `maxDev` times the configured multiple on every lane. `attestReentry`, the reference set and the quarantine bit were deleted, so a dark lane heals only through a normal push under the capped band; the reference tier remains a separate instance with its own push roster. The SDK decodes version six against a fixture that is byte-identical to the contract's.

Commits: [`230d0190`](https://github.com/btr-protocol/dex-evm/commit/230d0190f87724144807d9d2e18762f5b9f4a7b3), [`11cc7dd9`](https://github.com/btr-protocol/dex-evm/commit/11cc7dd9029c2cfb53710b87f78549f968d98700), [`20f7af74`](https://github.com/btr-protocol/dex-evm/commit/20f7af7416efada125171baa685490e7b903edc7), [`81c8bf96`](https://github.com/btr-protocol/dex-evm/commit/81c8bf96328c83d344fda40980fdfb4c53ea81a3), [`761a7f07`](https://github.com/btr-protocol/dex-evm/commit/761a7f07f938ccba6f2970ea4fdf150649fc68ca), [`25afd339`](https://github.com/btr-protocol/dex-evm/commit/25afd3390d08a7f21f451c8df68532956d1f892b), [`33d56272`](https://github.com/btr-protocol/dex-evm/commit/33d562722dacd4ef456707902e83499cc4b2a681), [`e092951d`](https://github.com/btr-protocol/dex-evm/commit/e092951d174a45ade242dbc7704c2ef22603a00c), [`7a183eac`](https://github.com/btr-protocol/dex-evm/commit/7a183eac95aad0adb26cc5141e03dd4eb8f24cc4), [`19b65df5`](https://github.com/btr-protocol/dex-evm/commit/19b65df582cbe4e2e765b3a9ec27d990545edcfc), [`47423b03`](https://github.com/btr-protocol/dex-evm/commit/47423b031edff4f9ca51f39968a1b146cc729605), [`060af9d`](https://github.com/btr-protocol/sdk/commit/060af9d9612d3731860608cb7bca43f0ae3c5cf1), [`91b84d0`](https://github.com/btr-protocol/sdk/commit/91b84d084128914814ad56eec5ad7ac0bd5ef7dd).

#### Status

Fixed, verified 2026-09-14 and 2026-09-15. One row is closed with no action: the bounded reentry nonce reverted at its 16-bit ceiling, which is moot now that the lever is deleted. The off-chain quote producer still emits the prior wire version; that cutover is tracked outside this scope and is not a contract change.

**Rows.** 12 rows.

### F-36  Hook yield was booked at par while donations were booked at coverage, and the face conversion divided by zero

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/Pool.sol:692-693`, `dex-evm/src/libraries/PoolLiquidityLib.sol:183`, `dex-evm/src/Pool.sol:701` |

**Severity rationale.** Every harvest moved coverage on a live book, so the divergence compounded with harvest frequency rather than needing an attacker.

#### Description

`hookCreditYield` credited the liability at par while `donate` credited it at coverage, so the two liability-credit sites disagreed on the unit they book in. Coverage therefore moved on every harvest, transferring value between legs.

```solidity
// dex-evm/src/Pool.sol:692
    $.assetHooks[t].lastCreditAt = uint32(block.timestamp);
    a.reserves += uint128(amount);
    a.liabilities += uint128(amount);
```

The remediation introduced two follow-on defects on the same path. Booking at face added an oracle dependency to harvest, so `hookCreditYield` reverted on any unusable non-target mark and one bad leg blocked every harvest. The daily rate bucket then mixed units, capping a token amount against the face book, which leaves the effective cap loose by the inverse of coverage whenever coverage is not one. Separately, the face conversion divides by coverage in deposit, donate, `hookCreditYield` and `hookWriteDown`, and only the swap-liability site carried the zero guard, so a zero coverage produced a panic; donate and `hookCreditYield` also lacked the zero-face guard.

#### Impact

Hook yield booked at the wrong unit shifts coverage, and coverage is what the wall and the toll are priced from, so the error is a cross-leg value transfer rather than an accounting cosmetic. The missing zero guard turns a degenerate coverage into a revert on four user-facing paths.

#### Remediation

Hook yield and the LP fee are booked at `face = amount * WAD / C`, the cap applies to the face while the whole push is booked, and an unusable mark degrades fail-closed rather than reverting the harvest. Five face-at-coverage sites refuse `C == 0` and the zero-face guards were added, with tests on each.

Commits: [`2e5e8363`](https://github.com/btr-protocol/dex-evm/commit/2e5e836318d5f79724f9ecbfc21170b9b55bf60c), [`eaf555ff`](https://github.com/btr-protocol/dex-evm/commit/eaf555ff348a60c08d1bf124b1b350d8eb04426f), [`9e00d331`](https://github.com/btr-protocol/dex-evm/commit/9e00d33194adff4fd7bd003ed4a92e75af76c53d), [`bd689d6b`](https://github.com/btr-protocol/dex-evm/commit/bd689d6beb635a10f4811e65e74b0e1bcafa22e9), [`d9a6b556`](https://github.com/btr-protocol/dex-evm/commit/d9a6b556b0f3292e4ef4aa0e98fd36741404cc59), [`518c2c2f`](https://github.com/btr-protocol/dex-evm/commit/518c2c2f2d44583d6efda5baac707b4f8c63fcaa), [`9dc67a46`](https://github.com/btr-protocol/dex-evm/commit/9dc67a466c48311e051a56cee835cb9557290eba).

#### Status

Fixed 2026-09-15. The harvest degrade row is conditional on the solvency-degrade decision holding: if the degrade were reverted to a hard failure, harvest would re-block on a funded dead leg, which is an accepted fail-closed outcome with keeper retry.

**Rows.** 4 rows.

### F-37  Hook cancel and write-down authority was protocol-scoped, so a foreign pool's hook slot was reachable from outside

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/hooks/YieldHook.sol:65-69`, `dex-evm/src/Pool.sol:614` |

**Severity rationale.** The reachable action is a write-down or a queued-operation cancel on a pool the caller does not administer; it requires a privileged role, not an anonymous caller.

#### Description

`YieldHook`'s cancel authority was the protocol guardian or owner rather than the pool's own seat, so a foreign pool's seat could not cancel its own queued hook operation while the protocol could cancel one belonging to a foreign seat. The hook seat authority was not pool-scoped on the write-down and recall paths either, so a keeper push could reach another pool's hook slot.

```solidity
// dex-evm/src/hooks/YieldHook.sol:65
  modifier onlyGuardianOrOwner() {
    AccessControl ac_ = AccessControl(AC);
    if (!ac_.isGuardianOrAuth(msg.sender, ac_.owner())) revert Err.NotAuth();
    _;
  }
```

#### Impact

Authority over a foreign pool's hook lifecycle sat with the protocol rather than with the pool that owns the assets, in both directions: the protocol could cancel a foreign seat's operation, and the foreign seat could not cancel its own.

#### Remediation

Cancels are seat-routed on foreign pools, the write-down and recall paths are target-bound, and the hook's pool is immutable, so a keeper push cannot reach another pool's slot.

Commits: [`e08f8f2e`](https://github.com/btr-protocol/dex-evm/commit/e08f8f2e1363cd0eed323794dd29b4740ac55e4f), [`5360dc33`](https://github.com/btr-protocol/dex-evm/commit/5360dc3320f5d42044198785c77c521531cf5f08).

#### Status

Fixed. Both rows closed in the same change.

**Rows.** 2 rows.


### F-39  The claim periphery accepted a malformed venue tree, and its first claim after funding hits the anti-JIT window by design

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Closed | Audit | `dex-evm/src/periphery/WombexClaim.sol:91`, `dex-evm/src/periphery/WombexClaim.sol:140-163`, `dex-evm/src/LPToken.sol:145-160` |

**Severity rationale.** Both rows require a deployment-time mistake or a specific one-transaction sequence rather than an adversary, and the failure mode is a stranded or mispriced leg in a periphery contract, not a loss from a pool.

#### Description

The claim contract's constructor validated the leg array but not the tree root, and did not reject duplicate legs. A root of zero or a repeated leg produced a tree that walks without reverting and can mis-price or strand a leg.

```solidity
// dex-evm/src/periphery/WombexClaim.sol:91
  constructor(address pool_, address funder_, bytes32 root_, address[] memory legs_) {
    if (pool_ == address(0) || funder_ == address(0)) revert Err.ZeroAddr();
    uint256 n = legs_.length;
    if (n == 0 || n > MAX_LEGS) revert BadLegs();
    pool = IPool(pool_);
    funder = funder_;
    root = root_;
    legCount = n;
    for (uint256 k; k < n; ++k) {
      if (legs_[k] == address(0)) revert Err.ZeroAddr();
      legs[k] = legs_[k];
    }
  }
```

Separately, the first claim following `fund` reverts with `CooldownActive`: the distributor's own LP receipt inherits the pool's mint freeze, so funding and claiming cannot settle in one transaction. The revert is pinned by an existing test.

#### Impact

A mis-shaped tree deployed once would mis-price or strand a leg for the life of the contract. The cooldown revert is friction on an operational sequence that is never run as one transaction.

#### Remediation

A zero root and duplicate legs are refused at construction, pinned by unit tests. The anti-JIT window is kept deliberately.

Commits: [`4c7fe903`](https://github.com/btr-protocol/dex-evm/commit/4c7fe903f2cbd89da8552cec7b89aa1ac954bbb0), [`05213a8e`](https://github.com/btr-protocol/dex-evm/commit/05213a8e0a03c877c0b855c458f745f43ef9accb).

#### Status

Closed. The tree validation is fixed; the cooldown is closed as intended behaviour, re-verified 2026-09-14, because funding and claiming are never issued in one transaction.

**Rows.** 2 rows.

### F-40  An issuer pause or blocklist on a listed token turns an LP cross exit into an unrefillable leg

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Closed | Audit | `dex-evm/src/libraries/PoolLiquidityLib.sol:340` |

**Severity rationale.** It requires a third-party issuer action rather than an attacker, but the affected legs are listed centrally issued tokens where that action is a real and exercised capability.

#### Description

The withdraw entrypoint gates both endpoints on the pool's own halt flags. It has no view of the token issuer's state, so a leg whose issuer has paused transfers or blocklisted the pool still passes the gate and the cross branch cannot be refilled.

```solidity
// dex-evm/src/libraries/PoolLiquidityLib.sol:360
      // HALT_MASK check on BOTH endpoints: withdrawTo is a value-moving user
      // entrypoint (esp. cross-asset, priced off the output mark). Without this a halt is
      // bypassed: draining a halted asset's reserves, or pushing a good asset
      // out priced by a halted/compromised feed. Interior-node halts (Pricing) don't cover
      // endpoints, and the direct spoke→base case has no interior node at all.
      PoolIOLib.checkRiskFlags(assetFrom.flags, 0);
      PoolIOLib.checkRiskFlags(assetTo.flags, 0);
```

#### Impact

While an issuer pause or blocklist is in force, the affected leg cannot be refilled through a cross exit. The residual is a liveness constraint imposed from outside the protocol, not a solvency defect.

#### Remediation

The hub token is probed off chain through its `baseToken()` view and a revert, never an RPC error, is treated as a verdict; a blocked hub pages the operators. The issuer-pause residual is owner-accepted and remains page-only, since no on-chain control can pre-empt an issuer action.

#### Status

Closed. Monitoring landed on the second remediation revision and the accepted residual was re-signed off on 2026-09-14.

**Rows.** 1 row.

### F-41  The interior displacement ceiling was a field-width artifact charged to every leg

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/libraries/PricingLib.sol:45,48-50`, `dex-evm/src/libraries/PricingLib.sol:236-250`, `dex-evm/src/libraries/PoolConfigLib.sol:73-80` |

**Severity rationale.** No fund loss and no attacker path; the cost was that a risk-shaped bound was in fact a storage-packing bound, applied to legs it could not describe.

#### Description

The 50 bp interior displacement ceiling was derived from the width of the `uint16` spread field, not from any property of a leg's move distribution. The fence is `ceil(x * PBPS / (PBPS - cap/2))`, and at most `MAX_INTERIOR_LEGS = 6` interior legs must sum under 65535, which solves to a cap of about 10862 pbps. That worst-case number was then applied to all pools and all legs regardless of depth.

It bound `minDispersionPbps` at write time even on depth-1 legs that can never be routed as an interior node, which is why one equity leg could not carry its tail. Being a constant fraction of mark, it is simultaneously far too wide for a stable pair and too narrow for an equity at the open. The same field width also let the composed spread saturate a `uint16` at peak confidence interval, waiving the tail premium at a 3.28% maximum fee against 11% modelled, though the fence component itself never saturates.

```solidity
// dex-evm/src/libraries/PricingLib.sol:44-50
  uint256 private constant FENCE_BUDGET_PBPS = uint256(type(uint16).max) / MAX_INTERIOR_LEGS;
  /// @notice Interior-leg mid swingPbps ceiling, PBPS - SOLVED FOR, never chosen. br.market/docs.
  /// @dev DERIVATION: `_fenceOfSwingPbps` is ceil(x*P/(P - cap/2)) over P = PBPS, so the budget line
  ///      `N*fence <= uint16.max` at the worst case x = cap is ceil(cap*P/(P - cap/2)) <= B, i.e.
  ///      cap <= 2*B*P/(2P + B) - this expression. MAX_DEPTH 4 => N 6, B 10922, cap 10862, per-leg
  ///      fence 10922, composed 65532 <= 65535.
```

Two adjacent documentation defects sat on the same code. The interior-fence guard suite, fifteen tests, had been red since the 2026-08-21 removal of sigma damping. The `_legMid` natspec asserted the mid is never zero, which is overbroad: `flooredOffsetPrice(0)` is zero and the guarantee actually lives upstream at the mark gate.

#### Impact

Legs that can never be interior were denied dispersion they could safely carry, and the ceiling gave no protection proportional to any leg's actual volatility. The red guard suite meant the property the fence is supposed to hold was unverified for the duration.

#### Remediation

Interior-capability scoping is in tree: `PoolConfigLib.dispersionCeiling` and `isInteriorCapable` charge the cap only to legs that can be routed as an interior node, which closes the substantive complaint. The depth-aware storage re-pack was considered and dropped; the `uint16` saturation is retained deliberately, since the fence component is bounded at 65532 and cannot saturate, and reverting instead would deny service to legitimate quotes. The guard suite is green at 895 passing. The `_legMid` guarantee is restated at its true locus.

#### Status

Fixed for the scoping and the test suite. The saturation behaviour is accepted as designed and the residual re-pack is recorded as decided against, with one workplan entry noted as contradicting that decision.

**Rows.** 4 rows.

### F-42  The Rust mirror and off-chain replicas drifted from the shipped Solidity pricing law

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `core: src/fixed.rs`, `core: src/pricing.rs`, `core: src/mitch.rs` |

**Severity rationale.** The mirror is the quoting and simulation authority off-chain, so a constant that lags the chain produces confidently wrong quotes with no revert to signal it.

#### Description

`STALE_Z` was raised from 100 to 472 in Solidity only. The Rust mirror, the SDK's generated constants and the published documentation all stayed at 100, and the mirror's own parity test pinned the stale value, so the drift was invisible to the suite that exists to catch it.

```rust
// core: src/fixed.rs
// PricingLib.sol
pub const STALE_Z: U256 = U256::from_u64(100);
pub const STALE_GRACE_CAP_SECS: U256 = U256::from_u64(30);
pub const MAX_DISPERSION_PBPS: U256 = U256::from_u64(900_000); // PoolConstantsLib
```

The same mirror was not updated for the 2026-09-04 interior swing cap. Its merged single-source-of-truth module also contradicted itself, declaring `MAX_INSTRUMENT_TYPE` as `FUND` (0xC) while a `STRUCTURED` type existed above it, and carried roughly 67 lines of unreferenced code including two constant tables that had silently gone stale.

A related family of view-versus-execution divergences sat on the same law. `_legExecPrice` reverted `ZeroValue` when the price floored to zero on an analytics-gated view path while execution settled the same dust, so off-chain routing failed on executable dust. Fee and toll flooring rounded one wei toward the trader rather than the pool. The full-drain preview clamped to reserves and omitted cross fees and caps, while execution reverted or confirmed through `minAmountOut`. Off-chain decimal fallbacks guessed 18 on route, liability and allowance paths, with the money path fail-closed behind on-chain bounds.

#### Impact

A replica quoting against a stale staleness multiplier prices the staleness premium wrong in the direction that understates it. The dust revert broke off-chain routing for orders the chain would have settled. The rounding direction handed the pool's dust to the trader. None of these moved funds beyond dust, but each made the off-chain price disagree with the chain.

#### Remediation

`STALE_Z` and the interior swing cap are now derived in the mirror rather than written as literals, with `sol_const_pin.rs` diffing the derivation against live Solidity source and pinning the values per function. The instrument and class tables are derived as the single source of truth, which also removes the unreferenced tables. `_legExecPrice` returns zero on dust instead of reverting, and fee and toll now round toward the pool with the mirror following.

Commits: `core@590b3e1`, `core@b671bc9`, `core@3f71c59`, [`1511d696`](https://github.com/btr-protocol/dex-evm/commit/1511d6962a8bc055c652b89e91b62d565b1685b6), [`4f8a42a`](https://github.com/btr-protocol/sdk/commit/4f8a42af7f784beea3a89e16614dfeb71a51a7f0), [`ccded83`](https://github.com/btr-protocol/content/commit/ccded83c416e211da8e6652d2ffa5fda5b64f9c4).

#### Status

Fixed on 2026-09-11. Pinned by `pricing_parity.rs` and `PricingRounding.t.sol:30,:56,:67,:78`. The remaining informational items were closed in the 2026-09-10 review.

**Rows.** 8 rows.

### F-43  The swap submit path re-anchored its floor and admitted a double submit

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `front/src/components/features/swap/SwapForm.tsx:1163-1198`, `front/src/components/features/swap/SwapForm.tsx:1153-1157`, `front/src/components/features/swap/SwapForm.tsx:749` |

**Severity rationale.** Both defects need only an ordinary user on an ordinary tape; the outcome is a fill below the displayed floor or a duplicated order, each of which the user must still sign.

#### Description

On submit the form replaced the displayed plan with a fresh route plan and derived every floor from it, never comparing against the `amountOut` the user was shown off an eight-second poll and never re-prompting. An adverse move inside the display window therefore executed below the on-screen minimum received, whose tooltip promised a revert. The SDK's `refloorRouterPlan` takes the minimum of quoted and fresh output, which guards the favourable direction only; the caller obligation to refuse or warn when the fresh output falls below the slipped quoted output was documented but unimplemented. The gap is bounded by adverse drift over at most eight seconds plus fetch latency, and the executed price is still fresh market minus tolerance, default 0.5%.

The re-quote, the balance re-check and the plan build all ran before `setSubmitting`, leaving roughly an 800 ms window with no spinner in which a second click launched a parallel batch on sequential nonces, debiting the input amount twice. Each flight required its own wallet confirmation, so there was no silent double spend, but the absent spinner invited the second click.

```ts
// front/src/components/features/swap/SwapForm.tsx:1165-1180
    try {
      const spendable = spendableOf(fromSym, rawBalanceOf(fromSym));
      if (spendable !== undefined && exactAmountIn !== undefined && exactAmountIn > spendable) {
        return void addNotification('error', `Not enough ${fromD}: balance moved since the quote.`);
      }
    } catch {
      // Unparseable size: the batch builder below reports it, not the funds gate.
    }

    const slipFrac = slippagePct / 100;
    const rawLegs = planToLegs(best, {
      slippageFrac: slipFrac,
      tokenOf: getToken,
```

Adjacent to these, the cross-withdraw fallback used when the routing backend was unreachable previewed a full-face USD value with no haircut, spread or fees, deriving a minimum output too high and producing a `ThresholdViolation` revert. The send-path slippage fraction was unclamped, so auto mode above 100% displayed a negative minimum before the assertion aborted the send. Liquidity-provider execution sent aged floors with no send-time re-quote, a preflight dry run failing cleanly before the prompt. Step context was dropped on liquidity-provider submit, the success line printed the quote rather than the receipt, and the swap interface read only the feed gate, so a halted-leg swap built and then reverted.

#### Impact

A user could receive less than the minimum received figure the interface displayed and described as enforced. A second click during the pre-submit window produced two mined swaps for one intent. The backend-down cross-withdraw path produced a guaranteed revert rather than a conservative preview, costing gas.

#### Remediation

The form now encodes the displayed floor and refuses on a lower re-quote, with the SDK carrying the matching caller-side check. A latch drops a second click before the asynchronous re-quote and plan build. The unfloored USD fallback is gone and cross-withdraw without a plan is gated. The remaining display and friction items were reviewed and closed below the low-severity bar.

Commits: [`48587b54`](https://github.com/btr-protocol/front/commit/48587b54b24820dde04608759c4048ad237fc82a).

#### Status

Fixed on 2026-09-11, pinned by `swapTx.test.ts`. Informational rows closed on 2026-09-09 and 2026-09-10.

**Rows.** 8 rows.

### F-44  The router library trusted backend-supplied pool addresses and mis-scaled chained floors

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `sdk/src/router/index.ts:352-369`, `sdk/src/router/index.ts:277-293`, `sdk/src/router/lpRoutes.ts:451-475` |

**Severity rationale.** The library grants an approval to whatever pool address it is handed, so a consumer that builds plans from an untrusted route service loses the approved balance; the shipped front end defuses it, direct consumers do not.

#### Description

`planToLegs` accepted the backend-supplied `poolAddr` verbatim with no allowlist. The legacy N-call approval path grants per pool, so a rogue pool address served in a route plan received an approval and could drain it, amplified when `approveMax` was set. The front end defuses this by sending the token universe to the backend, tag-allowlisting the result, defaulting `approveMax` to false, preferring the router and backstopping with `UnknownPool`; direct SDK and legacy consumers had none of that.

```ts
// sdk/src/router/index.ts:350-369
export function buildApprovalCalls(legs: ExecLeg[], opts: BuildOpts): ExecCall[] {
  const wnative = opts.wrappedNative?.toLowerCase();
  const { wrapValue } = validateLegs(legs, wnative);
  const exactByKey = new Map<string, bigint>();
  for (const leg of legs) {
    const key = `${leg.tokenIn.toLowerCase()}:${leg.pool.toLowerCase()}`;
    exactByKey.set(key, (exactByKey.get(key) ?? 0n) + leg.amountIn);
  }
  const approveAmt = (key: string): bigint =>
    opts.approveMax ? MAX_UINT256 : (exactByKey.get(key) ?? 0n);
```

The route enumerator existed twice. The second copy in `route.ts` had no production caller and had already drifted: its three-hop arm was gated on no two-hop route existing, so a better three-hop route was unreachable whenever any two-hop route was found.

Three further defects sat on the same surface. The chained two-leg batch undersized leg two and over-floored it with zero margin, producing spurious `ThresholdViolation` reverts on ordinary noise. The liquidity-provider route and liability replica still modelled the deleted per-leg haircut, applying coverage on input and then output, while the chain settles at pool-level coverage, giving wrong route rankings and wrong floors in both directions. The unwrap path had a recipient mismatch: the router pays wrapped native to the recipient while the SDK's unwrap call withdraws from the sender, so a recipient different from the sender either reverted empty or debited the sender from a prior balance. Scales above 18 decimals were truncated, unreachable today because listing rejects them.

#### Impact

A consumer building plans from an untrusted route service could have an approved balance taken by an address of the service's choosing. The duplicate enumerator silently excluded better three-hop routes. The mirror drift produced floors that did not match how the chain settles, and the chained floor produced avoidable reverts.

#### Remediation

`planToLegs` now requires `opts.isOfficialPool` per hop, and approvals default to the exact amount with `approveMax` opt-in. The duplicate enumerator was removed, leaving one. The chained leg-two floor is scaled by the ratio of leg one's minimum output to its quoted output. The route and liability replicas mirror pool-level coverage, with the front-end callers updated in the same change. `assertUnwrapSelfDirected` refuses a native-out plan whose recipient is not the sender, and the front end passes the sender into the unwrap path.

Commits: [`3a03ab7`](https://github.com/btr-protocol/sdk/commit/3a03ab78ad4bb3b997faa51599017364db382389), [`bfd0170`](https://github.com/btr-protocol/sdk/commit/bfd017027bfec2e53f989ba88506d4d721c89d53), [`8b822d6`](https://github.com/btr-protocol/sdk/commit/8b822d6f2219383f789096813b111febe6cbd78d), [`095a741`](https://github.com/btr-protocol/sdk/commit/095a741c9aff268f559f4b46e0e84d1b8a77decd), [`b3d71b06`](https://github.com/btr-protocol/front/commit/b3d71b067a66ecad0212536fe2e86a859ea1b343), [`4f452bc0`](https://github.com/btr-protocol/front/commit/4f452bc0f27ebc502f2550163a884697f6b465c6).

#### Status

Fixed on 2026-09-10 and 2026-09-11. Pinned by `liability.test.ts:40` and `lpRoutes.test.ts:215`. The decimal-scale item was closed below the low-severity bar on 2026-09-10.

**Rows.** 6 rows.

### F-45  Trading routes rendered before the access gate and continuous integration had not run since 09-14

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | QA | `front/.github/workflows/ci.yml:44`, `front/src/App.tsx:318`, `front/src/pages/metrics/metricsModel.tsx:121` |

**Severity rationale.** A broken pipeline and an ungated route are both certain, not probabilistic; the impact is loss of the gate that was supposed to hold, not loss of funds.

#### Description

The workflow pinned bun 1.4.3, a version with no GitHub release, so every run on the main branch since 2026-09-14 died at the setup step. No gate had actually executed in that window.

```yaml
# front/.github/workflows/ci.yml:42-44
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: "1.4.3"
```

The `/swap-form` and `/chart` routes rendered before the invite and disclaimer gate, so a direct link reached the trading interface without passing it. Three smaller defects sat alongside: the vite dev plugin referenced an undeclared source variable, merged-asset `strategyApr` took the first non-null value instead of a value weighted by total value locked, and `prependCandles` evicted the newest candles rather than the oldest once past the retention limit.

#### Impact

For two days the pipeline reported failure at setup rather than running any check, so nothing merged in that window was gated. The ungated routes exposed the trading interface without the invite and disclaimer step. The aggregate rate displayed on the metrics page was not representative of the merged position.

#### Remediation

The bun pin moved to a released version and the gates run again. Every trading route now sits behind the invite and disclaimer gate. The dev plugin declares its source, and displayed aggregates are weighted by total value locked. The candle eviction order was reviewed and accepted as it stands.

Commits: [`fe4fdcc`](https://github.com/btr-protocol/front/commit/fe4fdcc56b489128abe96521567a1855704890f1), [`5a56106`](https://github.com/btr-protocol/front/commit/5a56106d33ef3dae7c65ee935fe42edb50480322), [`e2694a8`](https://github.com/btr-protocol/front/commit/e2694a8a5a1c24a5a42839435097befc23630ec2), [`92885d3`](https://github.com/btr-protocol/front/commit/92885d3ca8c369c5a8e8753624dfb5a6ebddc512), [`0fcd1b9`](https://github.com/btr-protocol/front/commit/0fcd1b925ce95e34d4e1a9d306291e6bf5027b81).

#### Status

Closed on 2026-09-16, with the candle eviction row accepted, no change required.

**Rows.** 5 rows.

### F-46  The wallet send lifecycle could report a broadcast transaction as cancelled

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `front/src/lib/walletCalls.ts:208`, `front/src/lib/wallet.tsx:477`, `front/src/components/shared/TxSteps.tsx:79` |

**Severity rationale.** A user acting on a false "cancelled, no gas spent" message will resubmit, and the resubmission is a second real transaction; no attacker is required.

#### Description

`sentDespiteRejection` decided whether a rejected prompt had nevertheless broadcast by comparing the pending nonce before and after, retrying once after 1.2 s. A transaction that reached the mempool after that window read as unchanged, and the interface certified it as cancelled with no gas spent.

```ts
// front/src/lib/walletCalls.ts:203-214
export async function sentDespiteRejection(
  provider: Eip1193Provider,
  from: Address,
  before: number | undefined,
): Promise<boolean> {
  if (before === undefined) return false;
  for (const waitMs of [0, 1200]) {
    if (waitMs) await new Promise((r) => setTimeout(r, waitMs));
    const after = await pendingNonce(provider, from);
    if (after !== undefined && after > before) return true;
  }
  return false;
}
```

In a non-atomic batch, calls that had already mined never emitted a confirmation once a later receipt reverted, so the interface showed nothing for work that had actually settled. The transaction overlay labelled every approval rung with hop one's token, pool and amount, so a multi-hop plan displayed the wrong subject on each step. Two separate gas-reserve formulas existed and could disagree.

#### Impact

A user told a broadcast swap was cancelled will retry and pay twice. A user whose earlier batch calls mined saw no record of them. Mislabelled approval rungs meant the wallet prompt and the interface disagreed about what was being approved.

#### Remediation

The send lifecycle now reports what was actually signed, broadcast and mined, and refreshes balances and allowances afterwards. Mined predecessor calls emit their confirmations even when a later call in the batch reverts. Each approval rung carries its own token, pool and amount. The duplicated gas-reserve formulas were reviewed and accepted as they stand.

Commits: [`93383ae`](https://github.com/btr-protocol/front/commit/93383ae7d489d2ccf88673e218a9ae3359e78ca3), [`710cbbf`](https://github.com/btr-protocol/front/commit/710cbbf890711f79abb61b42f1ee3d9c58a86219), [`1d7b649`](https://github.com/btr-protocol/front/commit/1d7b6498272ae66b54f904ec8759d069f70bfc44), [`0193040`](https://github.com/btr-protocol/front/commit/0193040d9e448dc23677e2fa4348b30cc74612cd), [`6e9238c`](https://github.com/btr-protocol/front/commit/6e9238cf8471524cba4b2f1814a3dda91dfd8330).

#### Status

Closed on 2026-09-16, with the gas-reserve row accepted, no change required.

**Rows.** 4 rows.

### F-47  A pool with reserves and no liabilities returned a full coverage rate and handed the surplus to the next depositor

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/libraries/PoolSolvency.sol:79`, `dex-evm/src/libraries/PoolSolvency.sol:131` |

**Severity rationale.** The stranded state is reachable through ordinary use, a full same-asset exit below par, and the next depositor captures the stranded reserves with no special access.

#### Description

`solvency` returned `WAD` whenever the claim book was empty, including when the pool still held reserves. A full same-asset exit at a coverage rate below one strands reserves with zero liabilities, and the next depositor mints face value against a book that already holds surplus, capturing it. The natspec claimed the pooled rate was preserved.

```solidity
// dex-evm/src/libraries/PoolSolvency.sol:76-80
      navBase += (r * px) / SC.WAD;
      claimBase += (l * px) / SC.WAD;
    }
    if (claimBase == 0) return (SC.WAD, true, navBase, claimBase);
    cWad = (navBase * SC.WAD) / claimBase;
```

The first integrated fix returned `(0, false)` for that state, which hard-refused it: `mintRate` reverted `FeedUnavailable`, so deposit, donate, liability swap and hook credit all froze with no heal lever until value was added or the claim book re-seeded, while the same-asset exit cap still paid the last good coverage rate. A separate concern, that `lastGoodCWad` had no age bound, applied to the degrade fallback.

#### Impact

Before the fix, reserves left behind by a below-par exit accrued to whoever deposited next rather than to the exiting liquidity providers. After the first fix, the same state froze every credit path in the pool instead.

#### Remediation

A stranded book with reserves and no liabilities now reads `(WAD, false, A, 0)`, and only the pool owner's deposit re-seeds it, so the surplus is neither captured by an arbitrary depositor nor permanently frozen. The degrade fallback that needed an age bound was removed together with the fail-closed revert of the par-degrade.

Commits: [`1a02316d`](https://github.com/btr-protocol/dex-evm/commit/1a02316dc78bda6d5b8f86d680e477fb8f9e5d0e), [`d460cf93`](https://github.com/btr-protocol/dex-evm/commit/d460cf93e124dc420c3f74e67195468f94c3acc0), [`62065e8d`](https://github.com/btr-protocol/dex-evm/commit/62065e8d01d73008b6b091d5c2c276ad5e525da7), [`c4e13474`](https://github.com/btr-protocol/dex-evm/commit/c4e13474fdaf01dfa90b85f5c5d3bda74ae29e8c), [`96fc09ac`](https://github.com/btr-protocol/dex-evm/commit/96fc09ac39ae05ece7eb7f6f0ebc95767d5b062a), [`230ebc23`](https://github.com/btr-protocol/dex-evm/commit/230ebc23b5fe26bbf6b9af2c21b147e6db35e905), [`afeff22a`](https://github.com/btr-protocol/dex-evm/commit/afeff22a8262f56d9773519062d7948f39667699).

#### Status

Closed. Pinned by `PoolSolvency.t.sol:569`.

**Rows.** 2 rows.

### F-48  Deploy and operator scripts still targeted retired oracle surfaces and levers that no deployed contract exposes

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/script/Deploy.s.sol:80-88`, `dex-evm/script/OracleV4Unwedge.s.sol:61-64`, `dex-evm/script/lib/ChainParams.sol:74-78` |

**Severity rationale.** Every failure mode here is fail-closed at broadcast rather than a mispricing, but the incident-time scripts are exactly the ones whose failure matters most, and a new listing built against a retired contract surface would have to be re-run.

#### Description

The listing path still compiled against the V1 `ExternalOracle` surface. `Deploy.s.sol` used the V1 type as the shared feed ABI and `ArcOracleDeploy.s.sol:180` constructed it, so a listing performed with the shipped scripts would have targeted a retired contract. The same ceremony shape persisted throughout `ArcPoolDeploy.s.sol`: the legacy-oracle gate reverted without a `REF_ORACLE` bypass, the mirror and anchor helpers called a V1-only `addFeed`, the mark helper required seed keys a greenfield run never writes, and `FeedOrderLib.write` required a `getFeedIds` the live contract does not expose.

```solidity
// dex-evm/script/Deploy.s.sol:80
  function _broadcastDeployWith(address acOverride) internal returns (Addrs memory a) {
    uint256 pk = vm.envUint("DEPLOYER_PK");
    a.deployer = vm.addr(pk);
    // If DEPLOYER is set it must match the PK-derived address.
    try vm.envAddress("DEPLOYER") returns (address d) {
      require(d == a.deployer, "DEPLOYER/DEPLOYER_PK mismatch");
    } catch {}
    a.treasury_owner = _resolveTreasury(a.deployer);

    vm.startBroadcast(pk);
```

The unwedge ceremony had the inverse problem. The scripts are typed against a `pendingFeedWiden` / `requestFeedWiden` / `executeFeedWiden` / `cancelFeedWiden` lever that landed after the live oracles were deployed, so those selectors are absent from every deployed runtime blob. The script's `preview()` touches only read functions, so the dry run was clean and the failure was deferred to broadcast. `execute()` also ignored the per-lane selection filter, releasing all pending entries rather than the still-selected ones. Alongside these, `SafetyOps.s.sol` was a bare `Script` with no chain assertion, so an incident-time halt aimed at the wrong RPC would revert or mis-halt; `UpgradePoolImpl.s.sol` minted a Pool without enforcing the library pins its own comments mandated and logged a hardcoded 6 h delay; `ChainParams` read a `.chain.govDelays` manifest key that no script parses, contradicting its own natspec; the CREATE3 fleet manifest carried no chain-id binding; and the legacy and mocks gates could be bypassed by pointing `RISK_PARAMS` at a fixture file.

#### Impact

A listing run would have deployed against a retired oracle surface. The unwedge and halt ceremonies, both incident-time tools, would fail at broadcast rather than in the dry run, and the batch release could touch lanes the operator had deselected. The testnet faucet let any address self-whitelist, so its per-address cap was not a cap.

#### Remediation

The V1 oracle sources and the V1 ceremony surface were deleted and the listing scripts ported to the V4 `registerFeed` surface with `REF_ORACLE` mandatory, with `forge build` green. `OracleV4Unwedge` now probes for lever presence by code and selector and prints `LEVER ABSENT` instead of reverting at broadcast, and its `execute()` releases only still-selected, unexpired lanes. `SafetyOps` extends `ChainParams` with `_assertChain` on each entry point and documents the guardian halt path. `UpgradePoolImpl` asserts all four derived library pins against the linked library address before broadcast, so a stale or foreign link reverts instead of minting. `ChainParams` refuses a manifest `.chain.govDelays` key and pins `PROD_DELAYS`, reads the CREATE3 fleet from a chain-id-bound manifest, and refuses a `RISK_PARAMS` override off a local-class chain.

[`46622d71`](https://github.com/btr-protocol/dex-evm/commit/46622d71991e78b3c1e961c5f5cbf2db166c2f5f), [`93fd1951`](https://github.com/btr-protocol/dex-evm/commit/93fd1951af28e0a9fbbd4cbfbf2c99d00610a77a), [`a236b6f2`](https://github.com/btr-protocol/dex-evm/commit/a236b6f25b9ee4124806a0645cbe7cb4e8476f66), [`6623561b`](https://github.com/btr-protocol/dex-evm/commit/6623561b40bcc916a1be6bdc8fed1e74b2c14a13), [`df7a2c35`](https://github.com/btr-protocol/dex-evm/commit/df7a2c3583346b639ab84afb3e5b05719dbfb7be), [`fb975b2b`](https://github.com/btr-protocol/dex-evm/commit/fb975b2bc306c0d04ce66258ef31b5f18bbc85cf)

#### Status

Fixed, verified 2026-09-10 and 2026-09-11 and at the rev2 chair review. Four informational rows were closed rather than fixed: the testnet faucet is testnet-only and is replaced by a pool-side deposit allowlist; the `FORCE_EXECUTE` dark-leg repoint is a deliberate opt-in lever, default off; the hardcoded 6 h delay in the `UpgradePoolImpl` log is correct on the chain it runs on; and the standing claim that no production file imported the V1 oracle was corrected in the record.

**Rows.** 15 rows.

### F-49  Launch manifests shipped dispersion floors and feed-id bindings that did not match the signed feed set

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/deployments/arc-risk-params.json:593`, `dex-evm/script/OracleV4Deploy.s.sol:127-133`, `dex-evm/deployments/bnb-risk-params.json` |

**Severity rationale.** Too-narrow dispersion is the direction that picks off liquidity providers, and an identity base depeg lane means the halt that protects every swap on the chain can never fire; both ship in the manifest and neither is caught by a code-level cap.

#### Description

The 2026-09-03 patch set `minDispersion` flat at 1950 pbps (19.5 bp) for all ten equity names. The measured q999 tail for those names runs 22.4 to 38.7 bp, so the floor was narrower than the tail on all ten. Per-name dispersion had never been written. The value passes every code cap; the defect is that the floor underprices tail dispersion.

The BNB scaffold had two binding defects. The deploy registered feed ids as `keccak(token, quote)` while the push side addresses feeds by their MITCH `bytes32(ticker_id)`, so every genesis lane, the base USDC-USD lane included, would have gone stale on day one. Separately, the manifest pinned the `.USD` unit to USDC itself, which makes the base depeg lane `keccak(base, base)`: an identity that nothing quotes, so a Binance-Peg USDC depeg trips no halt. The manifest called that lane "owner pinned" with no owner record behind it.

```solidity
// dex-evm/script/OracleV4Deploy.s.sol:130
  /// @dev P1-RISK-1 (owner, 2026-09-11): production halts on a REAL USDC-USD depeg. A USD unit equal
  ///      to the base makes that lane keccak(base, base), an identity no signer prices, so the halt
  ///      it exists for can never fire. Refused on a production chain.
  function _usdUnit() internal view returns (address u) {
    u = _tokenOf("USD");
    require(
      !_isProductionChain() || u != _tokenOf(_baseSym()),
      "USD unit == base on a production chain: the depeg lane is an inert identity (P1-RISK-1)"
    );
  }
```

#### Impact

Under the flat equity floor, quoted dispersion sat inside the measured tail on all ten names, which transfers value from liquidity providers to informed flow on tail moves. On BNB, the id mismatch would have staled every genesis lane at listing, and the identity USD lane would have left a real stablecoin depeg with no halt.

#### Remediation

Per-name dispersion floors were written on chain on 2026-09-04 and verified on all ten names; three legs remain deliberately clamped short of the measured tail. BNB genesis now binds MITCH ids, with the deploy record and the push-side manifest asserted equal by `OracleDeployScripts.t.sol`, and the base depeg lane is a real signed USDC-USD lane halting at 5 percent per `PoolConstantsLib.sol:94`. The USD unit is refused outright when it equals the base on a production chain.

#### Status

Fixed. Per-name equity floors were applied on chain 2026-09-04; the BNB bindings landed with the V5 genesis. The residual blanket `optional=true` on the push-side feed list and the stale "inert" wording in the BNB manifest are tracked as a separate open row. One duplicate informational row on the same equity floors was closed under the informational purge of 2026-09-09.

**Rows.** 4 rows.

### F-50  The SDK build-time ABI integrity check was a tautology and fell back to stale artifacts

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `sdk/scripts/fetch-abis.ts:84`, `sdk/src/abis/fetch.ts:7-19`, `sdk/src/cache.ts:38-53` |

**Severity rationale.** The check that was meant to authenticate a fetched ABI derived its expected value from the fetched payload itself, so a hostile response passed; reaching it requires control of the serving API, which is a trusted tier.

#### Description

`fetch-abis.ts` pinned each ABI by function names plus one selector. The selector comparison recomputed the selector from the signature string it had just read out of the fetched payload, so both sides of the comparison came from the same untrusted input and any payload that named the right functions passed.

```ts
// sdk/scripts/fetch-abis.ts:84
  for (const [sig, want] of Object.entries(target.pins)) {
    const e = fns.find((f) => sigOf(f) === sig);
    if (!e) throw new Error(`integrity: ${target.name} ABI missing pinned ${sig}`);
    const got = selectorOf(sig);
    if (got !== want) throw new Error(`integrity: ${target.name} ${sig} -> ${got}, want ${want}`);
  }
}
```

The same script fell back to keeping the existing on-disk ABI and exiting zero when the fetch failed, with only a warning, so offline or pinned-reference container builds could ship a silently stale ABI. At runtime, `fetchAbi` and `fetchVenues` applied no integrity check at all and cached results in `localStorage` with no version key, so a poisoned entry persisted across a version bump. The oracle ABI export was documented as a V2 alias when it was the V4 ABI plus V2-only entries, and a V1 ABI shipped beside a V4 fleet.

#### Impact

A build-time or runtime ABI substitution would not have been refused by the integrity check. Value-moving calls encode against static ABIs and a mis-encode fails closed, so the realistic outcome is a stale or wrong read surface rather than a wrong transfer.

#### Remediation

The ABI pin is now a content hash over normalised entries, recorded in `abis.lock.json`, and `fetch-abis` fails closed on a mismatch or a missing pin rather than keeping a stale artifact. The cold ABI cache and the runtime venue fetch were deleted and pinned ABIs are served locally. The V2 alias was dropped from the docs, leaving `EXTERNAL_ORACLE_V4_ABI` as the only oracle export.

[`3a03ab7`](https://github.com/btr-protocol/sdk/commit/3a03ab78ad4bb3b997faa51599017364db382389), [`7298d9c`](https://github.com/btr-protocol/sdk/commit/7298d9c64b57ff1380d6ed7aa5e5fdaf63d66244), `d930595`

#### Status

Fixed, verified 2026-09-10 and 2026-09-11. Covered by `abi-pin.test.ts`.

**Rows.** 5 rows.

### F-51  SDK transaction encoding and nonce allocation produced unsendable or permanently gapped transactions

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `sdk/src/eth/rlp.ts:42-44`, `sdk/src/eth/client.ts:113`, `sdk/src/eth/abi.ts:335-338` |

**Severity rationale.** The RLP defect corrupts the signing preimage of any EIP-1559 transaction carrying an access list, and the nonce defect can stall a signer permanently; both are liveness failures with no path to an incorrect transfer.

#### Description

`encodeRlp` returned the empty-string prefix `0x80` for a zero-length input, and the EIP-1559 encoder passed the empty access list through that path. RLP requires the empty-list prefix `0xc0`. The signing preimage was therefore wrong and the signed transaction was rejected.

```ts
// sdk/src/eth/rlp.ts:42
  // Empty string
  if (bytes.length === 0) {
    return new Uint8Array([0x80]);
  }

  // Single byte < 0x80
  if (bytes.length === 1 && bytes[0] < 0x80) {
    return bytes;
```

Nonce handling had two successive defects. `getTransactionCount` sat behind a request deduper, so concurrent `signTransaction` calls in the same tick shared one result and reused a nonce, producing a replacement or a dropped transaction. The fix then introduced a regression: the allocator recorded `lastIssued` at allocation rather than at acceptance, so a single failed `estimateGas` or send left a nonce gap that never healed for the lifetime of the process. Alongside these, the client trusted the chain id the RPC reported rather than checking it against an expected value, `getPlan` resolved ABI entries first-name-wins with no overload dispatch, `signTypedData` crashed on a bigint chain id because `JSON.stringify` throws on bigints, `withDecodedRevert` replaced a typed revert error with a `SyntaxError` on truncated revert data, and the contract read and write helpers never surfaced decoded revert data at all.

#### Impact

Transactions carrying a non-empty access list were rejected outright. A signer that hit one send failure stopped being able to send at all until restarted. The missing chain guard leaves a signed payload's chain binding dependent on whatever endpoint answered.

#### Remediation

The RLP encoder emits `0xc0` for an empty access list, with transaction vectors added. Same-tick nonce reuse was removed and the allocator now releases a nonce on a failed send, so a failure no longer leaves a gap. `signTransaction` refuses an endpoint chain id that differs from the expected one and a `tx.chainId` mismatch at the preimage, and the healthy-RPC selector fails closed. `getPlan` refuses an ambiguous overload unless the full signature is named, `signTypedData` uses a bigint-safe JSON replacer, and `withDecodedRevert` keeps the typed revert error on clipped data.

[`3a03ab7`](https://github.com/btr-protocol/sdk/commit/3a03ab78ad4bb3b997faa51599017364db382389), [`84bc248`](https://github.com/btr-protocol/sdk/commit/84bc2485ebf54a0d60560d8cd7823264e5c348c7), [`d378c53`](https://github.com/btr-protocol/sdk/commit/d378c5311325f1b41d85229b19023497b54ae9c0), [`62d05b9`](https://github.com/btr-protocol/sdk/commit/62d05b988125ddea60430818f74ed874354e20d6)

#### Status

Fixed, verified 2026-09-10 and 2026-09-11. Covered by `tx-vectors.test.ts:223` and `correctness.test.ts:319`. Four informational rows were closed under the purge of 2026-09-10 rather than fixed: undecoded revert data on the generic contract helpers, multicall split-batch tearing against a stale RPC, a float-based haircut preview that is display-only, and the absent `getCode` check on the canonical multicall address.

**Rows.** 11 rows.

### F-52  The oracle page presented a governance push band as extractable value

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `front/src/components/features/oracle/FeedsTable.tsx:45-46`, `front/src/components/features/oracle/useOracleData.ts:244-245`, `front/src/components/features/oracle/oracleMarks.ts` |

**Severity rationale.** The figure was displayed to operators as live extractable value while being a governance parameter product, overstating it by 25 to 50 times; no contract reads it, so the impact is decision quality rather than funds.

#### Description

The feeds table computed a per-feed figure from `maxDevBps`, the class-pinned `sigmaPbps` and the feed TTL, and labelled it "OEV". That product is the widest band a push may legally carry, a governance ceiling, not value anyone extracted. It read 75 to 274 bp of TVL per feed against measured live deviations of 0.0 to 5.5 bp, and it is static by construction because sigma is pinned per asset class.

```ts
// front/src/components/features/oracle/FeedsTable.tsx:45
/** Widest band the next accepted push can carry, evaluated at the feed's own TTL (the widest
 *  legal source gap) so the OEV figure is a ceiling rather than a snapshot. */
const oevBandBps = (row: FeedRow): number =>
  pushBandBps(row.maxDevBps, row.sigmaPbps, row.ttl);
```

A second defect in the same area was a literal NUL byte committed as a memo separator inside a string literal in the feed-gate helper. Git classified the entire 2169-line file as binary, so it produced no diff in review, no three-way merge, and `git show` and `grep` returned nothing for it. The merge had to be resolved by hand and the defect was invisible to every prior review of that file.

#### Impact

Operators read a static governance ceiling as a live loss figure. The NUL byte removed a whole file from code review and from text tooling for as long as it was present.

#### Remediation

The oracle page now distinguishes the governance push band from measured live deviation. The NUL byte was replaced by the equivalent backslash-u escape, which keeps the file text and restores diffs, merges and grep.

[`9b21e60c`](https://github.com/btr-protocol/front/commit/9b21e60cbc478607086764a039bc8df87d8b5f5e), [`e4033a91`](https://github.com/btr-protocol/front/commit/e4033a91db61fae00c8ff5b4a76614b58f9ff24a)

#### Status

Fixed. The oracle page change is on the `audit/oev-live` branch and is not yet merged. Two informational rows were closed under the purge of 2026-09-10: stale marks are unfiltered in the portfolio revaluation path, which is display-only since the swap gate filters them, and the gate feed row drops a source timestamp that is currently redundant because the clocks are equal under V4.

**Rows.** 4 rows.

### F-53  Natspec and comments overclaimed against the shipped constants and levers

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV5.sol`, `dex-evm/src/libraries/PricingLib.sol:52`, `dex-evm/src/oracles/OracleBeacon.sol` |

**Severity rationale.** Documentation-only divergence, but on the interior swing cap and the beacon upgrade gate it describes safety properties the code does not have, which is what operators act on.

#### Description

Three comment defects shipped alongside the code they described. The interior swing cap was cited as 10000 when the shipped constant is 10862, with the interior-scoped dispersion ceiling documented against the old number, and Admin natspec carried stale 5000 and 1000 figures. The beacon layout natspec overclaimed upgrade safety, and the `adminBackfillLegs` comment understated a fail-closed brick.

The third was live rather than stale: after the preceding quarantine fix, the quarantine bit became write-only. `clearQuarantine` cleared a bit that no longer affected `_allowed`, and no getter exposed it, so the natspec claim that clearing re-arms the lane was false.

```solidity
// dex-evm/src/oracles/ExternalOracleV5.sol:586
  /// @dev TIGHTENING: clears ONLY the lane's quarantine bit, re-arming the X cap on a dark lane.
  function clearQuarantine(bytes32 feedId) external {
    requireGuardianOrOwner(AC);
    OracleStorageV5.OracleData storage $ = OracleStorageV5.get();
    (uint256 gi,) = _giCfg(feedId);
    uint256 lane = gi % LANES_PER_SLOT;
    $.riskSlot[uint32(gi / LANES_PER_SLOT)] &=
      ~bytes32(uint256(1) << (QUARANTINE_SHIFT + lane));
    emit QuarantineCleared(feedId, msg.sender);
  }
```

#### Impact

An operator following the beacon natspec would have assumed an upgrade-safety check that the gate does not perform, and one following the quarantine natspec would have believed a dead call re-armed a dark lane.

#### Remediation

The cap numbers were made symbolic or corrected to 10862 in both the contracts and the published documentation; the stale Admin natspec was deleted. The beacon layout and backfill comments were corrected. The quarantine bit and `clearQuarantine` were deleted outright rather than documented, with zero remaining references across the oracle sources, the interface and the client packages.

[`235d6b2d`](https://github.com/btr-protocol/dex-evm/commit/235d6b2d626a7cba945b8cb32a2ba009adbc8031), [`f9e19361`](https://github.com/btr-protocol/dex-evm/commit/f9e19361204aed958ea52404179262202784681f), [`c42848dd`](https://github.com/btr-protocol/dex-evm/commit/c42848dd4fe7f5fab7c82dcfbbc11040d345abd8), [`f039e1e7`](https://github.com/btr-protocol/dex-evm/commit/f039e1e73318021111ab3b26f3444ea7eda9a164), [`3d215a2c`](https://github.com/btr-protocol/dex-evm/commit/3d215a2c213fdeaa2994f255f9901145dae1cdc4), [`99eef0f6`](https://github.com/btr-protocol/dex-evm/commit/99eef0f628b8689e3712151a86844ae13edbe63f), [`761a7f07`](https://github.com/btr-protocol/dex-evm/commit/761a7f07f938ccba6f2970ea4fdf150649fc68ca), [`b58f78d`](https://github.com/btr-protocol/content/commit/b58f78d7ddf7134d88d76618b82876dc1a9d1db2), [`3e4834e`](https://github.com/btr-protocol/content/commit/3e4834e04ab42080952676fc3613ec09538ccf20)

#### Status

Fixed and closed, verified at the rev2 signoff. One stale natspec residual left by the quarantine deletion is tracked as a separate row.

**Rows.** 3 rows.

### F-54  The V5 sigma floor was one-way and was not applied on read, so releasing a dark lane left a permanent wide band

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV5.sol:450`, `dex-evm/src/oracles/ExternalOracleV5.sol:436` |

**Severity rationale.** Releasing a dark lane is a routine operation and the only mechanism available raised a floor that could never be lowered, leaving that lane permanently accepting pushes up to ten times its configured maximum deviation.

#### Description

The only way to release a dark lane was to raise `sigmaFloor`, and `sigmaFloor` had no downward path. The released lane therefore kept a per-push band of ten times `maxDev` for the rest of its life. The self-heal path ignored the sigma carried by the blob it had just refused, so the floor could not converge back on the observed volatility.

```solidity
// dex-evm/src/oracles/ExternalOracleV5.sol:344
    uint256 sigmaStored = (rw >> (c.lane * RISK_STRIDE)) & RISK_SIGMA_MASK;
    uint256 sigmaFloor = uint32(c.scfg >> SCFG_SIGMA_FLOOR_SHIFT);
    uint256 sigma = sigmaStored > sigmaFloor ? sigmaStored : sigmaFloor;
    uint256 newMark = _decodeLane(c.nl);
    (uint256 devBps, uint256 allowed) =
      FeedMathLib.bandVals(c.prevMark, newMark, dt, uint16(c.scfg), uint32(sigma));
    if (devBps <= allowed) return true;
```

Separately, `PoolIOLib` natspec claimed a relay-only, one-push-per-block property that `push()` does not enforce.

#### Impact

Any lane that had been released once accepted pushes inside a band an order of magnitude wider than its configured deviation cap, permanently, which is the band that bounds how far a single accepted push can move the mark a pool prices against.

#### Remediation

`sigmaFloor` is now the single lower bound on sigma applied on every read, and it is adjustable in both directions under the timelock, so a released lane can be tightened again.

[`a8c5463`](https://github.com/btr-protocol/dex-evm/commit/a8c54633d6f3d81221f827f9b08850613b39652c), [`ac57e66`](https://github.com/btr-protocol/dex-evm/commit/ac57e66a66e914182a73967ddef9d267e3e969dc), [`00d7abc`](https://github.com/btr-protocol/dex-evm/commit/00d7abce6ed5a03453fc557aa33a86678bf98a54)

#### Status

Fixed 2026-09-16. The absence of a `ttlSecs` ceiling below the `uint16` maximum of 18.2 h on `registerFeed` and `requestFeedWiden` was accepted on 2026-09-16 with no code change. The `PoolIOLib` natspec residual is tracked in a separate open row.

**Rows.** 2 rows.

### F-55  Published PoolFactory ABIs diverged from the contract, so consumers encoded selectors that do not exist

| Severity | Status | Class | Component |
|---|---|---|---|
| MEDIUM | Fixed | Audit | `dex-evm/src/PoolFactory.sol`, `sdk/src/abis` |

**Severity rationale.** A consumer encoding a removed selector produces a call that reverts rather than one that succeeds incorrectly, but the divergence covered the official-pool grant path, which is a routing credential.

#### Description

The pinned consumer ABIs lagged the contract. `executeOfficial` and `cancelOfficial` existed on `PoolFactory` but were missing from the published ABI, while the removed `setProtocolDeployer` was still exported. Consumers therefore had no way to encode the two live calls and a way to encode one that no longer exists.

```solidity
// dex-evm/src/PoolFactory.sol:326
  function executeOfficial(address pool) external override onlyAdmin {
    uint256 eta = pendingOfficial[pool];
    if (eta == 0) revert Err.NoPending();
    if (block.timestamp < eta) revert Err.NotReady();
    if (block.timestamp > eta + SC.GRACE_PERIOD_SECS) revert Err.Expired();
    delete pendingOfficial[pool];
    // The shape is re-asserted HERE, not only at request: the delay is exactly the window in which
    // the pool's sink / seal / listing can move, and branding is a routing credential.
    _grantOfficial(pool);
  }
```

#### Impact

The queued official-pool grant could not be executed or cancelled through a published ABI, and a consumer could construct a call to a selector the contract no longer implements.

#### Remediation

ABIs are now generated in `dex-evm` and consumed by path rather than hand-copied, `IOracle.json` is published, and the hashes are locked and recomputed equal across consumers. No CI parity gate exists yet for this surface.

[`f8ecb4ce`](https://github.com/btr-protocol/dex-evm/commit/f8ecb4ce9e194abcf5c484e58dea89a0c795b162), [`9d44a18`](https://github.com/btr-protocol/sdk/commit/9d44a18c6bbd3528688333c33f133105a57deb24), [`19260a7`](https://github.com/btr-protocol/sdk/commit/19260a79471283e5c71b4f87c52db533a13206e5)

#### Status

Fixed and closed.

**Rows.** 1 row.

### F-56  Session grants, sequence numbers and signer-set changes did not bound what their documentation claimed

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/src/oracles/ExternalOracleV4.sol:292-293`, `dex-evm/src/oracles/ExternalOracleV4.sol:334-341`, `dex-evm/src/oracles/NxrSignerSet.sol:159` |

**Severity rationale.** Every item here is liveness or documentation accuracy at the shipping configuration; the strongest, a public push at a frozen mark, is unprofitable against the live gate.

#### Description

Session `seq` was bounded by `maxSeq` only and never stored, so the number of pushes in a session was effectively unbounded. The fix stored a session-global `lastSeq` and required strict monotonicity, which then rejected honest pushes: on the shipped wire `seq` is the minimum observed value across the blob, not a monotone counter, and the chain orders by transaction, not by that field. The stored gate bounded nothing it claimed to bound while costing honest pushes, and the accompanying natspec asserted a property that does not hold at the shipped grant shape. Session and bias nonces were checked `uint16` additions, so the arithmetic reverts permanently at 65535.

An open session is in-band mark authorship for the granted relay address: `pushV4` verifies no signature, only the sender, the expiry and `seq <= maxSeq`, bounded thereafter by the per-lane deviation band, monotonic replay and the reference band. Three operator conclusions stated a relay key could not author a mark. The durable cutoff is `revokeSigner` below threshold, which makes every future `openSession` unsatisfiable, not `revokeSession` alone. Separately, a permissionless push against a frozen mark is reachable on the V1 oracle path, and a guardian mass revoke followed by serial single-slot re-grants produces a fleet-wide push outage of N times the BASE delay.

#### Impact

Liveness and operator accuracy. The rejected-honest-push regressions stall feeds rather than exposing value, the nonce overflow is a permanent denial of a governance path at a bound no deployment reaches, and the documentation defects would misdirect an on-call responder holding a leaked relay key.

#### Exploit scenario

1. An address with an open session observes a lane whose mark is frozen.
2. It lands an in-band push in the same block as its own trade, at up to the band's gross allowance.
3. The deviation available is bounded by theta plus drift while the gate charges a spread of at least two theta, so the round trip is net negative at the live configuration. The primitive is latent only on wide-band legs.

#### Remediation

The session sequence gate is reworked so it no longer rejects honest same-second or older-source pushes, and the natspec is corrected to state what it actually bounds. Nonce arithmetic no longer reverts at the `uint16` bound. The guardian and keeper operations pages state that an open session is in-band mark authority and that `revokeSession` is not a durable cutoff. The mark-push client dual-sends privately and never falls back to a public rescue push. V5 grants a batch of signers under one BASE operation and holds the revoke floor at the threshold.

[`502feb1`](https://github.com/btr-protocol/dex-evm/commit/502feb1df5277235521f6af1ee3098691dd884ec), [`d930595`](https://github.com/btr-protocol/content/commit/d9305957b7687a8932a52f13d67f2d49b3b67c87)

#### Status

Fixed, verified in the following round. The documentation rows are verified 2026-09-11. The V5 signer-set changes retire the V4 residual at the repoint.

**Rows.** 12 rows.

### F-57  Oracle natspec, interfaces and rollout notes drifted from the shipped code

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | Audit | `dex-evm/src/oracles/ExternalOracleV4.sol:80-95`, `dex-evm/src/interfaces/IExternalOracleV4.sol:78-79`, `dex-evm/src/oracles/ExternalOracle.sol:349` |

**Severity rationale.** Documentation and code-hygiene defects with no reachable on-chain consequence; the operational rows would have cost time during a rollout, not value.

#### Description

The rollout note added in an earlier round instructed a `grantSigner` call that does not exist and prescribed a signer step that does not match the shipped surface. The same note's storage-layout argument was moot, since the oracle is not behind a proxy. The packing natspec for the new public pending-widen getter omitted a field. A natspec trim dropped the only record of the oracle salt re-mine ceremony rule. "Shipping in the next release" was applied inconsistently to levers already present in `src` at head. The source tree still carried four full oracle generations with the signer-governance surface duplicated across them. Three further rows are interface ergonomics: the canonical interface declares no signer functions, an `updateFeed` comment was stale after the widen lever began writing the same word, and the view surface omits getters for the band anchor, lane configuration and exponent bias, all of which are derivable from shipped events and public mappings.

#### Impact

None on chain. An integrator or an incident responder reading the interface or the rollout note would have to reconstruct the real behaviour from the implementation.

#### Remediation

The rollout note is corrected against the shipped functions, the packing natspec is completed, and the salt re-mine ceremony rule is recorded in the oracles chapter of the documentation. The "next release" language is removed from the published content. `src/oracles` now holds only the current oracle and the signer set; V1 is deleted.

[`46622d7`](https://github.com/btr-protocol/dex-evm/commit/46622d71991e78b3c1e961c5f5cbf2db166c2f5f), [`d930595`](https://github.com/btr-protocol/content/commit/d9305957b7687a8932a52f13d67f2d49b3b67c87), [`e99fe79`](https://github.com/btr-protocol/content/commit/e99fe7943d77c4e6e4e411139b4f5cb64281dc3e)

#### Status

Fixed for the code and content rows. Three informational rows are Closed with no change, below the low-severity minimum bar, 2026-09-10.

**Rows.** 10 rows.

### F-58  Deploy ceremonies signed unbound salts and zeroed risk parameters

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | Audit | `dex-evm/script/lib/Create3Base.sol:37-50`, `dex-evm/script/lib/OracleV4Base.sol:27-41`, `dex-evm/script/ArcRiskRestore.s.sol:806-849` |

**Severity rationale.** Both defects require a signer to execute a prepared ceremony; neither is reachable by a third party, and the risk-restore case emits a visible warning.

#### Description

`_create3` read the salt and the expected address from the same role row, and the salt assertion was a no-op everywhere except the router deployment. A wrong role therefore self-agrees and can squat the address another role had reserved; the code's own natspec named the hazard. Separately, the risk-restore preview emitted blobs with `minLiquidity` and flags at zero, with a warning only, so signing the preview verbatim would restore those zeroes.

#### Impact

A misconfigured ceremony could consume a reserved deterministic address, or restore a pool configuration with no minimum liquidity, in both cases from a payload that was presented as ready to sign.

#### Remediation

The salt is bound in every live ceremony, with a role re-derivation in the oracle deployments and a preimage pin in the router deployment, held by a deterministic-deployment fleet test. The risk-restore preview now signs the live `minLiquidity`.

[`076eb79`](https://github.com/btr-protocol/dex-evm/commit/076eb798ee230f41d3e25b275f6a20ef438f21b5)

#### Status

Closed on the second revision, signed off 2026-09-14; the preview fix landed 2026-09-15. The remaining ordered upgrade and re-seed is tracked as the next ceremony step. Two legacy scripts still inherit the no-op salt assertion; they are unreachable under the current deployment shape and are scheduled for deletion.

**Rows.** 2 rows.

### F-59  An owner unhalt could relist a re-anchored leg without re-attestation

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | Audit | `dex-evm/src/libraries/PoolConfigLib.sol`, `dex-evm/src/Admin.sol:299-303` |

**Severity rationale.** Requires the owner role and a leg that has already collapsed and been re-anchored, but the unhalt silently clears a latch that exists to force a fresh attestation.

#### Description

The collapse halt reused the guardian halt bit. Because `unhaltAsset`, the batch risk operation and the fleet-wide unhalt all clear that bit, an owner unhalt relisted a leg whose anchor had been changed with no re-attestation of the new anchor. The access model's non-composability rule for this case was not expressed in code.

#### Impact

A leg could return to quoting against a re-anchored configuration that no one attested after the re-anchor.

#### Remediation

The collapse halt gets its own anchor-owned latch on chain, mirrored in the SDK and in the pricing core, and the interface sends only the settable halt mask on unhalt. Pinned by a halt-source test.

[`dacde55`](https://github.com/btr-protocol/dex-evm/commit/dacde55aa053f1c6ede67805f273f006840aaf2e), [`cd8d703`](https://github.com/btr-protocol/sdk/commit/cd8d7034a024e4618f7c39b48883e29560dbaf10), `core@6823437`, [`ea7150f`](https://github.com/btr-protocol/front/commit/ea7150fb9b97259a026b47dd8f56539ac13576bd)

#### Status

Closed, 2026-09-15.

**Rows.** 1 row.

### F-60  Router output floors were measured at the wrong place and quote views omitted execution guards

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | QA | `dex-evm/src/Router.sol:190-195`, `dex-evm/src/Pool.sol:285`, `dex-evm/src/LPToken.sol:154` |

**Severity rationale.** All of these need an owner-listed fee-on-transfer token or produce a revert rather than a loss, and none is live at the shipping configuration.

#### Description

The router floored on the amount it gained rather than on the amount the recipient received, so a fee-on-transfer output leg passed the check while the recipient netted less. Two rows recorded the same accounting distinction from different angles, and a third recorded the mirror on the input leg: the router approves and pulls the nominal amount, so a fee-on-transfer input reverts at the pool pull, fail closed and self-inflicted.

`getSwapQuote` omitted the execution-only guards, so it returned a non-zero quote on a path that would revert at execution: the swap-enabled bit, the pre-outflow liquidity check and the reference band are all checked in the swap and not in the view. `getAsset` returned a zero struct for an unlisted asset rather than reverting not-found, which reads as maximum coverage on a zero liability.

On the share token, a burn never decremented the frozen amount, so a partial burn left shares over-locked for up to the maximum cooldown, and raising the flow cooldown applied retroactively to positions already held. A delayed just-in-time liquidity argument against the same cooldown was analysed and closed: the profit was not demonstrated once gas and inventory seasoning are counted.

#### Impact

Bounded accounting differences and reverting views. No theft path, and the fee-on-transfer rows require the owner to list such a token first.

#### Remediation

The router measures the recipient's own balance delta and floors on the amount delivered rather than the amount gained, with a regression test pinning both rows. The quote-view guard omissions are documented. The remaining rows were closed as informational under the minimum-severity bar.

Commits: [`e36dbb86`](https://github.com/btr-protocol/dex-evm/commit/e36dbb86c4e0d36a0b57893a168f9875bfb9aba1), [`0e323816`](https://github.com/btr-protocol/dex-evm/commit/0e323816c0ebc5195b38cfcb0b016c7728700a02), [`89b058f5`](https://github.com/btr-protocol/dex-evm/commit/89b058f50e307653b2769a29d3b4110bdb4441fa).

#### Status

Fixed for the router floor and the quote-view documentation, verified 2026-09-10. The informational rows were closed on 2026-09-09 and 2026-09-10 under the minimum-severity bar.

**Rows.** 10 rows.

### F-61  A hook that refused recall could not be replaced, and a halted leg blocked the keeper's evacuation

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | QA | `dex-evm/src/libraries/PoolConfigLib.sol:764-772`, `dex-evm/src/hooks/YieldHook.sol:108-117`, `dex-evm/src/libraries/PoolConfigLib.sol:801` |

**Severity rationale.** Liveness only and conditioned on an already-compromised or failing hook with a non-zero invested balance; no extraction path.

#### Description

Replacing or clearing an asset hook was locked while the invested balance was non-zero, so a hook that refused recall left the operator with neither lever and blocked outflows on that leg. The rebalance path reverted on a halted or over-cap leg, so the keeper could neither trim nor evacuate a venue during exactly the incident that made evacuation necessary.

Two accounting and binding defects sat alongside. The hook credit rate bucket capped a token amount against the face book, so the effective daily cap was loose by a factor of the coverage ratio whenever it was not one. `setAssetHook` did not bind the hook target to the pool and access-control surface, so a hook constructed against a sentinel was accepted on the wrong pool.

#### Impact

A failing hook could hold a leg's outflows hostage until an operator intervention that did not exist, and the daily credit cap did not bind as specified off parity.

#### Remediation

An `adminForceClearHook` lane clears a hook that refuses recall, with the served and packaged ABIs re-pinned to carry it. Evacuation stays live on a halted leg. The hook credit cap and the LP fee are booked at the coverage ratio, and the whole hook push is booked with only the face capped. `setAssetHook` checks `pool()` and `AC()` on the target.

Commits: [`42efec95`](https://github.com/btr-protocol/dex-evm/commit/42efec95888ec05a9d860a17068bc323a3a3d968), [`45f88d84`](https://github.com/btr-protocol/dex-evm/commit/45f88d8464d71e4d15d122657cb22ad01f793550), [`eaf555ff`](https://github.com/btr-protocol/dex-evm/commit/eaf555ff348a60c08d1bf124b1b350d8eb04426f), [`9e00d331`](https://github.com/btr-protocol/dex-evm/commit/9e00d33194adff4fd7bd003ed4a92e75af76c53d), [`58cdad56`](https://github.com/btr-protocol/dex-evm/commit/58cdad56f1f20c8958c4946e9ca234d3421ed4a4), [`99bc688`](https://github.com/btr-protocol/sdk/commit/99bc688fb9fd3bf5e30c675f6a9a604b4416b9d8).

#### Status

Closed, with one residual carried into the deployment plan as ceremony step C-9: the venue adapter's virtual-balance read exists only on the newer lending-pool revision, so the target chain's fork must be confirmed to expose it before the first hook install. Landed 2026-09-15.

**Rows.** 4 rows.

### F-62  The guardian arming runbook named files and secrets that no longer exist

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | QA | `shared/evm/src/access/AccessControl.sol:278-280`, `dex-evm/src/Admin.sol:370-408`, `dex-evm/src/oracles/ExternalOracle.sol:334-336` |

**Severity rationale.** Documentation drift on a dormant role, not an availability gap: the owner authority is a strict superset of the guardian authority on every named lever, so the freeze is one transaction today.

#### Description

The single granted guardian address has never transacted and the guardian quorum policy is not armed. The arming procedure could not be executed as written: its first step invokes a deployment target whose values file was deleted, its second names a secret that does not exist, an alerting page cites a runbook file that does not exist, and a deployment comment contradicts a flag the tool actually accepts.

The harm claim that every guardian lever is therefore owner-only was checked against the authority code and does not hold: `AccessControl.isGuardianOrAuth` is a disjunction, and every named lever routes through it with the owner as the alternate authority. The halt, unhalt, batch risk, timelock cancel, session revoke, rebias and feed pause paths are all reachable by the owner without a timelock today. What is genuinely missing is role separation and automation, both already tracked separately.

#### Impact

An operator following the runbook during an incident would stop at a missing file. The freeze itself remains available through the owner path.

#### Remediation

No code change. Arming at least two guardian Safes is deployment plan step C-4, a fresh disjoint signer roster is step C-3, and the record sync is step C-2.

#### Status

Closed 2026-09-15 into the deployment ceremony steps C-2, C-3 and C-4.

**Rows.** 1 row.

### F-63  Governance lever scope: a steward raise voided unrelated owner operations and a sentinel-pool guardian halt carried no release clock

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/src/Admin.sol:552`, `dex-evm/src/Admin.sol:330`, `dex-evm/src/oracles/OracleBeacon.sol:88` |

**Severity rationale.** Each item is a bounded authority-scope defect on a lever that already requires a privileged role; the worst case is a repeatable denial of a governance operation whose magnitude the owner had already ratified.

#### Description

`raiseKappa` dropped any live owner `UPDATE_RISK` operation on the same leg, whether or not the queued operation touched the field the steward was raising.

```solidity
// dex-evm/src/Admin.sol:558
  function raiseKappa(address pool, address token, uint16 kappaCovBps) external {
    _onlySteward(pool, true);
    bytes32 key = _keyToken(pool, OP_UPDATE_RISK, token);
    if (_live(key)) _drop(key, pool, uint8(IPool.OpType.UPDATE_RISK));
    AdminParams.raiseKappa(pool, token, kappaCovBps);
  }
```

A guardian halt on a sentinel pool was not stamped, so a seat taking the pool later lifted it with no release delay. The reference-tier access control root also stayed deployer-owned with no handover step.

#### Impact

A steward key could repeatedly cancel an owner-ratified risk operation, and a guardian halt could be cleared immediately once a seat existed, removing the delay that halt is supposed to buy.

#### Remediation

`raiseKappa` voids only a conflicting operation. Guardian halts are stamped and carry the release delay. The handover step and an owner assertion were added, so every control root requires a disjoint two-party owner.

Commits: [`41e10bc`](https://github.com/btr-protocol/dex-evm/commit/41e10bcbbc1ae3202d3ef9c23aeb742d51722842), [`56020da`](https://github.com/btr-protocol/dex-evm/commit/56020da059415019e8420d3cfdeed55406a54dfd), [`0e0cd78`](https://github.com/btr-protocol/dex-evm/commit/0e0cd78f06495d715b2bc55b18a02bdc3e5c220b), [`957d3b0`](https://github.com/btr-protocol/shared/commit/957d3b0939a6a3ff156a8cd30eef3b594a70760f).

#### Status

Fixed 2026-09-16 for the two code rows. Five rows are accepted with no code change, recorded 2026-09-16: the oracle beacon upgrade sits at the LISTING tier with a nominal validation and one owner key reaching both beacons; a foreign-pool seat can widen its own fences and write inside them in one transaction, which will be stated in the LP documentation; `revokeSigner` refuses to go below the signing threshold, so at exactly the threshold the leak response is a per-feed pause; the guardian can seal a sentinel pool mid-ceremony; and `_tier` has no `UPDATE_ASSET_PARAMS` arm, so a request with that operation type reverts.

**Rows.** 7 rows.

### F-64  Yield adapters trusted venue-reported amounts and left owner levers untimelocked

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/src/hooks/ERC4626YieldHook.sol:43`, `dex-evm/src/hooks/YieldHook.sol:122-134`, `dex-evm/src/hooks/MorphoBlueYieldHook.sol:27` |

**Severity rationale.** No hook is installed on a live pool and installation is itself privileged, so every item here requires a privileged action or a misbehaving venue before it has any effect.

#### Description

`_venueWithdraw` returned the requested amount rather than the measured token delta on both the ERC-4626 and the Compound V2 paths, so a venue with a withdrawal fee or a lossy share price caused the ledger to over-credit and `hookRecall` to over-decrement.

```solidity
// dex-evm/src/hooks/ERC4626YieldHook.sol:43
  function _venueWithdraw(uint256 assets) internal override returns (uint256 got) {
    uint256 before = _tokenBalance(address(this));
    vault.withdraw(assets, address(this), address(this));
    got = _tokenBalance(address(this)) - before;
  }
```

Harvest credited a donation-inflated NAV with no deposit check. On the Morpho adapter, loan-token rewards were never sweepable because the token and the position were both skipped with no override, and bare hook cash was not counted in NAV, so it was a permanent strand even for the owner; the adapter also held an unbounded venue allowance with no revoker, unlike its siblings which approve exactly and then zero. The owner's `setBuffer` and `forceWriteDown` were instant with no hook timelock, although installing a hook is itself a high-tier operation, which is inconsistent tiering. One commit at the head had been flagged unreviewed by its author and had stripped the ERC-4626 deposit-loss invariant.

#### Impact

A lossy or fee-charging venue over-credited the hook ledger, and a donation-inflated NAV booked yield that did not exist. Stranded loan-token rewards were unrecoverable. The untimelocked owner levers allow an arbitrary LP haircut, which is griefing rather than extraction since it pays the owner nothing.

#### Remediation

Both adapters return the measured delta. Harvest realizes the measured venue delta before crediting, and the credit path proves the balance covers liabilities, protocol fees and the credited amount, so a donation-inflated NAV cannot book phantom claim. The Morpho adapter counts idle loan tokens in NAV and pushes hook idle before the venue on recall, and approves exactly then zeroes. Risk-increasing hook levers are timelocked at the risk-up delay with de-risking instant and a guardian cancel. The deposit-loss invariant was restored.

Commits: [`3565c671`](https://github.com/btr-protocol/dex-evm/commit/3565c6718023eb399e6f35b2520afae444078482), [`c9761fbd`](https://github.com/btr-protocol/dex-evm/commit/c9761fbdcdaf6c8d4c8e5c55017030200e8485d1), [`ea976690`](https://github.com/btr-protocol/dex-evm/commit/ea9766902eaf8fec16bfbd416367657d2ac95052), [`ad55c693`](https://github.com/btr-protocol/dex-evm/commit/ad55c693b64e1abc28d9824cbeb3330a237609f4).

#### Status

Fixed, verified 2026-09-10 and 2026-09-11.

**Rows.** 6 rows.

### F-65  Hook and periphery ledgers: force-clear ran inside flash context, rounding dust became a permanent index cut, and over-WAD claim weights stranded the last claimants

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/src/Pool.sol:756`, `dex-evm/src/hooks/YieldHook.sol:393`, `dex-evm/src/periphery/WombexClaim.sol:141` |

**Severity rationale.** Each needs a privileged or adversarially timed call, and the periphery case needs a malformed Merkle root at publication time.

#### Description

`adminForceClearHook` lacked `requireNoFlash`, so a write-down could be sized against a coverage value observed mid-loan rather than at rest, and the `hookCreditYield` natspec contradicted the code about where the over-cap slice lands. `_harvest` booked any NAV below book as a loss, so a one-wei share-rounding difference wedged rebalance on an unrelated stale feed and turned a transient NAV dip into a permanent index cut. In the periphery claim contract, Merkle weights summing above WAD hard-reverted for the final claimants against an immutable root with no rescue path.

#### Impact

A write-down sized at a mid-loan coverage is not the write-down the operator intended. Rounding dust blocking rebalance is a liveness problem on a healthy leg. The claim case permanently disables the last claimants once the root is published.

#### Remediation

Force-clear refuses flash context. Venue rounding is absorbed instead of reverting. Claim weights are bounded so the final claimants always settle.

Commits: [`5188023`](https://github.com/btr-protocol/dex-evm/commit/51880239d7b76bafb5535c547e9a6ab8c17a4012), [`df0367d`](https://github.com/btr-protocol/dex-evm/commit/df0367db996532cb848f63db8492145af1b651c1), [`1ff4d9f`](https://github.com/btr-protocol/dex-evm/commit/1ff4d9f4951b6b5f4f65d7cb68377a8b41324f63).

#### Status

Fixed 2026-09-16.

**Rows.** 3 rows.

### F-66  A pool de-listing left no on-chain record

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/src/PoolFactory.sol:207` |

**Severity rationale.** Observability only, with no fund path; the consequence is that an off-chain monitor cannot reconstruct when a pool left the pending set.

#### Description

A pending-official de-listing emitted nothing, so the transition was invisible to off-chain monitoring. The report also raised an unbounded chain-reads allowlist with no per-instance count.

#### Impact

A silent de-listing could not be audited from events alone.

#### Remediation

The pending-official flag is dropped inside `deregisterPool` and `PoolDeregistered` is emitted. The allowlist half is moot: no writer and no chain-reads allowlist exist, and the storage word is reserved.

Commits: [`c692b601`](https://github.com/btr-protocol/dex-evm/commit/c692b601b9ed9dc5a8aa8057374f14f749a198cb), [`3dc4c45a`](https://github.com/btr-protocol/dex-evm/commit/3dc4c45acda289f8897e13819f4d346de2af360a).

#### Status

Fixed.

**Rows.** 1 row.

### F-67  Fee-free LP flows were a toll-free substitute for a swap, and the internal depeg breaker compared the wrong pair

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | Audit | `dex-evm/src/libraries/PoolLiquidityLib.sol:414`, `dex-evm/src/libraries/PoolIOLib.sol:368`, `dex-evm/src/libraries/PoolConfigLib.sol:808` |

**Severity rationale.** Each row needs a specific configuration or a migration step to bind, and the measured gap is a fee-scale advantage rather than a drain of principal.

#### Description

LP flows settled at the oracle rate without the swap toll, so they could be used as a cheaper trading channel: exiting ahead of a predictable push, straddling the skew, converting through deposit and cross exit, or pairing a liability swap with a same-asset exit.

The internal-mode depeg breaker compared the primary feed to the reference feed, not to the peg at which the pool actually prices the leg, so it did not test what the configuration promises. `setBaseToken` re-anchored anchor-unit spokes without re-denominating them, leaving up to roughly twice the 5% band mispriced until each anchor update landed.

Three positions were accepted rather than changed. One ungateable leg, including a leg carrying surplus but no LPs, freezes deposit, donate, cross exit, liability swap and hook credit pool-wide, which is the intended fail-closed direction. Every armed spoke swap requires a fresh reference tier, making the reference fleet a second hard liveness dependency; it is alarmed off chain at half the time-to-live. Cross withdrawal and interior hops do not check the swap-enabled bit, and the spread field saturates at its 16-bit ceiling, discarding volatility and staleness premium above 6.55%.

#### Impact

Until the toll was applied, a trader could route around the swap fee through LP flows at a measurable saving. The breaker mismatch meant an internal-mode depeg could pass a check it should have failed. The base migration left a mispricing window.

#### Remediation

LP flows now carry the toll on the paths that could substitute for a swap, the depeg breaker compares against what the configuration promises, and `setBaseToken` re-denominates on migration. The accepted positions are recorded in the audit report.

Commits: [`e464872`](https://github.com/btr-protocol/dex-evm/commit/e464872820cd0a8cb0e77e55f0d2331a8bfd0350), [`a9fd0e2`](https://github.com/btr-protocol/dex-evm/commit/a9fd0e21da8cb18546679d1cb9b2f92a1776943e), [`ff2b49c`](https://github.com/btr-protocol/dex-evm/commit/ff2b49c9097e83bf1943f7458a73e12c1f92633b), [`08b9981`](https://github.com/btr-protocol/dex-evm/commit/08b9981bd80fe12ce17bbbbef71e9bc01abca4a9).

#### Status

Closed 2026-09-16. Four rows fixed, three accepted as design positions with the reference-tier dependency alarmed off chain.

**Rows.** 7 rows.

### F-68  The off-chain quote mirror could not carry feed confidence and defaulted a missing coverage wall to zero

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `sdk/src/amm/index.ts:105`, `sdk/src/amm/aimm.ts:559`, `sdk/src/amm/aimm.ts:260` |

**Severity rationale.** The mirror prices no settlement of its own, so divergence from the chain is a wrong displayed quote or a refused route rather than a wrong fill.

#### Description

The leg builder had no field for feed confidence or staleness excess, so every spoke reached the pricing service as uncertain and was refused, and a missing coverage wall defaulted to zero, which is the fail-open value.

The curve serialiser pinned the curve median at 5000 and overran the median field at 14 knots. Three producers wrote a `maxIn` field that no consumer read. Two informational divergences were recorded and closed below the reporting bar: float packing is lossy past 2^53, with relative error around 1e-7 or roughly 0.001 basis points, and the mirror's interior swing cap and dispersion cap were tighter than the values the chain accepts, rejecting legal configurations off chain in the fail-closed direction.

#### Impact

The confidence gap made the mirror refuse routes the chain would have priced. The zero coverage-wall default understated the toll in the same direction as the front-end defect. The serialiser defect produced a curve that did not round-trip at high knot counts.

#### Remediation

The coverage wall is a required input on every leg and the legacy leg-construction surface was deleted. The leg builder carries feed confidence and staleness excess to the wire, with confidence encoded as null rather than zero when absent and pinned by wire tests. The serialiser packs a real median and only interior boundaries. The write-only field was removed across the client, the mirror and the pricing service in one change.

Commits: [`61c4063`](https://github.com/btr-protocol/sdk/commit/61c4063065df0a15f4ef3866a5697cd3abd4386a), [`7af5592`](https://github.com/btr-protocol/sdk/commit/7af55929e635ca854b027ae0309aba01712d8443), [`3a03ab7`](https://github.com/btr-protocol/sdk/commit/3a03ab78ad4bb3b997faa51599017364db382389).

#### Status

Fixed on the second remediation revision; the write-only field was removed on 2026-09-11. The two informational rows were closed on 2026-09-09 and 2026-09-10 under the low-minimum bar.

**Rows.** 5 rows.

### F-69  Chained two-hop router legs were not self-directed, and the chain-56 registry named the wrong native token

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Closed | Audit | `sdk/src/router/index.ts:436`, `sdk/src/eth/tokens.ts:41`, `sdk/src/venues/deployments.generated.ts:12` |

**Severity rationale.** The chained-leg shape requires a recipient different from the sender on a two-hop route, and the registry labels are a naming defect on a chain not yet carrying a venue record.

#### Description

On a chained two-hop leg, hop one paid the recipient while hop two pulled the intermediate asset from the sender. Only the unwrap path enforced that sender and recipient are the same address, so the other chained shapes could be built with a mismatched pair and the second hop would have nothing to pull.

The chain-56 token registry labelled the wrapped native token as native BNB and labelled Binance-peg ETH as WETH. There is also no chain-56 venue record, and the deployments file is hand-maintained with no generator.

#### Impact

A chained two-hop route with a recipient other than the sender fails at the second hop. Mislabelled registry entries misidentify the native and wrapped tokens to any consumer that trusts the registry.

#### Remediation

Chained legs are self-directed. Chain-56 records name the correct native and wrapped tokens. The missing venue record is tracked as a close-out step in the deployment plan rather than a code change.

Commits: [`e454eeb`](https://github.com/btr-protocol/sdk/commit/e454eebc121ca65f41fb4bed53980f19f5d3bf07), [`1bc8d97`](https://github.com/btr-protocol/sdk/commit/1bc8d97c7dbbb05666ed04bcfbba8a0294c0c094).

#### Status

Closed 2026-09-16.

**Rows.** 3 rows.

### F-70  Display surfaces overstated what the pool would actually quote or settle

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `front/src/lib/density.ts:170`, `front/src/lib/density.ts:120`, `front/src/pages/pools/poolPosition.ts:65-78` |

**Severity rationale.** None of these rows is read by sizing or execution; each one can mislead the reader about the state of a pool while the transaction path enforces the true values.

#### Description

The chart's installed band was drawn from the impact curve and labelled as the quotable book, although it omits the coverage wall, the toll and fees. A halted tape kept showing the last density with no badge, leaving a picture several seconds stale. A live self-pair defect compared a roster symbol to a feed symbol and requested a density for an asset against itself.

The pool and portfolio surfaces carried the same class of overstatement. Position value was computed as face times mark with no haircut, a 44% gap in the observed example, while the send path uses the haircut maximum. Pool value fell back to a hard-coded reference mark when the feed was missing, inventing a value on a dead oracle. Independent polls at 10, 30 and 12 seconds could be read together and tear. Halted legs rendered at face with the manage control always available, blocked downstream by a hard gate. Deep scroll-back evicted the live edge of the candle store, recoverable by reloading. Observed density width mixed timeframe-variant and timeframe-invariant references, in a derived overlay that is disclosed, off by default and read by nothing. Stale-fit fields on the chart were unconsumed, with the production endpoint serving nothing.

#### Impact

A reader could take a displayed band, position value or pool value as the executable one. Execution is gated elsewhere in every case, so no incorrect fill follows from these rows.

#### Remediation

The chart legend now labels the impact curve as such and the tooltip states that the net book is not drawn, pinned by a display-honesty test. A gated leg threads a "Feed gated" badge into the density legend. The density key compares like symbols, so the self-pair request is gone.

Commits: [`c9b0b44a`](https://github.com/btr-protocol/front/commit/c9b0b44a45e0a770a36697f6ccfba1b6c268efce).

#### Status

Fixed for the chart honesty and gating rows, verified 2026-09-11; the self-pair defect is fixed on the safety and chart branch. The remaining rows were closed on 2026-09-10 as informational under the low-minimum bar.

**Rows.** 10 rows.

### F-71  Client transaction plumbing did not rebuild after approvals, explain every revert, or survive blocked browser storage

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `front/src/lib/txError.ts:15-53`, `front/src/lib/settings.tsx:47`, `front/src/hooks/usePoolData.ts:611-786` |

**Severity rationale.** These are friction, availability and clarity defects on the client; the preflight simulation and the chain's own reverts prevent a bad transaction from landing in every case.

#### Description

LP writes baked their deadline and built their floors once, with no rebuild after an approval landed, so a slow approval could leave an expired action to be caught by preflight. The error taxonomy covered two pool copies and five selectors, leaving the expired and stale-data reverts to surface as raw pre-prompt text. Settings and theme read and wrote browser storage unguarded at initialisation, which throws during render in a browser with storage blocked and takes the page down with no boundary.

Six smaller rows were recorded and closed below the reporting bar: allowance preview over-reported the approval need by one wei; the pool version key omitted the hub endpoint so a hub-only write could serve a toll up to eight seconds stale; the cost model was inert on the client path, with ranking done on gross rather than net; response payloads were assigned with no schema validation, so a missing metric raised a type error instead of a no-feed state; amount normalisation collapsed precision through a float round trip on the display path; and a transport failure was painted as an empty book rather than as a failure, with no refetch loop. Two consolidation rows were verified already closed: the rate-limit latch is a single shared module with no standalone copies left, and the chart primitive attach and detach boilerplate is now one base class with no standalone implementations.

#### Impact

A user could meet an unexplained revert message, an expired action on a slow approval, or a blank application in a storage-blocked browser. No gas was lost to the expired path because preflight blocks it before the prompt.

#### Remediation

The LP route rebuilds its action calls after approvals when the calls are not bundled. The error taxonomy covers the expired, stale-data, feature-disabled and base-depegged reverts. Browser storage sits behind one guarded layer whose accessors never throw.

Commits: [`48587b54`](https://github.com/btr-protocol/front/commit/48587b54b24820dde04608759c4048ad237fc82a).

#### Status

Fixed, verified 2026-09-11. The informational rows were closed on 2026-09-10 under the low-minimum bar, and the two consolidation rows were verified closed in code.

**Rows.** 11 rows.

### F-72  Wallet transport stamped a stale chain identity and dropped failure detail from batches

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `front/src/lib/wallet.tsx:406-469`, `front/src/lib/walletConnect.ts:151-178`, `front/src/lib/walletCalls.ts:104-140` |

**Severity rationale.** Every outcome is a failed or aborted transaction with gas burned, never a mis-executed one, because EIP-155 replay protection stops a wrong-chain send from landing.

#### Description

The batch send path stamped the chain identifier held in component state and never re-read it live; only the atomic-requirement check re-read. The sequential path did re-read. Because pools are deployed per chain through CREATE3 and EIP-155 blocks cross-chain execution, a chain moved under the interface produced a failed or aborted transaction rather than a wrong-chain fill. The WalletConnect transport went further and hardcoded the chain identifier from a session constant, permanently stale until reconnect, with the safety reads using the same stale value and therefore self-consistent.

The failed-batch path reported only a hash and lost the revert reason, while the confirmed path replayed and decoded it. The replay helper itself dropped the call value that the dry run forwards, so a payable call on the wrap path was misattributed. A time-of-check to time-of-use gap remained between the batch preflight and the send, spanning the wallet prompt with no re-simulation, so a feed push, a halt or a slippage shift during the prompt produced a mined revert.

On the depth display, the denomination toggle relabelled without converting the cumulative column, the invert control flipped labels without reordering rungs, and a bounded net-times-gross approximation was presented as exact.

#### Impact

A user whose wallet moved chain mid-flow burned gas on a transaction that could not land, with no message explaining why. A failed batch gave no reason at all. The depth panel could be read with the wrong unit or the wrong orientation, though fills re-quote through the router.

#### Remediation

`sendCalls` re-reads the live chain identifier from the provider and refuses a moved chain. The WalletConnect transport holds a mutable chain updated on `chainChanged` and on wallet-initiated switches. The failed-batch path replays the plan for a decoded reason, and the replay helper forwards the call value. The prompt-dwell gap is accepted: the whole plan is re-preflighted immediately before the prompt and there is no in-application lever over the dwell itself, with the cost bounded to gas.

Commits: [`55eaa59a`](https://github.com/btr-protocol/front/commit/55eaa59a4bd3acfc33cd8fceca3df94bac0c7f7f), [`48587b54`](https://github.com/btr-protocol/front/commit/48587b54b24820dde04608759c4048ad237fc82a).

#### Status

Fixed on 2026-09-11. The prompt-dwell gap is accepted and the display rows were closed below the low-severity bar on 2026-09-10.

**Rows.** 8 rows.

### F-73  Coverage and upgrade-order test pins were absent, bare or vacuous

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `dex-evm/test/unit/CoverageProofs.t.sol:753-796`, `dex-evm/test/unit/PoolHooks.t.sol:1472-1499`, `dex-evm/test/unit/Base.t.sol:84-86` |

**Severity rationale.** No production code is affected; the exposure is that the invariants relied on for the coverage findings were passing without exercising the paths they claim to cover.

#### Description

The fuzz handler behind the coverage floor invariant drove only part of the surface. Cross withdraw, liability swap, donate and base were missing, four selectors, while the invariant claimed arbitrary interleavings. The invariant was therefore green exactly where the original coverage finding lives.

Several closure pins were stale or weak in the same way: weight-cap tests used bare `expectRevert` rather than a typed error, liability-swap slicing and two hook findings were unpinned, and one pin still allowed 100 bps. The two named ledger invariants had no on-chain pins at all, confidence pairing was tested on lane zero only, the mark packing fuzz test exercised a local copy rather than the library, golden fixtures were not read, the fork test could not run, and the upgrade-order matrix covered three of eighteen selectors. A `vm.skip(true)` placeholder sat in the fork directory as a test that could never execute.

#### Impact

Findings were closed against evidence that did not exercise the closing path. A regression in any of the four missing selectors, or in the fifteen uncovered upgrade-order selectors, would not have been caught.

#### Remediation

The coverage floor handler was widened so the invariant is no longer vacuous. The listed bare reverts are typed and the full upgrade-order matrix restored. The named ledger pins landed along with a hook-writer handler, a mark-packing pin against the real library, golden fixture reads, confidence lockstep and the coverage-strength gaps. The `vm.skip(true)` fork placeholder was deleted; the fork directory now holds one test with a real body behind a chain-identifier gate. The upgrade-order rehearsal is a deployment runbook step rather than a test.

Commits: [`82e2b23c`](https://github.com/btr-protocol/dex-evm/commit/82e2b23c6f11a1f1c82e739ab755fc6111b663c4), [`51b4a44c`](https://github.com/btr-protocol/dex-evm/commit/51b4a44cd5f30c1ab96ff13f1cd83275390bcff3), [`89fae854`](https://github.com/btr-protocol/dex-evm/commit/89fae854eb1f5161a64e86e60ff5ba1e14fa8024), [`296cac19`](https://github.com/btr-protocol/dex-evm/commit/296cac197b62834843c7f73698e20cea7a04c4ec), [`cc796065`](https://github.com/btr-protocol/dex-evm/commit/cc796065bb3d57564ec7f873945275e04b38b9f9), [`f65f959d`](https://github.com/btr-protocol/dex-evm/commit/f65f959dc55dbc476ee8826aef0f8f806513f515), [`3b0503b3`](https://github.com/btr-protocol/dex-evm/commit/3b0503b3dc52360e2e5f2c45bd0bfaf7ae0fc5bf), [`a9d23050`](https://github.com/btr-protocol/dex-evm/commit/a9d23050c34e37ae04638130edcb34d94df768c0), [`2e0760d0`](https://github.com/btr-protocol/dex-evm/commit/2e0760d04330d09b0b6c2536ca9710cf783e5c66), [`9b014388`](https://github.com/btr-protocol/dex-evm/commit/9b01438824256cbf17459e4e26f2c7bd8eaf0762).

#### Status

Closed on 2026-09-15.

**Rows.** 3 rows.

### F-74  RPC endpoints were trusted without chain attestation and the ABI encoder accepted malformed input

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `sdk/src/eth/chains.ts:116-123`, `sdk/src/eth/transport.ts:136-148`, `sdk/src/eth/abi.ts:169-177` |

**Severity rationale.** Exploiting the transport requires a malicious endpoint in the configured ring, and transport-layer security rules out a man in the middle on the defaults; the encoder defect fails closed on revert rather than losing value.

#### Description

The transport held a ring of public endpoint URLs and failed over by attempt index with no chain identifier or height attestation of any kind. The health probe checked only that the HTTP response was successful, so a poisoned read from a hostile endpoint was accepted as truth.

The ABI encoder performed no range or shape checks. An address of the wrong length, a `uint8` given 300, a negative value that wraps, and a `bytesN` of the wrong length all encoded without complaint. The resulting call reverts on chain rather than silently losing funds, but the failure surfaces late and without a usable message.

#### Impact

A consumer configured with a hostile endpoint could act on fabricated chain state, including balances, allowances and quote reads. Malformed encoder input produced opaque on-chain reverts instead of a local error naming the offending argument.

#### Remediation

Each endpoint now attests to the chain it serves before use, a wrong-chain endpoint is evicted once, the retry moves past it without backoff, and the private-key client's transport is pinned to its chain. The front end pins every read provider to the chain it serves. The encoder checks exact length for `bytesN`, element count for fixed arrays, and address, boolean and integer bounds, with dynamic byte strings validated against a hexadecimal pattern.

Commits: [`f7bec7e`](https://github.com/btr-protocol/sdk/commit/f7bec7e5d038a22bb95185cb21afc90651493cb8), [`2871558`](https://github.com/btr-protocol/sdk/commit/2871558ae6b47a5ca45a303707a94c5bf3d9f440), [`66e877c`](https://github.com/btr-protocol/sdk/commit/66e877c7b59283229dcf52990a071461d4021db6), [`85fc982`](https://github.com/btr-protocol/sdk/commit/85fc982a6711b153d7f54268b0da52ccf990b3bb), [`62d05b9`](https://github.com/btr-protocol/sdk/commit/62d05b988125ddea60430818f74ed874354e20d6), [`48a4b30`](https://github.com/btr-protocol/sdk/commit/48a4b30716275d93549a861860cac1bf45c3f063), [`1dc17c90`](https://github.com/btr-protocol/front/commit/1dc17c900e5b59943e4faebb141553a2f61ff538).

#### Status

Closed on 2026-09-15. Pinned by `abi.test.ts:315-322` and `abi.test.ts:325-329`.

**Rows.** 2 rows.

### F-75  Shared access control let a live pending rotation be overwritten silently and let a compromised treasury owner veto its own eviction

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | Audit | `shared/evm/src/access/AccessControl.sol:337-344`, `shared/evm/src/access/AccessControl.sol:385`, `shared/evm/src/base/UpgradeGate.sol:106` |

**Severity rationale.** Both defects require an already-compromised owner key or an operator mistake, and both move in the safe direction on their own, but they remove the cancel signal monitoring depends on and one of them made an eviction unreachable.

#### Description

`queueRole` overwrote a live pending rotation without the `AlreadyPending` ban and without emitting a cancellation event, unlike its sibling queues. Overwriting restarts the full delay, which is the safe direction, but monitoring loses the signal that a queued rotation was replaced.

```solidity
// shared/evm/src/access/AccessControl.sol:337
  /// @dev Re-queueing overwrites and restarts the full delay; the queued address is never live, so
  ///      there is no exit-notice clock to silently restart (contrast `Admin.requestOp`, which
  ///      bans a re-queue for exactly that reason).
  function queueRole(Role role, address a) external onlyOwner {
    _validateAddr(a);
    uint64 eta = uint64(block.timestamp) + _rotationDelay(role);
    pendingRole[role] = Queued(a, eta);
```

A compromised `treasuryOwner` could veto every `TREASURY_OWNER` rotation indefinitely: the guardian is excluded from that path, the bootstrap override is spent once, and no other override exists. The rule requiring a quorum on timelock-shaped queues had reached four of six such queues, leaving `UpgradeGate.requestUpgrade` among those it had not. A shared test asserting that the zero schedule is refused at deploy time was red, which gated a deploy-time guard; the cause was a `vm.setEnv` race across concurrent test functions, not a product defect.

#### Impact

A compromised treasury-owner key could block its own eviction forever, converting a key compromise into a permanent governance deadlock on that pointer. Fee custody remains bounded per pool throughout.

#### Remediation

`queueRole` now refuses a live pending rotation with `AlreadyPending` and emits `RoleCancelled` before overwriting an expired one. The treasury-owner self-veto is capped at one per queued rotation, pinned by `AccessControl.t.sol`. The quorum rule was extended to the remaining timelock-shaped queues, and the red shared test was fixed at its actual cause.

[`81cd310`](https://github.com/btr-protocol/shared/commit/81cd31034fc654d2341a5a1fd55902f65d5ce985), [`1145df1`](https://github.com/btr-protocol/shared/commit/1145df15518d2ddf3b9abf50cdf8f3c1da1b4b31), [`4d2a068`](https://github.com/btr-protocol/shared/commit/4d2a06886f8313a462652b28cead501d5284e7f2)

#### Status

Fixed, verified 2026-09-11. The veto cap reverses the balance in the other direction: a compromised owner can now evict an honest treasury owner after one veto. That residual was ratified as "incumbent once" in the governance timelock decision record and accepted, verified at the rev2 chair review. The dead `TREASURY` and `FACTORY` pointers, which have no on-chain readers, were closed as governance-ceremony hygiene under the purge of 2026-09-09.

**Rows.** 6 rows.

### F-76  The SDK build pinned ABIs but fetched them from the live production API, and mirror constants drifted from chain

| Severity | Status | Class | Component |
|---|---|---|---|
| LOW | Fixed | QA | `sdk/scripts/fetch-abis.ts:41`, `sdk/src/amm/aimm.ts:191`, `sdk/README.md:46` |

**Severity rationale.** A release or rollback performed in the wrong order could have failed every SDK and front build at once; no on-chain behaviour depends on either the mirror constant or the documentation.

#### Description

The build pinned its ABIs but sourced them from the live production API, so build success depended on live production state and on the order in which a release or rollback was performed. The SDK mirror of the interior swing cap was 10_000 while both the contracts and the integer core use 10_862, and the documentation still described an off-chain `@sdk/amm` pricer after the f64 replica had been deleted and swaps moved to the `/v2` quote path.

```ts
// sdk/src/amm/aimm.ts:191
export const INTERIOR_SWING_CAP_PBPS = 10_000;
export const MAX_DISPERSION_PBPS = 900_000;
```

#### Impact

Every SDK and front build shared a single live dependency with no local fallback. The stale mirror constant and documentation misdescribe the shipped quote path to integrators.

#### Remediation

The pinned-ABI build no longer depends on live production state, the mirror constants equal the chain constants, and the documentation describes the shipped quote path.

[`154904a`](https://github.com/btr-protocol/sdk/commit/154904a37465a0f31a39f24b5b1e203536b31aad), [`ec109f1`](https://github.com/btr-protocol/sdk/commit/ec109f1dff70e7d11f2cf5f8491103a4cb8d28a1), [`faade8d`](https://github.com/btr-protocol/sdk/commit/faade8d25b1e503594dd5c0926a9638963b4acca), [`2185748`](https://github.com/btr-protocol/sdk/commit/2185748a737fec358ae07152f0a4ab69a12685d6)

#### Status

Fixed 2026-09-16 and closed.

**Rows.** 3 rows.

### F-77  keccak256 panicked on inputs whose length was a non-zero multiple of the rate

| Severity | Status | Class | Component |
|---|---|---|---|
| INFO | Fixed | QA | `core: src/keccak.rs` |

**Severity rationale.** A panic on a well-formed input of a specific length, in a pure hashing primitive with no on-chain consumer at the audited revision.

#### Description

The pure-standard-library `keccak256` implementation panicked when the input length was a non-zero multiple of 136 bytes, the sponge rate, rather than absorbing a final padded block.

#### Impact

Any caller hashing an input of such a length aborted instead of returning a digest.

#### Remediation

The exported `keccak256` now equals the EVM `KECCAK256` opcode for every input length.

Commits: `core@8ef3ef6`.

#### Status

Fixed 2026-09-16.

**Rows.** 1 row.

## 6. Reporting a finding

Findings against deployed contracts go to **security@btr.markets**. Please do not open a public
issue for anything exploitable. We confirm receipt, say whether the finding is already in the
private ledger, and tell you when the fix is deployed; once it is, the finding is published with
attribution unless you ask otherwise. See [Bug Bounty](/docs/3-4-overview) for scope, rewards and
safe harbour.
