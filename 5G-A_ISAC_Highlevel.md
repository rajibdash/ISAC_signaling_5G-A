# 5G-Advanced ISAC Signaling Flow & Parameter Specification

This document details the control plane and physical layer signaling architecture for **5G-Advanced (5G-A) Integrated Sensing and Communication (ISAC)** based on 3GPP Release 19 requirements and Release 20 enhancements.

The Standards Context: 5G-Advanced and 6G
ISAC isn't just an academic curiosity — it's being actively standardized by every major wireless body:

* 3GPP Release 19 (5G-Advanced): Introduced study items on "Sensing" as a new NR (New Radio) capability. The study item SP-231458 examines sensing use cases, channel models, and requirements. 3GPP TR 22.837 catalogs specific sensing scenarios and service requirements for NR-based sensing.
* ITU-R IMT-2030 Framework (Recommendation M.2160): The ITU's vision for 6G explicitly identifies "Integrated Sensing and Communication" as one of the six usage scenarios for next-generation wireless. This places ISAC on the same level of strategic importance as enhanced mobile broadband (eMBB) was for 5G.
* IEEE 802.11bf (Wi-Fi Sensing): The Wi-Fi community is standardizing sensing capabilities within WLAN, enabling your home router to detect presence, gestures, and even vital signs.
* ETSI ISG on ISAC: The European Telecommunications Standards Institute has established a dedicated Industry Specification Group for ISAC system architecture and requirements.

The message from the industry is unmistakable: every major standards body has recognized ISAC as a foundational pillar of future wireless systems. This is not a niche academic research topic — it is the mainstream strategic direction.

What is being designed (6G ISAC): True radio sensing — where the mobile network acts like a radar to detect weather, atmospehere, traffic, or physical objects — is a cornerstone of Ericsson's 6G architecture. Ericsson's researchers have noted that while the goal is to reuse existing hardware with minimal tweaks, implementing it requires new __sensing-capable__ base stations, specialized sensing software functions, and updated workflow frameworks that are not yet natively present in commercial 5G deployments.

When 6G rolls out, the RAN node will indeed control these sensing workflows, decide if user equipment (UE) should assist, and ingest that reception data, but it remains an experimental and developmental framework as of today. Do not know if sensing would be default enabled with 6G roll out plan. Where the sensing fusion server will sit for multiple sensors information storage.

---

## 🏗️ Protocol Stack Architecture & Signaling Topology

The 5G-A ISAC framework maps high-level application sensing requests down to sub-millisecond physical layer slots. 
The pipeline moves from the core network down to the Distributed Unit (DU) and Radio Unit (RU) for integrated execution of 
communication and sensing operations.

```
[ Core Network / Sensing Function (SF) ]
                    │
                    │ (1) L3: Sensing Session Initiation (TS 29.545 SBI)
                    ▼
          ┌───────────────────┐
          │     gNB - CU      │ (Layer 3 - Control Plane)
          └───────────────────┘
                    │
                    │ (2) L3/F1AP: Sensing Resource Coordination Information IE
                    ▼
          ┌───────────────────┐
          │     gNB - DU      │ (Baseband / MAC Layer Scheduling)
          └───────────────────┘
                    │
                    │ (3) L2/MAC-CE: Dynamic Sensing Activation & Subframe Muxing
                    ▼
          ┌───────────────────┐
          │     gNB - RU      │ (Layer 1 - Physical Transmission & Echo Rx)
          └───────────────────┘
                    │
                    │ (4) L1: Repurposed Reference Waveforms (PRS/SRS/CSI-RS)
                    ▼
         [ Target / UAV ] ──(Echo)──► [ gNB RX Array ]
```

---

## 📡 1. Layer 3 (L3) Control Plane Signaling

The L3 layer handles session establishment, target geofencing, and multi-node topology assignment. 
The core network's **Sensing Function (SF)** initiates requests via service APIs over the Service-Based Interface (SBI) following __TS 29.545__ standardization.

### TS 29.545 / TS 23.137 Service Exposure Payload (JSON)
The following parameter block represents the standardized information element sent from the Core Function to the gNB-CU to spin up an ISAC session:

```json
{
  "SensingSessionID": "isac-sfn-zone-4029",
  "SensingOperationMode": "TRP_Monostatic", 
  "TargetType": "UAV_LOW_ALTITUDE",
  "Sensing_QoS_Profile": {
    "RangeResolution_m": 1.5,
    "MaxRange_m": 800,
    "VelocityResolution_mps": 0.5,
    "MaxVelocity_mps": 45,
    "SensingUpdateRate_ms": 100,
    "ConfidenceLevel_percent": 95
  },
  "GeofencingConstraints": {
    "LatitudeStart": 51.5074,
    "LongitudeStart": -0.1278,
    "AltitudeCeiling_m": 120
  }
}
```

* **SensingOperationMode Selection:** Configures the operational topology (`TRP_Monostatic`, `TRP-TRP Bistatic`, or `UE-Assisted/Device-Assisted`).
* **F1AP Mapping:** The gNB-CU translates these QoS parameters into the `Sensing Resource Coordination Information IE` across the F1 application interface to bind hardware blocks within the DU.

---

## 🎛️ 2. Baseband / Distributed Unit (DU) Layer Control

The gNB-DU is the functional entity responsible for translating high-level sensing requirements into real-time baseband signal processing and resource scheduling. It bridges the gap between control-plane directives from the gNB-CU and physical-layer execution at the gNB-RU, implementing dynamic adaptation and coordinating coexistence of communication and sensing payloads.

### 2.1 DU Functional Architecture for ISAC

```
┌─────────────────────────────────────────────────────────────────┐
│                        gNB-DU ISAC Module                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  F1AP Interface Handler                                         │
│  ├─ Parse Sensing Resource Coordination Information IE          │
│  ├─ Validate QoS constraints & resource pools                   │
│  └─ Map to internal DU scheduling policies                      │
│                                                                 │
│  MAC Scheduler (Dual-Mode)                                      │
│  ├─ Communication Scheduler (PDSCH/PUSCH allocation)            │
│  ├─ Sensing Scheduler (PRS/SRS/CSI-RS burst generation)         │
│  ├─ Multiplexing Policy Engine (TDM/FDM/SDM arbitration)        │
│  └─ Priority Queue Manager (QoS enforcement)                    │
│                                                                 │
│  Beam Management & MIMO Control                                 │
│  ├─ Precoding Matrix Computation (W_comm, W_sense)              │
│  ├─ Beam Index Table Generator                                  │
│  └─ Spatial Adaptation Engine                                   │
│                                                                 │
│  Self-Interference Cancellation (SIC) Controller                │
│  ├─ TX-RX Leakage Estimation                                    │
│  ├─ Digital Cancellation Filter Configuration                   │
│  └─ Monostatic Echo Suppression Parameters                      │
│                                                                 │
│  L1 Signal Processing Coordinator                               │
│  ├─ Waveform Generation Instructions (PRS comb config)          │
│  ├─ Pulse Repetition Interval (PRI) timing                      │
│  ├─ Frequency allocation & subcarrier mapping                   │
│  └─ Phase reference and time synchronization                    │
│                                                                 │
│  Feedback Collection & Aggregation                              │
│  ├─ L1 Echo Report Parser (Delay-Doppler matrices)              │
│  ├─ AoA Extraction & Beamforming feedback                       │
│  ├─ RCS estimation accumulator                                  │
│  └─ Tracking prediction error monitor                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 DU Internal Scheduling Engine Parameters (3GPP TR 38.765 Baseline)

```ini
[Carrier_Settings]
CarrierFrequency = 4.0 GHz        # Target mid-band testing frequency
SubcarrierSpacing = 30 kHz        # FR1 numerology (μ=1)
TotalBandwidth_RBs = 273          # ~100 MHz active allocation channel
NumRBsPerSubarray = 48            # Per-beam resource granularity

[Sensing_Slot_Muxing]
MultiplexingMethod = TDM          # Time-Division Multiplexing (Comm vs Sensing slots)
SensingPeriodicity_Slots = 80     # Trigger sensing profile every 80 slots (1.25 ms PRI)
CombSize = 4                      # Frequency density spacing pattern (Comb-4 structure)
TotalActiveSubcarriers = 125      # Spectral footprint allocated per sensing comb
SensingDutyCycle_percent = 15     # Percentage of frame budget dedicated to ISAC

[SIC_Configuration]
SIC_Isolation_Target_dB = 70      # Baseline TX-RX cancellation isolation threshold
DigitalCancellationTaps = 256     # FIR filter length for SIC processing
Monostatic_Echo_Suppression = ENABLED  # Cross-correlation echo blanking

[MAC_CE_Behavior]
DynamicActivationLatency_us = 50  # Time from MAC-CE transmission to physical activation
BeamSwitchingTime_us = 100        # Analog beam steering latency
FastActivationMode = SUPPORTED     # Enable sub-frame reactive toggling
```

### 2.3 F1AP Resource Coordination & DU-CU Handshake

The gNB-DU receives the **Sensing Resource Coordination Information IE** from the gNB-CU via the F1 Application Protocol. This IE encapsulates the following binding parameters:

```
F1AP Message: "DL RRC Message Transfer"
├─ Sensing Resource Coordination Information IE
│  ├─ SensingSessionID (Unique per core network request)
│  ├─ Sensing Resource Pool Definition
│  │  ├─ Resource Block allocation mask (bitmap)
│  │  ├─ Slot periodicity (e.g., every 80 slots)
│  │  ├─ Reference signal density (Comb-2/4/6 pattern)
│  │  └─ Time/Frequency domain mapping rules
│  ├─ Beam Configuration Set
│  │  ├─ Transmit beam indices (0 to 63 for Massive MIMO)
│  │  ├─ Receive beam indices (codebook-based or beam refinement)
│  │  └─ Analog & digital precoding instructions
│  ├─ Periodicity & Timing Advance
│  │  ├─ Sensing trigger slot offsets within frame
│  │  ├─ Multi-frame alignments
│  │  └─ Priority escalation thresholds
│  └─ SIC Configuration & Monostatic Echo Blanking
│     ├─ TX leakage compensation coefficients
│     ├─ Cancellation filter weights
│     └─ Clutter suppression mask
```

**DU Processing Steps Upon Reception:**

1. **Parse & Validate** – Extract all IE fields and cross-check against local baseband capability database (max PRS pulses/frame, MIMO ranks, etc.).
2. **Resource Binding** – Map abstract resource pool definitions to physical RB indices in the frequency-time grid.
3. **Conflict Detection** – Check for collisions with ongoing communication sessions (PDSCH, PUSCH) and flag to MAC scheduler.
4. **Beam Table Construction** – Index transmit/receive codebook entries or compute custom precoding matrices for optimal sensing geometry.
5. **SIC Filter Initialization** – Pre-compute monostatic cancellation filter states based on RU antenna coupling metrics.
6. **Timing Synchronization** – Align sensing slot triggers with DU clock, factoring in F1AP transport latency.

### 2.4 MAC Scheduler: Dual-Mode Operation (Comm & Sensing)

The DU MAC scheduler manages the joint optimization of communication throughput and sensing detection performance. It maintains two parallel scheduling queues and arbitrates resource allocation in real time.

#### A. Time-Division Multiplexing (TDM) Architecture

**Concept:** Reserved time slots are locked exclusively for sensing transmission/reception, eliminating communication during sensing phases.

```
DU Scheduling Matrix (Example: 10ms Frame, 80 Slots @ 15kHz SCS)

Frame Structure:
┌─────────────────────────────────────────────────────────┐
│ Slot │ Type │ Type │ Type │ Type │ Type │ Type │ Type │ │
│  0   │ Comm │ Comm │ Sense│ Comm │ Comm │ Sense│ Comm │ │
│  1   │ Comm │ Sense│ Comm │ Comm │ Sense│ Comm │ Comm │ │
│ ...  │  ... │  ... │  ... │  ... │  ... │  ... │  ... │ │
│ 79   │ Comm │ Comm │ Comm │ Sense│ Comm │ Comm │ Sense│ │
└─────────────────────────────────────────────────────────┘

Resource Allocation Decision Logic:
IF SensingSlot(t) == TRUE:
    └─ Reserve ALL RBs for PRS waveform transmission
    └─ Suppress MAC PDU scheduling (communicate = 0)
    └─ Lock RU receive chains for echo capture
ELSE:
    └─ Allocate RBs to PDSCH (DL comm) & PUSCH (UL comm)
    └─ Monitor leftover resource budget for next sensing trigger
```

**Advantages:** Complete isolation; no cross-channel interference.  
**Trade-off:** Reduced communication throughput (~15% duty cycle reduction).

#### B. Frequency-Division Multiplexing (FDM) Architecture

**Concept:** Sensing pilots (narrowband, low-density comb patterns) coexist on dedicated sub-band allocations while wideband communication proceeds on complementary RBs.

```
Frequency Domain (Per Slot):
┌────────────────────────────────────────────────────────┐
│ RB0  │ RB1  │ RB2  │ RB3  │ RB4  │ RB5  │ RB6  │ RB7   │
├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────-┤
│ PRS  │ PDSCH│ PDSCH│ PRS  │ PDSCH│ PDSCH│ PRS  │ PDSCH │
│ (Comb-4 Pattern)  │ (Communication Data Payload)       │
└────────────────────────────────────────────────────────┘

DU Scheduler State Machine:
STATE = "FDM_ACTIVE"
├─ Allocate {RB0, RB3, RB6, ...} → L1 PRS Generation (Sensing)
├─ Allocate {RB1, RB2, RB4, RB5, RB7, ...} → MAC PDSCH Queue (Comm)
├─ Monitor PHY feedback for PRS demodulation success
└─ If PRS SNR < threshold:
   └─ Escalate to TDM mode (full resource reallocation)
```

**Advantages:** Higher communication efficiency; parallel sensing & data.  
**Trade-off:** Increased co-channel interference (requires robust SIC post-processing at L1).

#### C. Spatial Multiplexing (SDM) with Massive MIMO

**Concept:** Transmit and receive beamforming vectors ($\mathbf{W}_{\text{comm}}$ and $\mathbf{W}_{\text{sense}}$) are designed to isolate communication and sensing payloads in the spatial domain.

```
MIMO Precoding Strategy:

For M-element antenna array (e.g., 32 elements):
  
  TX Beamforming:
  ├─ Communication vector: w_comm = [w1_c, w2_c, ..., w32_c]ᵀ
  │  └─ Steered toward registered UE (angle θ_ue)
  ├─ Sensing vector: w_sense = [w1_s, w2_s, ..., w32_s]ᵀ
  │  └─ Steered toward target direction (angle θ_target)
  └─ Orthogonality constraint: w_commᴴ · w_sense ≈ 0 (cross-correlation minimized)

  RX Beamforming (Echo Reception):
  ├─ Combine RX antenna signals with matched filter for sensing echo
  └─ Spatial filtering suppresses direct TX leakage component

DU Beam Design Equation:
  w_optimal = arg min { ||w||² subject to: w_commᴴ·w_sense ≤ ε_isolation }
  
  where ε_isolation is typically -60 dB to -70 dB.
```

**Advantages:** Highest spectrum efficiency; simultaneous full-rate comm & sensing.  
**Trade-off:** Requires advanced channel estimation and feedback overhead.

### 2.5 DU L2 MAC Control Elements (MAC-CE) for Fast Activation

To achieve sub-millisecond reactive control, the DU can bypass RRC reconfiguration and issue **MAC Control Elements** or **Downlink Control Information (DCI)** fields to instantly toggle beam indices, activate sensing bursts, or suppress communication frames.

```
MAC-CE Frame Format (3GPP TS 38.321):

Byte 0-1: MAC Subheader
├─ E bit: More MAC elements follow
├─ LCID: 0x3A → Reserved for Sensing Control
└─ Length: 2-4 bytes

Byte 2-5: Sensing Activation Payload
├─ SensingActivationBit: 1 = Enable, 0 = Disable
├─ BeamIndex: 6-bit beam selector (0-63)
├─ PRI_Offset: 4-bit slot timing offset for next PRS burst
├─ PowerReduction: 2-bit TX power scaling instruction
└─ SIC_CancellationWeight: 8-bit digital cancellation tap adjustment

Example Transmission Sequence:
T=0ms:     MAC-CE issued → "Enable PRS, Beam 12, PRI_Offset=2"
T=0.05ms:  DU forwards to RU via S-plane interface
T=0.15ms:  RU DSP receives, configures FPGA coefficients
T=0.2ms:   Waveform generation begins with new beam steering
```

**Latency Profile:**
- MAC-CE composition in DU: ~10 µs
- F1 transport (DU→RU): ~30–50 µs (functional split dependent)
- RU DSP configuration: ~10–20 µs
- **Total end-to-end:** 50–80 µs (sub-frame reactive control achieved)

### 2.6 DU Feedback Aggregation & Reporting to CU

The DU collects raw or partially processed echo data from the RU and applies aggregation logic before forwarding to the gNB-CU for core network consumption.

```
DU Feedback Processing Pipeline:

┌──────────────────────────────────────┐
│  L1 RU Echo Reports                  │
│  (Delay-Doppler matrices, AoA, RCS)  │
└──────────────┬───────────────────────┘
               │ (Received every 1.25 ms per PRI)
               ▼
┌──────────────────────────────────────┐
│  DU Sensing Report Aggregator        │
│  ├─ Buffer 4–8 L1 reports            │
│  ├─ Temporal averaging (reduce noise)│
│  ├─ Cross-correlation peak detection │
│  └─ Track state vector [X,Y,Z,v,RCS] │
└──────────────┬───────────────────────┘
               │ (Aggregated every 10–40 ms)
               ▼
┌──────────────────────────────────────┐
│  F1AP Encapsulation                  │
│  ├─ SensingSessionID                 │
│  ├─ Timestamp (millisecond precision)│
│  ├─ Extracted Target List            │
│  └─ Quality Metrics (SNR, Cond. #)   │
└──────────────┬───────────────────────┘
               │ (F1 transport)
               ▼
┌──────────────────────────────────────┐
│  gNB-CU Sensing Report Sink          │
│  → Forward to Core SF via N68 SBI    │
└──────────────────────────────────────┘
```

**Reporting Abstraction Layers (Configurable):**

1. **Sample Level:** Raw IQ vectors (I/Q pairs @ baseband sampling rate). Bandwidth: ~500 Mbps per RU.
2. **Delay-Doppler Level:** Compressed 2D matrices (range vs. velocity bins). Bandwidth: ~50–100 Mbps.
3. **Point Cloud Level:** Parsed object states (X, Y, Z, Δv, RCS, confidence). Bandwidth: ~1–10 Mbps.

**DU Selection Logic:**
- High-precision sensing (UAV identification): Use Sample or Delay-Doppler level.
- Standard tracking (airspace monitoring): Use Point Cloud with confidence filtering.

### 2.7 Real-Time Adaptation & Tracking Loop Integration

The DU implements a feedback loop to monitor tracking performance and dynamically adjust sensing parameters without core network intervention.

```
DU Adaptive Tracking Loop (Sub-100ms Cycle):

┌─────────────────────────────────────────────┐
│ CU Issues: "Track UAV in sector 2, Z>50m"   │
└──────────────┬──────────────────────────────┘
               ▼
         ╔════════════════╗
         ║ T=0ms: Initial │
         ║ Beam Steering  │
         ║ Toward Sector 2║
         ╚════════════════╝
               │
               ▼ (Every 5ms)
         ╔════════════════════════════════════╗
         ║ Evaluate Tracking Error:           ║
         ║ error = |predicted_pos - reported  │
         ║ if error < 10m:                    │
         ║   └─ Maintain current beam config  │
         ║ elif error = 10–30m:               │
         ║   └─ Slight beam refinement        │
         ║ elif error > 30m:                  │
         ║   └─ Escalate: TDM full burst      │
         ║   └─ Increase PRI to every 40 slots│
         ╚════════════════════════════════════╝
               │
               ▼ (Report back to CU every 50ms)
         ╔════════════════════════╗
         ║ Send F1AP Feedback:    │
         ║ "Tracking confidence:  │
         ║ 98%, RCS stable"       │
         ╚════════════════════════╝
```

**Example Adaptation Triggers:**
- **Prediction Error > 50m:** Switch from FDM → TDM (higher resolution).
- **Target RCS Fluctuation > 3dB:** Increase integration pulses per burst.
- **Multiple Targets Detected:** Allocate additional beams via spatial multiplexing.
- **SNR Drop Below 10dB:** Enable SIC filter coefficient re-optimization.

---

## 3. MAC Layer & Software Scheduler Engineering

The MAC scheduler balances user cellular SLAs against the periodic insertion of deterministic sensing waveforms. Advanced algorithms coordinate resource allocation and adapt multiplexing strategies dynamically.

### A. Multiplexing Implementation Architectures
* **Time-Division Multiplexing (TDM):** Dedicated slots or individual OFDM symbols are locked exclusively for sensing pulses (e.g., PRS bursts). This eliminates self-interference entirely but causes throughput reduction (~10–20%).
* **Frequency-Division Multiplexing (FDM):** Interleaves sensing pilots on specific resource blocks alongside active PDSCH/PUSCH data channels. This allows parallel execution, but requires the software layer to post-process interference suppression at the receiver.
* **Spatial Multiplexing (SDM):** Utilizes Massive MIMO to calculate isolated precoding vectors ($\mathbf{W}_{\text{comm}}$ and $\mathbf{W}_{\text{sense}}$), firing data toward a user while simultaneously transmitting sensing waveforms on orthogonal spatial modes.

### B. Comb Pattern Selection Frameworks
To preserve spectral resources, the scheduler maps sparse pilots instead of full-grid allocations:
* **Comb-2 Structure:** Maximum energy capture per pulse, extending the **Maximum Detectable Range**. However, it halves the resource grid block availability for standard data pipelines.
* **Comb-4 / Comb-6 Structure:** Maximizes commercial user capacity but increases **Range Ambiguity** constraints, forcing the software layer to process secondary math equations to filter out "ghost targets" via range-Doppler disambiguation algorithms.

### C. Dynamic Frame Restructuring & Adaptive Bursts
The scheduler monitors prediction errors from tracking loops. If an uncooperative target (e.g., a drone) exhibits unpredictable acceleration, the MAC layer overrides standard communication frames to support high-priority sensing intervals.

```
[ Frame N: Communications Focus Mode ]
 ┌───────┬───────┬───────┬───────┐
 │ Slot0 │ Slot1 │ Slot2 │ Slot3 │   -> 95% Communication Data Payload
 └───────┴───────┴───────┴───────┘   -> 5% Static Reference Pilots
 
[ Frame N+1: High-Priority ISAC Event Triggered ]
 ┌───────┬───────┬───────┬───────┐
 │ Slot0 │ Slot1 │ Slot2 │ Slot3 │   -> 40% Frame Capacity Intercepted
 └───┬───┴───────┴───┬───┴───────┘
     ▼               ▼
 [Sensing Beam 1] [Sensing Beam 2]   -> Real-time targeted spatial sweep
```

---

## ⚡ 4. Layer 1 (L1) Physical Layer Configuration

The physical layer dictates the exact over-the-air waveform geometry used to maximize target echo extraction. The gNB-RU executes DU-provided DSP instructions and returns raw or processed echo data.

### L1 Over-the-Air Waveform Profile (3GPP TR 38.914 / TR 38.765)
```ini
[Waveform_Generation]
BaseSignalType = PRS              # Positioning Reference Signal (Downlink Monostatic)
TxPower_dBm = 37.0                # Configurable RF transmission power line
TotalPulsesPerBurst = 128         # Integration pulses to build Doppler velocity profile
PulseRepetitionInterval = 1.25 ms # Corresponds to a maximum unambiguous range of ~187 km

[Antenna_MIMO_Array]
PanelConfiguration = 4x8          # 4 Vertical x 8 Horizontal elements per TRP
SubarrayStructure = 2x1           # Spatial subarray binning 
ElementSpacing_H = 0.0375 m       # Half-wavelength optimization for spatial AoA resolution
Polarization = Dual_Linear        # Resolves target polarization/Radar Cross Section (RCS)
```

### Abstraction Reporting Tiers (L1 to Core/App Stack)
Once the echo is captured by the antenna arrays, the L1 processing engine parses the raw signals into one of three **3GPP Release 20 abstraction layers** before pushing data back up the stack:

1. **Sample Level (I/Q Data):** Raw phase/amplitude values indexable by time, frequency, and antenna beam. Provides maximum flexibility for external AI/ML pattern engines but demands high link bandwidth (~500 Mbps).
2. **Parameter Profile Level:** Compiled **Delay-Doppler matrices** or intermediate **Angle-of-Arrival (AoA)** angular profiles to compress reporting payloads (~50–100 Mbps).
3. **Detected Point Level (Point Cloud):** Cleaned and parsed object states mapping directly to the application tracker:
   $$\text{State Vector} = [X, Y, Z, \Delta v, \sigma_{\text{RCS}}, t_{\text{timestamp}}]$$
   *Where $\sigma_{\text{RCS}}$ is the calculated **Radar Cross Section** (expressed in dBsqm), enabling classification filters to separate targets (e.g., distinguishing an unauthorized drone from a bird or atmospheric clutter).

---

## 📊 5. Protocol Stack & Information Exchange Matrix

| Phase | Layer | Primary Interfaces | Core Parameter Payload Elements | DU Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **1.Request** | L3 Core to CU | HTTP/SBI, NGAP | `SensingSessionID`, `SensingOperationMode`, `TargetType`, `Sensing_QoS_Profile`, Geo-coordinates. | Receive & validate constraints. |
| **2.Allocation** | L3 CU to DU | F1AP, RRC Reconfig | `Sensing Resource Coordination Information IE`, Resource Pool allocation maps, Reference Signal Type. | Parse F1AP, bind resources, compute beams, initialize SIC. |
| **3.Activation**| L2 DU to RU/UE| MAC-CE, DCI | Immediate slot assignments, Beam Index configurations, Fast activation toggles. | Schedule MAC-CE, drive RU DSP config. |
| **4.Execution** | L1 Over-the-Air| OFDM Radio Links | Repurposed PRS/SRS/CSI-RS comb configurations, pulse timing bursts, phase echo tracking. | Generate L1 waveform instructions, sync timing. |
| **5.Reporting** | Cross-Layer | L1/L2 Feedbacks ➔ F1AP | Delay-Doppler matrices, extracted AoA profiles, state tracking vectors $[X, Y, Z, \Delta v, \sigma_{\text{RCS}}]$. | Aggregate L1 reports, filter noise, encapsulate F1AP, forward to CU. |

---

## 6. DU-Specific Performance Metrics & Monitoring

The gNB-DU continuously monitors ISAC system performance and reports quality indicators back to the gNB-CU:

| Metric | Unit | Typical Range | DU Measurement Method |
| :--- | :--- | :--- | :--- |
| **Sensing SNR** | dB | 10–30 | Echo power vs. noise floor at RU receive chains. |
| **Range Resolution Achieved** | m | 0.5–5 | Delay-domain pulse compression bandwidth. |
| **Velocity Accuracy** | m/s | ±0.1–1 | Doppler FFT bin spacing analysis. |
| **Beam Steering Latency** | µs | 50–200 | Time from MAC-CE to RU beamforming activation. |
| **TDM Duty Cycle** | % | 10–40 | Sensing slot occupancy vs. total frame budget. |
| **SIC Isolation Achieved** | dB | 50–80 | TX-RX leakage cancellation depth (monostatic). |
| **Target Track Continuity** | % | 95–99.5 | Frame-to-frame target detection success rate. |

```python

import json
import math
import time

# Mock telemetry payload simulating a live L1 reporting interval 
# structured following 3GPP TR 38.765/TS 23.137 configurations
# Enhanced with DU processing metadata

MOCK_ISAC_STREAM = """
{
  "SensingSessionID": "isac-sfn-zone-4029",
  "Timestamp": 1789234567.89,
  "Configuration": {
    "SensingOperationMode": "TRP_Monostatic",
    "CarrierFrequency_GHz": 4.0,
    "SubcarrierSpacing_kHz": 30,
    "MultiplexingMode": "TDM",
    "SensingDutyCycle_percent": 15
  },
  "DU_ProcessingMetrics": {
    "AggregationWindow_ms": 10,
    "L1_ReportsProcessed": 8,
    "SIC_IsolationAchieved_dB": 68.5,
    "BeamSteeringLatency_us": 65,
    "SchedulerQueueDepth": 3
  },
  "L1_Reports": [
    {
      "TargetID": 101,
      "Coordinates": {"X": 154.2, "Y": -42.8, "Z": 85.5},
      "DeltaV_mps": 18.5,
      "RCS_dBsqm": -5.2,
      "SNR_dB": 22.3,
      "ConfidenceLevel_percent": 97
    },
    {
      "TargetID": 102,
      "Coordinates": {"X": 12.1, "Y": 305.4, "Z": 12.0},
      "DeltaV_mps": -1.2,
      "RCS_dBsqm": -18.5,
      "SNR_dB": 15.8,
      "ConfidenceLevel_percent": 89
    }
  ]
}
"""

def classify_target_by_rcs(rcs_dbsqm):
    """
    Classifies targets by mapping Radar Cross Section (RCS) back into 
    approximate hardware envelopes aligned with 6G ISAC simulation models.
    """
    if rcs_dbsqm >= 0:
        return "Commercial Aircraft / Large Vehicle"
    elif -10 <= rcs_dbsqm < 0:
        return "Class I/II UAV (Medium Enterprise Drone)"
    elif -20 <= rcs_dbsqm < -10:
        return "Micro-UAV / Large Bird"
    else:
        return "Clutter / Small Bird / Atmospheric Artifact"

def process_isac_telemetry(raw_stream):
    try:
        # Step 1: Parse string stream payload
        payload = json.loads(raw_stream)
        
        session_id = payload.get("SensingSessionID")
        config = payload.get("Configuration", {})
        du_metrics = payload.get("DU_ProcessingMetrics", {})
        reports = payload.get("L1_Reports", [])
        
        print(f"🚀 INGESTING ISAC STREAM | Session: {session_id}")
        print(f"📡 System Mode: {config.get('SensingOperationMode')} | Carrier: {config.get('CarrierFrequency_GHz')} GHz")
        print(f"📊 DU Metrics: SIC Isolation: {du_metrics.get('SIC_IsolationAchieved_dB')} dB | Beam Latency: {du_metrics.get('BeamSteeringLatency_us')} µs")
        print("=" * 75)
        
        # Step 2: Loop through parsed L1 target arrays
        for target in reports:
            target_id = target.get("TargetID")
            coords = target.get("Coordinates", {})
            v_doppler = target.get("DeltaV_mps")
            rcs = target.get("RCS_dBsqm")
            snr = target.get("SNR_dB")
            confidence = target.get("ConfidenceLevel_percent")
            
            # Step 3: Deriving absolute range vector: R = sqrt(x^2 + y^2 + z^2)
            abs_range = math.sqrt(coords.get('X', 0)**2 + coords.get('Y', 0)**2 + coords.get('Z', 0)**2)
            
            # Step 4: Resolve signature profiles
            target_class = classify_target_by_rcs(rcs)
            
            # Output structured parsing results
            print(f"🔹 [Target #{target_id}] - Signature Match: {target_class}")
            print(f"   ├─ Real-space Pos : X: {coords.get('X')}m, Y: {coords.get('Y')}m, Z: {coords.get('Z')}m")
            print(f"   ├─ Absolute Range : {abs_range:.2f} meters from TRP antenna")
            print(f"   ├─ Doppler Velocity: {v_doppler} m/s ({'Approaching' if v_doppler < 0 else 'Receding'})")
            print(f"   ├─ Target RCS      : {rcs} dBsqm")
            print(f"   ├─ Detection SNR   : {snr} dB")
            print(f"   └─ Confidence      : {confidence}%")
            print("-" * 75)
            
    except json.JSONDecodeError as json_error:
        print(f"❌ Error decoding 3GPP network bitstream: {json_error}")
    except KeyError as key_error:
        print(f"❌ Missing expected Stage-3 schema parameter: {key_error}")

if __name__ == "__main__":
    process_isac_telemetry(MOCK_ISAC_STREAM)

```

---

### 5G-A sensing progress in 3GPP WGs (July 2026)

1. SA1 Service requirements for ISAC (TS 22.137 - Use cases & requirements).
2. SA2 Study on ISAC (TR 23.700-14 - Study for 5G sensing architecture).
3. SA2 ISAC (TS 23.137 – Specification for 5G sensing architecture).
4. SA3 Study on Security and Privacy Aspects of ISAC (TR 33.777).
5. CT3/CT4 ISAC Sensing Function Services (TS 29.545 - Protocol aspects of the new Sensing Function).
6. RAN1/RAN3 Study on ISAC for NR (TR 38.765 - Performance metric, Performance evaluation, Measurements, RAN-CN procedures and signalling).
7. RAN3 Core part: ISAC for NR (New TSs for Ns general aspects and principles, Ns Layer 1, Ns Signalling transport, NG-RAN; Ns Application Protocol) See RP-261566.

### 6G ISAC references
* https://www.sharetechnote.com/html/6G/6G_Research_ISAC.html

### Note on 6G Spectrum: To solve the severe propagation gap between FR1 and FR2, 6G introduces FR3 (7.125 GHz to 24.25 GHz) as a "golden mid-band" to bridge the coverage of FR1 with the high bandwidth of FR2.

## *Disclaimer*: The above informations needs to be refined with more understanding and knowledge while things will be unfolded by the time. 
