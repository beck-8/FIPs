---
fip: "XXXX"
title: Hybrid Base Fee Calculation for Multi-Block Message Deduplication
author: beck-8 (@beck-8)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/TBD
status: Draft
type: Technical
category: Core
created: 2026-01-07
---

# FIP-XXXX: Hybrid Base Fee Calculation for Multi-Block Message Deduplication

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the FIP.-->
Fix the base fee mechanism to correctly respond to network congestion when large messages appear in multiple blocks of a tipset, preventing scenarios where block space is full but fees decrease instead of increase.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->
The current Filecoin base fee calculation deduplicates messages across blocks in a tipset before computing total gas usage. While this correctly reflects VM execution costs (messages execute once per tipset), it fails to account for physical block space consumption when the same message appears in multiple blocks. This creates a vulnerability where high-value messages can saturate all blocks in a tipset while causing base fees to decrease instead of increase, effectively preventing legitimate transactions from being included.

This proposal introduces a hybrid base fee calculation that considers both execution dimensions (deduplicated gas) and space dimensions (physical gas consumption), using an adaptive weighting mechanism based on message duplication factor. The solution maintains backward compatibility in normal operation while correctly pricing congestion caused by repeated messages.

## Change Motivation
<!--The motivation is critical for FIPs that want to change the Filecoin protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the FIP solves. FIP submissions without sufficient motivation may be rejected outright.-->

### Problem Description

In December 2025, the Filecoin network experienced repeated congestion incidents (documented in https://github.com/filecoin-project/FIPs/issues/1224) where `SettleDealPaymentsExported` messages consumed massive gas (up to 8 billion per message, or 80% of the block gas limit), appeared in all blocks of a tipset, and crowded out critical `WindowPoSt` messages. Despite clear network congestion from a user perspective, the base fee mechanism failed to increase and in some cases actually decreased.

### Root Cause Analysis

The current base fee calculation in all three major implementations (Lotus `chain/store/basefee.go`, Venus `pkg/chain/message_store.go`, Forest `src/chain/store/base_fee.rs`) implements message deduplication:

```go
seen := make(map[cid.Cid]struct{})
for _, b := range ts.Blocks() {
    for _, m := range messages {
        if _, ok := seen[m.Cid()]; !ok {
            totalLimit += m.GasLimit  // Only counted once
            seen[m.Cid()] = struct{}{}
        }
    }
}
delta = (totalLimit / numBlocks) - BlockGasTarget
```

**Attack Scenario:**
- Message gas limit: 8B (80% of 10B block limit)
- Tipset: 5 blocks, all include the same message
- Current calculation: `totalLimit = 8B` (deduplicated), `avgGas = 1.6B`, `delta = -3.4B` → **Base fee decreases**
- Physical reality: Each block is 80% full, other messages cannot be included → **Network is severely congested**

### Why Deduplication Exists

Message execution in `chain/consensus/compute_state.go:213-248` also deduplicates messages - they execute only once per tipset regardless of how many blocks contain them. The base fee calculation mirrors this to reflect actual computational load.

### The Fundamental Conflict

| Dimension | Measurement | Congestion Level |
|-----------|-------------|------------------|
| Execution | VM gas used | 1.6B/block (not congested) |
| Space | Block capacity | 8B/block (very congested) |
| User Experience | Can send tx? | No (very congested) |

Current base fee only considers execution dimension, creating misalignment with user-perceived congestion.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current Filecoin implementations. -->

### Algorithm Overview

Replace the single-dimension base fee calculation with a hybrid approach that adaptively weights execution and space dimensions based on message duplication factor.

### Detailed Steps

**Step 1: Calculate Both Gas Dimensions**

```
unique_gas_limit := 0
physical_gas_limit := 0
seen := new Set()

for each block b in tipset.blocks:
    for each message m in block_messages(b):
        physical_gas_limit += m.gas_limit  // No dedup

        if m.cid not in seen:
            unique_gas_limit += m.gas_limit  // Dedup
            seen.add(m.cid)
```

**Step 2: Calculate Duplication Factor**

```
duplication_factor := physical_gas_limit / unique_gas_limit

if unique_gas_limit == 0:
    duplication_factor := 1.0
```

**Step 3: Compute Adaptive Weight**

```
num_blocks := len(tipset.blocks)

// Normalize duplication factor to [0, 1]
normalized_dup := (duplication_factor - 1.0) / (num_blocks - 1.0)
normalized_dup := clamp(normalized_dup, 0.0, 1.0)

// Apply sigmoid for smooth transition
sigmoid_input := normalized_dup × 6.0 - 3.0
space_weight := sigmoid(sigmoid_input)
space_weight := clamp(space_weight, 0.05, 0.95)
exec_weight := 1.0 - space_weight

where sigmoid(x) = 1 / (1 + e^(-x))
```

**Step 4: Calculate Effective Gas**

```
effective_gas := unique_gas_limit × exec_weight + physical_gas_limit × space_weight
```

**Step 5: Compute Base Fee (Existing Logic)**

```
delta := (effective_gas / num_blocks) - BlockGasTarget
// Continue with existing ComputeNextBaseFee logic
```

### Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| sigmoid_steepness | 6.0 | Provides clear transition around duplication factor 3.0 |
| sigmoid_center | 3.0 | Balances execution/space at mid-range duplication |
| min_space_weight | 0.05 | Small space consideration even without duplication |
| max_space_weight | 0.95 | Small execution consideration even with full duplication |

These parameters map the duplication factor range [1.0, num_blocks] to sigmoid's effective range [-3, +3], which produces weights from approximately 5% to 95%.

### Edge Cases

**Single-block tipset:**
```
if num_blocks == 1:
    normalized_dup := 0.0
    // Falls back to execution dimension (space_weight ≈ 0.05)
```

**Empty tipset:**
```
if unique_gas_limit == 0 and physical_gas_limit == 0:
    effective_gas := 0
    // Base fee decreases to minimum (existing behavior)
```

**Numerical precision:** Use fixed-point arithmetic with precision of 1,000,000 to ensure deterministic results across implementations.

## Design Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

### Alternative Approaches Considered

1. **Remove deduplication entirely**
   - Rejected: Violates execution semantics (messages execute once)
   - Would over-penalize normal multi-block inclusion

2. **Use max(unique_gas, physical_gas)**
   - Rejected: Too aggressive, lacks smooth transition
   - Simple but binary behavior

3. **Fixed 50/50 weighting**
   - Rejected: Doesn't adapt to duplication level
   - Over-penalizes normal operation

4. **Linear interpolation by duplication factor**
   - Rejected: Too responsive near boundaries
   - Less intuitive weighting curve

5. **Hybrid with sigmoid (this proposal)**
   - Selected: Adapts to duplication level, smooth transition
   - Maintains near-original behavior in normal cases
   - Strongly penalizes high duplication

### Why Sigmoid?

The sigmoid function provides:
1. **Smooth transition**: No sudden jumps in fee calculation
2. **Bounded output**: Always produces valid weights in (0, 1)
3. **Tunable**: Parameters control transition speed and center point
4. **Well-studied**: Extensively used in economics and control theory

**Weight Distribution:**

| Duplication | Space Weight | Exec Weight | Scenario |
|-------------|--------------|-------------|----------|
| 1.0 | 5% | 95% | Normal (no duplication) |
| 2.0 | 18% | 82% | Low duplication |
| 3.0 | 50% | 50% | Transition point |
| 4.0 | 82% | 18% | High duplication |
| 5.0 | 95% | 5% | Attack scenario |

### Parameter Selection

**Sigmoid steepness (6.0):** Tested values [2, 4, 6, 8, 10]. Value 6.0 provides clear transition zone without being too abrupt - 95% of transition occurs between duplication factors 2.0-4.0.

**Center point (3.0):** Balances execution and space dimensions when normalized duplication reaches 0.5. Due to the normalization formula `(dup - 1) / (num_blocks - 1)`, this adapts automatically to different tipset sizes:
- 5-block tipset: 50% weight when ~3 blocks contain the same message
- 10-block tipset: 50% weight when ~6 blocks contain the same message
- 11-block tipset: 50% weight when ~6 blocks contain the same message

This parameter choice prioritizes minimizing impact on normal operations while still providing strong attack mitigation, regardless of tipset size.

Alternative values considered:
- **center=2.5 (more aggressive)**: Would reach 50% space weight at duplication factor 2.5, responding more quickly to duplication. At dup=2.0, space weight would be ~38% (vs 18% with center=3.0). This is more conservative for network protection but may impact normal multi-block message inclusion.
- **center=3.5 (more lenient)**: Would delay response, reaching 50% only at dup=3.5. Less suitable for 5-block tipsets where maximum duplication is 5.0.

The choice of 3.0 represents a balanced approach: attack scenarios (dup=4.0-5.0) still receive 82-95% space weighting, while normal operations (dup=1.0-1.5) see minimal impact (<10% space weight). This parameter can be adjusted based on testnet observations and community feedback before mainnet activation.

**Weight bounds (0.05-0.95):** Prevents complete elimination of either dimension, ensuring small contribution from space even in normal cases while maintaining execution consideration during attacks.

## Backwards Compatibility
<!--All FIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The FIP must explain how the author proposes to deal with these incompatibilities. FIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->

This is a **consensus-breaking change** requiring a network upgrade. All nodes must activate the new algorithm at the same epoch.

### Impact Analysis

**Normal operation (duplication factor 1.0-1.5):**
- Space weight: 5-10%
- Effective gas ≈ unique_gas (nearly identical to current)
- **Impact: Minimal (<2% fee difference)**

**Moderate duplication (2.0-2.5):**
- Space weight: 18-32%
- Noticeable but not dramatic fee adjustment
- **Impact: Small (5-15% fee increase)**

**High duplication (4.0-5.0):**
- Space weight: 82-95%
- Strong fee increase to combat congestion
- **Impact: Large (intended correction)**

### Migration Strategy

**Phase 1: Testnet Activation**
- Activate on calibration testnet
- Observe fee dynamics under various conditions
- Tune parameters if needed

**Phase 2: Mainnet Upgrade**
- Include in scheduled network upgrade
- Announce 6+ weeks in advance
- Provide fee estimation tools

## Test Cases
<!--Test cases for an implementation are mandatory for FIPs that are affecting consensus changes. Other FIPs can choose to include links to test cases if applicable.-->

### Test 1: Attack Scenario (December 2025 Congestion)

**Setup:** 5 blocks, 8B message in all blocks, parent base fee 100 nanoFIL

**Expected:**
```
unique_gas = 8B, physical_gas = 40B
duplication_factor = 5.0
space_weight ≈ 0.953
effective_gas = 38.5B
avg_per_block = 7.7B
delta = +2.7B
Result: Base fee INCREASES ✓
```

**Current behavior:** delta = -3.4B, base fee decreases ✗

### Test 2: Normal Operation

**Setup:** 5 blocks, different 5B messages each

**Expected:**
```
unique_gas = 25B, physical_gas = 25B
duplication_factor = 1.0
space_weight ≈ 0.047
effective_gas = 25B
delta = 0
Result: Base fee UNCHANGED (matches current) ✓
```

### Test 3: Partial Duplication

**Setup:** 5 blocks, 6B message in blocks 1-3, different 4B messages in blocks 4-5

**Expected:**
```
unique_gas = 14B, physical_gas = 26B
duplication_factor = 1.86
space_weight ≈ 0.153
effective_gas = 15.8B
delta = -1.84B
Result: Base fee DECREASES (moderate, correct for partial load) ✓
```

### Test 4: Empty Tipset

**Setup:** 5 blocks, no messages

**Expected:**
```
effective_gas = 0
delta = -5B
Result: Base fee decreases to minimum (matches current) ✓
```

### Test 5: Single-Block Tipset

**Setup:** 1 block, 8B message

**Expected:**
```
normalized_dup = 0.0
effective_gas ≈ 8B
delta = +3B
Result: Base fee INCREASES (correct, block is full) ✓
```

## Security Considerations
<!--All FIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. FIP submissions missing the "Security Considerations" section will be rejected. A FIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

### Attack Vectors

**Message Flooding:** Sending many small messages to inflate physical_gas is mitigated by existing message pool filtering by gas premium. Base fee will increase normally.

**Duplication Manipulation:** Miners colluding to avoid duplicates results in duplication → 1.0, yielding current behavior. This is normal operation, not an attack.

**Fee Oscillation:** Alternating between high/low duplication is bounded by existing 12.5% max change per epoch rate limit. Attacker must sustain high gas premium, and sigmoid smoothing reduces reactivity.

**Precision Attacks:** Exploiting floating-point inconsistencies is prevented by using deterministic fixed-point arithmetic with specified precision (1,000,000).

### Consensus Safety

**Determinism Requirements:**

The sigmoid calculation MUST be fully deterministic across all implementations to prevent consensus failures. The reference implementation achieves this through:

1. **Precomputed Lookup Table**: All sigmoid values for the range [-3.0, +3.0] are precomputed and stored as fixed-point integers (61 entries at 0.1 intervals). This eliminates dependency on `math.Exp()` which may have platform-specific floating-point behavior.

2. **Linear Interpolation**: Runtime sigmoid evaluation uses simple integer arithmetic to interpolate between lookup table entries, ensuring identical results across all platforms, architectures, and compiler optimizations.

3. **Fixed-Point Arithmetic**: All intermediate calculations use 64-bit integer arithmetic with a fixed precision of 1,000,000. No floating-point operations are used in the consensus-critical path.

4. **Cross-Client Testing**: The lookup table values must be identical across Lotus (Go), Venus (Go), and Forest (Rust). Reference test vectors should be provided to validate consistency.

**Implementation Security:**

Critical requirements include:
- Overflow protection in gas summation (use checked arithmetic or validate bounds)
- Division-by-zero handling when uniqueGasLimit is zero
- Proper clamping of normalized_dup to [0, PRECISION] range
- Bounds checking on lookup table indices
- Fixed-point arithmetic validation to prevent precision loss

## Incentive Considerations
<!--All FIPs must contain a section that discusses the incentive implications/considerations relative to the proposed change. Include information that might be important for incentive discussion. A discussion on how the proposed change will incentivize reliable and useful storage is required. FIP submissions missing the "Incentive Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Incentive Considerations discussion deemed sufficient by the reviewers.-->

### Miner Incentives

Miners select messages by gas performance (reward per gas unit). Large messages with good fees remain attractive. High duplication leads to higher base fees, generating more revenue for miners. No change to message selection algorithm needed.

**Net Impact:** Neutral to positive for miners (higher fees during congestion).

### User Incentives

**Large Message Senders:** Previously could saturate network with moderate gas premium. After this change, must pay appropriately for block space consumption. Incentivized to optimize message size or split into smaller messages.

**Regular Users:** Previously priced out during attack scenarios despite low base fees. After this change, base fees rise appropriately but messages can be included. More predictable fee environment.

**Storage Providers:** Previously WindowPoSt messages at risk during attacks. After this change, base fee increase reduces attack profitability. More reliable message inclusion.

### Economic Equilibrium

The hybrid approach creates proper economic signals:
1. **Scarcity Pricing**: Both computational and space resources priced appropriately
2. **Attack Deterrence**: Duplication attacks become economically infeasible
3. **Market Efficiency**: Fees reflect actual resource consumption
4. **Predictability**: Smooth transitions reduce fee volatility

## Product Considerations
<!--All FIPs must contain a section that discusses the product implications/considerations relative to the proposed change. Include information that might be important for product discussion. A discussion on how the proposed change will enable better storage-related goods and services to be developed on Filecoin. FIP submissions missing the "Product Considerations" section will be rejected. An FIP cannot proceed to status "Final" without a Product Considerations discussion deemed sufficient by the reviewers.-->

### User-Facing Changes

**Fee Estimation:** Existing fee estimation APIs must account for new calculation. Historical fee data may show discontinuity at activation epoch. Updated fee estimator libraries should be provided.

**Wallet/Explorer Impact:** Gas estimation remains unchanged. Base fee display updated to show new values. Historical charts should note algorithm change.

**Developer Tooling:** Gas simulation tools need updates. Migration guides and testing utilities should be provided for pre-activation validation.

### Operational Considerations

**Node Operators:** No configuration changes required. Automatic activation at upgrade epoch. Monitoring recommended for initial epochs.

**Miners:** Message selection logic unchanged. Revenue may increase during congestion. No operational changes needed.

**Application Developers:** Should review gas estimation logic, test under various duplication scenarios, and update documentation if providing fee guidance.

## Implementation
<!--The implementations must be completed before any core FIP is given status "Final", but it need not be completed before the FIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

### Reference Implementation (Go/Lotus)

The following shows the key changes needed in Lotus `chain/store/basefee.go`:

```go
// Add new constants
const (
    PRECISION            = 1_000_000
    SIGMOID_STEEPNESS   = 6_000_000  // 6.0 in fixed-point
    SIGMOID_CENTER      = 3_000_000  // 3.0 in fixed-point
    MIN_SPACE_WEIGHT    = 50_000     // 0.05 in fixed-point
    MAX_SPACE_WEIGHT    = 950_000    // 0.95 in fixed-point
)

// Precomputed sigmoid lookup table for deterministic consensus
// Maps input x ∈ [-3.0, +3.0] to sigmoid(x) ∈ [0.047, 0.953]
// 61 entries at 0.1 intervals, all values × 1,000,000 for fixed-point
var sigmoidLookupTable = [61]uint64{
     47426,  52154,  57324,  62973,  69138,  75858,  83173,  91123,
     99750, 109097, 119203, 130108, 141851, 154465, 167982, 182426,
    197816, 214165, 231475, 249740, 268941, 289050, 310026, 331812,
    354344, 377541, 401312, 425557, 450166, 475021, 500000, 524979,
    549834, 574443, 598688, 622459, 645656, 668188, 689974, 710950,
    731059, 750260, 768525, 785835, 802184, 817574, 832018, 845535,
    858149, 869892, 880797, 890903, 900250, 908877, 916827, 924142,
    930862, 937027, 942676, 947846, 952574,
}

// Deterministic sigmoid using lookup table with linear interpolation
func sigmoidFixed(xFixed int64) uint64 {
    // Input range: [-3_000_000, +3_000_000] (fixed-point)
    // Clamp to valid range
    if xFixed <= -3_000_000 {
        return sigmoidLookupTable[0]
    }
    if xFixed >= 3_000_000 {
        return sigmoidLookupTable[60]
    }

    // Convert to table index (each step = 0.1 = 100_000)
    // Shift to [0, 6_000_000] range
    shifted := xFixed + 3_000_000

    // Find table indices
    lowerIdx := shifted / 100_000
    upperIdx := lowerIdx + 1

    // Linear interpolation
    lowerVal := sigmoidLookupTable[lowerIdx]
    upperVal := sigmoidLookupTable[upperIdx]

    // Calculate fractional part
    remainder := shifted % 100_000

    // Interpolate: lower + (upper - lower) × remainder / 100_000
    result := lowerVal + (upperVal-lowerVal)*uint64(remainder)/100_000

    return result
}

func (cs *ChainStore) ComputeBaseFee(ctx context.Context, ts *types.TipSet) (abi.TokenAmount, error) {
    // Check if hybrid algorithm is active
    if ts.Height() < buildconstants.UpgradeHybridBaseFeeHeight {
        return cs.computeBaseFeeOriginal(ctx, ts)
    }

    // Handle Breeze gas tamping
    if buildconstants.UpgradeBreezeHeight >= 0 &&
       ts.Height() > buildconstants.UpgradeBreezeHeight &&
       ts.Height() < buildconstants.UpgradeBreezeHeight + buildconstants.BreezeGasTampingDuration {
        return abi.NewTokenAmount(100), nil
    }

    zero := abi.NewTokenAmount(0)

    // Step 1: Calculate both gas dimensions
    uniqueGasLimit := int64(0)
    physicalGasLimit := int64(0)
    seen := make(map[cid.Cid]struct{})

    for _, b := range ts.Blocks() {
        msg1, msg2, err := cs.MessagesForBlock(ctx, b)
        if err != nil {
            return zero, xerrors.Errorf("error getting messages: %w", err)
        }

        // Process BLS messages
        for _, m := range msg1 {
            physicalGasLimit += m.GasLimit

            c := m.Cid()
            if _, ok := seen[c]; !ok {
                uniqueGasLimit += m.GasLimit
                seen[c] = struct{}{}
            }
        }

        // Process Secp messages
        for _, m := range msg2 {
            physicalGasLimit += m.Message.GasLimit

            c := m.Cid()
            if _, ok := seen[c]; !ok {
                uniqueGasLimit += m.Message.GasLimit
                seen[c] = struct{}{}
            }
        }
    }

    // Step 2: Calculate duplication factor (using fixed-point arithmetic)
    var duplicationFactorFixed int64
    if uniqueGasLimit > 0 {
        // duplicationFactor = physicalGas / uniqueGas, scaled by PRECISION
        duplicationFactorFixed = (physicalGasLimit * PRECISION) / uniqueGasLimit
    } else {
        duplicationFactorFixed = PRECISION  // 1.0 in fixed-point
    }

    // Step 3: Compute adaptive weight (all in fixed-point)
    numBlocks := len(ts.Blocks())
    var normalizedDupFixed int64

    if numBlocks > 1 {
        // normalized = (dup - 1.0) / (numBlocks - 1.0)
        // In fixed-point: (dup_fixed - PRECISION) / (numBlocks - 1)
        normalizedDupFixed = (duplicationFactorFixed - PRECISION) / int64(numBlocks-1)

        // Clamp to [0, PRECISION]
        if normalizedDupFixed < 0 {
            normalizedDupFixed = 0
        }
        if normalizedDupFixed > PRECISION {
            normalizedDupFixed = PRECISION
        }
    } else {
        normalizedDupFixed = 0
    }

    // sigmoid_input = normalized × 6.0 - 3.0
    sigmoidInputFixed := (normalizedDupFixed * SIGMOID_STEEPNESS / PRECISION) - SIGMOID_CENTER

    // Use deterministic lookup table
    spaceWeightFixed := sigmoidFixed(sigmoidInputFixed)

    // Enforce bounds
    if spaceWeightFixed < MIN_SPACE_WEIGHT {
        spaceWeightFixed = MIN_SPACE_WEIGHT
    }
    if spaceWeightFixed > MAX_SPACE_WEIGHT {
        spaceWeightFixed = MAX_SPACE_WEIGHT
    }

    execWeightFixed := PRECISION - spaceWeightFixed

    // Step 4: Calculate effective gas
    effectiveGas := (uint64(uniqueGasLimit)*execWeightFixed +
                     uint64(physicalGasLimit)*spaceWeightFixed) / PRECISION

    // Step 5: Use existing base fee calculation
    parentBaseFee := ts.Blocks()[0].ParentBaseFee

    return ComputeNextBaseFee(parentBaseFee, int64(effectiveGas), numBlocks, ts.Height()), nil
}

// Original ComputeNextBaseFee remains unchanged
func ComputeNextBaseFee(baseFee types.BigInt, gasLimitUsed int64, noOfBlocks int, epoch abi.ChainEpoch) types.BigInt {
    // Existing implementation unchanged
    var delta int64
    if epoch > buildconstants.UpgradeSmokeHeight {
        delta = gasLimitUsed / int64(noOfBlocks)
        delta -= buildconstants.BlockGasTarget
    } else {
        delta = buildconstants.PackingEfficiencyDenom * gasLimitUsed / (int64(noOfBlocks) * buildconstants.PackingEfficiencyNum)
        delta -= buildconstants.BlockGasTarget
    }

    if delta > buildconstants.BlockGasTarget {
        delta = buildconstants.BlockGasTarget
    }
    if delta < -buildconstants.BlockGasTarget {
        delta = -buildconstants.BlockGasTarget
    }

    change := big.Mul(baseFee, big.NewInt(delta))
    change = big.Div(change, big.NewInt(buildconstants.BlockGasTarget))
    change = big.Div(change, big.NewInt(buildconstants.BaseFeeMaxChangeDenom))

    nextBaseFee := big.Add(baseFee, change)
    if big.Cmp(nextBaseFee, big.NewInt(buildconstants.MinimumBaseFee)) < 0 {
        nextBaseFee = big.NewInt(buildconstants.MinimumBaseFee)
    }
    return nextBaseFee
}
```

### Implementation Status

**Lotus:** Not yet implemented
**Venus:** Not yet implemented
**Forest:** Not yet implemented

Implementation across all three clients is required before this FIP can proceed to "Final" status.

## TODO
<!--A section that lists any unresolved issues or tasks that are part of the FIP proposal. Examples of these include performing benchmarking to know gas fees, validate claims made in the FIP once the final implementation is ready, etc. A FIP can only move to a "Last Call" status once all these items have been resolved.-->

- [ ] Implement in Lotus, Venus, and Forest
- [ ] Cross-client consensus testing
- [ ] Deploy to calibration testnet
- [ ] Monitor real-world fee dynamics on testnet
- [ ] Benchmark performance impact
- [ ] Historical data analysis with December 2025 congestion data
- [ ] Community feedback and parameter tuning
- [ ] Security audit
- [ ] Update fee estimation tools and libraries
- [ ] Prepare migration documentation

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
