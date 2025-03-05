# Security Audit Portfolio

Welcome to my security audit portfolio repository. This collection showcases my participation in various smart contract security audits and highlights my contributions to securing blockchain projects.

## 🔍 Overview

This repository contains detailed reports and findings from security audits I've participated in, demonstrating my expertise in smart contract security analysis and vulnerability detection.

## 📊 Statistics

- Total Audits Participated: [6]
- High Severity Findings: [1]
- Medium Severity Findings: [2]
- Low Severity Findings: [1]

## 🏆 Audit Experience

Below is a chronological list of audits I've participated in, along with key findings:

### Zaros Part 2 - January 2025 

- High severity finding: getAutoDeleverageFactor in Market.sol can return 1 even if it should never, while delaverage mode is triggered

- Medium severity finding: Assuming the proposed initial curve in UsdTokenSwapConfig for getPremiumDiscountFactor will get applied it will DoS the whole functions execution as soon as vaultDebtUsdX18 has a negative value

[Compiled Findings](./Zaros/Zaros-Part-2.md)

[Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)

### BENQI: Ignite - January 2025 - 4th place

- Private Audit

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

