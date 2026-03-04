## Datasets
The main dataset is the [Xatu beacon events dataset](https://ethpandaops.io/data/xatu/schema/beacon_api_/), which aggregates events from numerous beacon nodes across the Ethereum network to provide a global overview of what the network sees and when. In particular, the following tables will be used: 
- [`beacon_api_eth_v1_events_block_gossip`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_block_gossip): This table contains information about block discovery events in the Ethereum beacon network.
- [`beacon_api_eth_v1_events_data_column_sidecar`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_data_column_sidecar): This table contains information about blob column sidecar discovery events in the Ethereum beacon network. This is important because each validator must receive a large enough fraction of these columns before it will attest to a block as valid.
- [`mev_relay_proposer_payload_delivered`](https://ethpandaops.io/data/xatu/schema/mev_relay_/#mev_relay_proposer_payload_delivered): This table contains information about when a relay delivered a winning payload to the proposer, which approximates the time at which that payload was released to the P2P network.

Another dataset is the SentryNet message dataset (`sentrynet_message_log`), which will be a log of boosted beacon block hashes, blob sidecar hashes, and their timestamps at each node. This will allow us to run a comparative analysis between boosted blocks and non-boosted blocks. This log
will be stored in Clickhouse.

## Methodology
The core question that we'd like to answer is the following: _how much of a broadcast advantage does SentryNet offer to a proposer over the regular GossipSub network?_

We can break this down into the following sub-questions:
1. **Latency**: How fast can we propagate blocks (and their blob sidecars) compared to the regular GossipSub network?
2. **Effectiveness**: How much does latency advantage translate into proposer effectiveness? Metrics included here are **attestation rates** and **reorg rates**.
3. **Yield**: Given this increased effectiveness, how and by how much does this translate into increased validator yield?

What follows is the methodology breakdown for each question.

### Latency

#### Internal Latency
_Goal_

Measure the latency impact of the DoubleZero-based fast path compared to regular GossipSub _internally_.

_Description_

Every boosted message (identified by its hash) will have 2 timestamps recorded: the DZ arrival timestamp and the GossipSub arrival timestamp (baseline). These timestamps will be recorded in the `sentrynet_message_log` and can be used to analyze mean, median, p90 and p99 diffs at each node.

> [!NOTE]
> Since our nodes will have more aggressive GossipSub configurations that optimize for latency, this difference will not be entirely comparable with what a regular beacon node would see.

#### External Latency
_Goal_

Measure the latency impact of the DoubleZero-based fast path compared to regular GossipSub, as observed from external nodes.

_Description_

We restrict our sample size for $T_{start}$ to those that are propagated by our partner relay. Xatu's [`mev_relay_proposer_payload_delivered`](https://ethpandaops.io/data/xatu/schema/mev_relay_/#mev_relay_proposer_payload_delivered) table
contains data on when winning payloads where delivered to proposers, which approximates when they release them to the P2P network. For each sample (block & its blob columns), this will be the starting point. 

Measuring latency in a global P2P network is tricky, since there is no bird's eye view of the network and it all depends on the point of reference. Our global point of reference for the arrival times ($T_{end}$) will be the nodes in the Xatu dataset, which are sufficiently numerous and distributed to provide an accurate picture. We'll consult the following Xatu tables for this data:
- `beacon_api_eth_v1_events_block_gossip`: for block arrival times
- `beacon_api_eth_v1_events_data_column_sidecar`: for blob column arrival times

The final metric then will be the differences between $T_{start}$ and $T_{end}$ ($\Delta$), for both boosted and non-boosted blocks. With sufficient sample sizes (1 week of data), we will be able to draw a conclusion from these aggregate data points, even if we can't do a per-message comparison like we could with internal latency.

### Effectiveness

_Goal_

Measure how latency improvements translate to effectiveness metrics like correct head vote rates and reorg rates.

_Description_

We consult historical beacon chain data from the Beacon-API for the following metrics:
- **Correct head vote rate**: the number of same-slot, correct head votes as a fraction of the assigned committee for that slot (inclusion delay = 1).
- **Reorg rate**: aggregate metric for reorged blocks. Number of boosted blocks that got reorged out / total boosted blocks, compared to the same ratio for non-boosted blocks.

We can then compare both of these rates for blocks that we boosted, vs. the baseline, non-boosted blocks.
We use the same control group as the latency study (i.e. blocks from the same relay). These metrics will also be categorized by number of blobs
in the block, because we expect there to be a correlation.

> [!NOTE]
> Reorgs are rare events, so this will require a longer data collection period.

### Yield

_Goal_

Quantify how SentryNet's latency advantage translates into increased validator yield, and explore how timing games can further amplify this effect.

_Description_

Yield improvements come from two sources:

#### 1. Reorg Protection
Faster propagation does not directly increase the proposer's consensus rewards — attestation rewards accrue to the attesters, and inclusion rewards go to the next slot's proposer. However, faster propagation does increase the fork choice weight behind the proposed block, reducing the probability of it being reorged out. Since a reorg causes the proposer to lose all execution layer rewards (MEV tips + priority fees) for that slot, reorg protection preserves the proposer's most valuable revenue stream.

We estimate this by comparing the reorg rate of boosted vs. non-boosted blocks (from the Effectiveness section) and multiplying the reorg rate delta by the average block value:

$$\Delta_{yield} = \Delta_{reorg} \times \overline{V_{block}}$$

where $\overline{V_{block}}$ is the average execution layer value per block. This gives us a per-slot expected value gain that can be extrapolated over the proposer's expected number of slots per year.

#### 2. Timing Games
If SentryNet can reliably propagate blocks faster, proposers can afford to delay their proposal time (e.g. from $t=1s$ to $t=2s$ into the slot) to capture more MEV value from the builder auction, without sacrificing attestation inclusion. This is the core timing game tradeoff: later proposals capture more value but risk losing attestations.

We measure this by:
- Varying proposal times for SentryNet-boosted validators across controlled intervals (e.g. $t=1s, 1.5s, 2s, 2.5s$).
- For each interval, recording the correct head vote rate and the block value (execution layer rewards + MEV tips) from the relay payload.
- Plotting the Pareto frontier: the optimal proposal delay that maximizes total yield (consensus rewards + execution rewards) without degrading attestation performance.

The control group remains non-boosted blocks from the same relay, proposed at their natural times.

> [!NOTE]
> Timing game experiments require coordination with partner validators and will be conducted in controlled phases to avoid unnecessary risk.
