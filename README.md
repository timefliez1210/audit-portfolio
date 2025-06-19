# Security Audit Portfolio

Welcome to my security audit portfolio repository. This collection showcases my participation in various smart contract security audits and highlights my contributions to securing blockchain projects.


## Overview

This repository contains detailed reports and findings from security audits I've participated in, demonstrating my expertise in smart contract security analysis and vulnerability detection.

## Statistics

- Total Audits Participated: [6]
- High Severity Findings: [10]
- Medium Severity Findings: [14]
- Low Severity Findings: [2]

## Audit Experience

Below is a chronological list of audits I've participated in, along with key findings:

### Alchemix V3 - May 2025 - TBD

- **Humble contribution of two lesser duplicated findings**
- **[Compiled Findings](./Alchemix-v3/Alchemix.md)**
- **[Contest Page](https://cantina.xyz/competitions/e68909e6-3491-4a94-a707-ecf0c89cf72a)**

### Regnum Aurum Aquisition Corp - February 2025 - TBD

- **A total of 20 Valid findings contributed**
- **[Compiled Findings](./RAAC/Raac-Core.md)**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-02-raac)**

### Gamma Limit Orders - April 2025 - unranked

- **Informational severity finding:** Non Inclusion of actual target tick in calculated tick range causes spread on limit orders.
- **[Compiled Findings](./Gamma-Limit-Orders/Gamma-Limit-Orders.md)**
- **[Contest Page](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)**

### Zaros Part 2 - January 2025 - 10th place

- **High severity finding:** `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while delaverage mode is triggered. This could lead to a failure in the deleveraging process, putting funds at risk.
- **Medium severity finding:** Assuming the proposed initial curve in `UsdTokenSwapConfig` for `getPremiumDiscountFactor` will get applied it will DoS the whole functions execution as soon as `vaultDebtUsdX18` has a negative value. This could halt key functionalities of the protocol.
- **[Compiled Findings](./Zaros/Zaros-Part-2.md)**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)**

### BENQI: Ignite - January 2025 - 4th place

- **Private Audit Competition**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)**

### Alchemix - Transmuter - December 2024 - unranked, first valid contest finding

- **Medium severity finding:** Incorrect asset accounting in `_harvestAndReport` function. This could lead to a miscalculation of rewards and potential loss of yield for users.
- **[Compiled Findings](./Alchemix-Transmuter/Alchemix-Transmuter.md)**
- **[Contest Page](https://codehawks.cyfrin.io/c/2024-12-alchemix/results)**

## Tools & Methodologies

Common tools and approaches I use:
- Manual Code Review
- Symbolic Execution and Formal Verification (Certora, Halmos)
- Stateless and Stateful Fuzzing (Foundry, Medusa)

## Contact

For audit inquiries or security consultations:
- Twitter: [@0xTimefliez](https://x.com/0xTimefliez)
- Email: [crfabig@gmail.com](mailto:crfabig@gmail.com)
