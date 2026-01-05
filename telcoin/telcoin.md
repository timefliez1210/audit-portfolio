# Telcoin - Findings Report

## Table of Contents
- Informational Findings
    - [I-01. Potential DoS during adversarial network conditions due to lack of fallback leader](#I-01)

---

## Contest Summary

**Sponsor:** Telcoin

**Dates:** TBD

---

## Results Summary

| Severity      | Count |
|---------------|-------|
| High          | 0     |
| Medium        | 0     |
| Low           | 0     |
| Informational | 1     |

---

# Informational Findings

## <a id='I-01'></a>I-01. Potential DoS during adversarial network conditions due to lack of fallback leader

### Summary
Referring to the original papers of Bullshark here and here the proof of liveness, and difference to previous DAG based consensus protocols, is that Bullshark can work asynchronous and in adversarial network conditions, however the Telcoin Network fails to implement it.

### Finding Description
As a brief summary of the paper, it is notable that, the partially synchronous version of Bullshark, still requires backup leaders which are randomly elected to guarantee liveness during adversarial or asynchronous network conditions.

Quote from the Original Paper Bullshark: DAG BFT Protocols Made Practical:

"[...] A further challenge is to take advantage of a common-case synchronous network without sacrificing latency in the asynchronous worst case. To this end, BullShark introduces two types of votes - steady-state for the predefined leader and fallback for the random one. Similarly to DAG-Rider [25], BullShark rounds are grouped in waves, each of which consists of 4 rounds. Intuitively, each wave encodes a consensus logic. The first round of a wave has two potential leaders - a predefined steady-state leader and a leader that is chosen in retrospect by the randomness produced in the fourth round of the wave.[...]"

The Random Leader selection guarantees liveness, in case a Node delivers it's vertex extremely late, not at all, or intentionally withholds it from the network. Such a scenario could brick the progress of the network to a point of entirely halting it.

### Impact Explanation
A high impact seems suitable for a complete shutdown of operations.

### Likelihood Explanation
Basically no pre-conditions have to be met, any adversarial node or DoSed node could cause a network breakdown.

### Recommendation
Implement a fallback leader to ensure liveness during adversarial or extended asynchronous network periods.