# Security Audit Portfolio

This Repo gives you a quick overview of what I have done after school and keeps track of my contributions to make web3 a safer place.

## Public Profiles

CodeHawks [0xTimefliez](https://profiles.cyfrin.io/u/0xtimefliez)

Cantina [timefliez](https://cantina.xyz/u/timefliez)

## A Quick about Me

2009 - Finished School

2009/2010 - Mandatory Military Service

2010 - 2014 - Studied Physics at the Humboldt University in Berlin

2015/2016 - Worked at Corporate Consultancy

2016 - 2020 - Freelance Frontend Work with focus on SEO Performance

2021 - 2024 - Dive Instructor for Recreational and Technical Diving

2023 - 2024 - Transitioned into an own Diving Business with Landside Property and Boat

September 2024 - Sold the Diving Business and started Web3 Security

And now here I am:

## My Top 3 Findings:

1. **From Ammalgam DLEX on Cantina:**

  Naturally occurring DoS of _repay() due to rounding in calcTrancheAtStartOfLiquidation(). [Find the details here](./Top3/first.md)

2. **From Zaros Part 2 on Codehawks:**

  `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while delaverage mode is triggered. This could lead to a failure in the deleveraging process, putting funds at risk. [Find the details here](./Top3/second.md)

3. **From Zaros Part 2 on Codehawks:**

  Assuming the proposed initial curve in `UsdTokenSwapConfig` for `getPremiumDiscountFactor` will get applied it will DoS the whole functions execution as soon as `vaultDebtUsdX18` has a negative value. This could halt key functionalities of the protocol. [Find the details here](./Top3/third.md)

## Statistics

- Total Audits Participated: [9]
- High Severity Findings: [15]
- Medium Severity Findings: [14]
- Low Severity Findings: [3]

## Contest Experience

Below are the contests I have participated in, in chronological order

### **Ammalgam DLEX - July 2025**

- Contribution of 1 High Severity (Solo) and 1 Info Finding

- **Solo High severity finding:**

  Naturally occurring DoS of _repay() due to rounding in calcTrancheAtStartOfLiquidation().

- [Compiled Findings](./Ammalgam/ammalgam.md)
- [Contest Page](https://cantina.xyz/code/02c29467-cb27-4beb-b2ef-500ad95e1a51/overview)

### Telcoin Network - July 2025

### **Jigsaw Protocol - June 2025**

- Contribution of 3 High Severity and 1 Low Severity Finding
- [Compiled Findings](./Jigsaw/Jigsaw.md)
- [Contest Page](https://cantina.xyz/competitions/7a40c849-0b35-4128-b084-d9a83fd533ea)

### **Alchemix V3 - May 2025**

- 1 High and 1 Medium Severity Finding
- [Compiled Findings](./Alchemix-v3/Alchemix.md)
- [Contest Page](https://cantina.xyz/competitions/e68909e6-3491-4a94-a707-ecf0c89cf72a)

### **Gamma Limit Orders - April 2025**

- 1 Informational Finding
- [Compiled Findings](./Gamma-Limit-Orders/Gamma-Limit-Orders.md)
- [Contest Page](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)

### **Regnum Aurum Aquisition Corp - February 2025**

- A total of 20 Valid findings contributed
- [Compiled Findings](./RAAC/Raac-Core.md)
- [Contest Page](https://codehawks.cyfrin.io/c/2025-02-raac)


### **Zaros Part 2 - January 2025**

- **High severity finding:**

  `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while delaverage mode is triggered. This could lead to a failure in the deleveraging process, putting funds at risk.
- **Medium severity finding:**

  Assuming the proposed initial curve in `UsdTokenSwapConfig` for `getPremiumDiscountFactor` will get applied it will DoS the whole functions execution as soon as `vaultDebtUsdX18` has a negative value. This could halt key functionalities of the protocol.
- [Compiled Findings](./Zaros/Zaros-Part-2.md)
- [Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)

### **BENQI: Ignite - January 2025 - 4th place**

- Private Audit Competition
- [Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)

### **Alchemix - Transmuter - December 2024**

- Contribution of 1 Medium Severity Finding
- [Compiled Findings](./Alchemix-Transmuter/Alchemix-Transmuter.md)
- [Contest Page](https://codehawks.cyfrin.io/c/2024-12-alchemix/results)

## Tools & Methodologies

Common tools and approaches I use:
- Manual Code Review
- Symbolic Execution and Formal Verification (Certora, Halmos)
- Stateless and Stateful Fuzzing (Foundry, Medusa)

## Contact

For audit inquiries or security consultations:
- Twitter: [@0xTimefliez](https://x.com/0xTimefliez)
- Email: [crfabig@gmail.com](mailto:crfabig@gmail.com)
