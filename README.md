# ISAC Signaling for 5G-Advanced / 6G

Architecture and design research for **Integrated Sensing and Communication (ISAC)** in 5G-Advanced (3GPP Rel-19/20) and 6G networks. The project captures how sensing topologies, transmitters/receivers and signaling fit together across the RAN protocol stack, plus component-level designs and the reference literature behind them.

## Key Concepts

- **ISAC** – the network acts as a radar: communication waveforms and resources are reused for object detection, tracking and target measurement (range, Doppler, angle).
- **Sensing topologies** – monostatic (same node Tx/Rx), bistatic/distributed (Tx and Rx on different nodes, coordinated over Xn/F1), and UE-assisted sensing.
- **Signaling stack** – L3 sensing session control (CU/SF), L2 MAC scheduling and MAC-CE activation (DU), L1 reference waveforms (PRS/SRS/CSI-RS) at the RU.
- **Sensing data plane** – DASH aggregates, secures and exposes multi-sensor data (ISAC, radar, lidar, camera) to near-RT RIC and dApps.
- **Sensor fusion** – combining cellular ISAC with passive RF and other sensors.
- **Sensing waveforms** – NR reference signals such as PRS, SRS, and CSI-RS are reused as controlled sensing signals, with waveform choice balanced against communication capacity and target-estimation fidelity.
- **Unknown target handling** – low-confidence, ambiguous, or novel returns should be reported as unknown until evidence supports a class assignment.

## Repository Layout

```
.
├── README.md
├── LICENSE
├── docs/
│   ├── ARCHITECTURE.md                      # Architecture overview & reading guide
│   ├── architecture/
│   │   └── 5G-A_ISAC_Highlevel.md           # ISAC signaling flow & parameter specification
│   └── design/
│       ├── DESIGN_INDEX.md                  # Component design index
│       └── components/
│           ├── dash_design_specification.md # Data Access and Security Hub (DASH)
│           └── isac_mac_scheduler_design.md # ISAC-aware gNB MAC scheduler
└── research/
    ├── PAPERS_INDEX.md                      # Reference papers by topic
    └── papers/                              # PDFs
```

## Navigation Guide

| If you want to… | Start here |
|---|---|
| Understand the overall system | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| Read the signaling flow and parameters (L3/L2/L1) | [docs/architecture/5G-A_ISAC_Highlevel.md](docs/architecture/5G-A_ISAC_Highlevel.md) |
| Browse component designs | [docs/design/DESIGN_INDEX.md](docs/design/DESIGN_INDEX.md) |
| Study the sensing data plane / security | [DASH specification](docs/design/components/dash_design_specification.md) |
| Study scheduler integration | [MAC scheduler design](docs/design/components/isac_mac_scheduler_design.md) |
| Find the supporting literature | [research/PAPERS_INDEX.md](research/PAPERS_INDEX.md) |
| Review sensing waveform design and unknown-target analysis | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) |

## Status

Working research notes; content is being refined as 3GPP Rel-19/20 and 6G standardization evolves.

## License

See [LICENSE](LICENSE).
