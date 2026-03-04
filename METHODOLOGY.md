## Datasets
The main dataset is the [Xatu beacon events dataset](https://ethpandaops.io/data/xatu/schema/beacon_api_/), which aggregates events from numerous beacon nodes across the Ethereum network to provide a global overview of what the network sees and when. In particular, the following tables will be used: 
- [`beacon_api_eth_v1_events_block_gossip`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_block_gossip): This table contains information about block discovery events in the Ethereum beacon network.
- [`beacon_api_eth_v1_events_blob_sidecar`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_blob_sidecar): This table contains information about blob sidecar discovery events in the Ethereum beacon network.

Another dataset is the SentryNet message dataset (`sentrynet_message_log`), which will be a log of boosted beacon block hashes, blob sidecar hashes, and their timestamps at each node. This will allow us to run a comparative analysis between boosted blocks and non-boosted blocks.

## Methodology
The core question that we'd like to answer is the following: _how much of a broadcast advantage does SentryNet offer to a proposer over the regular GossipSub network?_

We can break this down into the following sub-questions:
1. **Latency**: How fast can we propagate blocks (and their blob sidecars) compared to the regular GossipSub network?
2. **Effectiveness**: How much does latency advantage translate into proposer effectiveness? Metrics included here are **attestation rates** and **reorg rates**.
3. **Yield**: Given this increased effectiveness, how and by how much does this translate into increased validator yield?

What follows is the methodology breakdown for each question.

### Latency

#### Internal Latency
**Goal**

Measure the latency impact of the DoubleZero-based fast path compared to regular GossipSub _internally_.

**Description**

Every boosted message (identified by its hash) will have 2 timestamps recorded: the DZ arrival timestamp and the GossipSub arrival timestamp (baseline). These timestamps will be recorded in a Clickhouse database for later analysis (mean, median, p90, p99).

> [!NOTE]
> Since our nodes will have more aggressive GossipSub configurations that optimize for latency, this difference will not be entirely comparable with what a regular beacon node would see.

#### External Latency
**Goal**

Measure the latency impact of the DoubleZero-based fast path compared to regular GossipSub, as observed from external nodes.

**Description**

We restrict our sample size for $T_{start}$ to those that are propagated by our partner relay. Xatu's [`mev_relay_proposer_payload_delivered`](https://ethpandaops.io/data/xatu/schema/mev_relay_/#mev_relay_proposer_payload_delivered) table
contains data on when winning payloads where delivered to proposers, which approximates when they release them to the P2P network. For each sample (block & its blob columns), this will be the starting point. 

Measuring latency in a global P2P network is tricky, since there is no bird's eye view of the network and it all depends on the point of reference. Our global point of reference for the arrival times ($T_{end}$) will be the nodes in the Xatu dataset, which are sufficiently numerous and distributed to provide an accurate picture. We'll consult the following Xatu tables for this data:
- [`beacon_api_eth_v1_events_block_gossip`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_block_gossip): This table contains information about block discovery events in the Ethereum beacon network.
- [`beacon_api_eth_v1_events_blob_sidecar`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_blob_sidecar): This table contains information about blob sidecar discovery events in the Ethereum beacon network.

The final metric then will be the differences between $T_{start}$ and $T_{end}$ ($\Delta$), for both boosted and non-boosted blocks. With sufficient sample sizes (1 week of data), we will be able to draw a conclusion from these aggregate data points, even if we can't do a per-message comparison like we could with internal latency.

### Effectiveness
Effectiveness will be measured by 2 metrics:
- Attestation rate
- Reorg rate


### Yield
