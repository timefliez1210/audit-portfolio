# Alignerz Audit Contest - Findings Report

## Table of Contents
- High Risk Findings
    - [H-01. Interface Mismatch DoS](#H-01)
    - [H-02. DoS in DividendDistributor due to Unbounded Loops](#H-02)
    - [H-03. Inverted Logic in Distributor](#H-03)
    - [H-04. Incorrect Split Logic leads to Compounding Reduction](#H-04)
    - [H-05. DoS in claimTokens for Multi-Flow Vesting](#H-05)
    - [H-06. Cumulative fee subtraction in FeesManager causes loss of funds](#H-06)
    - [H-07. Uninitialized memory in _computeSplitArrays causes DoS in splitTVS](#H-07)
    - [H-08. Missing loop increment in FeesManager causes infinite loop and underflow](#H-08)
    - [H-09. Unallocated memory in FeesManager causes DoS in splitTVS and mergeTVS](#H-09)
    - [H-10. DoS in A26ZDividendDistributor due to O(NM) complexity](#H-10)
    - [H-11. Infinite loop in getUnclaimedAmounts in A26ZDividendDistributor causes DoS](#H-11)
- Medium Risk Findings
    - [M-01. getToalUnclaimedAmounts in A26ZDividendDistributor incorrectly loops from NFT ID 0](#M-01)
    - [M-02. _setDividends in A26ZDividentDistributor incorrectly loops from NFT ID 0](#M-02)

---

## Contest Summary

**Sponsor:** Alignerz

**Dates:** November 2025

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 11    |
| Medium   | 2     |
| Low      | 0     |

---

# High Risk Findings

## <a id='H-01'></a>H-01. Interface Mismatch DoS

### Summary

There is a mismatch between the `IAlignerzVesting` interface and the `AlignerzVesting` implementation regarding the `allocationOf` function. The interface expects a full `Allocation` struct (including arrays), but the implementation's public mapping getter returns a tuple that omits the arrays. This causes all calls from `A26ZDividendDistributor` to `vesting.allocationOf` to revert.

### Root Cause

In `IAlignerzVesting.sol`:
```solidity
    function allocationOf(uint256 nftId) external view returns (Allocation memory);
```

In `AlignerzVesting.sol`:
```solidity
    mapping(uint256 => Allocation) public allocationOf;
```

Solidity's auto-generated getter for public mappings with structs containing arrays does not return the arrays. It returns `(bool isClaimed, IERC20 token, uint256 assignedPoolId)`. The distributor expects the full struct, leading to a decoding error.

### Impact

**High**. The `A26ZDividendDistributor` is completely non-functional. Any attempt to calculate or distribute dividends will revert.

### Mitigation

Implement a custom getter function in `AlignerzVesting` that returns the full struct.

```solidity
    function getAllocation(uint256 nftId) external view returns (Allocation memory) {
        return allocationOf[nftId];
    }
```
And update the interface and distributor to use this function.

## <a id='H-02'></a>H-02. DoS in DividendDistributor due to Unbounded Loops

### Description
In `A26ZDividendDistributor.sol`, the functions `_setDividends` and `getTotalUnclaimedAmounts` iterate from 0 to `nft.getTotalMinted()`.
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            // ... complex logic ...
        }
    }
```
As the number of NFTs grows, the gas cost of these functions will grow linearly. Eventually, it will exceed the block gas limit, making it impossible to set or distribute dividends.
Since `_setDividends` is required to enable claiming, the entire dividend mechanism will become permanently stuck.

### Impact
**High**. Permanent Denial of Service for dividend distribution once the user base grows sufficiently large.

### Recommendation
Implement a pull-based mechanism or process dividends in batches. Avoid iterating over all NFTs in a single transaction.

## <a id='H-03'></a>H-03. Inverted Logic in Distributor

### Summary

In `A26ZDividendDistributor.sol`, the function `getUnclaimedAmounts` contains a logic error that effectively disables dividend distribution. It checks if the project token matches the NFT token, but returns `0` if they match, instead of if they don't match.

### Root Cause

In `getUnclaimedAmounts`:

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        // ...
    }
```

The check `address(token) == address(vesting.allocationOf(nftId).token)` evaluates to `true` for valid NFTs of the project. The function then returns `0`. It only proceeds if the tokens *do not* match (or if the NFT holds a different token).

### Impact

**High**. The dividend distribution mechanism is completely broken. Valid users receive 0 dividends regardless of their holdings.

### Mitigation

Invert the check to ensure the tokens DO match, or return 0 if they DO NOT match.

```diff
- if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+ if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```

## <a id='H-04'></a>H-04. Incorrect Split Logic leads to Compounding Reduction

### Summary

The `AlignerzVesting::splitTVS` function contains an oversight where the storage of the original NFT is updated *during* the loop, affecting subsequent calculations. Specifically, when splitting an NFT, the function iterates through the percentages. For the first iteration (`i=0`), it updates the original NFT's allocation to its new partial amount. For subsequent iterations (`i>0`), it reads the *already reduced* amount from storage to calculate the next share, resulting in significantly lower amounts than intended.

### Root Cause

In `splitTVS`:

```solidity
        for (uint256 i; i < nbOfTVS; ) {
            // ...
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            // ...
            Allocation storage newAlloc = ...; // Points to `allocation` if i=0
            _assignAllocation(newAlloc, alloc); // Updates storage!
```

In `_computeSplitArrays`:

```solidity
    function _computeSplitArrays(Allocation storage allocation, ...) ... {
        uint256[] memory baseAmounts = allocation.amounts; // Reads from storage
        // ...
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

**The Sequence:**
1.  **Iteration `i=0` (e.g., 50%):**
    *   Reads `baseAmounts` (1000).
    *   Calculates 50% = 500.
    *   Updates `allocation` storage to 500.
2.  **Iteration `i=1` (e.g., 50%):**
    *   Reads `baseAmounts` from `allocation` (which is now 500!).
    *   Calculates 50% of 500 = 250.
    *   Result: NFT 2 gets 250 tokens instead of 500.

### Impact

**High**. Users lose a significant portion of their funds when splitting NFTs. In a 50/50 split, 25% of the total funds are permanently lost (vanished from the allocation tracking).

### Mitigation

Cache the original allocation amounts in memory *before* the loop, and pass this memory array to `_computeSplitArrays` instead of reading from storage inside the loop.

## <a id='H-05'></a>H-05. DoS in claimTokens for Multi-Flow Vesting

### Summary

The `AlignerzVesting::claimTokens` function reverts if *any* flow within the NFT's allocation has not yet started or has 0 claimable tokens. This creates a Denial of Service (DoS) for multi-flow NFTs (created via `mergeTVS`), where users cannot claim tokens from *active* flows if they are merged with *future* flows.

### Root Cause

In `AlignerzVesting::claimTokens`, the code iterates over all flows and calls `getClaimableAmountAndSeconds`:

```solidity
        for (uint256 i; i < nbOfFlows; i++) {
            // ...
            (
                uint256 claimableAmount,
                uint256 claimableSeconds
            ) = getClaimableAmountAndSeconds(allocation, i);
            // ...
        }
```

However, `getClaimableAmountAndSeconds` reverts in two scenarios:

1.  **Underflow (Not Started):**
    ```solidity
    } else {
        secondsPassed = block.timestamp - vestingStartTime; // <--- Reverts if now < vestingStartTime
    }
    ```
2.  **Zero Claimable:**
    ```solidity
    require(claimableAmount > 0, No_Claimable_Tokens()); // <--- Reverts if amount is 0
    ```

If an NFT has multiple flows (e.g., Flow A starts now, Flow B starts next year), `claimTokens` will fail for Flow A because Flow B triggers a revert.

### Impact

**High**. Users are effectively locked out of their funds if they use the `mergeTVS` feature with disparate schedules. This breaks the composability of the vesting tokens.

### Mitigation

Modify `getClaimableAmountAndSeconds` to return `(0, 0)` instead of reverting when tokens are not claimable.
Update `claimTokens` to handle the `0` return gracefully (e.g., `continue`).

```diff
    function getClaimableAmountAndSeconds(...) public view returns (...) {
        // ...
        if (block.timestamp < vestingStartTime) {
            return (0, 0);
        }
        // ...
        if (claimableAmount == 0) {
            return (0, 0);
        }
        return (claimableAmount, claimableSeconds);
    }
```

## <a id='H-06'></a>H-06. Cumulative fee subtraction in FeesManager causes loss of funds

### Summary

The `FeesManager` contract contains a critical logic bug in `calculateFeeAndNewAmountForOneTVS`. It calculates fees cumulatively, subtracting the sum of all previous fees from the current flow's amount. This results in severe overcharging for users with multiple vesting flows, potentially wiping out their allocations.

### Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
1. @>       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
2. @>       newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

At marker 1, `feeAmount` accumulates the fee for *each* flow.
At marker 2, this *cumulative* `feeAmount` is subtracted from `amounts[i]`.

**Example Scenario:**
*   User has 3 flows of 100 tokens each. Fee is 1%.
*   **Iteration 0**: `feeAmount` = 1. `newAmounts[0]` = 100 - 1 = 99. (Correct)
*   **Iteration 1**: `feeAmount` = 1 + 1 = 2. `newAmounts[1]` = 100 - 2 = 98. (Incorrect, should be 99)
*   **Iteration 2**: `feeAmount` = 2 + 1 = 3. `newAmounts[2]` = 100 - 3 = 97. (Incorrect, should be 99)

Users are charged the fees of all previous flows again for every subsequent flow. For a large number of flows, this can exceed the flow amount, causing an underflow revert or zero allocation.

### Impact

**High**. Loss of funds for users. Users with many vesting flows will be significantly penalized.

### Mitigation

Use a local variable for the current fee, not the cumulative total, when calculating the new amount.

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;
+           uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
+           newAmounts[i] = amounts[i] - currentFee;
+           feeAmount += currentFee;
        }
    }
```

## <a id='H-07'></a>H-07. Uninitialized memory in _computeSplitArrays causes DoS in splitTVS

### Summary

The `AlignerzVesting` contract contains a critical bug in `_computeSplitArrays` where it attempts to populate the arrays of a memory `Allocation` struct without allocating memory for them. This causes `splitTVS` to revert with an "Array Out of Bounds" panic, rendering the function unusable.

### Root Cause

In `AlignerzVesting.sol`:

```solidity
    function _computeSplitArrays(...) internal view returns (Allocation memory alloc) {
        // ...
        for (uint256 j; j < nbOfFlows; ) {
1. @>       alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            // ...
```

The return variable `alloc` is of type `Allocation memory`. It is not initialized. Accessing `alloc.amounts[j]` causes a panic because the array has no allocated storage.

### Impact

**High**. The `splitTVS` function is broken and unusable.

### Mitigation

Allocate memory for all dynamic arrays in the `alloc` struct before the loop.

```diff
    ) internal view returns (Allocation memory alloc) {
        // ...
+       alloc.amounts = new uint256[](nbOfFlows);
        // ... (allocate other arrays)

        for (uint256 j; j < nbOfFlows; ) {
            // ...
        }
    }
```

## <a id='H-08'></a>H-08. Missing loop increment in FeesManager causes infinite loop and underflow

### Summary

The `FeesManager` contract contains a critical bug in `calculateFeeAndNewAmountForOneTVS` where the `for` loop is missing an increment statement for the loop counter `i`. This results in an infinite loop where the fee is repeatedly accumulated until it exceeds the flow amount, causing an arithmetic underflow revert.

### Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
1. @>   for (uint256 i; i < length; ) { 
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

At marker 1, `i` is never incremented.

### Impact

**High**. `splitTVS` and `mergeTVS` are completely unusable due to Revert/DoS.

### Mitigation

Add the increment statement to the loop.

```diff
-       for (uint256 i; i < length; ) {
+       for (uint256 i; i < length; ++i) {
```

## <a id='H-09'></a>H-09. Unallocated memory in FeesManager causes DoS in splitTVS and mergeTVS

### Summary

The `FeesManager` contract contains a critical bug in `calculateFeeAndNewAmountForOneTVS` where it returns a memory array `newAmounts` without allocating memory for it. This causes `splitTVS` and `mergeTVS` to revert due to invalid memory access.

### Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
1. @>       newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

`newAmounts` is not initialized with `new uint256[](length)`.

### Impact

**High**. The `splitTVS` and `mergeTVS` functions are completely broken.

### Mitigation

Allocate memory for `newAmounts` before the loop.

```diff
+       newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
```

## <a id='H-10'></a>H-10. DoS in A26ZDividendDistributor due to O(NM) complexity

### Summary

The `_setDividends` function iterates through all minted NFTs. Inside this loop, it calls `getUnclaimedAmounts`, which iterates through all vesting flows for that NFT. This O(N*M) complexity will inevitably cause the transaction to exceed the block gas limit as the number of NFTs and flows increases, leading to a permanent DoS of the dividend distribution system.

### Impact

**High**. Permanent Denial of Service of the core dividend distribution functionality.

### Mitigation

Refactor the dividend distribution mechanism to avoid iterating over all NFTs in a single transaction. Use a pull-based mechanism or batched processing.

## <a id='H-11'></a>H-11. Infinite loop in getUnclaimedAmounts in A26ZDividendDistributor causes DoS

### Summary

Within `A26ZDividendDistributor::getUnclaimedAmounts` the function iterates through vesting allocations to calculate unclaimed amounts. However, conditional `continue` statements bypass the loop increment, causing an infinite loop when `claimedFlows[i]` is true or `claimedSeconds[i]` is 0.

### Root Cause

```solidity
        for (uint i; i < len; ) {
1. @>       if (claimedFlows[i]) continue;
2. @>       if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            // ...
            unchecked {
3. @>           ++i;
            }
        }
```

If the loop hits a `continue`, `i` is not incremented, causing an infinite loop.

### Impact

**High**. Denial of Service for the dividend distribution mechanism.

### Mitigation

Ensure the loop counter is incremented before `continue`.

# Medium Risk Findings

## <a id='M-01'></a>M-01. getToalUnclaimedAmounts in A26ZDividendDistributor incorrectly loops from NFT ID 0

### Summary

Within `A26ZDividentDistributor::getToalUnclaimedAmounts` the function is looping through existing, owned NFTs to distribute dividends, however an oversight causes the last NFT to not be counted towards dividend calculations.

### Root Cause

```solidity
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len; ) {
```

The loop iterates from 0 to `len - 1`. NFTs are minted starting from ID 1. This loop skips the last minted NFT (ID `len`).

### Impact

**Medium**. Critical calculations for dividends will be incorrect, skipping one NFT.

### Mitigation

Loop from 1 to `len` (inclusive).

```diff
-       for (uint i; i < len; ) {
+       for (uint i = 1; i <= len; ) {
```

## <a id='M-02'></a>M-02. _setDividends in A26ZDividentDistributor incorrectly loops from NFT ID 0

### Summary

Within `A26ZDividentDistributor::_setDividends` the function is looping through existing, owned NFTs to distribute dividends, however an oversight causes the last NFT to not get an allocation.

### Root Cause

```solidity
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len; ) {
```

The loop iterates from 0 to `len - 1`. NFTs are minted starting from ID 1. This loop skips the last minted NFT (ID `len`).

### Impact

**Medium**. The owner of the last NFT ID is entirely skipped regarding the dividends.

### Mitigation

Loop from 1 to `len` (inclusive).

```diff
-       for (uint i; i < len; ) {
+       for (uint i = 1; i <= len; ) {
```
