# SentryNet Data Room: Methodology

## 1. Overview

The core question we aim to answer is:

> _How much of a broadcast advantage does SentryNet offer to a proposer over the regular GossipSub network?_

We decompose this into three layers of analysis:

1. **Latency**: How fast can we propagate blocks and their data column sidecars compared to regular GossipSub?
2. **Effectiveness**: How does this latency advantage translate into proposer effectiveness (correct head vote rates, reorg rates)?
3. **Yield**: How does improved effectiveness translate into increased validator yield, and how can timing games amplify this?

Each layer builds on the previous: latency drives effectiveness, effectiveness drives yield.

## 2. Notation

| Symbol | Description | Domain |
|---|---|---|
| $s$ | Slot index | $\mathbb{N}$ |
| $b$ | Block identifier (block root) | — |
| $c$ | Data column sidecar index for block $b$ | — |
| $\mathcal{B}$ | Set of sampled blocks | — |
| $\mathcal{J}$ | Set of external observer nodes (Xatu vantage points) | — |
| $G_b$ | Group label for block $b$ | $\{\text{boosted},\ \text{control}\}$ |
| $S_b$ | Start timestamp — approximate network release time of block $b$ | milliseconds |
| $A_{b,j}$ | Arrival timestamp of block $b$ at vantage point $j \in \mathcal{J}$ | milliseconds |
| $A^{(c)}_{b,j}$ | Arrival timestamp of column sidecar $c$ for block $b$ at vantage point $j$ | milliseconds |
| $L_{b,j}$ | Block propagation latency: $A_{b,j} - S_b$ | milliseconds |
| $L^{(c)}_{b,j}$ | Column propagation latency: $A^{(c)}_{b,j} - S_b$ | milliseconds |
| $n$ | SentryNet internal node index | — |
| $A^{DZ}_{m,n}$ | Arrival time of message $m$ at node $n$ via DoubleZero | milliseconds |
| $A^{GS}_{m,n}$ | Arrival time of message $m$ at node $n$ via GossipSub | milliseconds |
| $D_{m,n}$ | Internal latency gain: $A^{GS}_{m,n} - A^{DZ}_{m,n}$ | milliseconds |
| $H_b$ | Correct head vote rate for block $b$ | $[0,1]$ |
| $R_b$ | Reorg indicator for block $b$ | $\{0,1\}$ |
| $V_b$ | Execution-layer value of block $b$ (priority fees + MEV) | ETH |
| $K_b$ | Blob count for block $b$ | $\mathbb{N}_0$ |
| $d$ | Proposer delay (ms into slot at which the block is released) | milliseconds |
| $\alpha$ | Significance level for hypothesis tests | $0.05$ |

**Conventions.** All timestamps use a single time basis (Unix milliseconds). Unless stated otherwise, external latency uses observations indexed by $(b, j)$; effectiveness and yield use block-level observations indexed by $b$. Results are reported overall and stratified by blob count $K_b$ where relevant.

## 3. Datasets

### 3.1 Xatu Beacon Events

The [Xatu dataset](https://ethpandaops.io/data/xatu/schema/beacon_api_/) aggregates events from numerous beacon nodes across the Ethereum network. We use the following tables:

| Table | Purpose |
|---|---|
| [`beacon_api_eth_v1_events_block_gossip`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_block_gossip) | Block arrival times $A_{b,j}$ at Xatu observer nodes |
| [`beacon_api_eth_v1_events_data_column_sidecar`](https://ethpandaops.io/data/xatu/schema/beacon_api_/#beacon_api_eth_v1_events_data_column_sidecar) | Column sidecar arrival times $A^{(c)}_{b,j}$. Validators must receive a sufficient fraction of columns before attesting. |
| [`mev_relay_proposer_payload_delivered`](https://ethpandaops.io/data/xatu/schema/mev_relay_/#mev_relay_proposer_payload_delivered) | Relay payload delivery timestamps, used to approximate network release time $S_b$ |

### 3.2 SentryNet Message Log

The `sentrynet_message_log` (stored in ClickHouse) records, for each boosted message $m$ and each SentryNet node $n$:
- The DoubleZero arrival timestamp $A^{DZ}_{m,n}$
- The GossipSub arrival timestamp $A^{GS}_{m,n}$

This enables paired, per-message comparison of the two propagation paths.

## 4. Control Group & Confounders

### 4.1 Control Group Definition

Because boosting is not randomly assigned, careful control group selection is essential. We define the **control group** as blocks that are:

1. Sourced from the **same relay(s)** used for boosted blocks (controlling for builder/relay differences),
2. Within the **same observation window** (controlling for time-varying network conditions), and
3. **Stratified by blob count** $K_b$ (controlling for data availability load).

Formally, each block $b \in \mathcal{B}$ is assigned a group label $G_b \in \{\text{boosted}, \text{control}\}$.

### 4.2 Confounders & Mitigations

| Confounder | Risk | Mitigation |
|---|---|---|
| Relay/builder effects | Different relays have different delivery timing and value distributions | Eliminated by randomized boosting within the same relay |
| Time-varying network conditions | Propagation depends on global load and daily patterns | Eliminated by randomized boosting (balanced in expectation); additionally stratify by time-of-day |
| Block characteristics | Blob count and block size affect propagation speed | Eliminated by randomized boosting (balanced in expectation); additionally stratify by blob count $K_b$ for subgroup analysis |
| Proposer infrastructure | SentryNet proposers may have better setups independent of boosting | Eliminated by randomized boosting, both groups contain the same proposer population |

### 4.3 Randomized Boosting

To move beyond associational results, we employ a **randomized controlled design** for blocks sourced from our partner relay. For each incoming block, a hash-based coin flip determines whether it is boosted:

$$G_b = \begin{cases} \text{boosted} & \text{if } \text{hash}(b) \mod 100 < X \\ \text{control} & \text{otherwise} \end{cases}$$

where $X$ is the target boost percentage (tunable). Because the block hash is independent of proposer identity, infrastructure, blob count, and all other confounders, this produces groups that are balanced in expectation.

### 4.4 Interpretation

With randomized boosting, results from the external latency, effectiveness, and yield analyses can be interpreted as **causal**: any observed difference between groups is attributable to the boosting itself, not to confounders.

Internal latency is causal by construction (paired measurement: same message, same node, two paths).

For timing-game experiments with controlled assignment of proposal delays, estimates can similarly be interpreted as causal, subject to remaining measurement limitations.

## 5. Methodology

### 5.1 Latency

#### 5.1.1 Internal Latency

**Goal.** Measure the latency advantage of the DoubleZero fast path over GossipSub, as observed internally at SentryNet nodes.

**Method.** For each boosted message $m$ observed at SentryNet node $n$, we compute the paired difference:

$$D_{m,n} \triangleq A^{GS}_{m,n} - A^{DZ}_{m,n}$$

A positive $D_{m,n}$ indicates that the DoubleZero path was faster. We report the empirical distribution of $D$ via:

- Summary statistics: median, p90, and p99 of $D$
- Empirical CDF:

$$\hat{F}_D(t) = \frac{1}{N} \sum_{i=1}^{N} \mathbf{1}\{D_i \le t\}$$

**Hypothesis test.** One-sided Wilcoxon signed-rank test:
- $H_0$: $\text{median}(D) \le 0$ (no improvement)
- $H_1$: $\text{median}(D) > 0$ (DoubleZero is faster)

**Confidence intervals.** Bootstrap 95% CIs for median, p90, and p99 of $D$.

> [!NOTE]
> SentryNet nodes use more aggressive GossipSub configurations optimized for latency. The internal $D$ values therefore represent a _conservative_ estimate of the advantage a regular beacon node would experience.

#### 5.1.2 External Latency

**Goal.** Measure the propagation speed advantage of SentryNet-boosted blocks as observed by external nodes across the global network.

**Method.** For each block $b$, we define the start time $S_b$ from the relay's `mev_relay_proposer_payload_delivered` timestamp (approximating network release time). We restrict to blocks propagated by our partner relay.

Blocks and their data column sidecars propagate independently over the P2P network. The primary metric in this section is **block propagation latency** — column count does not affect it. For each vantage point $j \in \mathcal{J}$, we compute:

$$L_{b,j} \triangleq A_{b,j} - S_b \quad \text{(block latency)}$$

We additionally record column sidecar latency as a supplementary data point for future analysis:

$$L^{(c)}_{b,j} \triangleq A^{(c)}_{b,j} - S_b \quad \text{(column latency)}$$

To reduce dependence on the number of observers and make the unit of analysis clearly "a block," we first compute **block-level summaries**:

$$\tilde{L}_b = \text{median}_{j \in \mathcal{J}}(L_{b,j})$$

We then compare the distributions of $\tilde{L}_b$ between groups. For each group $g \in \{\text{boosted}, \text{control}\}$:

$$\hat{F}_g(t) = \frac{1}{N_g} \sum_{b: G_b = g} \mathbf{1}\{\tilde{L}_b \le t\}$$

**Primary effect size.** Quantile shift at key percentiles, to determine the latency differences at different percentiles:

$$\Delta q(p) = \hat{q}_{\text{control}}(p) - \hat{q}_{\text{boosted}}(p), \quad p \in \{0.50, 0.90, 0.99\}$$

**Hypothesis test.** One-sided Mann–Whitney U test on $\{\tilde{L}_b\}$: boosted blocks have lower typical latency than control blocks.

**Confidence intervals.** Bootstrap 95% CIs for $\Delta q(p)$, clustered by block.

**Missingness.** If a block is never observed by some Xatu node, that $(b, j)$ pair is excluded. All exclusions are reported with counts.

### 5.2 Effectiveness

**Goal.** Measure how latency improvements translate into proposer effectiveness metrics.

#### 5.2.1 Correct Head Vote Rate

While blocks propagate independently of their column sidecars, attesters cannot vote for a block's head until they have received a sufficient fraction of its data columns (for data availability). This means that even if a block arrives quickly, attestation depends on column completeness, which means that blocks with more columns are harder to attest to in time.

For each block $b$, define:

$$H_b \triangleq \frac{C_b}{N_b}$$

where $C_b$ is the number of correct head votes at inclusion delay = 1, and $N_b$ is the committee size for that slot.

**Primary estimand** (overall and stratified by blob count $K_b$, since higher blob counts require more columns to be received before attestation):

$$\Delta_H = \mathbb{E}[H_b \mid G_b = \text{boosted}] - \mathbb{E}[H_b \mid G_b = \text{control}]$$

**Hypothesis test.** Two-proportion z-test on aggregated counts, or bootstrap-based test on block-level $\{H_b\}$.

**Confidence intervals.** Block-level bootstrap CI for $\Delta_H$ (resampling by slot). Wilson CIs for individual group rates.

#### 5.2.2 Reorg Rate

Define the reorg indicator $R_b \in \{0, 1\}$ and the group-level reorg probability:

$$\pi_g \triangleq \mathbb{P}(R_b = 1 \mid G_b = g), \quad \hat{\pi}_g = \frac{1}{N_g} \sum_{b: G_b = g} R_b$$

**Effect sizes:**
- Risk difference: $\Delta_\pi = \pi_{\text{control}} - \pi_{\text{boosted}}$
- Risk ratio: $\text{RR} = \pi_{\text{boosted}} / \pi_{\text{control}}$

**Hypothesis test.** Fisher's exact test on the 2×2 contingency table (recommended for rare events).

**Confidence intervals.** Exact (Clopper–Pearson) CIs for each group rate; Newcombe CI for the risk difference.

> [!NOTE]
> Reorgs are rare events. If the baseline reorg rate is on the order of 0.1–1%, meaningful inference may require weeks to months of data collection.

### 5.3 Yield

**Goal.** Quantify how SentryNet's latency advantage translates into increased validator yield, and explore how timing games can amplify this effect.

Yield improvements come from two sources:

#### 5.3.1 Reorg Protection

Faster propagation increases fork choice weight behind the proposed block, reducing the probability of a reorg. A reorg causes the proposer to lose all execution-layer rewards for that slot. Reorg protection therefore preserves one of the proposer's revenue streams.

Let $V_b$ be the execution-layer value if the block remains canonical. The expected realized value for group $g$ is:

$$\mathbb{E}[\text{EL}_b \mid G_b = g] \approx (1 - \pi_g) \cdot \overline{V}_g$$

where $\overline{V}_g$ is the mean execution-layer value for group $g$.

Assuming comparable value distributions across groups, the **per-block reorg protection gain** is:

$$\Delta_{\text{reorg}} \triangleq (\pi_{\text{control}} - \pi_{\text{boosted}}

#### 5.3.2 Timing Games

If SentryNet can reliably propagate blocks faster, proposers can afford to delay their proposal time to capture more MEV value from the builder auction, without sacrificing attestation inclusion. This is the core timing game tradeoff: later proposals capture more value but risk losing attestations and being reorged.

Define the proposer delay $d$ (milliseconds into the slot at which the block is released). For each delay $d$, let:

- $\pi(d)$: reorg probability at delay $d$
- $V(d)$: expected execution-layer value at delay $d$
- $H(d)$: expected correct head vote rate at delay $d$

The expected total per-slot yield is:

$$Y(d) \triangleq (1 - \pi(d)) \cdot V(d) + C(d)$$

where $C(d)$ represents the consensus reward component (a function of $H(d)$).

The **timing game objective** is to find the delay $d$ that maximizes yield subject to an attestation constraint:

$$\max_d \ Y(d) \quad \text{s.t.} \quad H(d) \ge H_{\min}$$

**Measurement protocol:**
- Vary proposal times for SentryNet-boosted validators across controlled intervals (e.g. $d \in \{1.0, 1.5, 2.0, 2.5\}$ seconds).
- For each interval, record $\hat{H}(d)$, $\hat{\pi}(d)$, and $\hat{V}(d)$ from relay payload data.
- Plot the Pareto frontier of $(\hat{V}(d),\ \hat{H}(d))$ and the implied $\hat{Y}(d)$.

The control group remains non-boosted blocks from the same relay, proposed at their natural times.

> [!NOTE]
> Timing game experiments require coordination with partner validators and will be conducted in controlled phases to avoid unnecessary risk.

## 6. Statistical Methodology

### 6.1 Reporting Standards

- All point estimates are accompanied by **95% confidence intervals**.
- For latency, we emphasize **distributional summaries** (median, p90, p99) rather than means alone.
- Hypothesis tests use significance level $\alpha = 0.05$.
<!--- When multiple percentiles or strata are tested simultaneously, we apply Holm–Bonferroni correction or clearly label results as exploratory.-->

### 6.2 Sample Size Guidance

| Analysis | Effective sample size | Target |
|---|---|---|
| External latency | Number of blocks per group (not observer pairs) | Thousands per group per stratum for stable p99 estimates (up to a week of data) |
| Correct head vote rate | Number of blocks per group | Thousands per group for tight CIs, especially when stratifying by blob count (up to a week of data) |
| Reorg rate | Number of blocks per group | Weeks of data, baseline rate is 0.1–1% (up to 2-3 weeks of data) |

### 6.3 Data Quality & Exclusions

We predefine exclusion rules to avoid post-hoc bias:
- Remove observations with missing $S_b$ or implausible timestamps (e.g. negative latencies).
- Deduplicate by block root and observer.
- Document handling of missing observer arrivals.

All exclusions are reported with counts.
