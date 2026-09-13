# 5G-Advanced ISAC Signaling Flow & Parameter Specification

This document details the control plane and physical layer signaling architecture for **5G-Advanced (5G-A) Integrated Sensing and Communication (ISAC)** based on 3GPP Release 19 requirements and Release 20 architectural studies (**TS 22.137**, **TS 23.137**, and **TR 38.765**).

---

## 🏗️ Protocol Stack Architecture & Signaling Topology

The 5G-A ISAC framework maps high-level application sensing requests down to sub-millisecond physical layer slots. The pipeline moves from the core network down to the Distributed Unit (DU) and Radio Unit (RU).

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

The L3 layer handles session establishment, target geofencing, and multi-node topology assignment. The core network's **Sensing Function (SF)** initiates requests via service APIs over the Service-Based Interface (SBI).

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

The DU interprets the L3 constraints and converts them into explicit subcarrier schedules, time slots, and spatial beams.

### DU Internal Scheduling Engine Parameters (3GPP TR 38.765 Baseline)
```ini
[Carrier_Settings]
CarrierFrequency = 4.0 GHz        # Target mid-band testing frequency
SubcarrierSpacing = 30 kHz        # FR1 numerology (μ=1)
TotalBandwidth_RBs = 273          # ~100 MHz active allocation channel

[Sensing_Slot_Muxing]
MultiplexingMethod = TDM          # Time-Division Multiplexing (Comm vs Sensing slots)
SensingPeriodicity_Slots = 80     # Trigger sensing profile every 80 slots (1.25 ms PRI)
CombSize = 4                      # Frequency density spacing pattern (Comb-4 structure)
TotalActiveSubcarriers = 125      # Spectral footprint allocated per sensing comb
```

* **Self-Interference Cancellation (SIC) Thresholds:** For monostatic architectures, the DU enforces cross-correlation digital cancellation vectors. It targets a baseline isolation threshold of **&ge; 90 to 110 dB** across the active transceivers to prevent active high-power downlink bleed from masking weak target echoes.
* **Activation via MAC-CE:** To optimize scheduling delay, the DU fires off rapid **MAC Control Elements (MAC-CE)** or **Downlink Control Information (DCI)** toggles to alter beam directions or pause sensing routines if prioritized data traffic spikes.

---

## 3. MAC Layer & Software Scheduler Engineering

The MAC scheduler balances user cellular SLAs against the periodic insertion of deterministic sensing waveforms.

### A. Multiplexing Implementation Architectures
* **Time-Division Multiplexing (TDM):** Dedicated slots or individual OFDM symbols are locked exclusively for sensing pulses (e.g., PRS bursts). This eliminates self-interference entirely but causes a "throughput cliff" by pausing user traffic.
* **Frequency-Division Multiplexing (FDM):** Interleaves sensing pilots on specific resource blocks alongside active PDSCH/PUSCH data channels. This allows parallel execution, but requires the software to schedule guard bands to prevent target Doppler shifts from bleeding into user data.
* **Spatial Multiplexing (SDM):** Utilizes Massive MIMO to calculate isolated precoding vectors ($\mathbf{W}_{	ext{comm}}$ and $\mathbf{W}_{	ext{sense}}$), firing data toward a user while simultaneously sweeping an analytical sensing beam across a target sector.

### B. Comb Pattern Selection Frameworks
To preserve spectral resources, the scheduler maps sparse pilots instead of full-grid allocations:
* **Comb-2 Structure:** Maximum energy capture per pulse, extending the **Maximum Detectable Range**. However, it halves the resource grid block availability for standard data pipelines.
* **Comb-4 / Comb-6 Structure:** Maximizes commercial user capacity but increases **Range Ambiguity** constraints, forcing the software layer to process secondary math equations to filter out "ghost targets."

### C. Dynamic Frame Restructuring & Adaptive Bursts
The scheduler monitors prediction errors from tracking loops. If an uncooperative target (e.g., a drone) exhibits unpredictable acceleration, the MAC layer overrides standard communication frames to spin up a high-frequency tracking burst:

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

The physical layer dictates the exact over-the-air waveform geometry used to maximize target echo extraction.

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

1. **Sample Level (I/Q Data):** Raw phase/amplitude values indexable by time, frequency, and antenna beam. Provides maximum flexibility for external AI/ML pattern engines but demands high link bandwidth.
2. **Parameter Profile Level:** Compiled **Delay-Doppler matrices** or intermediate **Angle-of-Arrival (AoA)** angular profiles to compress reporting payloads.
3. **Detected Point Level (Point Cloud):** Cleaned and parsed object states mapping directly to the application tracker:
   $$\text{State Vector} = [X, Y, Z, \Delta v, \sigma_{\text{RCS}}, t_{\text{timestamp}}]$$
   *Where $\sigma_{\text{RCS}}$ is the calculated **Radar Cross Section** (expressed in dBsqm), enabling classification filters to separate targets (e.g., distinguishing an unauthorized drone from a bird).*

---

## 📊 5. Protocol Stack & Information Exchange Matrix

| Phase | Layer | Primary Interfaces | Core Parameter Payload Elements |
| :--- | :--- | :--- | :--- |
| **1. Request** | L3 Core to CU | HTTP/SBI, NGAP | `SensingSessionID`, `SensingOperationMode`, `TargetType`, `Sensing_QoS_Profile`, Geo-coordinates. |
| **2. Allocation** | L3 CU to DU | F1AP, RRC Reconfig | `Sensing Resource Coordination Information IE`, Resource Pool allocation maps, Reference Signal Type. |
| **3. Activation**| L2 DU to RU/UE| MAC-CE, DCI | Immediate slot assignments, Beam Index configurations, Fast activation toggles. |
| **4. Execution** | L1 Over-the-Air| OFDM Radio Links | Repurposed PRS/SRS/CSI-RS comb configurations, pulse timing bursts, phase echo tracking. |
| **5. Reporting** | Cross-Layer | L1/L2 Feedbacks ➔ F1AP | Delay-Doppler matrices, extracted AoA profiles, state tracking vectors $[X, Y, Z, \Delta v, \sigma_{\text{RCS}}]$. |


```python

import json
import math
import time

# Mock telemetry payload simulating a live L1 reporting interval 
# structured following 3GPP TR 38.765/TS 23.137 configurations
MOCK_ISAC_STREAM = """
{
  "SensingSessionID": "isac-sfn-zone-4029",
  "Timestamp": 1789234567.89,
  "Configuration": {
    "SensingOperationMode": "TRP_Monostatic",
    "CarrierFrequency_GHz": 4.0,
    "SubcarrierSpacing_kHz": 30
  },
  "L1_Reports": [
    {
      "TargetID": 101,
      "Coordinates": {"X": 154.2, "Y": -42.8, "Z": 85.5},
      "DeltaV_mps": 18.5,
      "RCS_dBsqm": -5.2
    },
    {
      "TargetID": 102,
      "Coordinates": {"X": 12.1, "Y": 305.4, "Z": 12.0},
      "DeltaV_mps": -1.2,
      "RCS_dBsqm": -18.5
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
        reports = payload.get("L1_Reports", [])
        
        print(f"🚀 INGESTING ISAC STREAM | Session: {session_id}")
        print(f"📡 System Mode: {config.get('SensingOperationMode')} | Carrier: {config.get('CarrierFrequency_GHz')} GHz")
        print("=" * 65)
        
        # Step 2: Loop through parsed L1 target arrays
        for target in reports:
            target_id = target.get("TargetID")
            coords = target.get("Coordinates", {})
            v_doppler = target.get("DeltaV_mps")
            rcs = target.get("RCS_dBsqm")
            
            # Step 3: Deriving absolute range vector: R = sqrt(x^2 + y^2 + z^2)
            abs_range = math.sqrt(coords.get('X', 0)**2 + coords.get('Y', 0)**2 + coords.get('Z', 0)**2)
            
            # Step 4: Resolve signature profiles
            target_class = classify_target_by_rcs(rcs)
            
            # Output structured parsing results
            print(f"🔹 [Target #{target_id}] - Signature Match: {target_class}")
            print(f"   ├─ Real-space Pos : X: {coords.get('X')}m, Y: {coords.get('Y')}m, Z: {coords.get('Z')}m")
            print(f"   ├─ Absolute Range : {abs_range:.2f} meters from TRP antenna")
            print(f"   ├─ Doppler Velocity: {v_doppler} m/s ({'Approaching' if v_doppler < 0 else 'Receding'})")
            print(f"   └─ Target RCS      : {rcs} dBsqm")
            print("-" * 65)
            
    except json.JSONDecodeError as json_error:
        print(f"❌ Error decoding 3GPP network bitstream: {json_error}")
    except KeyError as key_error:
        print(f"❌ Missing expected Stage-3 schema parameter: {key_error}")

if __name__ == "__main__":
    process_isac_telemetry(MOCK_ISAC_STREAM)

```


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
