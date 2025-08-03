# Security Audit Portfolio

This Repo gives you a quick overview of what I have done after school and keeps track of my humble contributions to make web3 a safer place.

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

1. From Zaros Part 2 on Codehawks:

  `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while delaverage mode is triggered. This could lead to a failure in the deleveraging process, putting funds at risk.

2. From Ammalgam DLEX on Cantina:

  Repayment of debt blocked incl. liquidations.

3. From Zaros Part 2 on Codehawks:

  Assuming the proposed initial curve in `UsdTokenSwapConfig` for `getPremiumDiscountFactor` will get applied it will DoS the whole functions execution as soon as `vaultDebtUsdX18` has a negative value. This could halt key functionalities of the protocol.

## Statistics

- Total Audits Participated: [7]
- High Severity Findings: [13]
- Medium Severity Findings: [14]
- Low Severity Findings: [3]

## Audit Experience

Below are the contests I have participated in, in chronological order

### Ammalgam DLEX - July 2025

### Telcoin Network - July 2025

### Jigsaw Protocol - June 2025

- **Humble contribution of 3 High Severity and 1 Low Severity Finding**
- **[Compiled Findings](./Jigsaw/Jigsaw.md)**
- **[Contest Page](https://cantina.xyz/competitions/7a40c849-0b35-4128-b084-d9a83fd533ea)**

### Alchemix V3 - May 2025

- **Humble contribution of two lesser duplicated findings**
- **[Compiled Findings](./Alchemix-v3/Alchemix.md)**
- **[Contest Page](https://cantina.xyz/competitions/e68909e6-3491-4a94-a707-ecf0c89cf72a)**

### Gamma Limit Orders - April 2025

- **Informational severity finding:** Non Inclusion of actual target tick in calculated tick range causes spread on limit orders.
- **[Compiled Findings](./Gamma-Limit-Orders/Gamma-Limit-Orders.md)**
- **[Contest Page](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)**

### Regnum Aurum Aquisition Corp - February 2025

- **A total of 20 Valid findings contributed**
- **[Compiled Findings](./RAAC/Raac-Core.md)**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-02-raac)**


### Zaros Part 2 - January 2025

- **High severity finding:** `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while delaverage mode is triggered. This could lead to a failure in the deleveraging process, putting funds at risk.
- **Medium severity finding:** Assuming the proposed initial curve in `UsdTokenSwapConfig` for `getPremiumDiscountFactor` will get applied it will DoS the whole functions execution as soon as `vaultDebtUsdX18` has a negative value. This could halt key functionalities of the protocol.
- **[Compiled Findings](./Zaros/Zaros-Part-2.md)**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)**

### BENQI: Ignite - January 2025 - 4th place

- **Private Audit Competition**
- **[Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)**

### Alchemix - Transmuter - December 2024

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
