# Security Audit Portfolio

Welcome to my security audit portfolio repository. This collection showcases my participation in various smart contract security audits and highlights my contributions to securing blockchain projects.

## 🔍 Overview

This repository contains detailed reports and findings from security audits I've participated in, demonstrating my expertise in smart contract security analysis and vulnerability detection.

## 📊 Statistics

- Total Audits Participated: [5]
- High Severity Findings: [1]
- Medium Severity Findings: [2]
- Low Severity Findings: [1]

## 🏆 Audit Experience

Below is a chronological list of audits I've participated in, along with key findings:

### Gamma Limit Orders - April 2025

- Informational severity finding: Non Inclusion of actual target tick in calculated tick range causes spread on limit orders

[Compiled Findings](./Gamma-Limit-Orders/1-Gamma-Limit-Orders.md)

[Contest Page](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)

### Regnum Aurum Aquisition Corp - February 2025 - 

[Compiled Findings](./Zaros/Zaros-Part-2.md)

[Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)

### Zaros Part 2 - January 2025 - 10th place

- High severity finding: getAutoDeleverageFactor in Market.sol can return 1 even if it should never, while delaverage mode is triggered

- Medium severity finding: Assuming the proposed initial curve in UsdTokenSwapConfig for getPremiumDiscountFactor will get applied it will DoS the whole functions execution as soon as vaultDebtUsdX18 has a negative value

[Compiled Findings](./Zaros/Zaros-Part-2.md)

[Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)

### BENQI: Ignite - January 2025 - 4th place

- Private Audit Competition

[Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)


### Alchemix - Transmuter - December 2024 - unranked, first valid contest finding

- Medium severity finding: Incorrect asset accounting in ```_harvestAndReport``` function. 

[Compiled Findings](./Alchemix-Transmuter/1-Alchemix-Transmuter.md)

[Contest Page](https://codehawks.cyfrin.io/c/2024-12-alchemix/results)



## 🛠 Tools & Methodologies

Common tools and approaches I use:
- Manual Code Review
- Static Analysis Tools
- Symbolic Execution and Formal Verification
- Stateless and StatefulFuzzing


## 📫 Contact

For audit inquiries or security consultations:
- Twitter: [@0xTimefliez](https://x.com/0xTimefliez)
- Email: [crfabig@gmail.com](mailto:crfabig@gmail.com)

