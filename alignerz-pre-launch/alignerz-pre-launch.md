# Alignerz Pre-Launch - Findings Report

## Table of Contents
- High Risk Findings
- Medium Risk Findings
    - [M-01. Multi-Batch Allocation DoS Due to Index Desynchronization](#M-01)
    - [M-02. Mathematical error in split/merge fee calculation leads to excessive token loss](#M-02)

---

## Contest Summary

**Sponsor:** Alignerz

**Dates:** January 2026

---

## Results Summary

| Severity | Count |
|----------|-------|
| High     | 0     |
| Medium   | 2     |
| Low      | 0     |

---

# Medium Risk Findings

## <a id='M-01'></a>M-01. Multi-Batch Allocation DoS Due to Index Desynchronization

### Summary

In Alignerz.sol's setTVSAllocation function the KOL index calculation uses a monotonically increasing counter (nbOfKOLs) instead of the actual array length. When any KOL claims their TVS allocation (which removes them from the tracking array via swap-and-pop), subsequent batch allocations will assign incorrect indices. This causes:

Data Corruption: Most users in subsequent batches will unknowingly corrupt other users' records when claiming.
Permanent DoS: The last N users of subsequent batches (where N = number of intermediate claims) will experience an array out-of-bounds panic and be permanently unable to claim their allocated tokens.

### Finding Description

**Root Cause**

In setTVSAllocation (line 463), the index is calculated as:

```solidity
rewardProject.kolTVSIndexOf[kol] = rewardProject.nbOfKOLs + i;
```

However, nbOfKOLs only ever increases (line 466: rewardProject.nbOfKOLs += length). It is never decremented when users claim and are removed from kolTVSAddresses.

In contrast, claimRewardTVS uses swap-and-pop (lines 501-510) to remove the claimer from kolTVSAddresses, which decreases the array length.

**The Desync**

| Event | nbOfKOLs | kolTVSAddresses.length |
|-------|----------|------------------------|
| Initial | 0 | 0 |
| Batch 1 (200 users) | 200 | 200 |
| 1 User Claims | 200 ❌ | 199 ✓ |
| Batch 2 (200 users) | 400 | 399 |

When Batch 2 is processed:

User 0 gets index 200 + 0 = 200, but is physically at array position 199.
User 199 gets index 200 + 199 = 399, but the array only has 399 elements (valid indices: 0-398).

**Realistic Scenario**

Project Launch: Admin launches a Reward Project for 1000 KOLs.
Batch 1: Admin allocates to first 200 KOLs (gas limits prevent larger batches).
Claim: One enthusiastic KOL claims immediately after allocation.
Batch 2: Admin allocates to next 200 KOLs.
Result: The 200th user in Batch 2 is permanently DoS'd. All other Batch 2 users will corrupt each other's records when claiming.

This is highly realistic because:
1. Gas limits necessitate batched allocations for large KOL sets.
2. There is no mechanism to pause claims during batch processing.
3. Claims are permissionless - users can claim anytime after their allocation.

### Impact Explanation

**Index Corruption (Non-Panic Path):**

When User 0 (Batch 2) claims, they access kolTVSAddresses[200], which holds User 1.
The swap-and-pop logic removes User 1's entry instead of User 0's.
User 1's claim will later fail or corrupt another user's entry.

**Permanent DoS (Panic Path):**

The last user(s) of Batch 2 have indices >= array length.
claimRewardTVS attempts to read kolTVSAddresses[399] when max index is 398.
This triggers Panic(0x32) (Array Out of Bounds).
These users permanently lose access to their allocated tokens.

### Likelihood Explanation

Likelihood: High - Batched allocations are standard practice; early claims are expected behavior.

### Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

import "forge-std/Test.sol";
import {Alignerz} from "../../src/contracts/vesting/Alignerz.sol";
import {A26Z} from "../../src/contracts/token/A26Z.sol";
import {TVSManager} from "../../src/contracts/vesting/TVSManager.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract MultiBatchDoSTest is Test {
    Alignerz alignerz;
    TVSManager tvsManager;
    A26Z token;
    AlignerzNFT nft;
    
    address owner = address(this);

    function setUp() public {
        token = new A26Z("Test", "TST");
        nft = new AlignerzNFT("NFT", "NFT", "http://test/");
        
        TVSManager tvsImpl = new TVSManager();
        Alignerz alignerzImpl = new Alignerz();
        
        ERC1967Proxy tvsProxy = new ERC1967Proxy(address(tvsImpl), "");
        tvsManager = TVSManager(address(tvsProxy));
        tvsManager.initialize(address(nft));
        
        ERC1967Proxy alignerzProxy = new ERC1967Proxy(address(alignerzImpl), "");
        alignerz = Alignerz(payable(address(alignerzProxy)));
        alignerz.initialize(address(nft), address(tvsManager));
        
        nft.addMinter(address(alignerz));
        nft.addMinter(address(tvsManager));
        tvsManager.setAlignerz(address(alignerz));
        tvsManager.setTreasury(owner);

        token.approve(address(alignerz), type(uint256).max);
        alignerz.launchRewardProject(address(token), 1_000_000e18, 500);
    }

    function test_MultiBatch_DoS() public {
        uint256 batchSize = 200;
        address[] memory kols1 = new address[](batchSize);
        uint256[] memory amounts1 = new uint256[](batchSize);
        
        for(uint256 i=0; i<batchSize; i++) {
            kols1[i] = address(uint160(0x1000 + i));
            amounts1[i] = 100e18;
        }
        
        // Batch 1 allocation
        alignerz.setTVSAllocation(0, 1000, 100e18 * batchSize, kols1, amounts1);

        // One user claims between batches
        vm.prank(kols1[0]);
        alignerz.claimRewardTVS(0);
        
        // Batch 2 allocation
        address[] memory kols2 = new address[](batchSize);
        uint256[] memory amounts2 = new uint256[](batchSize);
        for(uint256 i=0; i<batchSize; i++) {
            kols2[i] = address(uint160(0x2000 + i));
            amounts2[i] = 100e18;
        }
        alignerz.setTVSAllocation(0, 1000, 100e18 * batchSize, kols2, amounts2);
        
        // Last user of Batch 2 is permanently DoS'd
        address victim = kols2[batchSize - 1];
        vm.prank(victim);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x32));
        alignerz.claimRewardTVS(0);
    }
}
```

Test Output:

```
[PASS] test_MultiBatch_DoS() (gas: 28633844)
```

### Recommendation

Use the actual array length after push instead of nbOfKOLs:

```diff
    for (uint256 i = 0; i < length; i++) {
        address kol = kolTVS[i];
        rewardProject.kolTVSAddresses.push(kol);
        uint256 amount = TVSamounts[i];
        amounts += amount;
        rewardProject.kolTVSRewards[kol] = amount;
-       rewardProject.kolTVSIndexOf[kol] = rewardProject.nbOfKOLs + i;
+       rewardProject.kolTVSIndexOf[kol] = rewardProject.kolTVSAddresses.length - 1;
        emit TVSAllocated(rewardProjectId, kol, amount, vestingPeriod);
    }
```

## <a id='M-02'></a>M-02. Mathematical error in split/merge fee calculation leads to excessive token loss

### Summary

The protocol fails to scale down a user's claimedAmount when applying a fee to the total amount during split or merge operations. This results in the user effectively paying the fee on tokens they have already claimed, causing a significant reduction in their remaining unclaimed balance beyond the stated fee percentage.

### Finding Description

Consider the following logic in src/libraries/TVSCalculator.sol:

When splitting or merging mid-vesting, a fee is calculated based on the unclaimedAmount. This fee is subtracted from the total amount of the flow.

```solidity
// src/libraries/TVSCalculator.sol:36
                uint256 fee = unclaimedAmount * feeRate / BASIS_POINT;
@>              newAmounts[i] = amount - fee;
                feeAmount += fee;
```

However, the claimedAmount counter for that flow is copied over unscaled to the new/updated NFT.

Later, when calculating the claimable amount, the contract uses:

```solidity
// src/contracts/vesting/TVSManager.sol:231
@> claimableAmount = ((block.timestamp - vestingStartTime) * amount / vestingPeriod) - claimedAmount;
```

Because amount (the total) has decreased due to the fee, but claimedAmount (the offset) has remained the same, the user's progress is incorrectly calculated. The user is effectively penalized as if the fee was applied to the entire amount, including the portion they already "owned" and claimed.

**Example:**

User has 100 tokens vesting over 100 seconds.
At t=40, they claim 40 tokens.
They split the NFT with a 2% fee.
Unclaimed was 60. Fee is 1.2.
New amount = 100 - 1.2 = 98.8.
claimedAmount is still 40.
At t=50, the user expects to have earned more tokens.
Mathematically, they had 60 left. After a 2% fee, they have 58.8 left.
Over the remaining 60 seconds, they should earn 58.8 / 60 = 0.98 per second.
10 seconds later (t=50), they should have earned 9.8 tokens.
Actual Result:
claimable = (50/100 * 98.8) - 40 = 49.4 - 40 = 9.4 tokens.
User lost 0.4 tokens extra (a ~4.1% loss of the earned portion instead of 2%).

### Impact Explanation

Significant and silent loss of user tokens. The deviation increases the further along the user is in their vesting schedule. Users who split or merge frequently will see their total claimable tokens significantly decay beyond the intended fee rates.

### Likelihood Explanation

This affects every split and merge operation performed after a user has made their first claim. Since these are core management features, the likelihood is High.

### Proof of Concept

Add the following test to test/AlignerzVestingProtocolTest.t.sol and run `forge test --mt test_SplitFeeMathError -vv`:

```solidity
    function test_SplitFeeMathError() public {
        uint256 amount = 100e18;
        uint256 period = 100;
        uint256 startTime = block.timestamp;

        token.transfer(address(tvsManager), amount);

        // 1. Setup NFT
        vm.prank(address(alignerz));
        uint256 nftId = nft.mint(bidders[1]);
        vm.prank(address(alignerz));
        tvsManager.setTVS(nftId, amount, period, startTime, token, 0, false, 0);

        // 2. Warp to 40% of vesting and claim
        vm.warp(startTime + 40);
        vm.prank(bidders[1]);
        tvsManager.claimTokens(nftId);
        // 3. Set split fee to 2% and split 100%
        vm.prank(owner);
        tvsManager.setSplitFeeRate(200);
        uint256[] memory percentages = new uint256[](1);
        percentages[0] = 10000; // 100%
        vm.prank(bidders[1]);
        (, uint256[] memory newIds) = tvsManager.splitTVS(percentages, nftId);
        uint256 nextId = newIds[0];
        TVSManager.Allocation memory alloc = tvsManager.getAllocationOf(nextId);

        // Fee on unclaimed 60 is 1.2. New amount = 98.8.
        assertEq(alloc.amounts[0], 98.8e18);
        assertEq(alloc.claimedAmounts[0], 40e18);
        // 4. Warp to 50% of original vesting (10 seconds later)
        vm.warp(startTime + 50);
        uint256 claimable = tvsManager.getClaimableAmount(alloc, 0);
        // Current calculation: (50/100 * 98.8) - 40 = 9.4 tokens.
        // If scaled correctly, it should be 9.8 tokens.
        console.log("Claimable at t=50: %s (Expected ~9.8e18)", claimable);

        // This confirms the user lost 0.4 tokens due to unscaled claimedAmount
        assertEq(claimable, 9.4e18);
    }
```

### Recommendation

The claimedAmount must be scaled down proportionally whenever the total amount is reduced by fees.

In TVSManager.sol, when updating the allocation state after a split/merge fee is deducted, calculate:
newClaimedAmount = (oldClaimedAmount * newAmount) / oldAmount
(Rounding should be handled carefully, preferably rounding claimedAmount UP to favor the protocol and prevent over-claims).
