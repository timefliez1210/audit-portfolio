# Issues Review for timefliez1210

## Issue # [000360] H15 Interface Mismatch DoS
- **Issue Folder:** H-9
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No
- **Tag:** finding_allocationOf_interface_mismatch_the_mapping
- **Likelihood:** High: Impossible to set dividends
- **Impact:** High, distributing dividends doesn't work

## Issue # [000361] M01 DoS in DividendDistributor
- **Issue Folder:** H-6
- **Status:** ✅ Valid
- **Guard Severity:** medium
- **Judge Severity:** High
- **Severity Changed:** ⚠️ Yes (from medium to High)

## Issue # [000362] H14 Inverted Logic in Distributor
- **Issue Folder:** H-12
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000363] H13 Cross Project Merge Dividend Theft
- **Issue Folder:** I-design_choice
- **Status:** ❌ Invalid
- **Guard Severity:** high
- **Judge Severity:** _Not assigned_
- **Severity Changed:** _N/A_
- **Judge Comment:** Dividends are for all NFT holders (A26Z token), merging between project is allowed.

## Issue # [000364] H12 Incorrect Split Logic leads to Compounding Reduction
- **Issue Folder:** H-1
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000365] L01 Merge does not check for duplicate NFTs to be passed, potentially locking user funds
- **Issue Folder:** I-user_mistake
- **Status:** ❌ Invalid
- **Guard Severity:** low
- **Judge Severity:** _Not assigned_
- **Severity Changed:** _N/A_

## Issue # [000366] H03 DoS in claimTokens for Multi-Flow Vesting
- **Issue Folder:** H-4
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000367] H11 Cumulative fee subtraction in FeesManager causes loss of funds
- **Issue Folder:** H-5
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000368] H10 Uninitialized memory in _computeSplitArrays causes DoS in splitTVS
- **Issue Folder:** H-3
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000369] H09 Missing loop increment in FeesManager causes infinite loop and underflow
- **Issue Folder:** H-7
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000370] H08 Unallocated memory in FeesManager causes DoS in splitTVS and mergeTVS
- **Issue Folder:** H-2
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000371] H07 DoS in A26ZDividendDistributor due to O(NM) complexity
- **Issue Folder:** H-6
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000372] H06 Infinite loop in getUnclaimedAmounts in A26ZDividendDistributor causes DoS
- **Issue Folder:** H-8
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** High
- **Severity Changed:** No

## Issue # [000373] H05 getToalUnclaimedAmounts in A26ZDividendDistributor incorrectly loops from NFT ID 0
- **Issue Folder:** M-4
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** Medium
- **Severity Changed:** ⚠️ Yes (from high to Medium)

## Issue # [000374] H04 _setDividends in A26ZDividentDistributor incorrectly loops from NFT ID 0
- **Issue Folder:** M-4
- **Status:** ✅ Valid
- **Guard Severity:** high
- **Judge Severity:** Medium
- **Severity Changed:** ⚠️ Yes (from high to Medium)

## Issue # [000375] H02 Immediate fee transfer in updateBid prevents full refunds
- **Issue Folder:** I-design_choice
- **Status:** ❌ Invalid
- **Guard Severity:** high
- **Judge Severity:** _Not assigned_
- **Severity Changed:** _N/A_
- **Judge Comment:** Those fees are not included in the refund.

## Issue # [000376] H01 Immediate fee transfer in placeBid prevents full refunds
- **Issue Folder:** I-design_choice
- **Status:** ❌ Invalid
- **Guard Severity:** high
- **Judge Severity:** _Not assigned_
- **Severity Changed:** _N/A_
- **Judge Comment:** Those fees are not included in the refund.
