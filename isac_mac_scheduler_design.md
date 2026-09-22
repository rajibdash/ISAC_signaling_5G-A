# ISAC-Modified Native gNB MAC Scheduler Design Specification

This document provides a highly technical architectural blueprint and implementation specification for modifying a native **3GPP-compliant gNB Medium Access Control (MAC) layer scheduler** to support an **Integrated Sensing and Communications (ISAC)** module. 

The design natively supports **Release 17/18/19 NextGen** frameworks across both **FR1 (Sub-6 GHz)** and **FR2 (mmWave)** spectrums. It accommodates dual-waveform frameworks (**MIMO-OFDM Unified Waveform** and **Split Radar/Cellular Multiplexing**) operating in either **Monostatic** or **Bistatic/Distributed** architectures.

---

## 1. System Architecture & Component Overhaul

To handle simultaneous communication allocation and sensing environmental evaluation, the native MAC layer must move beyond legacy single-objective processing loops.

```
+------------------------------------------------------------+
|                  MAC Scheduler Core Loop                   |
|  - Dynamic Weight Engine (Comm QoS + Sensing Priority)    |
|  - Unified TTI Resource Mapping (Unified / Split Modes)   |
+-----------------------------+------------------------------+
                              |
       +----------------------+----------------------+
       | (Monostatic Path)                           | (Bistatic Path)
       v                                             v
+-----------------------------+               +-----------------------------+
|    Local Sensing Manager    |               |  Xn-U/Xn-C Interface Ext.   |
| - RX Timing Window Control  |               | - Inter-gNB Phase Sync      |
| - Local Echo Triggering     |               | - Remote RX Triggering      |
+--------------+--------------+               +--------------+--------------+
               |                                             |
               +----------------------+----------------------+
                                      |
                                      v
                       +-----------------------------+
                       |       PHY Layer (L1)        |
                       +-----------------------------+
```

### 1.1 Local Sensing Manager (Monostatic Mode)
*   **Role:** Handles local reflection logic when the transmitter and sensing receiver are colocated on the same gNB.
*   **Modification:** Injects microsecond-accurate hardware control pulses down to the L1/RF interface. It coordinates the exact Transmission Time Interval (TTI) slots where the gNB transmits a sensing-heavy waveform and opens an internal localized RX window to capture environmental reflections.

### 1.2 Xn Interface Extensions (Bistatic / Distributed Mode)
*   **Role:** Controls remote nodes when one gNB transmits a waveform and a neighboring gNB captures the scatter/reflection.
*   **Modification:** Expands the **Xn-C (Control)** and **Xn-U (User)** messaging interfaces. When the transmitting MAC schedules a sensing burst, it calculates backhaul transmission delay and issues a cooperative scheduling grant to the target receiver gNB across the Xn link. This alerts the remote node to open its reception capture engine at a precise slot boundary.

---

## 2. Dual-Waveform Scheduling Framework

The modified scheduler executes adaptive allocation by switching between two distinct physical resource mapping methodologies on a per-slot basis:

| Feature / Metric | Unified Waveform (MIMO-OFDM) | Split Multiplexing (Radar + Cellular) |
| :--- | :--- | :--- |
| **Resource Mapping** | Embeds sensing capabilities within standard UE data blocks. | Dedicates isolated physical resources explicitly to radar/sensing functions. |
| **MAC Layer Modification** | Dynamically increases the density/comb configuration of **DMRS** and **CSI-RS** reference signals. | Applies a **Sensing-Reserved Resource Mask** to completely hide specific PRBs from standard UEs. |
| **Spectrum Application** | Optimizes spectral efficiency in **FR1 (Sub-6 GHz)** environments. | Ideal for high-bandwidth isolation in **FR2 (mmWave)** beamformed systems. |
| **Primary Challenge** | High processing overhead for echo clutter suppression at the receiver. | Requires inter-carrier and inter-beam guard bands to block intra-cell self-interference. |

---

## 3. Step-by-Step MAC Scheduling Loop

The sub-millisecond core scheduler loop (e.g., executing every $0.5	ext{ ms}$ for $30	ext{ kHz}$ SCS or $0.25	ext{ ms}$ for $60	ext{ kHz}$ SCS) integrates sensing queue evaluations chronologically:

1.  **Queue Assessment:** The scheduler fetches traditional communication Buffer Status Reports (BSR) while simultaneously evaluating the **Sensing Target Tracking Queue** (stating target update deadlines, required range resolution, and Doppler accuracy targets).
2.  **Metric Computation:** The scheduler computes composite allocation weights combining traditional Proportional Fair (PF) coefficients with dynamic sensing priorities and estimation error bounds (Cramér-Rao Lower Bound).
3.  **Resource Mapping Engine:** 
    *   *If Unified Mode:* maps data to standard UEs but forces an over-provisioned reference signal configuration (dense frequency combs).
    *   *If Split Mode:* strikes out specific Physical Resource Blocks (PRBs) using the reservation mask, allocating remaining slots to communication queues.
4.  **Hardware Dispatch:** Generates Downlink Control Information (DCI) blocks for UEs, triggers local L1 RF switching matrix parameters for Monostatic setups, or dispatches predictive Xn coordination messages for Bistatic configurations.

---

## 4. Native C/C++ Data Structures & Software Logic

The following data structures and scheduling logic outline the necessary additions to a native C/C++ gNB MAC scheduler codebase.

### 4.1 Header Definitions (`isac_mac_scheduler.h`)

```cpp
#ifndef ISAC_MAC_SCHEDULER_H
#define ISAC_MAC_SCHEDULER_H

#include <stdint.h>
#include <stdbool.h>

#define MAX_UE_COUNT         256
#define MAX_PRB_COUNT        273  // Standard for 100MHz bandwidth at 30kHz SCS
#define MAX_SENSING_TARGETS  16

typedef enum {
    WAVEFORM_MODE_UNIFIED_OFDM,
    WAVEFORM_MODE_SPLIT_MULTIPLEXING
} IsacWaveformMode;

typedef enum {
    ARCH_TYPE_MONOSTATIC,
    ARCH_TYPE_BISTATIC_TX,
    ARCH_TYPE_BISTATIC_RX
} IsacArchitectureType;

// Tracks active sensing jobs (e.g., drone tracking, traffic monitoring)
typedef struct {
    uint8_t  target_id;
    uint32_t last_updated_slot;
    uint32_t refresh_rate_slots; // Target latency deadline for tracking refresh
    float    required_crlb;      // Maximum tolerable Cramér-Rao Lower Bound
    float    priority_weight;    // Dynamic sensing priority indicator
    uint16_t estimated_azimuth;  // Spatial target area info for beamforming
    uint16_t estimated_elevation;
} SensingTargetContext;

// Tracks individual communication User Equipment metrics
typedef struct {
    uint16_t rnti;
    uint32_t buffer_size_bytes;
    uint8_t  cqi;
    float    historical_throughput;
    float    instantaneous_rate;
} UeCommContext;

// Master tracking structure for the modified MAC Scheduler Instance
typedef struct {
    IsacWaveformMode     current_mode;
    IsacArchitectureType arch_type;
    uint32_t             current_slot;
    
    uint16_t             total_active_ues;
    UeCommContext        ue_pool[MAX_UE_COUNT];
    
    uint16_t             total_sensing_targets;
    SensingTargetContext target_pool[MAX_SENSING_TARGETS];
    
    // Mask array marking PRB availability (true = reserved strictly for sensing)
    bool                 prb_sensing_mask[MAX_PRB_COUNT];
} IsacMacSchedulerContext;

// Core Scheduling functions
void Isac_Scheduler_Init(IsacMacSchedulerContext* ctx, IsacWaveformMode mode, IsacArchitectureType arch);
void Isac_Evaluate_Sensing_Mask(IsacMacSchedulerContext* ctx);
void Isac_Execute_Unified_Allocation(IsacMacSchedulerContext* ctx);
void Isac_Execute_Split_Allocation(IsacMacSchedulerContext* ctx);

#endif // ISAC_MAC_SCHEDULER_H
```

### 4.2 Implementation Framework (`isac_mac_scheduler.c`)

```cpp
#include "isac_mac_scheduler.h"
#include <stdio.h>
#include <math.h>

void Isac_Scheduler_Init(IsacMacSchedulerContext* ctx, IsacWaveformMode mode, IsacArchitectureType arch) {
    ctx->current_mode = mode;
    ctx->arch_type = arch;
    ctx->current_slot = 0;
    ctx->total_active_ues = 0;
    ctx->total_sensing_targets = 0;
    
    for (int i = 0; i < MAX_PRB_COUNT; i++) {
        ctx->prb_sensing_mask[i] = false;
    }
}

void Isac_Evaluate_Sensing_Mask(IsacMacSchedulerContext* ctx) {
    // Clear old sensing masks
    for (int i = 0; i < MAX_PRB_COUNT; i++) {
        ctx->prb_sensing_mask[i] = false;
    }

    if (ctx->current_mode != WAVEFORM_MODE_SPLIT_MULTIPLEXING) {
        return; // Unified mode doesn't block physical PRBs from communication
    }

    // Scan sensing targets to see if any require dedicated structural resource chunks
    for (int t = 0; t < ctx->total_sensing_targets; t++) {
        SensingTargetContext* target = &ctx->target_pool[t];
        uint32_t slots_passed = ctx->current_slot - target->last_updated_slot;
        
        // If a tracking threshold window is expiring, hard-allocate a physical radar block
        if (slots_passed >= target->refresh_rate_slots) {
            // Dedicate upper 25% of PRB grid exclusively to radar chirp generation
            uint16_t start_prb = (uint16_t)(MAX_PRB_COUNT * 0.75);
            for (uint16_t p = start_prb; p < MAX_PRB_COUNT; p++) {
                ctx->prb_sensing_mask[p] = true;
            }
            break; // Allocation completed for the urgent target block
        }
    }
}

void Isac_Execute_Split_Allocation(IsacMacSchedulerContext* ctx) {
    Isac_Evaluate_Sensing_Mask(ctx);

    // Loop through PRBs to distribute communication grants on unmasked channels
    for (int prb = 0; prb < MAX_PRB_COUNT; prb++) {
        if (ctx->prb_sensing_mask[prb] == true) {
            // This PRB is reserved for native radar or bistatic cross-triggering
            // Trigger internal L1 registers or dispatch Xn transmission framework instructions
            continue; 
        }

        int best_ue_index = -1;
        float highest_metric = -1.0f;

        // Process communication metrics using modified multi-objective scheduling logic
        for (int u = 0; u < ctx->total_active_ues; u++) {
            UeCommContext* ue = &ctx->ue_pool[u];
            if (ue->buffer_size_bytes == 0) continue;

            // Base metric: Proportional Fair score
            float pf_metric = ue->instantaneous_rate / (ue->historical_throughput + 1e-6f);
            
            if (pf_metric > highest_metric) {
                highest_metric = pf_metric;
                best_ue_index = u;
            }
        }

        if (best_ue_index != -1) {
            // Allocate the specific PRB block to the winning communication user channel
            ctx->ue_pool[best_ue_index].buffer_size_bytes -= 50; // Emulate packet distribution
        }
    }
}

void Isac_Execute_Unified_Allocation(IsacMacSchedulerContext* ctx) {
    // In Unified Mode, every PRB is accessible for communication, 
    // but the allocation framework optimizes for joint performance metrics.
    for (int prb = 0; prb < MAX_PRB_COUNT; prb++) {
        int best_ue_index = -1;
        float max_joint_metric = -10000.0f;

        for (int u = 0; u < ctx->total_active_ues; u++) {
            UeCommContext* ue = &ctx->ue_pool[u];
            if (ue->buffer_size_bytes == 0) continue;

            // 1. Calculate Communication Parameter weight
            float r_comm = ue->instantaneous_rate / (ue->historical_throughput + 1e-6f);

            // 2. Synthesize Sensing Benefit & Penalty variables
            float p_sense = 0.0f;
            float crlb_penalty = 0.0f;

            for (int t = 0; t < ctx->total_sensing_targets; t++) {
                SensingTargetContext* target = &ctx->target_pool[t];
                p_sense += target->priority_weight;
                
                // Emulate tracking prediction error additions (Cramér-Rao Lower Bound evaluation)
                crlb_penalty += (target->required_crlb * 0.1f); 
            }

            // Multi-Objective Utility Function Equation
            // Metric = (Wc * R_comm) + (Ws * P_sense) - (Lambda * CRLB)
            float w_c = 0.6f; 
            float w_s = 0.4f;
            float lambda = 0.2f;
            
            float joint_metric = (w_c * r_comm) + (w_s * p_sense) - (lambda * crlb_penalty);

            if (joint_metric > max_joint_metric) {
                max_joint_metric = joint_metric;
                best_ue_index = u;
            }
        }

        if (best_ue_index != -1) {
            // Allocate PRB to the communication UE while concurrently signaling the L1 PHY 
            // module to inject dense DMRS/CSI-RS comb configurations for simultaneous sensing tracking.
            ctx->ue_pool[best_ue_index].buffer_size_bytes -= 50; 
        }
    }
}
```

---

## 5. Sub-Millisecond Execution Timing Constraints

To maintain native real-time stability inside the gNB layer without dropping frames, the system architecture must strictly enforce these constraints:

### 5.1 Pre-Computed Lookups
Complex floating-point optimization problems, Lyapunov computations, or Deep Reinforcement Learning inferences for spatial channel predictions must **never** be executed directly inside the real-time slot loop. The scheduler should use static lookup arrays or extract weights updated asynchronously by non-real-time threads.

### 5.2 Bistatic Guarding Windows
Due to variable transport network delay across the Xn backhaul link, the transmitting gNB's MAC layer must issue the target sensing grant details **3 to 5 slots in advance**. This allows the receiver gNB's scheduler context enough overhead time to safely parse the instruction, mask out conflicting communication blocks, and accurately open its L1 reception framework.
