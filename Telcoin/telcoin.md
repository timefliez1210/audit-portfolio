# High 1 - Current timestamp validation can cause liveness and safety issues

## Summary

In the current implementation of Narwhal the truncation of time in ```now()``` in combination with the addition of 1 second within the creation of blocks can cause severe liveness and safety issues potentially resulting in forks if a validator is merely off by nanoseconds.

## Finding Description

Consider the following code:

```./crates/batch-builder/src/batch.rs```:

```rust
pub fn build_batch<P: TxPool>(

	// ***** SNIP ***** //

@>  let mut timestamp = now(); // @audit non deterministic
@>  if timestamp == parent_info.timestamp {
@>      warn!(target: "worker::batch_builder", "new block timestamp same as parent - setting offset by 1sec");
@>      timestamp = parent_info.timestamp + 1;
@>  }

    // batch
    let batch = Batch {
        transactions,
        parent_hash,
        beneficiary,
        timestamp,
        base_fee_per_gas,
        worker_id,
        received_at: None,
    };
    // return output
    BatchBuilderOutput { batch, mined_transactions }
}
```

```./crates/consensus/primary/src/network/handler.rs```:

```rust
		let now = now();
        if &now < header.created_at() {
            // wait if the difference is small enough
            if *header.created_at() - now
                <= self
                    .consensus_config
                    .network_config()
                    .sync_config()
                    .max_header_time_drift_tolerance
            {
      tokio::time::sleep(Duration::from_secs(*header.created_at() - now)).await;
            } else {
                // created_at is too far in the future
                warn!(
                    "Rejected header {:?} due to timestamp {} newer than {now}",
                    header,
                    *header.created_at()
                );

                return Err(HeaderError::InvalidTimestamp {
                    created: *header.created_at(),
                    received: now,
                }
                .into());
            }
        }
```

and ```./crates/types/src/primary/mod.rs```:

```rust
pub fn now() -> TimestampSec {
    match SystemTime::now().duration_since(SystemTime::UNIX_EPOCH) {
@>      Ok(n) => n.as_secs() as TimestampSec,
        Err(_) => panic!("SystemTime before UNIX EPOCH!"),
    }
}
```

Considering above code here is a scenario:

#### Scenario

1. We assume a header r-2 created at timestamp 10 from 10.00001 original precision.
2. A header at r-1 with timestamp 11 with 10.6 (randomly chosen as example)
	The header at r-1 will be 11 since the logic in code snippet 1.
What will happen now:
At round r an honest node proposes a header from the system time 11.0 exact, so proposed header will be dated on 12.0 and sends those header with a vote request through the network, however, there is a very realistic scenario in which many honest nodes of the network have a time of 10.99999 or slightly less, due to the truncation in the now() function this would be rounded towards 10.0 leading to an honest node rejecting another honest nodes header.

This scenario describes only the smallest possible time difference, however it is worth noting, that during extended asynchronous periods in the distributed network, this issue amplifies very rapidly.

Lastly it is worth noting, that Narwhals Paper and Original Implementation does not include any timestamp related checks, and timeouts are integrated into bullshark, which in fact means that, even if the Network does not fork under above circumstances, the continuous extension of the DAG would be interrupted by timeouts/not signing valid headers within Narwhal, which would ultimately nullify safety and liveness guarantees.
## Impact Explanation

The impact affects liveness and more importantly safety. Due to non-deterministic use of SystemTime during critical validation processes, honest nodes in the scenario above, can have different views of the DAG potentially leading to forks.

## Likelihood Explanation

System Clocks on different computer within a distributed network are expected to have time differences, especially that small.


## Recommendation

Since this is a complex issue, and I do not fully understand, why the system would need this sort of timestamp validation, my recommendation would be to completely remove it out of the Narwhal Integration (fall back to original and proofen implementation) and deal with the time, if necessary, within the Bullshark implementation, which is "eventually synchronous".
