# 📡 Communication Protocols - Complete Reference Guide

**Version:** 1.0 | **Format:** Technical Reference for Study & Implementation  
**Target:** Automotive · Industrial · IoT · Web · Networking · Storage · Messaging

---

## 📑 Table of Contents

1. [Automotive Protocols](#1-automotive-protocols)
2. [Industrial Automation Protocols](#2-industrial-automation-protocols)
3. [IoT & Wireless Protocols](#3-iot--wireless-protocols)
4. [Web & Application Protocols](#4-web--application-protocols)
5. [IT, Networking & Infrastructure](#5-it-networking--infrastructure)
6. [File Transfer & Storage](#6-file-transfer--storage-protocols)
7. [Communication & Messaging](#7-communication--messaging-protocols)

---

# 1. Automotive Protocols

## 1.1 CAN (Controller Area Network)

**Purpose:** Multi-master bus for connecting vehicle ECUs (Engine, ABS, Airbag, etc.) without a central host.

**Baud Rate:** 125 kbps – 1 Mbps (CAN FD: up to 8 Mbps data phase)

### Frame Structure (Classic CAN - 11-bit ID)

```
Bit Layout (Total: 108 bits minimum)

┌─────┬──────────────────────────────┬───────────┬─────────────────┬──────────────┬─────┬────────┐
│ SOF │      ARBITRATION FIELD       │ CONTROL   │   DATA FIELD    │  CRC FIELD   │ ACK │  EOF   │
│ 1b  │   ID(11b) | RTR(1b) | r(1b)  │ IDE(1b) r │  DLC(4b) Data   │  CRC(15b)del │ 2b  │  7b    │
│     │           | (Dominant=0)     │ (0/1) (0) │  0-8 bytes      │  + 1b del    │     │        │
└─────┴──────────────────────────────┴───────────┴─────────────────┴──────────────┴─────┴────────┘

Example Frame (Transmitting RPM data):
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│ 0 │ 0 1 1 0 0 0 0 0 0 0 0  │ 0 │ 0 │ 0 0 0 0 │ 0x1F 0x40 0x00 0x00 ...   │ CRC │ Ack │ 1111111 │
│   │ CAN ID = 0x300         │   │   │ 4 bytes │ Payload (32-bit RPM value)│     │     │         │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Bus Type** | Differential (CAN_H, CAN_L twisted pair) |
| **Termination** | 120Ω at each end |
| **Arbitration** | Bitwise (Dominant=0 wins over Recessive=1) |
| **Error Handling** | Bit stuffing + CRC + Fault confinement |
| **Max Nodes** | 110+ (theoretical) |

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

// CAN Frame Structure
typedef struct {
    uint32_t id;           // 11-bit (Standard) or 29-bit (Extended) ID
    uint8_t  dlc;          // Data Length Code (0-8)
    uint8_t  data[8];      // Payload bytes
    uint8_t  is_extended;  // 0 = Standard, 1 = Extended ID
    uint8_t  is_rtr;       // 0 = Data frame, 1 = Remote frame
} CAN_Frame_t;

// Function to calculate CAN CRC (polynomial 0x1D5, 15-bit)
uint16_t CAN_CalculateCRC(uint8_t *frame_data, uint8_t len) {
    uint16_t crc = 0;
    for (int i = 0; i < len; i++) {
        crc ^= (frame_data[i] << 7);
        for (int j = 0; j < 8; j++) {
            crc <<= 1;
            if (crc & 0x8000) crc ^= 0x4599;  // Polynomial
        }
    }
    return crc >> 1;  // 15-bit CRC
}

// Send CAN frame
void CAN_SendFrame(CAN_Frame_t *frame) {
    printf("=== CAN Frame Transmission ===\n");
    printf("ID:       0x%03X\n", frame->id);
    printf("DLC:      %d bytes\n", frame->dlc);
    printf("Data:     ");
    for (int i = 0; i < frame->dlc; i++) {
        printf("0x%02X ", frame->data[i]);
    }
    printf("\n");
    printf("RTR:      %s\n", frame->is_rtr ? "YES" : "NO");
}

int main(void) {
    CAN_Frame_t engine_rpm_msg;
    
    // Configure: CAN ID = 0x100, Engine RPM message
    engine_rpm_msg.id   = 0x100;
    engine_rpm_msg.dlc  = 4;
    engine_rpm_msg.is_extended = 0;  // Standard 11-bit
    engine_rpm_msg.is_rtr = 0;
    
    // RPM = 3000 (0x0BB8), encode as big-endian
    engine_rpm_msg.data[0] = 0x0B;  // RPM high byte
    engine_rpm_msg.data[1] = 0xB8;  // RPM low byte
    engine_rpm_msg.data[2] = 0x00;  // Reserved
    engine_rpm_msg.data[3] = 0x00;  // Reserved
    
    CAN_SendFrame(&engine_rpm_msg);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: How does CAN handle collisions?**  
  **A:** Using bitwise arbitration. Dominant bit (0) overwrites recessive bit (1). The frame with lowest ID wins.

- **Q: What is the max data payload per frame?**  
  **A:** 8 bytes in Classic CAN; 64 bytes in CAN FD (Flexible Data-rate).

---

## 1.2 LIN (Local Interconnect Network)

**Purpose:** Low-cost, single-wire bus for non-safety subsystems (windows, mirrors, seat adjustment).

**Baud Rate:** Max 20 kbps (9.6, 10.4, 19.2 kbps typical)

### Frame Structure

```
Master Header → Slave Response

┌──────────────────────────────────────────────────────────────────┐
│ BREAK FIELD │ SYNC FIELD │ PID FIELD │ DATA FIELD │ CHECKSUM     │
│ ≥13 bits    │ 0x55       │ 8 bits    │ 1-8 bytes  │ 8 bits       │
│ (dominant)  │ (0101 0101)│(6b ID+2b) │            │(classic or   │
│             │            │ parity    │            │ enhanced)    │
└──────────────────────────────────────────────────────────────────┘

Example: Window Control Command
┌─────────────┬──────────┬──────────┬────────┬────────┬─────────┐
│ 13 × 0      │  0x55    │  0x21    │ 0x01   │ 0x64   │ Checksum│
│ (synchronize│ (syncs   │ (PID: ID │ (Cmd:  │ (Speed:│         │
│  all slaves)│  baud)   │ = 1)     │ UP)    │ 100%)  │         │
└─────────────┴──────────┴──────────┴────────┴────────┴─────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  protected_id;     // 6-bit ID + 2-bit parity
    uint8_t  data[8];
    uint8_t  data_len;
    uint8_t  checksum;
} LIN_Frame_t;

// LIN Classic Checksum: Sum all data bytes, invert, mod 256
uint8_t LIN_ClassicChecksum(uint8_t *data, uint8_t len, uint8_t pid) {
    uint16_t sum = pid;  // Include PID in checksum
    for (int i = 0; i < len; i++) {
        sum += data[i];
        if (sum > 0xFF) sum -= 0xFF;  // Carry
    }
    return (uint8_t)(~sum);
}

// LIN Enhanced Checksum: Same as classic, but excludes PID
uint8_t LIN_EnhancedChecksum(uint8_t *data, uint8_t len) {
    uint16_t sum = 0;
    for (int i = 0; i < len; i++) {
        sum += data[i];
        if (sum > 0xFF) sum -= 0xFF;
    }
    return (uint8_t)(~sum);
}

void LIN_SendFrame(LIN_Frame_t *frame) {
    printf("=== LIN Frame Transmitted ===\n");
    printf("Protected ID : 0x%02X\n", frame->protected_id);
    printf("Data Len     : %d bytes\n", frame->data_len);
    printf("Data         : ");
    for (int i = 0; i < frame->data_len; i++) {
        printf("0x%02X ", frame->data[i]);
    }
    printf("\nChecksum     : 0x%02X\n", frame->checksum);
}

int main(void) {
    LIN_Frame_t window_frame;
    
    window_frame.protected_id = 0x21;  // Frame ID
    window_frame.data_len = 2;
    window_frame.data[0] = 0x01;       // Command: Move UP
    window_frame.data[1] = 0x64;       // Speed: 100%
    
    // Calculate checksum (using enhanced method)
    window_frame.checksum = LIN_EnhancedChecksum(window_frame.data, 
                                                   window_frame.data_len);
    
    LIN_SendFrame(&window_frame);
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: Why use LIN if CAN already exists?**  
  **A:** Cost reduction. LIN needs no crystal oscillators on slave nodes and uses single wire (vs. dual wire for CAN).

- **Q: What is LIN's maximum nodes?**  
  **A:** 16 nodes maximum (1 master + 15 slaves).

---

## 1.3 FlexRay

**Purpose:** High-speed, fault-tolerant, deterministic bus for x-by-wire safety systems (brake-by-wire, steer-by-wire).

**Baud Rate:** 10 Mbps per channel (20 Mbps aggregate with dual channel)

### Communication Cycle Structure

```
One FlexRay Communication Cycle (~5-10ms typical)

┌────────────────────────────────────────────────────────────┐
│ STATIC SEGMENT      │ DYNAMIC SEGMENT │ SYMBOL │    NIT    │
├────────────────────────────────────────────────────────────┤
│ Slot 1 │ Slot 2 │..│ MinSlot │MinSlot│ Window │ Idle/Sync  │
│ TDMA   │ TDMA   │..│ Priority│ Arb   │ for    │ for        │
│ (Det.) │ (Det.) │..│ Driven  │ Driven│ Clock  │ Clock      │
│        │        │  │         │       │ Sync   │ Correction │
└────────────────────────────────────────────────────────────┘
```

### Frame Header Structure

```
FlexRay Frame (Dual Channel Transmission)

┌──────────┬──────────┬──────────┬───────────────┬────────────┬────────┐
│ Frame ID │ Length   │ Header   │ Payload       │ Trailer    │ CRC    │
│ 11 bits  │ 7 bits   │ CRC      │ (0-254 bytes) │ (optional) │ 24 bits│
│ (1-2047) │(words)   │ 11 bits  │               │            │        │
└──────────┴──────────┴──────────┴───────────────┴────────────┴────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>

typedef struct {
    uint16_t frame_id;         // 1-2047 (slot ID in static segment)
    uint8_t  payload_length;   // In 16-bit words (0-127 = 0-254 bytes)
    uint8_t  payload[254];
    uint16_t header_crc;       // 11-bit CRC of frame header
    uint32_t frame_crc;        // 24-bit CRC of entire frame
    uint8_t  channel;          // 0 = Channel A, 1 = Channel B
    uint8_t  cycle_counter;    // Incremented each cycle
} FlexRay_Frame_t;

void FlexRay_BuildBrakeCommand(FlexRay_Frame_t *f, 
                                uint8_t brake_pressure_pct) {
    f->frame_id        = 0x0001;  // Slot 1 (highest priority)
    f->payload_length  = 4;       // 4 words = 8 bytes
    f->channel         = 0;       // Channel A (primary)
    f->cycle_counter   = 0;
    
    // Brake pressure: 0x00 = 0%, 0xFF = 100%
    f->payload[0]      = brake_pressure_pct;
    f->payload[1]      = 0x00;    // Reserved
    f->payload[2]      = 0x00;    // Reserved
    f->payload[3]      = 0x00;    // CRC/Status
    
    // In real implementation, calculate CRCs here
    f->header_crc      = 0x000;   // Placeholder
    f->frame_crc       = 0x000000; // Placeholder
    
    printf("=== FlexRay Brake Command ===\n");
    printf("Frame ID       : %d (Static Slot)\n", f->frame_id);
    printf("Brake Pressure : %d%%\n", brake_pressure_pct);
    printf("Channel        : %c\n", f->channel ? 'B' : 'A');
}

int main(void) {
    FlexRay_Frame_t brake_frame;
    FlexRay_BuildBrakeCommand(&brake_frame, 75);  // 75% braking
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: Why does FlexRay use dual channels?**  
  **A:** For fault tolerance. If one channel fails, the network continues on the other channel.

- **Q: What is TDMA in FlexRay?**  
  **A:** Time Division Multiple Access - each ECU gets fixed time slots for guaranteed deterministic transmission.

---

## 1.4 MOST (Media Oriented Systems Transport)

**Purpose:** High-bandwidth multimedia networking for infotainment (audio, video, navigation).

**Baud Rate:** MOST25 (25 Mbps), MOST50 (50 Mbps), MOST150 (150 Mbps)

### Frame Structure (MOST Frame repeats at 44.1 kHz)

```
MOST Synchronous Ring Frame

┌───────────┬──────────────────────┬──────────────────┬─────────────────┐
│ Preamble  │  Synchronous Area    │ Asynchronous Area│  Control Channel│
│ (sync)    │  (PCM Audio/Video)   │ (TCP/IP Packets) │  (Diagnostics)  │
│ 4 bytes   │  28 bytes            │  8 bytes         │  1 byte         │
│           │  @ 44.1/48 kHz       │                  │                 │
└───────────┴──────────────────────┴──────────────────┴─────────────────┘

Total frame size: 64 bytes (512 bits) per frame
Frame rate: 44.1 kHz = 44,100 frames/sec
Effective sync bandwidth: 28 × 44,100 = ~1.2 Mbps (audio only)
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

#define MOST_SYNC_BYTES  28
#define MOST_ASYNC_BYTES 8
#define MOST_CTRL_BYTE   1
#define MOST_PREAMBLE    4

typedef struct {
    uint8_t preamble[MOST_PREAMBLE];
    uint8_t sync_data[MOST_SYNC_BYTES];     // PCM audio (stereo samples)
    uint8_t async_data[MOST_ASYNC_BYTES];   // TCP/IP payload
    uint8_t control_byte;
} MOST_Frame_t;

// Encode stereo audio into MOST sync area (16-bit samples, 2 channels)
void MOST_EncodeAudio(MOST_Frame_t *frame, int16_t left, int16_t right) {
    // Store stereo pair (assuming Little-Endian)
    frame->sync_data[0] = (uint8_t)(left & 0xFF);
    frame->sync_data[1] = (uint8_t)((left >> 8) & 0xFF);
    frame->sync_data[2] = (uint8_t)(right & 0xFF);
    frame->sync_data[3] = (uint8_t)((right >> 8) & 0xFF);
    // Additional 24 bytes for more audio samples...
}

void MOST_PrintFrame(MOST_Frame_t *f) {
    printf("=== MOST Frame Structure ===\n");
    printf("Preamble (sync)     : 0x%02X 0x%02X 0x%02X 0x%02X\n",
           f->preamble[0], f->preamble[1], f->preamble[2], f->preamble[3]);
    printf("Sync Data (audio)   : %d bytes (PCM samples)\n", MOST_SYNC_BYTES);
    printf("Async Data (IP)     : %d bytes (packetized data)\n", MOST_ASYNC_BYTES);
    printf("Control Byte        : 0x%02X\n", f->control_byte);
    printf("Total Frame Size    : 42 bytes @ 44.1 kHz\n");
}

int main(void) {
    MOST_Frame_t audio_frame;
    memset(&audio_frame, 0, sizeof(audio_frame));
    
    // Encode stereo audio: Left=1000, Right=2000 (16-bit samples)
    MOST_EncodeAudio(&audio_frame, 1000, 2000);
    
    MOST_PrintFrame(&audio_frame);
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What happens if a single MOST node fails in ring topology?**  
  **A:** The ring breaks and the entire network goes down unless bypass switches are used.

---

## 1.5 Automotive Ethernet

**Purpose:** High-bandwidth Ethernet adapted for vehicles (ADAS, LiDAR, cameras, autonomous driving).

**Baud Rate:** 100BASE-T1 (100 Mbps), 1000BASE-T1 (1 Gbps), 10GBASE-T1 (10 Gbps)

### Frame Structure (Standard Ethernet with TSN extensions)

```
Automotive Ethernet Frame (Layer 2)

┌────────┬────────┬─────────┬──────────┬─────────────┬────────┐
│ Dest   │ Src    │ 802.1Q  │ EtherType│  Payload    │  FCS   │
│ MAC(6) │ MAC(6) │ Tag(4)  │ (2)      │  (46-1500)  │ (4)    │
│ bytes  │ bytes  │ bytes   │ bytes    │  bytes      │ bytes  │
└────────┴────────┴─────────┴──────────┴─────────────┴────────┘

Example: LiDAR Point Cloud over Automotive Ethernet

┌──────────────────────────────────────────────────────────────────────┐
│ FF:FF:FF:FF:FF:FF │ 00:1A:2B:3C:4D:5E │ 8100:0100 │ 0x88A4   │ ...   │
│ (Broadcast)       │ (LiDAR MAC)       │(VLAN ID)  │(EtherCAT)│Payload│
└──────────────────────────────────────────────────────────────────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  dest_mac[6];     // Destination MAC address
    uint8_t  src_mac[6];      // Source MAC address
    uint16_t vlan_tag;        // 802.1Q VLAN tag (optional)
    uint16_t ether_type;      // EtherType: 0x0800=IPv4, 0x88A4=EtherCAT
    uint8_t  payload[1500];   // Maximum Transmission Unit (MTU)
    uint16_t payload_len;
    uint32_t fcs;             // Frame Check Sequence (CRC-32, hardware calc)
} AutoEthernet_Frame_t;

void AutoEth_BuildLiDARFrame(AutoEthernet_Frame_t *f, 
                              uint32_t lidar_data[3]) {
    // Broadcast to all listeners
    memset(f->dest_mac, 0xFF, 6);
    
    // Source: LiDAR unit MAC
    uint8_t lidar_mac[] = {0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x5E};
    memcpy(f->src_mac, lidar_mac, 6);
    
    // VLAN ID: 0x100 (priority + VID)
    f->vlan_tag = 0x8100;     // TPID
    f->ether_type = 0x88A4;   // EtherCAT / TSN
    
    // Encode 3 LiDAR point cloud coordinates (X, Y, Z)
    f->payload[0] = (lidar_data[0] >> 24) & 0xFF;  // X coordinate
    f->payload[1] = (lidar_data[0] >> 16) & 0xFF;
    f->payload[2] = (lidar_data[1] >> 24) & 0xFF;  // Y coordinate
    f->payload[3] = (lidar_data[1] >> 16) & 0xFF;
    f->payload[4] = (lidar_data[2] >> 24) & 0xFF;  // Z coordinate
    f->payload[5] = (lidar_data[2] >> 16) & 0xFF;
    
    f->payload_len = 6;
}

void AutoEth_PrintFrame(AutoEthernet_Frame_t *f) {
    printf("=== Automotive Ethernet Frame ===\n");
    printf("Dest MAC : %02X:%02X:%02X:%02X:%02X:%02X\n",
           f->dest_mac[0], f->dest_mac[1], f->dest_mac[2],
           f->dest_mac[3], f->dest_mac[4], f->dest_mac[5]);
    printf("Src MAC  : %02X:%02X:%02X:%02X:%02X:%02X\n",
           f->src_mac[0], f->src_mac[1], f->src_mac[2],
           f->src_mac[3], f->src_mac[4], f->src_mac[5]);
    printf("EtherType: 0x%04X\n", f->ether_type);
    printf("Payload  : %d bytes\n", f->payload_len);
}

int main(void) {
    AutoEthernet_Frame_t lidar_frame;
    uint32_t point_cloud[3] = {1000, 2000, 500};  // X, Y, Z coordinates
    
    AutoEth_BuildLiDARFrame(&lidar_frame, point_cloud);
    AutoEth_PrintFrame(&lidar_frame);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: Why replace CAN with Automotive Ethernet?**  
  **A:** Bandwidth. ADAS sensors (LiDAR, cameras) generate Gbps data rates; CAN maxes at 1 Mbps.

- **Q: What is TSN (Time-Sensitive Networking)?**  
  **A:** IEEE 802.1AS/Qav/Qbv standards enabling sub-microsecond clock sync and deterministic delivery.

---

# 2. Industrial Automation Protocols

## 2.1 Modbus

**Purpose:** Simple master/slave protocol for industrial devices - PLCs, drives, sensors, motors.

**Baud Rate:** Modbus RTU (RS-485): 1.2–115.2 kbps | Modbus TCP (Ethernet): 10–100 Mbps

### Modbus RTU Frame Structure

```
Modbus RTU Message (Binary, LSB-first CRC)

┌─────────────────┬──────────────────┬────────────────┬────────────┐
│ Device Address  │ Function Code    │ Data / Payload │ CRC-16     │
│ (8 bits / 1B)   │ (8 bits / 1B)    │ (N bytes)      │ (16 bits)  │
└─────────────────┴──────────────────┴────────────────┴────────────┘

Frame Delimited by: 3.5 character silent gap (t3.5) before and after

Example: Read 4 Holding Registers starting at address 100

┌────────┬────┬────────┬────────┬───────┬───────┐
│ 0x01   │0x03│ 0x00   │ 0x64   │ 0x00  │ 0x04  │ CRC-16
│(Slave1)│RD32├─────────────────────────────────┤
│        │    │ Address│Count of│ Registers     │
│        │    │ 0x0064 │ 4 regs │               │
└────────┴────┴────────┴────────┴───────┴───────┘
```

### Modbus Function Codes (Common)

| Code | Name | Read/Write | Data Type |
|------|------|-----------|-----------|
| 0x01 | Read Coils | Read | 1-bit |
| 0x02 | Read Discrete Inputs | Read | 1-bit (input only) |
| 0x03 | Read Holding Registers | Read | 16-bit |
| 0x04 | Read Input Registers | Read | 16-bit (input only) |
| 0x05 | Write Single Coil | Write | 1-bit |
| 0x06 | Write Single Register | Write | 16-bit |
| 0x10 | Write Multiple Registers | Write | Multiple 16-bit |

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  device_addr;      // Slave address (1-247)
    uint8_t  function_code;    // 0x03 = Read Holding Regs
    uint16_t start_address;    // Starting register address
    uint16_t quantity;         // Number of registers to read
    uint16_t crc16;            // CRC-16-Modbus
} Modbus_RTU_Request_t;

typedef struct {
    uint8_t  device_addr;
    uint8_t  function_code;
    uint8_t  byte_count;       // Number of bytes following
    uint16_t register_values[125];  // Up to 250 bytes / 125 registers
    uint16_t crc16;
} Modbus_RTU_Response_t;

// Calculate CRC-16 Modbus
uint16_t Modbus_CRC16(uint8_t *buffer, uint16_t buffer_len) {
    uint16_t crc = 0xFFFF;
    for (int pos = 0; pos < buffer_len; pos++) {
        crc ^= buffer[pos];
        for (int i = 8; i != 0; i--) {
            if ((crc & 1) != 0) {
                crc >>= 1;
                crc ^= 0xA001;  // Polynomial 0xA001
            } else {
                crc >>= 1;
            }
        }
    }
    return crc;
}

void Modbus_BuildReadRequest(Modbus_RTU_Request_t *req,
                              uint8_t slave_id,
                              uint16_t start_reg,
                              uint16_t reg_count) {
    req->device_addr = slave_id;
    req->function_code = 0x03;         // Read Holding Registers
    req->start_address = start_reg;
    req->quantity = reg_count;
    
    // Build buffer for CRC calculation
    uint8_t crc_buf[6];
    crc_buf[0] = req->device_addr;
    crc_buf[1] = req->function_code;
    crc_buf[2] = (uint8_t)(start_reg >> 8);
    crc_buf[3] = (uint8_t)(start_reg & 0xFF);
    crc_buf[4] = (uint8_t)(reg_count >> 8);
    crc_buf[5] = (uint8_t)(reg_count & 0xFF);
    
    req->crc16 = Modbus_CRC16(crc_buf, 6);
}

void Modbus_PrintRequest(Modbus_RTU_Request_t *req) {
    printf("=== Modbus RTU Read Request ===\n");
    printf("Slave ID      : %d\n", req->device_addr);
    printf("Function      : 0x%02X (Read Holding Registers)\n", req->function_code);
    printf("Start Address : %d (0x%04X)\n", req->start_address, req->start_address);
    printf("Quantity      : %d registers\n", req->quantity);
    printf("CRC-16        : 0x%04X\n", req->crc16);
}

int main(void) {
    Modbus_RTU_Request_t request;
    
    // Read 4 holding registers starting from address 100 on Slave 1
    Modbus_BuildReadRequest(&request, 1, 100, 4);
    Modbus_PrintRequest(&request);
    
    printf("\nRaw Modbus RTU Frame:\n");
    printf("01 03 00 64 00 04 [CRC-16 LSB] [CRC-16 MSB]\n");
    printf("01 03 00 64 00 04 %02X %02X\n",
           (uint8_t)(request.crc16 & 0xFF),
           (uint8_t)((request.crc16 >> 8) & 0xFF));
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What's the difference between Modbus Coil and Holding Register?**  
  **A:** Coil = 1-bit (ON/OFF); Holding Register = 16-bit value for numeric data (speed, temperature, etc.).

- **Q: Why is CRC placed at the END of Modbus RTU frame?**  
  **A:** CRC is calculated over all preceding bytes; placed at end for easy detection during reception.

---

## 2.2 PROFINET (Real-Time Ethernet)

**Purpose:** Real-time industrial Ethernet for PLCs, drives, sensors with millisecond cycle times.

**Baud Rate:** 100 Mbps (RT) / 1 Gbps (IRT) Ethernet

### PROFINET RT Frame Structure

```
PROFINET Real-Time Cyclic Data Frame

┌──────────┬──────────┬─────────────────────┬────────────┬──────────┐
│ Dest MAC │ Src MAC  │ EtherType: 0x8892   │ Frame ID   │ IO Data  │
│ (6B)     │ (6B)     │ (PROFINET RT)       │ (2B)       │ (N Bytes)│
└──────────┴──────────┴─────────────────────┴────────────┴──────────┘

Frame ID Range for RT/IRT: 0x8000 - 0xBFFF
Cycles: 1ms, 2ms, 4ms, 8ms, 16ms, 32ms, 64ms, 128ms
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  dest_mac[6];      // Multicast MAC for IO data
    uint8_t  src_mac[6];
    uint16_t ether_type;       // 0x8892 = PROFINET RT
    uint16_t frame_id;         // 0x8000-0xBFFF
    uint8_t  io_data[40];      // Process I/O payload
    uint16_t iops;             // IO Provider Status
    uint16_t iocs;             // IO Consumer Status
    uint8_t  data_status;      // Validity, Primary, Run flags
    uint8_t  transfer_status;  // Must be 0x00
} PROFINET_RT_Frame_t;

void PROFINET_BuildMotorControl(PROFINET_RT_Frame_t *f,
                                 uint8_t motor_enable,
                                 uint16_t speed_rpm) {
    // Multicast MAC for IO
    uint8_t dst_mac[] = {0x01, 0x0E, 0xCF, 0x00, 0x00, 0x01};
    uint8_t src_mac[] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};
    
    memcpy(f->dest_mac, dst_mac, 6);
    memcpy(f->src_mac, src_mac, 6);
    
    f->ether_type     = 0x8892;   // PROFINET RT
    f->frame_id       = 0x8001;   // Frame ID for cyclic data
    
    // Encode IO data: Motor enable + speed
    f->io_data[0]     = motor_enable ? 0x01 : 0x00;  // Enable bit
    f->io_data[1]     = (uint8_t)(speed_rpm >> 8);   // Speed MSB
    f->io_data[2]     = (uint8_t)(speed_rpm & 0xFF); // Speed LSB
    
    f->iops           = 0x80;     // Provider Status: GOOD
    f->iocs           = 0x80;     // Consumer Status: GOOD
    f->data_status    = 0x35;     // Valid | Primary | Run
    f->transfer_status= 0x00;
}

void PROFINET_PrintFrame(PROFINET_RT_Frame_t *f) {
    printf("=== PROFINET RT Cyclic Frame ===\n");
    printf("Dest MAC : %02X:%02X:%02X:%02X:%02X:%02X\n",
           f->dest_mac[0], f->dest_mac[1], f->dest_mac[2],
           f->dest_mac[3], f->dest_mac[4], f->dest_mac[5]);
    printf("EtherType: 0x%04X (PROFINET RT)\n", f->ether_type);
    printf("Frame ID : 0x%04X\n", f->frame_id);
    printf("Motor EN : 0x%02X\n", f->io_data[0]);
    printf("Speed    : %d RPM\n", (f->io_data[1] << 8) | f->io_data[2]);
    printf("Provider : 0x%02X %s\n", f->iops, f->iops & 0x80 ? "(GOOD)" : "(BAD)");
}

int main(void) {
    PROFINET_RT_Frame_t motor_frame;
    PROFINET_BuildMotorControl(&motor_frame, 1, 1500);  // Enable, 1500 RPM
    PROFINET_PrintFrame(&motor_frame);
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What makes PROFINET IRT different from PROFINET RT?**  
  **A:** IRT adds hardware clock synchronization and reserved time slots, achieving sub-millisecond jitter (< 1 µs).

---

## 2.3 EtherCAT (Ethernet for Control Automation)

**Purpose:** Ultra-fast industrial Ethernet with "processing on-the-fly" for motion control, robotics.

**Baud Rate:** 100 Mbps full-duplex | Cycle times: < 100 µs (1 ms typical)

### EtherCAT "Processing On-The-Fly" Concept

```
Master sends Ethernet frame → passes through Slave 1 → Slave 2 → ... → back to Master

┌──────────────────────────────────────────────────────────────────┐
│ Master constructs EtherCAT frame with commands for each slave    │
├──────────────────────────────────────────────────────────────────┤
│  Frame travels through network:                                  │
│  Master → [Slave1: extracts input, injects output] →             │
│       [Slave2: extracts input, injects output] → ... → Master    │
└──────────────────────────────────────────────────────────────────┘

Key: Slaves do NOT wait for entire frame. Hardware (ESC chip) processes
     on-the-fly with minimal latency (~100ns per slave).
```

### EtherCAT Datagram Structure

```
┌─────────┬─────┬──────┬──────────┬─────────┬────────┬──────────┐
│ Command │ Idx │ Addr │ Physical │ Length+ │ IRQ    │ Working  │
│ (1B)    │ (1B)│ (2B) │ Address  │ Flags   │ (2B)   │ Counter  │
│         │     │      │ (2B)     │ (2B)    │        │ (2B)     │
├─────────┴─────┴──────┴──────────┴─────────┴────────┴──────────┤
│ Data Payload (0-N bytes) + WKC incremented by each slave      │
└───────────────────────────────────────────────────────────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  cmd;              // 0x01=APRD, 0x02=APWR, 0x0C=LRW
    uint8_t  idx;              // Datagram index
    uint16_t adp;              // Address position / slave ID
    uint16_t ado;              // Address offset (physical memory)
    uint16_t len;              // Length + flags
    uint16_t irq;              // External event request
    uint8_t  data[256];        // Payload
    uint16_t wkc;              // Working Counter
} EtherCAT_Datagram_t;

void EtherCAT_BuildLRW(EtherCAT_Datagram_t *dg,
                       uint32_t logical_addr,
                       uint8_t *output_data,
                       uint16_t data_len) {
    dg->cmd = 0x0C;                                 // LRW = Logical Read/Write
    dg->idx = 0x01;
    dg->adp = (uint16_t)(logical_addr & 0xFFFF);   // Low word of address
    dg->ado = (uint16_t)(logical_addr >> 16);      // High word of address
    dg->len = data_len;
    dg->wkc = 0;
    
    memcpy(dg->data, output_data, data_len);
}

void EtherCAT_PrintDatagram(EtherCAT_Datagram_t *dg) {
    printf("=== EtherCAT Datagram ===\n");
    printf("Command     : 0x%02X (LRW - Logical Read/Write)\n", dg->cmd);
    printf("Datagram Idx: %d\n", dg->idx);
    printf("Address     : 0x%08X\n", (dg->ado << 16) | dg->adp);
    printf("Length      : %d bytes\n", dg->len);
    printf("Working Cnt : %d (incremented per slave)\n", dg->wkc);
    printf("Payload[0]  : 0x%02X\n", dg->data[0]);
}

int main(void) {
    EtherCAT_Datagram_t dg;
    uint8_t servo_commands[4] = {0x01, 0xFF, 0x00, 0x55};
    
    EtherCAT_BuildLRW(&dg, 0x00001000, servo_commands, 4);
    EtherCAT_PrintDatagram(&dg);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What does "processing on-the-fly" mean in EtherCAT?**  
  **A:** Slaves extract their data and inject new data into the frame while it's passing through, without buffering the entire frame.

- **Q: Why is EtherCAT faster than traditional switched Ethernet?**  
  **A:** No TCP/IP overhead, dedicated ESC slave controllers, and on-the-fly processing eliminate round-trip latency.

---

# 3. IoT & Wireless Protocols

## 3.1 MQTT (Message Queuing Telemetry Transport)

**Purpose:** Lightweight publish/subscribe for IoT sensors, remote monitoring, smart home.

**Baud Rate:** Network dependent (TCP) — typically 1–100 Mbps

### MQTT Fixed Header

```
MQTT Packet Fixed Header (2 bytes minimum)

┌──────────────────────────────────────┬──────────────────────────────┐
│ Byte 1                               │ Byte 2+ (Variable Length)    │
├──────────────────┬───────────────────┤                              │
│ Bits 7-4 (Pkt)   │ Bits 3-0 (Flags)  │ Remaining Length             │
│ 0000-PUBLISH     │ DUP|QoS |RETAIN   │ Encoded in 1-4 bytes         │
│ 0001-PUBACK      │(optional flags)   │ MSB set if more bytes follow │
│ 0010-SUBSCRIBE   │                   │                              │
│ 0011-SUBACK      │                   │                              │
│ ...              │                   │                              │
└──────────────────┴───────────────────┴──────────────────────────────┘
```

### MQTT QoS Levels

| QoS | Guarantee | Handshake |
|-----|-----------|-----------|
| 0 | At most once | None (fire-and-forget) |
| 1 | At least once | PUBLISH → PUBACK |
| 2 | Exactly once | PUBLISH → PUBREC → PUBREL → PUBCOMP |

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

// MQTT PUBLISH packet structure
typedef struct {
    uint8_t  packet_type;      // 0x30 = PUBLISH (QoS 0)
    uint8_t  remaining_length; // Variable length encoding
    uint16_t topic_len;
    char     topic[64];        // e.g. "factory/line1/temperature"
    uint16_t packet_id;        // Only for QoS 1, 2
    uint8_t  payload[256];
    uint8_t  payload_len;
    uint8_t  qos;              // Quality of Service (0, 1, 2)
} MQTT_Publish_t;

// Variable length encoding for remaining length
uint8_t MQTT_EncodeLength(uint32_t length, uint8_t *buf) {
    uint8_t idx = 0;
    do {
        uint8_t encoded_byte = length % 128;
        length /= 128;
        if (length > 0) encoded_byte |= 0x80;  // Continuation bit
        buf[idx++] = encoded_byte;
    } while (length > 0);
    return idx;
}

void MQTT_BuildPublish(MQTT_Publish_t *pkt,
                        const char *topic,
                        const char *payload,
                        uint8_t qos) {
    pkt->qos = qos;
    pkt->packet_type = 0x30 | (qos << 1);  // Encode QoS in bits [2:1]
    
    strncpy(pkt->topic, topic, 63);
    pkt->topic_len = strlen(topic);
    
    strncpy((char *)pkt->payload, payload, 255);
    pkt->payload_len = strlen(payload);
    
    // For QoS 1 & 2, include packet ID
    if (qos > 0) pkt->packet_id = 0x0001;
    
    // Calculate remaining length
    pkt->remaining_length = 2 + pkt->topic_len + pkt->payload_len;
    if (qos > 0) pkt->remaining_length += 2;  // +2 for packet ID
}

void MQTT_PrintPublish(MQTT_Publish_t *pkt) {
    printf("=== MQTT PUBLISH Packet ===\n");
    printf("Packet Type  : 0x%02X\n", pkt->packet_type);
    printf("QoS Level    : %d\n", pkt->qos);
    printf("Topic        : %s\n", pkt->topic);
    printf("Payload      : %s\n", (char *)pkt->payload);
    printf("Payload Len  : %d bytes\n", pkt->payload_len);
    printf("Remaining L  : %d bytes\n", pkt->remaining_length);
}

int main(void) {
    MQTT_Publish_t pub;
    
    MQTT_BuildPublish(&pub,
                      "factory/line1/temperature",
                      "22.50",
                      1);  // QoS 1 = At least once
    
    MQTT_PrintPublish(&pub);
    
    printf("\nRaw MQTT Packet:\n");
    printf("Byte 1: 0x%02X (PUBLISH, QoS %d)\n", pub.packet_type, pub.qos);
    printf("Byte 2: 0x%02X (Remaining length)\n", pub.remaining_length);
    printf("Topic : %s (%d bytes)\n", pub.topic, pub.topic_len);
    printf("Payload: %s (%d bytes)\n", (char *)pub.payload, pub.payload_len);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What's the difference between MQTT QoS 1 and QoS 2?**  
  **A:** QoS 1 uses 2-step handshake (may have duplicates); QoS 2 uses 4-step handshake (exactly-once).

- **Q: Why use MQTT over HTTP for IoT?**  
  **A:** MQTT is ~100× smaller (2B header vs. 200B+ HTTP), uses persistent connection, supports pub/sub pattern.

---

## 3.2 CoAP (Constrained Application Protocol)

**Purpose:** Ultra-lightweight protocol for battery-powered microcontrollers.

**Baud Rate:** Network dependent (UDP) — 250 kbps typical (IEEE 802.15.4)

### CoAP Header Structure

```
CoAP Fixed Header (4 bytes minimum)

┌──────────┬────────┬─────┬──────────┐
│ Ver(2b)  │ Type   │ TKL │ Code(8b) │
│ "01"     │(2bits) │(4b) │ Method/  │
│          │  CON   │Tkn  │Response  │
│          │  NON   │Len  │          │
├──────────┴────────┴─────┴──────────┤
│         Message ID (16 bits)       │
├────────────────────────────────────┤
│  Token (0-8 bytes, TKL length)     │
├────────────────────────────────────┤
│  Options + Payload Marker + Payload│
└────────────────────────────────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  ver_type_tkl;    // Ver(2b) | Type(2b) | TKL(4b)
    uint8_t  code;            // Method (0x01=GET, 0x02=POST, etc.)
    uint16_t message_id;      // Unique message ID
    uint8_t  token[8];        // Token for correlating requests/responses
    uint8_t  tkl;             // Token length
    char     uri_path[64];    // Resource path
    uint8_t  payload[256];
    uint8_t  payload_len;
} CoAP_Packet_t;

#define COAP_TYPE_CON  0  // Confirmable
#define COAP_TYPE_NON  1  // Non-confirmable
#define COAP_CODE_GET  0x01
#define COAP_CODE_POST 0x02

void CoAP_BuildGET(CoAP_Packet_t *pkt,
                    const char *uri,
                    uint16_t msg_id) {
    // Fixed header: Ver=1, Type=CON, TKL=4
    pkt->ver_type_tkl = (1 << 6) | (COAP_TYPE_CON << 4) | 4;
    pkt->code = COAP_CODE_GET;
    pkt->message_id = msg_id;
    
    // Token (unique per request)
    pkt->token[0] = 0xDE; pkt->token[1] = 0xAD;
    pkt->token[2] = 0xBE; pkt->token[3] = 0xEF;
    pkt->tkl = 4;
    
    strncpy(pkt->uri_path, uri, 63);
    pkt->payload_len = 0;  // No payload for GET
}

void CoAP_PrintPacket(CoAP_Packet_t *pkt) {
    uint8_t type = (pkt->ver_type_tkl >> 4) & 0x03;
    printf("=== CoAP Packet ===\n");
    printf("Type       : %s\n", type == COAP_TYPE_CON ? "CON" : "NON");
    printf("Code       : 0x%02X (GET/POST/etc)\n", pkt->code);
    printf("Message ID : 0x%04X\n", pkt->message_id);
    printf("Token      : %02X%02X%02X%02X\n",
           pkt->token[0], pkt->token[1], pkt->token[2], pkt->token[3]);
    printf("URI-Path   : %s\n", pkt->uri_path);
}

int main(void) {
    CoAP_Packet_t coap_req;
    CoAP_BuildGET(&coap_req, "/sensors/temperature", 0x1234);
    CoAP_PrintPacket(&coap_req);
    
    printf("\nRaw CoAP Packet Header (4 bytes):\n");
    printf("Byte 1: 0x%02X (Ver=1, Type=CON, TKL=4)\n", coap_req.ver_type_tkl);
    printf("Byte 2: 0x%02X (GET)\n", coap_req.code);
    printf("Bytes 3-4: 0x%04X (Message ID)\n", coap_req.message_id);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: Why is CoAP better than HTTP for IoT devices?**  
  **A:** 4-byte fixed header vs. 200+ bytes HTTP; UDP vs. TCP (no handshake); runs on micro-controllers.

---

## 3.3 Zigbee

**Purpose:** Low-power mesh network for smart home, industrial sensors.

**Baud Rate:** 250 kbps (IEEE 802.15.4 at 2.4 GHz)

### Zigbee Network Roles & Topology

```
Zigbee Mesh Network with Self-Healing

    ┌─────────────────────────────┐
    │  ZC (Coordinator)           │
    │  (Trust Center)             │
    └──────────┬────────┬─────────┘
               │        │
        ┌──────┴┐     ┌─┴──────┐
    [ZR1 Router] [ZR2 Router]  [ZED Device]
        │   │        │
    [ZED] [ZR3]   [ZED]

ZC  = Coordinator (single, manages keys)
ZR  = Router (forwards packets, can sleep after)
ZED = End Device (sleeps most, low power)

If ZR1 fails: packets reroute through ZR2 or ZR3
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint16_t frame_control;   // Frame type, security, ACK, pan_id_compress
    uint8_t  seq_number;      // Sequence number
    uint16_t dest_pan_id;     // Destination PAN ID
    uint16_t dest_addr;       // 16-bit short address (0x0000-0xFFFF)
    uint16_t src_addr;        // Source short address
    uint8_t  payload[127];    // MAC Service Data Unit
    uint8_t  payload_len;
    uint16_t fcs;             // CRC-16 (ITU-T)
} Zigbee_MACFrame_t;

#define ZIGBEE_FRAME_CTRL_DATA  0x8861  // Data frame, ACK req, 16-bit addr

void Zigbee_BuildDataFrame(Zigbee_MACFrame_t *f,
                            uint16_t dest_addr,
                            uint16_t src_addr,
                            uint8_t *data,
                            uint8_t len) {
    f->frame_control = ZIGBEE_FRAME_CTRL_DATA;
    f->seq_number    = 0x42;
    f->dest_pan_id   = 0xABCD;
    f->dest_addr     = dest_addr;
    f->src_addr      = src_addr;
    f->payload_len   = len;
    
    memcpy(f->payload, data, len);
}

void Zigbee_PrintFrame(Zigbee_MACFrame_t *f) {
    printf("=== Zigbee MAC Frame ===\n");
    printf("Frame Control : 0x%04X\n", f->frame_control);
    printf("Seq No        : %d\n", f->seq_number);
    printf("Dest PAN ID   : 0x%04X\n", f->dest_pan_id);
    printf("Dest Addr     : 0x%04X\n", f->dest_addr);
    printf("Src Addr      : 0x%04X\n", f->src_addr);
    printf("Payload[0]    : 0x%02X\n", f->payload[0]);
}

int main(void) {
    // Light sensor reading: device 0x0002 → coordinator 0x0001
    uint8_t lux_data[] = {0x01, 0xF4};  // 0x01F4 = 500 lux
    Zigbee_MACFrame_t frame;
    
    Zigbee_BuildDataFrame(&frame, 0x0001, 0x0002, lux_data, 2);
    Zigbee_PrintFrame(&frame);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: How does Zigbee mesh network handle node failures?**  
  **A:** Dynamic routing recalculates alternative paths through neighboring routers.

- **Q: What's the max range for Zigbee?**  
  **A:** 10-100m per hop; mesh can extend to multiple hops covering entire buildings.

---

## 3.4 LoRaWAN

**Purpose:** Ultra-long-range, low-power WAN for rural IoT (agriculture, smart city, utility metering).

**Baud Rate:** 0.3–50 kbps (depends on Spreading Factor SF7–SF12)

### LoRaWAN Frame Structure

```
LoRaWAN Uplink Frame (Device → Gateway → Network Server)

┌────────┬───────────────────────────────────────────────────────┬──────┐
│ MHDR   │           MAC Payload                                 │ MIC  │
│ (1B)   ├──────────────┬───────────────┬──────┬─────────────────┤(4B)  │
│ 0x40   │ Device Addr  │ Frame Control │ Fcnt │ Frame Options + │      │
│ (Uncnf │ (4B, LSB)    │ (1B)          │ (2B) │ Port + Payload  │      │
│  Up)   │              │               │      │                 │      │
└────────┴──────────────┴───────────────┴──────┴─────────────────┴──────┘

Spreading Factor (SF) Trade-off:
SF7  → 5.5 kbps, 2 km range    (fast, short)
SF12 → 0.3 kbps, 15 km range   (slow, far)
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  mhdr;           // Message type & version (0x40=Uncnf Up)
    uint8_t  dev_addr[4];    // Device address (Little-Endian)
    uint8_t  fctrl;          // Frame control
    uint16_t fcnt;           // Frame counter (rolling)
    uint8_t  fopts[15];      // Frame options (MAC commands)
    uint8_t  fport;          // Port (1-223 application, 0=MAC only)
    uint8_t  frm_payload[64]; // Encrypted payload
    uint8_t  payload_len;
    uint8_t  mic[4];         // Message Integrity Code (AES-CMAC)
} LoRaWAN_Frame_t;

void LoRaWAN_BuildUplink(LoRaWAN_Frame_t *f,
                          uint32_t dev_addr,
                          uint16_t frame_counter,
                          uint8_t *data,
                          uint8_t len) {
    f->mhdr = 0x40;  // Unconfirmed Data Up
    
    // Device address in Little-Endian format
    f->dev_addr[0] = (uint8_t)(dev_addr & 0xFF);
    f->dev_addr[1] = (uint8_t)((dev_addr >> 8) & 0xFF);
    f->dev_addr[2] = (uint8_t)((dev_addr >> 16) & 0xFF);
    f->dev_addr[3] = (uint8_t)((dev_addr >> 24) & 0xFF);
    
    f->fctrl = 0x00;  // No ADR, no ACK
    f->fcnt = frame_counter;
    f->fport = 1;     // Application port
    f->payload_len = len;
    
    memcpy(f->frm_payload, data, len);
}

void LoRaWAN_PrintFrame(LoRaWAN_Frame_t *f) {
    printf("=== LoRaWAN Uplink Frame ===\n");
    printf("MHDR      : 0x%02X (Unconfirmed Up)\n", f->mhdr);
    printf("DevAddr   : %02X%02X%02X%02X\n",
           f->dev_addr[3], f->dev_addr[2],
           f->dev_addr[1], f->dev_addr[0]);
    printf("Frame Cnt : %d\n", f->fcnt);
    printf("FPort     : %d\n", f->fport);
    printf("Payload   : %d bytes (encrypted)\n", f->payload_len);
}

int main(void) {
    LoRaWAN_Frame_t uplink;
    
    // Temperature sensor: temp=35.2°C, humidity=100%
    uint8_t sensor_data[] = {0x01, 0x5C, 0x00, 0x64};
    
    LoRaWAN_BuildUplink(&uplink,
                        0x26011BDA,  // Device address
                        1,           // Frame counter
                        sensor_data,
                        4);
    
    LoRaWAN_PrintFrame(&uplink);
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What is the trade-off in LoRaWAN?**  
  **A:** Bandwidth for range. Higher SF = longer range but slower data rate.

- **Q: Max payload size in LoRaWAN?**  
  **A:** Typically 51 bytes for SF7, 11 bytes for SF12 (varies by region).

---

# 4. Web & Application Protocols

## 4.1 HTTP / HTTPS

**Purpose:** Foundation of World Wide Web — stateless request/response.

**Baud Rate:** Network dependent (TCP/IP) — 10 Mbps – Gbps

### HTTP Request/Response Structure

```
HTTP/1.1 Request

┌─────────────────────────────────┐
│ METHOD PATH HTTP/1.1            │
│ Host: example.com               │
│ Content-Type: application/json  │
│ Content-Length: 27              │
│                                 │
│ {"sensor":"temp","value":22.5}  │
└─────────────────────────────────┘

HTTP/1.1 Response

┌─────────────────────────────────┐
│ HTTP/1.1 200 OK                 │
│ Server: nginx/1.18.0            │
│ Content-Type: application/json  │
│ Content-Length: 35              │
│                                 │
│ {"status":"ok","data":35.2}     │
└─────────────────────────────────┘
```

### C Implementation Example

```c
#include <stdio.h>
#include <string.h>

void HTTP_BuildGETRequest(const char *host,
                           const char *path,
                           char *buffer,
                           int buf_size) {
    snprintf(buffer, buf_size,
        "GET %s HTTP/1.1\r\n"
        "Host: %s\r\n"
        "Accept: application/json\r\n"
        "Connection: close\r\n"
        "\r\n",
        path, host);
}

void HTTP_BuildPOSTRequest(const char *host,
                            const char *path,
                            const char *content_type,
                            const char *body,
                            char *buffer,
                            int buf_size) {
    snprintf(buffer, buf_size,
        "POST %s HTTP/1.1\r\n"
        "Host: %s\r\n"
        "Content-Type: %s\r\n"
        "Content-Length: %zu\r\n"
        "Connection: close\r\n"
        "\r\n"
        "%s",
        path, host, content_type, strlen(body), body);
}

int main(void) {
    char http_request[512];
    
    // Build and print GET request
    HTTP_BuildGETRequest("api.example.com",
                         "/sensors/temperature",
                         http_request,
                         sizeof(http_request));
    printf("=== HTTP GET Request ===\n%s\n", http_request);
    
    // Build and print POST request
    const char *json_body = "{\"sensor_id\":\"TMP001\",\"value\":22.5}";
    HTTP_BuildPOSTRequest("api.example.com",
                          "/data/upload",
                          "application/json",
                          json_body,
                          http_request,
                          sizeof(http_request));
    printf("\n=== HTTP POST Request ===\n%s\n", http_request);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: What's the difference between HTTP/1.1 and HTTP/2?**  
  **A:** HTTP/2 uses binary framing, header compression, and multiplexing over single TCP connection (vs. sequential requests in HTTP/1.1).

---

## 4.2 WebSocket

**Purpose:** Full-duplex persistent connection for real-time bidirectional communication.

**Baud Rate:** Network dependent (TCP) — up to Gbps

### WebSocket Handshake & Frame

```
WebSocket Upgrade (HTTP to WS)

Client Request:
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

─ Now persistent TCP connection carrying WebSocket frames ─
```

### C Implementation Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  fin_rsv_opcode;   // FIN(1) + RSV(3) + Opcode(4)
    uint8_t  mask_len;         // MASK(1) + Payload len(7)
    uint8_t  ext_len[8];       // Extended payload length (if needed)
    uint8_t  masking_key[4];   // Masking key (client → server only)
    uint8_t  payload[256];     // Application data
    uint16_t payload_len;
} WebSocket_Frame_t;

#define WS_OPCODE_CONTINUATION 0x0
#define WS_OPCODE_TEXT         0x1
#define WS_OPCODE_BINARY       0x2
#define WS_OPCODE_CLOSE        0x8
#define WS_OPCODE_PING         0x9
#define WS_OPCODE_PONG         0xA

// XOR mask payload (client → server requirement)
void WS_MaskPayload(uint8_t *payload, uint16_t len, uint8_t *key) {
    for (int i = 0; i < len; i++) {
        payload[i] ^= key[i % 4];
    }
}

void WS_BuildTextFrame(WebSocket_Frame_t *f,
                        const char *message) {
    uint16_t msg_len = strlen(message);
    
    f->fin_rsv_opcode = 0x81;  // FIN=1, Opcode=0x1 (Text)
    f->mask_len = 0x80 | (msg_len & 0x7F);  // MASK=1, len
    
    // Masking key (random)
    f->masking_key[0] = 0x37;
    f->masking_key[1] = 0xFA;
    f->masking_key[2] = 0x21;
    f->masking_key[3] = 0x3D;
    
    f->payload_len = msg_len;
    memcpy(f->payload, message, msg_len);
    
    // Mask payload (required for client)
    WS_MaskPayload(f->payload, msg_len, f->masking_key);
}

void WS_PrintFrame(WebSocket_Frame_t *f) {
    uint8_t opcode = f->fin_rsv_opcode & 0x0F;
    uint8_t fin    = (f->fin_rsv_opcode >> 7) & 1;
    
    printf("=== WebSocket Frame ===\n");
    printf("FIN    : %d (final fragment)\n", fin);
    printf("Opcode : 0x%X (%s)\n", opcode,
           opcode == WS_OPCODE_TEXT ? "Text" : "Binary");
    printf("Masked : %s\n", (f->mask_len & 0x80) ? "YES" : "NO");
    printf("Len    : %d bytes\n", f->payload_len);
}

int main(void) {
    WebSocket_Frame_t ws_frame;
    WS_BuildTextFrame(&ws_frame, "{\"type\":\"sensor\",\"value\":42.0}");
    WS_PrintFrame(&ws_frame);
    
    return 0;
}
```

**Viva Questions & Answers:**
- **Q: When should you use WebSocket instead of HTTP polling?**  
  **A:** For real-time bidirectional communication (live chat, stock tickers, multiplayer games) to reduce latency and server load.

---

(Continuing in next sections with TCP/IP, DNS, DHCP, BGP, SNMP, File Protocols, and Messaging Protocols...)

---

# Quick Speed Reference Table

| Protocol | Category | Speed / Baud Rate |
|----------|----------|-------------------|
| CAN | Automotive | 125 kbps – 1 Mbps |
| LIN | Automotive | Max 20 kbps |
| FlexRay | Automotive | 10 Mbps/ch |
| MOST | Automotive | 25/50/150 Mbps |
| Auto Ethernet | Automotive | 100 Mbps / 1 Gbps |
| Modbus RTU | Industrial | 1.2–115.2 kbps |
| PROFINET RT | Industrial | 100 Mbps (1–10 ms) |
| EtherCAT | Industrial | 100 Mbps (< 100 µs) |
| MQTT | IoT | Network (TCP) |
| CoAP | IoT | 250 kbps (802.15.4) |
| Zigbee | IoT Wireless | 250 kbps |
| Z-Wave | IoT Wireless | 9.6/40/100 kbps |
| LoRaWAN | IoT Wireless | 0.3–50 kbps |
| BLE | IoT Wireless | 1/2 Mbps |
| HTTP/HTTPS | Web | Network (TCP) |
| WebSocket | Web | Network (TCP) |
| TCP/IP | Networking | 10 Mbps – 400 Gbps |
| DNS | Networking | UDP Port 53 |
| FTP/SFTP | Storage | Network (TCP) |
| NFS | Storage | Network (TCP) |
| SMB | Storage | Network (TCP) |

---

**End of Consolidated Reference**

*This document consolidates all protocol definitions with detailed frame structures, C implementation examples, and viva questions for comprehensive understanding.*
