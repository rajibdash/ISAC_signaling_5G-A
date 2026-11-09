# 6G ISAC Research Ideas

This note turns the themes in the [ShareTechnote 6G ISAC research overview](https://www.sharetechnote.com/html/6G/6G_Research_ISAC.html) into candidate research questions for this repository. It also connects them to the existing 5G-Advanced signaling, MAC scheduling, and DASH data-plane designs. These are research directions, not claims about standardized 6G behavior.

## Context

Integrated Sensing and Communication (ISAC) uses shared radio infrastructure, spectrum, antennas, and/or waveforms to provide both connectivity and sensing. A 6G system could estimate properties such as target range, direction, velocity, and motion from radio observations, including reflections from objects that are not connected devices. The central design challenge is to improve sensing without compromising communication service, while making measurements trustworthy, privacy-aware, and useful to applications.

Relevant sensing arrangements include monostatic operation (transmitter and receiver at one node), bistatic operation (separate transmitter and receiver), and distributed or multistatic operation across coordinated nodes. The repository also considers UE-assisted sensing and fusion with non-cellular sensors.

## Candidate research directions

### 1. Adapt communication waveforms for sensing quality

**Question:** How can a network adapt known reference signals and data-bearing resources to meet changing communication and sensing needs?

Study PRS, SRS, and CSI-RS configurations alongside candidate joint waveforms such as OFDM and OTFS. Vary bandwidth, repetition interval, resource density, beam pattern, and sensing duty cycle. Compare a unified communication-and-sensing waveform with a split-resource design, including the cost of reserving dedicated sensing resources.

**Evaluate:** communication throughput and BLER; range, Doppler, and angle estimation error; detection and false-alarm rates; pilot/resource overhead; and robustness to mobility and multipath.

**Repository connection:** [Waveform discussion](../docs/ARCHITECTURE.md#21-sensing-waveforms-in-isac), [signaling specification](../docs/architecture/5G-A_ISAC_Highlevel.md), and [MAC scheduler design](../docs/design/components/isac_mac_scheduler_design.md).

### 2. Jointly schedule communication and sensing resources

**Question:** Can a scheduler allocate time, frequency, power, and spatial resources to satisfy both communication QoS and sensing accuracy targets under changing traffic and target conditions?

Compare static partitions with adaptive scheduling. Investigate priority and fairness policies, sensing deadlines, multi-user interference, beam conflicts, and graceful degradation when the network is overloaded. Include admission control so a new sensing session cannot silently violate existing communication commitments.

**Evaluate:** rate and latency distributions, BLER, sensing accuracy and freshness, resource utilization, fairness, and deadline-miss rates across load levels.

**Repository connection:** [ISAC-aware MAC scheduler](../docs/design/components/isac_mac_scheduler_design.md). The scheduler is a natural first implementation point because it already represents resource arbitration between the two functions.

### 3. Coordinate monostatic, bistatic, and distributed sensing

**Question:** When does collaboration between radio nodes materially improve coverage or estimation, and what coordination is required to realize that gain?

Compare single-node sensing with bistatic and multistatic arrangements across different node geometry, synchronization error, blockage, and mobility. Study selection of participating nodes, measurement alignment, handover between sensing areas, and resilient operation when a node or link becomes unavailable.

**Evaluate:** coverage and localization error, detection probability, robustness to blocked paths, coordination overhead, synchronization tolerance, and end-to-end report latency.

**Repository connection:** [Deployment topologies](../docs/ARCHITECTURE.md#3-deployment-topologies) and the [5G-A signaling flow](../docs/architecture/5G-A_ISAC_Highlevel.md).

### 4. Improve sensing measurements and signal processing

**Question:** How should radio observations be converted into reliable target estimates in cluttered, dynamic environments?

Investigate estimation and detection using delay, Doppler, and angle information; characterize the effects of multipath, interference, sparse observations, and calibration error. Explore learning-based processing as a complement to signal-processing baselines, with tests for generalization to new environments and changing radio conditions.

**Evaluate:** range/velocity/angle error, detection probability, false alarms, track continuity, compute cost, and performance under domain shift. Report measurement quality and uncertainty, not just a point estimate.

### 5. Treat unknown targets and uncertainty as first-class outputs

**Question:** How can an ISAC system maintain useful tracks without forcing an unreliable object classification?

Compare confidence-aware classification, anomaly scoring, and temporal track association. Preserve an unknown or low-confidence state when evidence is insufficient, and investigate how later observations or independent sensors can resolve ambiguity without retroactively overstating certainty.

**Evaluate:** calibration of confidence, false-classification rate, track continuity, time to resolve an unknown, and the quality of downstream decisions when the system abstains from classification.

**Repository connection:** [Unknown-target handling](../docs/ARCHITECTURE.md#22-unknown-target-handling-in-sensing-analysis).

### 6. Fuse cellular sensing with distributed and external sensors

**Question:** What fusion strategy produces timely, consistent, and traceable situational awareness from radio sensing and sources such as radar, lidar, cameras, or passive RF?

Compare centralized edge fusion with distributed preprocessing and hierarchical fusion. Study source identity, timestamp alignment, coordinate transforms, calibration, out-of-order observations, data freshness, and conflicting evidence. Preserve provenance and uncertainty through association and track management.

**Evaluate:** track accuracy and continuity, data age, fusion latency, tolerance to missing or faulty sources, bandwidth, and compute cost. Keep raw or sensitive sensor data local unless an explicit policy allows its release.

**Repository connection:** [Sensor Fusion Server and DASH](../README.md#sensor-fusion-server-logical-architecture) and the [DASH design specification](../docs/design/components/dash_design_specification.md).

### 7. Explore large arrays, high-frequency spectrum, and reconfigurable propagation

**Question:** How can emerging radio technologies improve sensing resolution or coverage, and what are their costs and limits?

Evaluate large-aperture and holographic MIMO, higher-frequency spectrum (including FR3), and reconfigurable intelligent surfaces as candidate enablers. Measure how aperture, bandwidth, propagation, blockage, calibration, and hardware constraints affect both sensing and communication. Compare practical gains against deployment complexity and power consumption.

**Evaluate:** spatial/range resolution, coverage under blockage, link and sensing reliability, calibration effort, power, hardware complexity, and communication capacity.

**Repository connection:** [FR3 reference](PAPERS_INDEX.md#spectrum) and the [architecture guide](../docs/ARCHITECTURE.md).

### 8. Design secure, privacy-aware sensing services

**Question:** How can a network expose useful sensing results while limiting access to sensitive observations and protecting against untrusted or faulty sources?

Study authenticated sensor identity, attestation, least-privilege subscriptions, provenance, auditability, and policy enforcement across edge and cloud placements. Compare privacy-preserving processing approaches and quantify their impact on sensing utility and latency. Define behavior for revoked sources, invalid measurements, and unavailable policy services.

**Evaluate:** unauthorized disclosure and tampering resistance, policy enforcement latency, audit completeness, privacy/utility trade-offs, and the ability to identify untrusted observations.

**Repository connection:** [DASH design specification](../docs/design/components/dash_design_specification.md).

### 9. Build an end-to-end 6G-oriented evaluation platform

**Question:** Which architectural placement and service procedures meet application needs from radio measurement through policy-controlled sensing output?

Prototype a repeatable scenario that exercises radio-resource control, one or more sensing topologies, measurement processing, fusion, and authorized data exposure. Compare local, DU-edge, and higher-layer processing placements. Document which interfaces are implementation choices versus standardized behavior; use the testbed to identify gaps in signaling, timing, and interoperability.

**Evaluate:** end-to-end latency and freshness, scalability, availability, interoperability, compute/network overhead, and reproducibility. Include a communication-only baseline and report conditions, datasets, and failure cases.

## Suggested first study

A tractable starting project is **adaptive MAC scheduling for joint communication and sensing**. Extend the existing scheduler evaluation with at least two traffic/target scenarios, compare fixed and adaptive resource policies, and report communication QoS and sensing accuracy together. Then add a distributed-sensing case to measure how coordination overhead changes the trade-off. This creates a measurable baseline before expanding into fusion, AI, or privacy mechanisms.

## Common evaluation checklist

For comparable results, report:

- **Communication:** throughput, latency, reliability/BLER, and resource overhead.
- **Sensing:** range/velocity/angle error, detection probability, false-alarm rate, uncertainty calibration, and track continuity.
- **System:** sensing freshness, end-to-end latency, energy/compute cost, coordination overhead, and performance under load or failure.
- **Trust and privacy:** source provenance, policy outcomes, access-control behavior, and the effect of privacy protections on utility.
- **Experimental conditions:** topology, carrier/bandwidth, waveform and scheduling configuration, mobility, clutter/multipath, synchronization assumptions, sensor calibration, and baseline methods.

## Source and related project documents

- ShareTechnote, [6G Research: ISAC](https://www.sharetechnote.com/html/6G/6G_Research_ISAC.html)
- [ISAC architecture guide](../docs/ARCHITECTURE.md)
- [5G-A ISAC signaling flow](../docs/architecture/5G-A_ISAC_Highlevel.md)
- [Component design index](../docs/design/DESIGN_INDEX.md)
- [Research papers index](PAPERS_INDEX.md)
