# ISAC Modified Native gNB MAC Scheduler Design Document

## 1. Architectural Overview & Context
This document outlines a highly technical architectural blueprint and implementation specification for modifying a native **3GPP-compliant gNB Medium Access Control (MAC) layer scheduler** to support an **Integrated Sensing and Communications (ISAC)** module. 

The design natively supports **Release 17/18/19 NextGen** frameworks across both **FR1 (Sub-6 GHz)** and **FR2 (mmWave)** spectrums. It accommodates dual-waveform frameworks (**MIMO-OFDM Unified Waveform** and **Split Radar/Cellular Multiplexing**) operating in either **Monostatic** or **Bistatic/Distributed** architectures or topologies.

---

## 2. System Architecture & Component Overhaul

To handle simultaneous communication allocation and sensing physical environmental evaluation, the native MAC layer must move beyond legacy single-objective processing loops.

```
+------------------------------------------------------------+
|                  MAC Scheduler Core Loop                   |
|  - Dynamic Weight Engine (Comm QoS + Sensing Priority)     |
|  - Unified TTI Resource Mapping (Unified / Split Modes)    |
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

### 2.1 Local Sensing Manager (Monostatic Mode)
*   **Role:** Handles local reflection logic when the transmitter and sensing receiver are colocated on the same gNB.
*   **Modification:** Injects microsecond-accurate hardware control pulses down to the L1/RF interface. It coordinates the exact Transmission Time Interval (TTI) slots where the gNB transmits a sensing-heavy waveform and opens an internal localized RX window to capture environmental reflections.

### 2.2 Xn Interface Extensions (Bistatic / Distributed Mode)
*   **Role:** Controls remote nodes when one gNB transmits a waveform and a neighboring gNB captures the scatter/reflection.
*   **Modification:** Expands the **Xn-C (Control)** and **Xn-U (User)** messaging interfaces. When the transmitting MAC schedules a sensing burst, it calculates backhaul transmission delay and issues a cooperative scheduling grant to the target receiver gNB across the Xn link. This alerts the remote node to open its reception capture engine at a precise slot boundary.

---

## 3. Dual-Waveform Scheduling Framework

The modified scheduler executes adaptive allocation by switching between two distinct physical resource mapping methodologies on a per-slot or per-TTI basis:

| Feature / Metric | Unified Waveform (MIMO-OFDM) | Split Multiplexing (Radar + Cellular) |
| :--- | :--- | :--- |
| **Resource Mapping** | Embeds sensing capabilities within standard UE data blocks. | Dedicates isolated physical resources explicitly to radar/sensing functions. |
| **MAC Layer Modification** | Dynamically increases the density/comb configuration of **DMRS** and **CSI-RS** reference signals. | Applies a **Sensing-Reserved Resource Mask** to completely hide specific PRBs from standard UEs. |
| **Spectrum Application** | Optimizes spectral efficiency in **FR1 (Sub-6 GHz)** environments. | Ideal for high-bandwidth isolation in **FR2 (mmWave)** beamformed systems. |
| **Primary Challenge** | High processing overhead for echo clutter suppression at the receiver. | Requires inter-carrier and inter-beam guard bands to block intra-cell self-interference. |

---

## 4. Step-by-Step MAC Scheduling Loop

The sub-millisecond core scheduler loop (e.g., executing every $0.5	ext{ ms}$ for $30	ext{ kHz}$ SCS or $0.25	ext{ ms}$ for $60	ext{ kHz}$ SCS) integrates sensing queue evaluations chronologically:

1.  **Queue Assessment:** The scheduler fetches traditional communication Buffer Status Reports (BSR) while simultaneously evaluating the **Sensing Target Tracking Queue** (stating target update deadlines, required range resolution, and Doppler accuracy targets).
2.  **Metric Computation:** The scheduler computes composite allocation weights combining traditional Proportional Fair (PF) coefficients with dynamic sensing priorities and estimation error bounds (Cramér-Rao Lower Bound).
3.  **Resource Mapping Engine:** 
    *   *If Unified Mode:* maps data to standard UEs but forces an over-provisioned reference signal configuration (dense frequency combs).
    *   *If Split Mode:* strikes out specific Physical Resource Blocks (PRBs) using the reservation mask, allocating remaining slots to communication queues.
4.  **Hardware Dispatch:** Generates Downlink Control Information (DCI) blocks for UEs, triggers local L1 RF switching matrix parameters for Monostatic setups, or dispatches predictive Xn coordination messages for Bistatic configurations.

---

## 5. Native C/C++ Data Structures & Software Logic

The following data structures and scheduling logic outline the necessary additions to a native C/C++ gNB MAC scheduler codebase.

* **Proposals**

### 5.1 Header Definitions (`isac_mac_scheduler.h`)

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

### 5.2 Implementation Framework (`isac_mac_scheduler.c`)

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

## 6. Sub-Millisecond Execution Timing Constraints

To maintain native real-time stability inside the gNB layer without dropping frames, the system architecture must strictly enforce these constraints:

### 6.1 Pre-Computed Lookups
Complex floating-point optimization problems, Lyapunov computations, or Deep Reinforcement Learning inferences for spatial channel predictions must **never** be executed directly inside the real-time slot loop. The scheduler should use static lookup arrays or extract weights updated asynchronously by non-real-time threads.

### 6.2 Bistatic Guarding Windows
Due to variable transport network delay across the Xn backhaul link, the transmitting gNB's MAC layer must issue the target sensing grant details **3 to 5 slots in advance**. This allows the receiver gNB's scheduler context enough overhead time to safely parse the instruction, mask out conflicting communication blocks, and accurately open its L1 reception framework.

## 7. Production Engineering Specification: ISAC MAC Scheduler Design for Native gNB Layer (Advanced design outline)

## 7.1 Advanced C/C++ Data Structures
To implement this architecture within a sub-millisecond Time Transmission Interval (TTI) scheduling loop, the standard MAC control blocks must be extended. Below are the optimized C/C++ data structure definitions.

```cpp
#include <stdint.h>
#include <stdbool.h>

#define MAX_NUM_UES 256
#define MAX_PRBS_FR1 273   // 100MHz SCS 30kHz
#define MAX_PRBS_FR2 264   // 400MHz SCS 120kHz
#define MAX_SENSING_TARGETS 32

typedef enum {
    WAVEFORM_MODE_UNIFIED_OFDM,
    WAVEFORM_MODE_SPLIT_MULTIPLEX
} IsacWaveformMode;

typedef enum {
    ISAC_TOPOLOGY_MONOSTATIC,
    ISAC_TOPOLOGY_BSTATIC_TX,
    ISAC_TOPOLOGY_BSTATIC_RX
} IsacTopologyType;

// Target profiles tracked by the Sensing Processing Unit (SPU)
typedef struct {
    uint32_t target_id;
    IsacTopologyType topology;
    uint32_t associated_gnb_id; // For bistatic operations over Xn
    
    // Kinematic & Tracking Accuracy Requirements
    float expected_range_m;
    float expected_velocity_mps;
    float current_crlb_m2;       // Cramér-Rao Lower Bound tracking error variance
    float threshold_crlb_m2;     // Maximum allowed tracking error variance
    
    uint16_t time_since_last_refresh_slots;
    uint16_t refresh_deadline_slots;
    uint8_t priority_weight;     // Dynamic priority assigned via Lyapunov logic
} IsacSensingTarget;

// Extended User Equipment Control Block (UE-CB) for Communication
typedef struct {
    uint16_t rnti;
    uint8_t cqi;
    uint32_t buffer_status_bytes;
    float average_throughput_bps;
    float alpha_fairness;
} IsacUeControlBlock;

// Real-time scheduling context per slot
typedef struct {
    uint32_t current_slot;
    uint16_t total_available_prbs;
    IsacWaveformMode current_mode;
    
    uint32_t active_ue_count;
    IsacUeControlBlock ue_pool[MAX_NUM_UES];
    
    uint32_t active_target_count;
    IsacSensingTarget target_pool[MAX_SENSING_TARGETS];
} IsacSchedulerContext;

// Allocation Grant Command Output to L1 PHY
typedef struct {
    uint16_t rnti;               // 0xFFFF if reserved for dedicated sensing radar chirps
    uint16_t start_prb;
    uint16_t num_prbs;
    uint8_t mcs;
    
    // ISAC Specific Controls
    bool is_sensing_allocated;
    uint8_t dmrs_density_pattern; // Comb-configuration for Unified mode
    uint8_t spatial_beam_id;      // TX/RX analog/hybrid beam index
    
    // Bistatic Triggering Parameters over Xn/L1 interface
    bool trigger_remote_rx;
    uint32_t target_rx_slot;
} IsacMacToPhyGrant;
```

---

## 7.2 Real-Time Sub-Millisecond Scheduling Loop
The native loop runs every slot (e.g., $500\,\mu	ext{s}$ for FR1 $30\,	ext{kHz}$ SCS, or $125\,\mu	ext{s}$ for FR2 $120\,	ext{kHz}$ SCS). To guarantee deadline fulfillment, multi-objective optimization metrics are evaluated via deterministic priority weighting.

```cpp
// Mathematical Metric Formulation:
// Score = Wc * (R_comm / Avg_Throughput) + Ws * (Sensing_Priority / Current_CRLB)
IsacMacToPhyGrant scheduled_grants[MAX_PRBS_FR1];
uint32_t grant_count = 0;

void ExecuteIsacMacSchedulingLoop(IsacSchedulerContext* ctx) {
    grant_count = 0;
    uint16_t remaining_prbs = ctx->total_available_prbs;
    uint16_t allocated_prb_idx = 0;

    // Phase 1: Handle Split Multiplexing Mode Resource Reservation
    if (ctx->current_mode == WAVEFORM_MODE_SPLIT_MULTIPLEX) {
        for (uint32_t i = 0; i < ctx->active_target_count; ++i) {
            IsacSensingTarget* target = &ctx->target_pool[i];
            
            // Critical condition check: Refresh deadline or unacceptable tracking error variance
            if (target->time_since_last_refresh_slots >= target->refresh_deadline_slots || 
                target->current_crlb_m2 > target->threshold_crlb_m2) {
                
                uint16_t sensing_prb_requirement = (ctx->total_available_prbs > 100) ? 32 : 12; // Bandwidth configuration based on FR band
                if (remaining_prbs >= sensing_prb_requirement) {
                    IsacMacToPhyGrant sensing_grant = {
                        .rnti = 0xFFFF, // Reserved Radar RNTI
                        .start_prb = allocated_prb_idx,
                        .num_prbs = sensing_prb_requirement,
                        .mcs = 0,       // Not applicable for raw radar chirp waveforms
                        .is_sensing_allocated = true,
                        .dmrs_density_pattern = 0,
                        .spatial_beam_id = 0, // Assigned based on tracking coordinates
                        .trigger_remote_rx = (target->topology == ISAC_TOPOLOGY_BSTATIC_TX),
                        .target_rx_slot = ctx->current_slot // Assuming zero-delay or propagation bounded slot index
                    };
                    
                    scheduled_grants[grant_count++] = sensing_grant;
                    remaining_prbs -= sensing_prb_requirement;
                    allocated_prb_idx += sensing_prb_requirement;
                }
            }
        }
    }

    // Phase 2: Compute Dynamic Multi-Objective Metric for UEs & Remaining PRBs
    while (remaining_prbs > 0 && ctx->active_ue_count > 0) {
        float best_score = -1.0f;
        int best_ue_idx = -1;
        
        for (uint32_t u = 0; u < ctx->active_ue_count; ++u) {
            IsacUeControlBlock* ue = &ctx->ue_pool[u];
            if (ue->buffer_status_bytes == 0) continue;

            float r_comm = (float)ue->cqi * 1024.0f; // Simplified transmission capacity mapper
            float w_c = 0.6f;                        // Baseline communication weight
            float w_s = 0.4f;                        // Baseline sensing dual weight
            
            // Unified Waveform Metric calculation injection
            float sensing_utility = 0.0f;
            if (ctx->current_mode == WAVEFORM_MODE_UNIFIED_OFDM) {
                for (uint32_t t = 0; t < ctx->active_target_count; ++t) {
                    IsacSensingTarget* target = &ctx->target_pool[t];
                    // Sensing value increases as accuracy drops (higher CRLB error)
                    sensing_utility += (float)target->priority_weight * (target->current_crlb_m2);
                }
            }

            float pf_comm_score = r_comm / (ue->average_throughput_bps + 1.0f);
            float current_score = (w_c * pf_comm_score) + (w_s * sensing_utility);

            if (current_score > best_score) {
                best_score = current_score;
                best_ue_idx = u;
            }
        }

        if (best_ue_idx == -1) break; // No UEs with active traffic buffers

        // Phase 3: Allocate to Selected Best Candidate
        uint16_t allocation_chunk = (remaining_prbs > 16) ? 16 : remaining_prbs;
        IsacUeControlBlock* target_ue = &ctx->ue_pool[best_ue_idx];

        IsacMacToPhyGrant ue_grant = {
            .rnti = target_ue->rnti,
            .start_prb = allocated_prb_idx,
            .num_prbs = allocation_chunk,
            .mcs = target_ue->cqi + 4, // Pseudo mapping to MCS index Table
            .is_sensing_allocated = (ctx->current_mode == WAVEFORM_MODE_UNIFIED_OFDM),
            .dmrs_density_pattern = (ctx->current_mode == WAVEFORM_MODE_UNIFIED_OFDM) ? 2 : 1, // High density comb assignment
            .spatial_beam_id = 1,
            .trigger_remote_rx = false,
            .target_rx_slot = 0
        };

        scheduled_grants[grant_count++] = ue_grant;
        remaining_prbs -= allocation_chunk;
        allocated_prb_idx += allocation_chunk;
        target_ue->buffer_status_bytes -= (target_ue->buffer_status_bytes > 4096) ? 4096 : target_ue->buffer_status_bytes;
    }
}
```

---

## 7.3 Asynchronous Dynamic Weight Engine (Lyapunov Optimization)
Executing drift-plus-penalty tracking optimizations directly in the TTI execution context violates execution deadlines. The design shifts this logic into an asynchronous background system thread utilizing a low-latency thread ring buffer.

```cpp
#include <pthread.h>
#include <unistd.h>

#define LYAPUNOV_V_PARAM 20.0f // Tradeoff parameter for drift stabilization vs sensing penalty

typedef struct {
    float queue_backlog_history[MAX_NUM_UES];
    float tracking_error_history[MAX_SENSING_TARGETS];
    bool is_running;
} IsacAsyncEngineContext;

IsacAsyncEngineContext g_async_ctx;
pthread_t async_engine_thread;

void* RunAsynchronousWeightEngine(void* arg) {
    IsacAsyncEngineContext* engine = (IsacAsyncEngineContext*)arg;
    
    while (engine->is_running) {
        // Step 1: Sleep for multi-slot observation window epoch (e.g., every 10 or 20 slots)
        usleep(10000); 

        // Step 2: Compute virtual queue drift and calculate modified priority weights
        // Minimize dynamic bounds: Drift + V * E[Penalty]
        for (int i = 0; i < MAX_SENSING_TARGETS; ++i) {
            // Placeholder: Access target_pool and tune values safely via thread synchronization primitives
            // Modifying target_pool[i].priority_weight based on cumulative backlog vs tracking drift
        }
    }
    return NULL;
}

void StartAsynchronousWeightEngine() {
    g_async_ctx.is_running = true;
    pthread_create(&async_engine_thread, NULL, RunAsynchronousWeightEngine, &g_async_ctx);
}

void StopAsynchronousWeightEngine() {
    g_async_ctx.is_running = false;
    pthread_join(async_engine_thread, NULL);
}
```

---

## 8. Downlink Control Information (DCI) Reference Design
To trigger varying reference signal grids for unified operation seamlessly without forcing structural modifications down to legacy UEs, the scheduler uses **3GPP Release 18 DCI Format 1_1 / 1_2 extensions**.

### DCI Field Multi-Bit Bitmask Configurations
1.  **DMRS Position & Density Field (2-3 Bits):** 
    *   `00`: Standard single-symbol pattern for optimization of clean spectral efficiency.
    *   `01`: Double-symbol enhanced pattern mapping for fast channels.
    *   `10`: **ISAC Unified High-Density Pattern** (Forces maximum subcarrier comb spacing, converting subcarriers into high-resolution radar range profiles).
2.  **Sensing Indicator Mapping Field (1 Bit):**
    *   `0`: Indicates typical consumer terminal frame composition.
    *   `1`: Informs advanced terminals that co-located spatial parameters contain radar structural beams, instructing the terminal to activate echo reflection suppression filters locally.

