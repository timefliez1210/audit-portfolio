# Security Audit Portfolio

This repo provides a quick overview of my journey as a security researcher in the web3 space.

---

## 🔗 Public Profiles

| Platform | Profile |
|----------|---------|
| CodeHawks | [0xTimefliez](https://profiles.cyfrin.io/u/0xtimefliez) |
| Cantina | [timefliez](https://cantina.xyz/u/timefliez) |

---

## 🏆 Top 3 Findings

1. **Solo Medium - Ammalgam DLEX on Cantina**  
   Naturally occurring DoS of `_repay()` due to rounding in `calcTrancheAtStartOfLiquidation()`.  
   → [View Finding](./top3/first.md)

2. **Solo High - Zaros Part 2 on Codehawks**  
   `getAutoDeleverageFactor` in `Market.sol` can return 1 even if it should never, while deleverage mode is triggered.  
   → [View Finding](./top3/second.md)

3. **Solo Medium - Zaros Part 2 on Codehawks**  
   Proposed initial curve in `UsdTokenSwapConfig::getPremiumDiscountFactor` will DoS execution when `vaultDebtUsdX18` is negative.  
   → [View Finding](./top3/third.md)

---

## 📊 Statistics

| Metric | Count |
|--------|-------|
| Total Audits Participated | 9 |
| High Severity Findings | 15 |
| Medium Severity Findings | 15 |
| Low Severity Findings | 2 |
| Informational Findings | 3 |

---

## 📋 Contest Experience

Contests listed in chronological order (newest first):

### Kuru CLOB — August 2025
- **Findings:** 1 High, 1 Medium
- [Compiled Findings](./kuru/kuru.md) | [Contest Page](https://cantina.xyz/code/cdce21ba-b787-4df4-9c56-b31d085388e7/overview/leaderboard)

### Ammalgam DLEX — July 2025 (4th Place 🏅)
- **Findings:** 1 Medium (Solo), 1 Informational
- **Highlight:** Solo Medium - DoS of `_repay()` due to rounding
- [Compiled Findings](./ammalgam/ammalgam.md) | [Contest Page](https://cantina.xyz/code/02c29467-cb27-4beb-b2ef-500ad95e1a51/overview)

### Telcoin — June 2025
- **Findings:** 1 Informational
- [Compiled Findings](./telcoin/telcoin.md) | [Contest Page](https://cantina.xyz/competitions/26d5255b-6f68-46cf-be55-81dd565d9d16)

### Jigsaw Protocol — May 2025
- **Findings:** 3 High, 1 Low
- [Compiled Findings](./jigsaw/jigsaw.md) | [Contest Page](https://cantina.xyz/competitions/7a40c849-0b35-4128-b084-d9a83fd533ea)

### Alchemix V3 — May 2025
- **Findings:** 1 High, 1 Medium
- [Compiled Findings](./alchemix_v3/alchemix.md) | [Contest Page](https://cantina.xyz/competitions/e68909e6-3491-4a94-a707-ecf0c89cf72a)

### Gamma Limit Orders — April 2025
- **Findings:** 1 Informational
- [Compiled Findings](./gamma_limit_orders/gamma_limit_orders.md) | [Contest Page](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)

### Regnum Aurum Acquisition Corp — February 2025
- **Findings:** 20 Valid (8 High, 11 Medium, 1 Low)
- [Compiled Findings](./raac/raac_core.md) | [Contest Page](https://codehawks.cyfrin.io/c/2025-02-raac)

### Zaros Part 2 — January 2025
- **Findings:** 1 High, 1 Medium
- **Highlights:**
  - High: `getAutoDeleverageFactor` can return 1 during deleverage mode
  - Medium: Premium discount factor curve DoS on negative vault debt
- [Compiled Findings](./zaros/zaros_part_2.md) | [Contest Page](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results)

### BENQI: Ignite — January 2025 (4th Place 🏅)
- Private Audit Competition
- [Contest Page](https://codehawks.cyfrin.io/c/2025-01-benqi/results)

### Alchemix Transmuter — December 2024
- **Findings:** 1 Medium
- [Compiled Findings](./alchemix_transmuter/alchemix_transmuter.md) | [Contest Page](https://codehawks.cyfrin.io/c/2024-12-alchemix/results)

---

## 🛠 Tools & Methodologies

| Category | Tools |
|----------|-------|
| Manual Review | Deep code analysis, threat modeling |
| Formal Verification | Certora, Halmos |
| Fuzzing | Foundry, Medusa (stateless & stateful) |

---

## 📬 Contact

For audit inquiries or security consultations:

| Platform | Contact |
|----------|---------|
| Twitter | [@0xTimefliez](https://x.com/0xTimefliez) |
| Email | [crfabig@gmail.com](mailto:crfabig@gmail.com) |
