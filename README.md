# SentryNet Data Room

SentryNet is a global network of high-performance, lightweight P2P sentries that are interconnected over [DoubleZero](https://doublezero.xyz) [Multicast](https://doublezero.xyz/journal/doublezero-introduces-multicast-support-smarter-faster-data-delivery-for-distributed-systems).
It provides a fast-path for certain types of data, such as blocks and blob sidecars, and offers more competitive latency than the traditional GossipSub-based network.

This repository contains the SentryNet data room, which includes [methodologies](./METHODOLOGY.md) and notebooks for analyzing SentryNet data.

## Requirements
- Clickhouse CLI (optional, for exporting Xatu DB data to Parquet)
- Quarto CLI (optional, for rendering notebooks): `brew install --cask quarto`
