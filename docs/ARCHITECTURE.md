# ISAC Architecture Guide

This guide gives an end-to-end view of the ISAC architecture and points to the detailed specifications.

## 1. System View

```
[ Core Network / Sensing Function (SF) ]  <-- sensing session, fusion server
                  │  L3 (SBI, e.g. TS 29.545)
            ┌─────▼─────┐
            │  gNB-CU   │  session control, topology assignment
            └─────┬─────┘
                  │  F1AP – Sensing Resource Coordination IE
            ┌─────▼─────┐        ┌────────────────────────┐
            │  gNB-DU   │◄──────►│ DASH (sensing data     │
            │ MAC sched │        │ plane, secure exposure)│
            └─────┬─────┘        └────────────────────────┘
                  │  MAC-CE activation
            ┌─────▼─────┐
            │  gNB-RU   │  L1: PRS/SRS/CSI-RS based sensing, echo Rx
            └───────────┘
```

## 2. Layered Responsibilities

| Layer | Node | Responsibility | Detail |
|---|---|---|---|
| L3 | Core/SF, CU | Sensing session initiation, geofencing, multi-node topology | [Highlevel §1](architecture/5G-A_ISAC_Highlevel.md) |
| L2 | DU | Sensing/communication resource arbitration, MAC-CE, scheduling | [Highlevel §2–3](architecture/5G-A_ISAC_Highlevel.md), [MAC scheduler](design/components/isac_mac_scheduler_design.md) |
| L1 | RU | Reference waveform configuration, echo reception | [Highlevel §4](architecture/5G-A_ISAC_Highlevel.md) |
| Data plane | RAN / near-RT RIC | Multi-sensor aggregation, privacy, exposure | [DASH](design/components/dash_design_specification.md) |

## 3. Deployment Topologies

- **Monostatic** – one gNB transmits and receives; handled by the local sensing manager in the MAC scheduler.
- **Bistatic / distributed** – Tx and Rx on different nodes; requires Xn extensions and guard windows.
- **UE-assisted** – UE contributes measurements or acts as Tx/Rx.

## 4. Standards Context

3GPP Rel-19 (sensing study, TR 22.837), Rel-20 enhancements, ITU-R IMT-2030 (M.2160), IEEE 802.11bf, ETSI ISG ISAC. See [Highlevel](architecture/5G-A_ISAC_Highlevel.md) and the [paper index](../research/PAPERS_INDEX.md).

## 5. Suggested Reading Order

1. [5G-A_ISAC_Highlevel.md](architecture/5G-A_ISAC_Highlevel.md) – signaling flow and parameters
2. [Design index](design/DESIGN_INDEX.md) – component designs
3. [Paper index](../research/PAPERS_INDEX.md) – supporting research
