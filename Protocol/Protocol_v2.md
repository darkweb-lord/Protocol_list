# Communication Protocols — Complete Reference Manual

> **Comprehensive technical documentation covering architecture, frame structures (bit-level), block diagrams, and embedded C code examples for 30+ protocols across 7 domains.**

---

## Table of Contents

| # | Module | Protocols |
|---|--------|-----------|
| 1 | Automotive Protocols | CAN, LIN, FlexRay, MOST, Automotive Ethernet |
| 2 | Industrial Automation Protocols | Modbus, PROFINET, EtherCAT, EtherNet/IP, BACnet |
| 3 | IoT & Wireless Protocols | MQTT, CoAP, Zigbee, Z-Wave, LoRaWAN, BLE |
| 4 | Web & Application Protocols | HTTP/HTTPS, WebSocket, gRPC, GraphQL |
| 5 | IT, Networking & Infrastructure | TCP/IP, DNS, DHCP, BGP, SNMP |
| 6 | File Transfer & Storage Protocols | FTP/SFTP, NFS, SMB, iSCSI |
| 7 | Communication & Messaging Protocols | SMTP/IMAP/POP3, SIP, XMPP |

---

# Module 1: Automotive Protocols

---

## 1.1 CAN (Controller Area Network)

### Overview

- **OSI Layer:** Layer 1 (Physical) & Layer 2 (Data Link)
- **Topology:** Multi-master linear bus
- **Medium:** Twisted pair (CAN_H, CAN_L) with 120 Ω termination
- **Speed:** Up to 1 Mbps (Classic CAN), 5 Mbps data phase (CAN FD)
- **Arbitration:** CSMA/CR (Carrier-Sense Multiple Access with Collision Resolution)

### Block Diagram

```
 ┌──────────┐    ┌──────────┐    ┌──────────┐
 │  ECU #1  │    │  ECU #2  │    │  ECU #3  │
 │(Engine)  │    │ (Brakes) │    │  (ABS)   │
 └────┬─────┘    └────┬─────┘    └────┬─────┘
      │               │               │
      │  CAN_H ───────┼───────────────┼──── 120Ω
      │  CAN_L ───────┼───────────────┼──── 120Ω
      │               │               │
  ┌───┴───┐       ┌───┴───┐       ┌───┴───┐
  │CAN    │       │CAN    │       │CAN    │
  │Trans- │       │Trans- │       │Trans- │
  │ceiver │       │ceiver │       │ceiver │
  └───────┘       └───────┘       └───────┘
```

### Frame Structure (Standard CAN 2.0A)

| Field | Bits | Description |
|-------|------|-------------|
| SOF (Start of Frame) | 1 | Single dominant bit (0) |
| Identifier | 11 | Message priority / ID (lower = higher priority) |
| RTR (Remote Transmission Request) | 1 | 0 = Data frame, 1 = Remote frame |
| IDE (Identifier Extension) | 1 | 0 = Standard (11-bit), 1 = Extended (29-bit) |
| r0 (Reserved) | 1 | Reserved bit (dominant) |
| DLC (Data Length Code) | 4 | Number of data bytes (0–8) |
| Data Field | 0–64 | Payload: 0 to 8 bytes |
| CRC Sequence | 15 | Cyclic Redundancy Check |
| CRC Delimiter | 1 | Recessive bit |
| ACK Slot | 1 | Receivers drive dominant to acknowledge |
| ACK Delimiter | 1 | Recessive bit |
| EOF (End of Frame) | 7 | Seven recessive bits |
| **Total (max)** | **108** | **With 8-byte payload** |

```
| SOF | Identifier (11) | RTR | IDE | r0 | DLC (4) | Data (0-64) | CRC (15) | CRC Del | ACK | ACK Del | EOF (7) |
|  1  |       11        |  1  |  1  |  1 |    4    |   0 - 64    |    15    |    1    |  1  |    1    |    7    |
```

### C Code Example

```c
/* CAN Transmission Example using SocketCAN (Linux) */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <linux/can.h>
#include <linux/can/raw.h>

int can_send(const char *iface, uint32_t id, uint8_t *data, uint8_t len) {
    int sock;
    struct sockaddr_can addr;
    struct ifreq ifr;
    struct can_frame frame;

    /* Create CAN socket */
    sock = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    if (sock < 0) { perror("socket"); return -1; }

    /* Bind to interface */
    strncpy(ifr.ifr_name, iface, IFNAMSIZ - 1);
    ioctl(sock, SIOCGIFINDEX, &ifr);
    addr.can_family  = AF_CAN;
    addr.can_ifindex = ifr.ifr_ifindex;
    bind(sock, (struct sockaddr *)&addr, sizeof(addr));

    /* Build CAN frame */
    frame.can_id  = id;        /* 11-bit standard ID */
    frame.can_dlc = len;       /* Data Length Code    */
    memcpy(frame.data, data, len);

    /* Transmit */
    if (write(sock, &frame, sizeof(frame)) != sizeof(frame)) {
        perror("write"); close(sock); return -1;
    }
    printf("CAN TX: ID=0x%03X DLC=%d\n", id, len);
    close(sock);
    return 0;
}

/* Usage example */
int main(void) {
    uint8_t payload[] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08};
    can_send("can0", 0x123, payload, 8);
    return 0;
}
```

---

## 1.2 LIN (Local Interconnect Network)

### Overview

- **OSI Layer:** Layer 1 (Physical) & Layer 2 (Data Link)
- **Topology:** Single-master, multi-slave linear bus
- **Medium:** Single wire + ground
- **Speed:** Max 20 kbps
- **Signaling:** NRZ, dominant = 0 V, recessive = V_BAT (~12 V)

### Block Diagram

```
  ┌────────────────┐
  │   LIN Master   │
  │ (Body Control) │
  └──────┬─────────┘
         │  LIN Bus (Single Wire)
    ┌────┴─────┬───────────┬─────────┐
    │          │           │         │
 ┌──┴───┐   ┌──┴───┐   ┌───┴──┐  ┌───┴──┐
 │Slave │   │Slave │   │Slave │  │Slave │
 │ #1   │   │ #2   │   │  #3  │  │  #4  │
 │Mirror│   │Window│   │Wiper │  │ Seat │
 └──────┘   └──────┘   └──────┘  └──────┘
```

### Frame Structure

**Master Header:**

| Field | Bits | Description |
|-------|------|-------------|
| Break Field | ≥13 | Dominant bits to signal frame start |
| Break Delimiter | 1 | Recessive bit |
| Sync Field | 8 | Fixed byte 0x55 for baud rate detection |
| PID (Protected ID) | 8 | 6-bit ID + 2 parity bits |

**Slave Response:**

| Field | Bits | Description |
|-------|------|-------------|
| Data | 8–64 | 1 to 8 bytes of payload |
| Checksum | 8 | Classic or Enhanced checksum |

```
Master Header:                    Slave Response:
| Break (≥13) | Sync (8) | PID (8) |  | Data (8-64) | Checksum (8) |
```

### C Code Example

```c
/* LIN Master Frame Transmission (Bare-metal UART) */
#include <stdint.h>

#define LIN_SYNC_BYTE   0x55
#define LIN_BREAK_BITS  13

/* Calculate LIN Protected ID (PID) from 6-bit frame ID */
uint8_t lin_calc_pid(uint8_t id) {
    uint8_t p0, p1;
    id &= 0x3F;  /* Mask to 6 bits */
    p0 = ((id >> 0) ^ (id >> 1) ^ (id >> 2) ^ (id >> 4)) & 0x01;
    p1 = ~((id >> 1) ^ (id >> 3) ^ (id >> 4) ^ (id >> 5)) & 0x01;
    return id | (p0 << 6) | (p1 << 7);
}

/* Calculate enhanced checksum over PID + data */
uint8_t lin_checksum(uint8_t pid, const uint8_t *data, uint8_t len) {
    uint16_t sum = pid;
    for (uint8_t i = 0; i < len; i++) {
        sum += data[i];
        if (sum > 0xFF) sum -= 0xFF;  /* Carry addition */
    }
    return (uint8_t)(~sum);
}

/* Send LIN master header via UART */
void lin_send_header(uint8_t frame_id) {
    /* Step 1: Send Break (13+ dominant bits) */
    uart_send_break(LIN_BREAK_BITS);

    /* Step 2: Send Sync Byte (0x55) */
    uart_send_byte(LIN_SYNC_BYTE);

    /* Step 3: Send Protected ID */
    uint8_t pid = lin_calc_pid(frame_id);
    uart_send_byte(pid);
}

/* Send LIN slave response */
void lin_send_response(uint8_t pid, const uint8_t *data, uint8_t len) {
    for (uint8_t i = 0; i < len; i++) {
        uart_send_byte(data[i]);
    }
    uart_send_byte(lin_checksum(pid, data, len));
}
```

---

## 1.3 FlexRay

### Overview

- **OSI Layer:** Layer 1 (Physical) & Layer 2 (Data Link)
- **Topology:** Dual-channel bus/star/hybrid
- **Medium:** 100 Ω differential twisted pair per channel
- **Speed:** Up to 10 Mbps per channel (20 Mbps aggregate)
- **Access:** TDMA (static) + FTDMA (dynamic)

### Block Diagram

```
  Channel A ════════════════════════════════════════
      │           │           │           │
  ┌───┴───┐   ┌───┴───┐   ┌───┴───┐   ┌───┴───┐
  │FlexRay│   │FlexRay│   │FlexRay│   │FlexRay│
  │Node 1 │   │Node 2 │   │Node 3 │   │Node 4 │
  │Steer  │   │Brake  │   │Engine │   │Suspen.│
  └───┬───┘   └───┬───┘   └───┬───┘   └───┬───┘
      │           │           │           │
  Channel B ════════════════════════════════════════
                  (Redundant Channel)
```

### Communication Cycle

```
┌──────────────────────────────────────────────────────────────┐
│                    FlexRay Communication Cycle               │
├───────────────────┬────────────────┬────────┬────────────────┤
│  Static Segment   │Dynamic Segment │ Symbol │    NIT         │
│  (TDMA Slots)     │ (Minislots)    │ Window │ (Network Idle) │
│                   │                │        │                │
│ Slot1|Slot2|Slot3 │ Mini|Mini|Mini │  MTS   │  Sync/Correct  │
│ (Deterministic)   │ (Event-driven) │        │                │
└───────────────────┴────────────────┴────────┴────────────────┘
```

### Frame Structure

| Field | Bits | Description |
|-------|------|-------------|
| Reserved | 1 | Reserved bit |
| Payload Preamble Indicator | 1 | 1 = network management vector present |
| Null Frame Indicator | 1 | 0 = null frame |
| Sync Frame Indicator | 1 | 1 = sync frame for clock sync |
| Startup Frame Indicator | 1 | 1 = startup frame |
| Frame ID | 11 | Slot number (1–2047) |
| Payload Length | 7 | Data words (0–127, in 16-bit words = 0–254 bytes) |
| Header CRC | 11 | CRC over Sync, Startup, Frame ID, Payload Length |
| Cycle Count | 6 | Current communication cycle (0–63) |
| Data | 0–2032 | Payload: up to 254 bytes (127 × 16 bits) |
| Trailer CRC | 24 | CRC over header + data |
| **Total (max)** | **~2096** | **With 254-byte payload** |

### C Code Example

```c
/* FlexRay Frame Builder (Conceptual) */
#include <stdint.h>
#include <string.h>

typedef struct {
    uint8_t  reserved       : 1;
    uint8_t  ppi            : 1;  /* Payload Preamble Indicator */
    uint8_t  null_frame     : 1;
    uint8_t  sync_frame     : 1;
    uint8_t  startup_frame  : 1;
    uint16_t frame_id       : 11; /* Slot number 1-2047 */
    uint8_t  payload_length : 7;  /* In 16-bit words */
    uint16_t header_crc     : 11;
    uint8_t  cycle_count    : 6;
    uint8_t  data[254];           /* Max payload */
    uint32_t trailer_crc    : 24;
} flexray_frame_t;

/* CRC-11 for FlexRay header (polynomial: 0x385) */
uint16_t flexray_header_crc(uint16_t frame_id, uint8_t payload_len,
                            uint8_t sync, uint8_t startup) {
    uint32_t header_data = ((uint32_t)sync << 19) |
                           ((uint32_t)startup << 18) |
                           ((uint32_t)frame_id << 7) |
                           payload_len;
    uint16_t crc = 0x01A;  /* Initial value */
    for (int i = 19; i >= 0; i--) {
        uint16_t bit = (header_data >> i) & 0x01;
        uint16_t msb = (crc >> 10) & 0x01;
        crc = (crc << 1) | bit;
        if (msb) crc ^= 0x385;
        crc &= 0x7FF;
    }
    return crc;
}

/* Build a FlexRay frame */
int flexray_build_frame(flexray_frame_t *frame, uint16_t slot_id,
                        const uint8_t *data, uint8_t len_bytes,
                        uint8_t cycle) {
    memset(frame, 0, sizeof(*frame));
    frame->frame_id       = slot_id;
    frame->payload_length = (len_bytes + 1) / 2; /* Convert to 16-bit words */
    frame->null_frame     = 1; /* Not a null frame */
    frame->cycle_count    = cycle & 0x3F;
    frame->header_crc     = flexray_header_crc(slot_id, frame->payload_length,
                                                frame->sync_frame,
                                                frame->startup_frame);
    memcpy(frame->data, data, len_bytes);
    return 0;
}
```

---

## 1.4 MOST (Media Oriented Systems Transport)

### Overview

- **OSI Layer:** Layers 1–7 (full stack)
- **Topology:** Physical ring
- **Medium:** Polymer Optical Fiber (POF) at 650 nm
- **Speed:** MOST25 (25 Mbps), MOST50 (50 Mbps), MOST150 (150 Mbps)

### Block Diagram

```
       ┌────────┐
       │ Timing │
       │ Master │
       └───┬────┘
           │ POF
    ┌──────┴──────┐
    │             │
┌───┴───┐     ┌───┴───┐
│ Head  │     │ Audio │
│ Unit  │◄───►│Amplif.│
└───┬───┘     └───┬───┘
    │             │
┌───┴───┐     ┌───┴───┐
│  CD   │     │  Nav  │
│Changer│◄───►│System │
└───────┘     └───────┘
   (Ring Topology — unidirectional optical)
```

### Frame Structure (MOST25)

| Field | Bits | Description |
|-------|------|-------------|
| Preamble | 8+ | Synchronization pattern |
| Boundary Descriptor | 4 | Defines sync/async boundary |
| Synchronous Data | 0–384 | Up to 24 stereo audio channels |
| Asynchronous Data | 0–384 | Packet data (TCP/IP, etc.) |
| Control Data | 32 | 2 bytes address + 2 bytes data |
| Frame Control | 2 | Status bits |
| **Total per block** | **512** | **At 44.1/48 kHz sample rate** |

### C Code Example

```c
/* MOST Network Interface Controller (Conceptual) */
#include <stdint.h>

typedef struct {
    uint8_t  boundary_desc;     /* Sync/Async boundary */
    uint8_t  sync_data[48];     /* Synchronous channels */
    uint8_t  async_data[48];    /* Asynchronous packet data */
    uint16_t ctrl_addr;         /* Control message address */
    uint16_t ctrl_data;         /* Control message data */
    uint8_t  frame_ctrl;        /* Frame control bits */
} most25_frame_t;

/* Allocate a synchronous streaming channel */
typedef struct {
    uint8_t  channel_id;
    uint8_t  bandwidth;     /* Number of bytes per frame */
    uint16_t source_addr;
    uint16_t sink_addr;
} most_channel_t;

int most_allocate_channel(most_channel_t *ch, uint8_t id,
                          uint8_t bw, uint16_t src, uint16_t dst) {
    ch->channel_id = id;
    ch->bandwidth  = bw;
    ch->source_addr = src;
    ch->sink_addr   = dst;

    /* Send allocation request via control channel */
    most25_frame_t frame = {0};
    frame.ctrl_addr = src;
    frame.ctrl_data = (id << 8) | bw;
    /* Transmit frame to MOST NIC hardware register */
    most_nic_write(&frame);
    return 0;
}

/* Read synchronous audio stream data */
int most_read_sync(uint8_t *buffer, uint8_t channel, uint8_t bytes) {
    most25_frame_t frame;
    most_nic_read(&frame);
    /* Extract channel data from synchronous area */
    uint8_t offset = channel * bytes;
    for (uint8_t i = 0; i < bytes; i++) {
        buffer[i] = frame.sync_data[offset + i];
    }
    return bytes;
}
```

---

## 1.5 Automotive Ethernet

### Overview

- **OSI Layer:** Layer 1 (Physical) & Layer 2 (Data Link)
- **Standard:** 100BASE-T1 (IEEE 802.3bw), 1000BASE-T1 (IEEE 802.3bp)
- **Medium:** Single unshielded twisted pair (UTP), full-duplex
- **Modulation:** PAM-3
- **Key Feature:** Time-Sensitive Networking (TSN) extensions

### Block Diagram

```
 ┌──────────┐     ┌──────────────┐     ┌──────────┐
 │  ADAS    │     │  Automotive  │     │  LiDAR   │
 │  ECU     ├─────┤   Ethernet   ├─────┤  Sensor  │
 └──────────┘     │   Switch     │     └──────────┘
                  │  (TSN-aware) │
 ┌──────────┐     │              │     ┌──────────┐
 │  Camera  ├─────┤              ├─────┤ Gateway  │
 │  Module  │     │              │     │(CAN/LIN) │
 └──────────┘     └──────────────┘     └──────────┘
                  Single Twisted Pair per link
```

### Ethernet Frame (IEEE 802.3 with VLAN/TSN)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Preamble | 7 | 56 | Alternating 10101010 pattern |
| SFD (Start Frame Delimiter) | 1 | 8 | 10101011 |
| Destination MAC | 6 | 48 | Destination address |
| Source MAC | 6 | 48 | Source address |
| 802.1Q VLAN Tag (optional) | 4 | 32 | TPID + PCP + DEI + VID |
| EtherType / Length | 2 | 16 | Protocol identifier |
| Payload | 46–1500 | 368–12000 | Data |
| FCS (Frame Check Sequence) | 4 | 32 | CRC-32 |
| **Total (max)** | **1522** | **12176** | **With VLAN tag** |

### C Code Example

```c
/* Automotive Ethernet Raw Frame Transmit (Linux) */
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <linux/if_packet.h>
#include <linux/if_ether.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <arpa/inet.h>
#include <unistd.h>

typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];
    uint8_t  src_mac[6];
    uint16_t ethertype;
    uint8_t  payload[1500];
} eth_frame_t;

int eth_send_frame(const char *iface, const uint8_t *dst_mac,
                   const uint8_t *data, uint16_t len) {
    int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock < 0) { perror("socket"); return -1; }

    /* Get interface index */
    struct ifreq ifr;
    strncpy(ifr.ifr_name, iface, IFNAMSIZ - 1);
    ioctl(sock, SIOCGIFINDEX, &ifr);

    /* Get source MAC */
    struct ifreq ifr_mac;
    strncpy(ifr_mac.ifr_name, iface, IFNAMSIZ - 1);
    ioctl(sock, SIOCGIFHWADDR, &ifr_mac);

    /* Build frame */
    eth_frame_t frame;
    memcpy(frame.dst_mac, dst_mac, 6);
    memcpy(frame.src_mac, ifr_mac.ifr_hwaddr.sa_data, 6);
    frame.ethertype = htons(0x88B5); /* IEEE 802.1 local experimental */
    memcpy(frame.payload, data, len);

    /* Destination address */
    struct sockaddr_ll addr = {0};
    addr.sll_ifindex  = ifr.ifr_ifindex;
    addr.sll_halen    = ETH_ALEN;
    memcpy(addr.sll_addr, dst_mac, 6);

    sendto(sock, &frame, 14 + len, 0,
           (struct sockaddr *)&addr, sizeof(addr));

    printf("Eth TX: %d bytes on %s\n", len, iface);
    close(sock);
    return 0;
}
```

---

# Module 2: Industrial Automation Protocols

---

## 2.1 Modbus (RTU / TCP)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Architecture:** Master/Slave (Client/Server)
- **Transport:** RS-485 (RTU), Ethernet (TCP)
- **Data Model:** Coils, Discrete Inputs, Holding Registers, Input Registers

### Block Diagram

```
 ┌──────────────┐
 │ Modbus Master│ (SCADA / HMI)
 │  (Client)    │
 └──────┬───────┘
        │  RS-485 Bus or Ethernet
  ┌─────┼──────────┬──────────┐
  │     │          │          │
┌─┴──┐ ┌┴───┐   ┌──┴───┐  ┌───┴──┐
│Slv1│ │Slv2│   │Slv3  │  │Slv4  │
│PLC │ │VFD │   │Temp  │  │Flow  │
│    │ │    │   │Sensor│  │Meter │
└────┘ └────┘   └──────┘  └──────┘
```

### Modbus RTU Frame

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Slave Address | 1 | 8 | Device ID (1–247) |
| Function Code | 1 | 8 | Operation (e.g., 0x03 = Read Holding Registers) |
| Data | N | N×8 | Variable: start addr, quantity, values |
| CRC-16 | 2 | 16 | Error check (polynomial 0xA001) |

### Modbus TCP Frame (MBAP Header + PDU)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Transaction ID | 2 | 16 | Request/response matching |
| Protocol ID | 2 | 16 | Always 0x0000 for Modbus |
| Length | 2 | 16 | Remaining bytes count |
| Unit ID | 1 | 8 | Slave device identifier |
| Function Code | 1 | 8 | Operation code |
| Data | N | N×8 | Register addresses and values |

### C Code Example

```c
/* Modbus RTU Master — Read Holding Registers */
#include <stdio.h>
#include <stdint.h>
#include <string.h>

/* CRC-16/Modbus calculation */
uint16_t modbus_crc16(const uint8_t *data, uint16_t len) {
    uint16_t crc = 0xFFFF;
    for (uint16_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++) {
            if (crc & 0x0001)
                crc = (crc >> 1) ^ 0xA001;
            else
                crc >>= 1;
        }
    }
    return crc;
}

/* Build Modbus RTU request: Read Holding Registers (FC 0x03) */
int modbus_read_holding(uint8_t *frame, uint8_t slave_id,
                        uint16_t start_addr, uint16_t quantity) {
    frame[0] = slave_id;
    frame[1] = 0x03;                        /* Function Code */
    frame[2] = (start_addr >> 8) & 0xFF;    /* Start Address High */
    frame[3] = start_addr & 0xFF;           /* Start Address Low */
    frame[4] = (quantity >> 8) & 0xFF;      /* Quantity High */
    frame[5] = quantity & 0xFF;             /* Quantity Low */

    uint16_t crc = modbus_crc16(frame, 6);
    frame[6] = crc & 0xFF;                  /* CRC Low */
    frame[7] = (crc >> 8) & 0xFF;           /* CRC High */

    return 8; /* Frame length */
}

/* Parse response */
int modbus_parse_response(const uint8_t *resp, uint16_t *values) {
    uint8_t byte_count = resp[2];
    uint8_t num_regs = byte_count / 2;
    for (uint8_t i = 0; i < num_regs; i++) {
        values[i] = (resp[3 + i*2] << 8) | resp[4 + i*2];
    }
    return num_regs;
}
```

---

## 2.2 PROFINET

### Overview

- **OSI Layer:** Layer 2 (RT/IRT) to Layer 7
- **Classes:** NRT (~100 ms), RT (1–10 ms), IRT (<1 ms, <1 μs jitter)
- **EtherType:** 0x8892
- **Transport:** Standard Ethernet (IEEE 802.3)

### Block Diagram

```
 ┌──────────────────┐
 │  PROFINET IO     │
 │  Controller (PLC)│
 └────────┬─────────┘
          │ Industrial Ethernet
   ┌──────┼──────────┬──────────┐
   │      │          │          │
┌──┴───┐ ┌┴─────┐  ┌─┴────┐  ┌──┴────┐
│IO Dev│ │IO Dev│  │IO Dev│  │IO     │
│Drive │ │Sensor│  │Valve │  │Superv.│
└──────┘ └──────┘  └──────┘  └───────┘
```

### PROFINET RT Frame

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Dest MAC | 6 | 48 | Destination |
| Src MAC | 6 | 48 | Source |
| VLAN Tag (opt) | 4 | 32 | 802.1Q priority/VLAN |
| EtherType | 2 | 16 | 0x8892 |
| Frame ID | 2 | 16 | Identifies data relationship |
| User Data | N | N×8 | I/O process data |
| IOPS/IOCS | 1 | 8 | Provider/Consumer status |
| Cycle Counter | 2 | 16 | Communication cycle count |
| Data Status | 1 | 8 | Valid/invalid indicators |
| Transfer Status | 1 | 8 | OK or fault condition |
| FCS | 4 | 32 | Ethernet CRC-32 |

### C Code Example

```c
/* PROFINET RT Frame Construction (Conceptual) */
#include <stdint.h>
#include <string.h>

#define PROFINET_ETHERTYPE  0x8892

typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];
    uint8_t  src_mac[6];
    uint16_t ethertype;      /* 0x8892 */
    uint16_t frame_id;       /* RT data identification */
    uint8_t  user_data[40];  /* Cyclic I/O data */
    uint8_t  iops;           /* IO Provider Status */
    uint16_t cycle_counter;
    uint8_t  data_status;
    uint8_t  transfer_status;
} profinet_rt_frame_t;

/* Build cyclic PROFINET RT frame */
void profinet_build_rt(profinet_rt_frame_t *frame,
                       const uint8_t *dst, const uint8_t *src,
                       uint16_t fid, const uint8_t *io_data,
                       uint8_t io_len, uint16_t cycle) {
    memcpy(frame->dst_mac, dst, 6);
    memcpy(frame->src_mac, src, 6);
    frame->ethertype       = htons(PROFINET_ETHERTYPE);
    frame->frame_id        = htons(fid);
    memset(frame->user_data, 0, sizeof(frame->user_data));
    memcpy(frame->user_data, io_data, io_len);
    frame->iops            = 0x80;  /* Good status */
    frame->cycle_counter   = htons(cycle);
    frame->data_status     = 0x35;  /* Valid, run, primary */
    frame->transfer_status = 0x00;  /* OK */
}
```

---

## 2.3 EtherCAT

### Overview

- **OSI Layer:** Layer 2 (Data Link)
- **EtherType:** 0x88A4
- **Topology:** Line/tree/star (logical ring)
- **Key Feature:** "Processing on the fly" — slaves insert/extract data without buffering

### Block Diagram

```
 ┌──────────────┐
 │  EtherCAT    │
 │  Master      │
 └──────┬───────┘
        │  Frame passes through sequentially
  ┌─────▼─────┐   ┌──────────┐   ┌──────────┐
  │  Slave #1 ├──►│ Slave #2 ├──►│ Slave #3 │
  │  (ESC)    │   │  (ESC)   │   │  (ESC)   │
  │  Servo    │   │  I/O     │   │  Sensor  │
  └───────────┘   └──────────┘   └────┬─────┘
                                      │
              ◄───────────────────────┘
              (Frame returns to master)
```

### EtherCAT Frame Structure

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Ethernet Header | 14 | 112 | Dst + Src MAC + EtherType 0x88A4 |
| EtherCAT Header | 2 | 16 | Length (11 bits) + Type (4 bits) + Reserved |
| **Datagram(s):** | | | |
| → Cmd | 1 | 8 | Command type (e.g., LRW, BRD, FPRD) |
| → Index | 1 | 8 | Frame index for matching |
| → Address | 4 | 32 | Slave address / offset |
| → Length | 2 | 16 | Data length + flags |
| → IRQ | 2 | 16 | Interrupt request register |
| → Data | N | N×8 | Process data |
| → WKC | 2 | 16 | Working Counter (incremented by each slave) |
| Ethernet FCS | 4 | 32 | CRC-32 |

### C Code Example

```c
/* EtherCAT Datagram Builder */
#include <stdint.h>
#include <string.h>

/* EtherCAT command types */
#define ECAT_CMD_NOP   0x00
#define ECAT_CMD_BRD   0x07  /* Broadcast Read */
#define ECAT_CMD_BWR   0x08  /* Broadcast Write */
#define ECAT_CMD_LRW   0x0C  /* Logical Read/Write */

typedef struct __attribute__((packed)) {
    uint8_t  cmd;
    uint8_t  index;
    uint32_t address;
    uint16_t len_flags;   /* [10:0]=length, [15:11]=flags */
    uint16_t irq;
    uint8_t  data[256];
    uint16_t wkc;         /* Working Counter */
} ecat_datagram_t;

typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];
    uint8_t  src_mac[6];
    uint16_t ethertype;   /* 0x88A4 */
    uint16_t ecat_header; /* length + type */
    ecat_datagram_t datagrams[4]; /* Multiple datagrams */
} ecat_frame_t;

/* Build a Logical Read/Write (LRW) datagram */
int ecat_build_lrw(ecat_datagram_t *dg, uint32_t logical_addr,
                   uint8_t *data, uint16_t len, uint8_t idx) {
    dg->cmd       = ECAT_CMD_LRW;
    dg->index     = idx;
    dg->address   = logical_addr;
    dg->len_flags = len & 0x07FF;  /* Length in lower 11 bits */
    dg->irq       = 0x0000;
    memcpy(dg->data, data, len);
    dg->wkc       = 0;             /* Master sets to 0; slaves increment */
    return sizeof(ecat_datagram_t) - 256 + len;
}

/* Check working counter after frame returns */
int ecat_verify_wkc(const ecat_datagram_t *dg, uint16_t expected) {
    if (dg->wkc != expected) {
        printf("WKC mismatch: got %d, expected %d\n", dg->wkc, expected);
        return -1;
    }
    return 0;
}
```

---

## 2.4 EtherNet/IP

### Overview

- **OSI Layer:** Layer 7 (CIP over TCP/UDP/IP)
- **Explicit Messaging:** TCP port 44818 (configuration)
- **Implicit Messaging:** UDP port 2222 (real-time I/O)

### Block Diagram

```
   ┌──────────────────┐
   │   EtherNet/IP    │
   │   Scanner (PLC)  │
   └────────┬─────────┘
            │ Standard Ethernet + TCP/UDP
   ┌────────┼──────┬───────────┐
   │        │      │           │
┌──┴────┐ ┌─┴───┐  │┌─────┐ ┌──┴────┐
│Adapter│ │Adpt.│  ││Adpt.│ │Adapter│
│VFD    │ │I/O  │  ││Valve│ │Robot  │
└───────┘ └─────┘  │└─────┘ └───────┘
   TCP:Config         UDP:Cyclic I/O
```

### CIP Encapsulation Frame

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Command | 2 | 16 | Encapsulation command (e.g., 0x006F = SendRRData) |
| Length | 2 | 16 | Data portion length |
| Session Handle | 4 | 32 | Assigned by target |
| Status | 4 | 32 | 0 = success |
| Sender Context | 8 | 64 | Originator reference |
| Options | 4 | 32 | Must be 0 |
| CIP Data | N | N×8 | Service code + path + data |

### C Code Example

```c
/* EtherNet/IP Register Session (TCP) */
#include <stdint.h>
#include <string.h>

#define ENIP_CMD_REGISTER_SESSION  0x0065
#define ENIP_CMD_SEND_RR_DATA     0x006F
#define ENIP_PORT                 44818

typedef struct __attribute__((packed)) {
    uint16_t command;
    uint16_t length;
    uint32_t session_handle;
    uint32_t status;
    uint8_t  sender_context[8];
    uint32_t options;
} enip_header_t;

/* Build Register Session request */
int enip_register_session(uint8_t *buffer) {
    enip_header_t *hdr = (enip_header_t *)buffer;
    memset(hdr, 0, sizeof(enip_header_t));
    hdr->command = ENIP_CMD_REGISTER_SESSION;
    hdr->length  = 4;  /* Protocol version + option flags */

    /* Session registration data */
    buffer[24] = 0x01;  /* Protocol version 1 */
    buffer[25] = 0x00;
    buffer[26] = 0x00;  /* Options flags */
    buffer[27] = 0x00;

    return 28;  /* Total packet length */
}

/* Build Read Tag Service request */
int enip_read_tag(uint8_t *buffer, uint32_t session,
                  const char *tag_name) {
    enip_header_t *hdr = (enip_header_t *)buffer;
    memset(hdr, 0, sizeof(enip_header_t));
    hdr->command        = ENIP_CMD_SEND_RR_DATA;
    hdr->session_handle = session;

    /* CIP payload follows after header */
    uint8_t *cip = buffer + sizeof(enip_header_t);
    int offset = 0;

    /* Interface handle + timeout */
    memset(cip, 0, 6);
    offset = 6;

    /* Item count = 2 (Null Address + Unconnected Data) */
    cip[offset++] = 0x02; cip[offset++] = 0x00;

    /* Address item: Null */
    cip[offset++] = 0x00; cip[offset++] = 0x00;
    cip[offset++] = 0x00; cip[offset++] = 0x00;

    hdr->length = offset;
    return sizeof(enip_header_t) + offset;
}
```

---

## 2.5 BACnet

### Overview

- **OSI Layer:** Layers 1–7
- **Transport:** MS/TP (RS-485), BACnet/IP (UDP), ARCNET
- **Data Model:** Object-Property architecture
- **Port:** UDP 47808 (0xBAC0)

### Block Diagram

```
 ┌────────────────────┐
 │  BACnet Workstation│ (BMS Software)
 └────────┬───────────┘
          │ BACnet/IP (UDP)
   ┌──────┼───────────────┐
   │      │               │
┌──┴────┐ │  ┌──────────┐ │ ┌─────────────┐
│Router │ │  │ BACnet   │ │ │ BACnet      │
│IP↔MSTP│ │  │Controller│ │ │ Controller  │
└──┬────┘ │  │ HVAC     │ │ │ Lighting    │
   │MSTP  │  └──────────┘ │ └─────────────┘
┌──┴────┐ │
│Sensor │ │
│(MSTP) │ │
└───────┘ │
```

### BACnet/IP Frame (BVLC + NPDU + APDU)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| **BVLC Header:** | | | |
| Type | 1 | 8 | 0x81 = BACnet/IP |
| Function | 1 | 8 | 0x0A = Original-Unicast-NPDU |
| Length | 2 | 16 | Total BVLC message length |
| **NPDU:** | | | |
| Version | 1 | 8 | 0x01 |
| Control | 1 | 8 | Bit flags: DNET, SNET, etc. |
| **APDU:** | | | |
| PDU Type | 4 bits | 4 | Confirmed/Unconfirmed request |
| Service Choice | 1 | 8 | ReadProperty=0x0C, WriteProperty=0x0F |
| Object ID | 4 | 32 | Object type (10 bits) + Instance (22 bits) |
| Property ID | 1+ | 8+ | Property to read/write |

### C Code Example

```c
/* BACnet ReadProperty Request Builder */
#include <stdint.h>
#include <string.h>

#define BACNET_IP_PORT       47808  /* 0xBAC0 */
#define BACNET_BVLC_TYPE     0x81
#define BACNET_BVLC_UNICAST  0x0A
#define BACNET_READ_PROPERTY 0x0C

typedef struct __attribute__((packed)) {
    uint8_t  type;
    uint8_t  function;
    uint16_t length;
} bvlc_header_t;

typedef struct __attribute__((packed)) {
    uint8_t version;
    uint8_t control;
} bacnet_npdu_t;

/* Encode BACnet Object Identifier */
uint32_t bacnet_encode_object_id(uint16_t type, uint32_t instance) {
    return ((uint32_t)(type & 0x3FF) << 22) | (instance & 0x3FFFFF);
}

/* Build ReadProperty request */
int bacnet_read_property(uint8_t *buffer, uint16_t obj_type,
                         uint32_t obj_instance, uint8_t property_id,
                         uint8_t invoke_id) {
    int offset = 0;

    /* BVLC Header */
    buffer[offset++] = BACNET_BVLC_TYPE;
    buffer[offset++] = BACNET_BVLC_UNICAST;
    int len_pos = offset;
    offset += 2;  /* Length filled later */

    /* NPDU */
    buffer[offset++] = 0x01;  /* Version 1 */
    buffer[offset++] = 0x04;  /* Expecting reply */

    /* APDU: Confirmed Request */
    buffer[offset++] = 0x00;       /* Confirmed request, no segmentation */
    buffer[offset++] = 0x05;       /* Max segments=0, max APDU=1476 */
    buffer[offset++] = invoke_id;
    buffer[offset++] = BACNET_READ_PROPERTY;

    /* Object Identifier (context tag 0) */
    uint32_t oid = bacnet_encode_object_id(obj_type, obj_instance);
    buffer[offset++] = 0x0C;  /* Context tag 0, length 4 */
    buffer[offset++] = (oid >> 24) & 0xFF;
    buffer[offset++] = (oid >> 16) & 0xFF;
    buffer[offset++] = (oid >> 8)  & 0xFF;
    buffer[offset++] = oid & 0xFF;

    /* Property Identifier (context tag 1) */
    buffer[offset++] = 0x19;  /* Context tag 1, length 1 */
    buffer[offset++] = property_id;

    /* Fill BVLC length */
    uint16_t total_len = offset;
    buffer[len_pos]     = (total_len >> 8) & 0xFF;
    buffer[len_pos + 1] = total_len & 0xFF;

    return offset;
}
```

---

# Module 3: IoT & Wireless Protocols

---

## 3.1 MQTT (Message Queuing Telemetry Transport)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP (port 1883, TLS on 8883)
- **Architecture:** Publish/Subscribe with central Broker
- **QoS Levels:** 0 (at most once), 1 (at least once), 2 (exactly once)

### Block Diagram

```
 ┌────────────┐         ┌───────────┐         ┌───────────┐
 │ Publisher  │         │   MQTT    │         │Subscriber │
 │(Temp Sens.)├────────►│  Broker   ├────────►│(Dashboard)│
 └────────────┘ PUBLISH │           │PUBLISH  └───────────┘
               Topic:   │  ┌─────┐  │
 ┌───────────┐ home/    │  │Topic│  │         ┌───────────┐
 │ Publisher │ temp     │  │Table│  │         │Subscriber │
 │(Humidity) ├─────────►│  └─────┘  ├────────►│(Alarm Sys)│
 └───────────┘          └───────────┘         └───────────┘
```

### MQTT Fixed Header

| Field | Bits | Description |
|-------|------|-------------|
| Packet Type | 4 | CONNECT=1, PUBLISH=3, SUBSCRIBE=8, etc. |
| Flags | 4 | DUP (1), QoS (2), RETAIN (1) |
| Remaining Length | 8–32 | Variable-length encoding (1–4 bytes) |

### MQTT PUBLISH Packet

| Field | Bytes | Description |
|-------|-------|-------------|
| Fixed Header | 2+ | Type + Flags + Remaining Length |
| Topic Length | 2 | UTF-8 encoded string length |
| Topic Name | N | e.g., "home/livingroom/temp" |
| Packet ID | 2 | Present only if QoS > 0 |
| Payload | M | Application data |

### C Code Example

```c
/* Minimal MQTT PUBLISH Packet Builder */
#include <stdint.h>
#include <string.h>
#include <stdio.h>

#define MQTT_PUBLISH    0x30
#define MQTT_CONNECT    0x10
#define MQTT_SUBSCRIBE  0x82

/* Encode MQTT Remaining Length (variable-length encoding) */
int mqtt_encode_remaining(uint8_t *buf, uint32_t length) {
    int i = 0;
    do {
        uint8_t byte = length % 128;
        length /= 128;
        if (length > 0) byte |= 0x80;
        buf[i++] = byte;
    } while (length > 0);
    return i;
}

/* Build MQTT PUBLISH packet */
int mqtt_build_publish(uint8_t *buffer, const char *topic,
                       const uint8_t *payload, uint16_t payload_len,
                       uint8_t qos, uint16_t packet_id) {
    int offset = 0;

    /* Fixed header: PUBLISH with QoS */
    buffer[offset++] = MQTT_PUBLISH | ((qos & 0x03) << 1);

    /* Calculate remaining length */
    uint16_t topic_len = strlen(topic);
    uint32_t remaining = 2 + topic_len + payload_len;
    if (qos > 0) remaining += 2;  /* Packet ID */

    offset += mqtt_encode_remaining(&buffer[offset], remaining);

    /* Topic */
    buffer[offset++] = (topic_len >> 8) & 0xFF;
    buffer[offset++] = topic_len & 0xFF;
    memcpy(&buffer[offset], topic, topic_len);
    offset += topic_len;

    /* Packet ID (QoS 1 or 2 only) */
    if (qos > 0) {
        buffer[offset++] = (packet_id >> 8) & 0xFF;
        buffer[offset++] = packet_id & 0xFF;
    }

    /* Payload */
    memcpy(&buffer[offset], payload, payload_len);
    offset += payload_len;

    return offset;
}

/* Example usage */
int main(void) {
    uint8_t packet[256];
    const char *topic = "home/livingroom/temp";
    const char *data  = "23.5";

    int len = mqtt_build_publish(packet, topic,
                                 (const uint8_t *)data, strlen(data),
                                 1, 0x0001);
    printf("MQTT PUBLISH: %d bytes\n", len);
    return 0;
}
```

---

## 3.2 CoAP (Constrained Application Protocol)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** UDP (port 5683, DTLS on 5684)
- **Header Size:** Fixed 4 bytes (vs. HTTP's ~700+ bytes)
- **Methods:** GET, POST, PUT, DELETE

### Block Diagram

```
 ┌───────────────┐              ┌──────────────┐
 │  CoAP Client  │   UDP/DTLS   │  CoAP Server │
 │ (Smartphone)  ├─────────────►│ (IoT Sensor) │
 │               │◄─────────────┤              │
 │  GET /temp    │   2.05 OK    │ Resource:    │
 │               │   "23.5°C"   │  /temp       │
 └───────────────┘              │  /humidity   │
                                └──────────────┘
```

### CoAP Message Format (4-byte header)

| Field | Bits | Description |
|-------|------|-------------|
| Version (Ver) | 2 | Always 01 (version 1) |
| Type (T) | 2 | CON=0, NON=1, ACK=2, RST=3 |
| Token Length (TKL) | 4 | 0–8 bytes |
| Code | 8 | Class (3 bits) + Detail (5 bits). E.g., 0.01=GET, 2.05=Content |
| Message ID | 16 | For matching ACK to CON |
| Token | 0–64 | Request/response correlation |
| Options | Variable | Delta + Length + Value |
| Payload Marker | 8 | 0xFF separates options from payload |
| Payload | Variable | Application data |

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |     Code      |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### C Code Example

```c
/* CoAP GET Request Builder */
#include <stdint.h>
#include <string.h>

#define COAP_VERSION      1
#define COAP_TYPE_CON     0  /* Confirmable */
#define COAP_TYPE_NON     1  /* Non-confirmable */
#define COAP_CODE_GET     1  /* 0.01 */
#define COAP_CODE_POST    2  /* 0.02 */
#define COAP_CODE_CONTENT 69 /* 2.05 */
#define COAP_OPT_URI_PATH 11

typedef struct {
    uint8_t  ver_type_tkl;
    uint8_t  code;
    uint16_t message_id;
    uint8_t  token[8];
    uint8_t  tkl;
} coap_header_t;

/* Build CoAP GET request */
int coap_build_get(uint8_t *buffer, uint16_t msg_id,
                   uint8_t *token, uint8_t tkl,
                   const char *uri_path) {
    int offset = 0;

    /* Header */
    buffer[offset++] = (COAP_VERSION << 6) | (COAP_TYPE_CON << 4) | (tkl & 0x0F);
    buffer[offset++] = COAP_CODE_GET;
    buffer[offset++] = (msg_id >> 8) & 0xFF;
    buffer[offset++] = msg_id & 0xFF;

    /* Token */
    memcpy(&buffer[offset], token, tkl);
    offset += tkl;

    /* URI-Path option (option number 11) */
    uint8_t path_len = strlen(uri_path);
    if (path_len <= 12) {
        buffer[offset++] = (COAP_OPT_URI_PATH << 4) | path_len;
    } else {
        buffer[offset++] = (COAP_OPT_URI_PATH << 4) | 13;
        buffer[offset++] = path_len - 13;
    }
    memcpy(&buffer[offset], uri_path, path_len);
    offset += path_len;

    return offset;
}

/* Parse CoAP response */
int coap_parse_response(const uint8_t *data, int len,
                        uint8_t *code, uint8_t *payload, int *plen) {
    *code = data[1];
    uint8_t tkl = data[0] & 0x0F;
    int offset = 4 + tkl;

    /* Skip options until payload marker 0xFF */
    while (offset < len && data[offset] != 0xFF) {
        uint8_t delta_len = data[offset++];
        uint8_t opt_len = delta_len & 0x0F;
        offset += opt_len;
    }
    if (offset < len && data[offset] == 0xFF) {
        offset++;
        *plen = len - offset;
        memcpy(payload, &data[offset], *plen);
    }
    return 0;
}
```

---

## 3.3 Zigbee

### Overview

- **OSI Layer:** Layers 3–7 (over IEEE 802.15.4)
- **Frequency:** 2.4 GHz (16 channels), 868/915 MHz
- **Modulation:** DSSS (Direct Sequence Spread Spectrum)
- **Topology:** Mesh (self-healing)
- **Roles:** Coordinator, Router, End Device

### Block Diagram

```
  ┌───────────────┐
  │   Zigbee      │
  │  Coordinator  │ (Trust Center)
  └──────┬────────┘
         │
    ┌────┴────┬───────────┐
    │         │           │
 ┌──┴───┐  ┌──┴───┐   ┌───┴──┐
 │Router│  │Router│   │Router│
 │  #1  │  │  #2  │   │  #3  │
 └──┬───┘  └──┬───┘   └──┬───┘
    │         │          │
 ┌──┴──┐   ┌──┴──┐   ┌───┴────┐
 │End  │   │End  │   │End     │
 │Dev. │   │Dev. │   │Device  │
 │Light│   │Lock │   │Therm.  │
 └─────┘   └─────┘   └────────┘
      (Mesh — self-healing routes)
```

### IEEE 802.15.4 MAC Frame + Zigbee NWK

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| **MAC Layer:** | | | |
| Frame Control | 2 | 16 | Type, security, addressing mode |
| Sequence Number | 1 | 8 | Frame counter |
| Dest PAN ID | 2 | 16 | Network identifier |
| Dest Address | 2/8 | 16/64 | Short or extended address |
| Src Address | 2/8 | 16/64 | Short or extended address |
| **NWK Layer:** | | | |
| Frame Control | 2 | 16 | Type, protocol version, route |
| Dest Address | 2 | 16 | Network destination |
| Src Address | 2 | 16 | Network source |
| Radius | 1 | 8 | Max hop count |
| Sequence | 1 | 8 | NWK frame counter |
| **APS/Payload** | N | N×8 | Application data |
| **MAC FCS** | 2 | 16 | CRC-16 |

### C Code Example

```c
/* Zigbee IEEE 802.15.4 Frame Builder */
#include <stdint.h>
#include <string.h>

#define ZB_FRAME_TYPE_DATA    0x01
#define ZB_ADDR_MODE_SHORT    0x02

typedef struct __attribute__((packed)) {
    uint16_t frame_control;
    uint8_t  seq_number;
    uint16_t dst_pan_id;
    uint16_t dst_addr;
    uint16_t src_addr;
} ieee802154_hdr_t;

typedef struct __attribute__((packed)) {
    uint16_t frame_control;
    uint16_t dst_addr;
    uint16_t src_addr;
    uint8_t  radius;
    uint8_t  seq_number;
} zigbee_nwk_hdr_t;

/* Build Zigbee data frame */
int zigbee_build_data(uint8_t *buffer, uint16_t pan_id,
                      uint16_t dst, uint16_t src,
                      const uint8_t *payload, uint8_t pay_len,
                      uint8_t seq) {
    int offset = 0;

    /* IEEE 802.15.4 MAC header */
    ieee802154_hdr_t mac = {0};
    mac.frame_control = 0x8861;  /* Data, PAN compress, short addr */
    mac.seq_number    = seq;
    mac.dst_pan_id    = pan_id;
    mac.dst_addr      = dst;
    mac.src_addr      = src;
    memcpy(&buffer[offset], &mac, sizeof(mac));
    offset += sizeof(mac);

    /* Zigbee NWK header */
    zigbee_nwk_hdr_t nwk = {0};
    nwk.frame_control = 0x0002;  /* Data frame, NWK v2 */
    nwk.dst_addr      = dst;
    nwk.src_addr      = src;
    nwk.radius        = 0x1E;   /* Max 30 hops */
    nwk.seq_number    = seq;
    memcpy(&buffer[offset], &nwk, sizeof(nwk));
    offset += sizeof(nwk);

    /* Payload */
    memcpy(&buffer[offset], payload, pay_len);
    offset += pay_len;

    return offset;  /* FCS appended by radio hardware */
}
```

---

## 3.4 Z-Wave

### Overview

- **OSI Layer:** Layers 1–7 (complete stack)
- **Frequency:** Sub-GHz (908 MHz US, 868 MHz EU)
- **Topology:** Source-routed mesh, up to 232 nodes
- **Range:** ~30 m indoor, up to 100 m line-of-sight

### Block Diagram

```
      ┌──────────────┐
      │   Primary    │
      │  Controller  │ (Z-Wave Hub)
      └──────┬───────┘
             │  Sub-GHz RF
    ┌────────┴┬───────────┐
    │         │           │
 ┌──┴──┐   ┌──┴───┐   ┌───┴──┐
 │Node │   │Node  │   │Node  │
 │Door │   │Dimmer│   │Therm │
 │Lock │   │      │   │ostat │
 └──┬──┘   └──────┘   └──┬───┘
    │                    │
 ┌──┴──┐             ┌───┴──┐
 │Node │             │Node  │
 │Siren│             │Plug  │
 └─────┘             └──────┘
```

### Z-Wave Frame Structure

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Home ID | 4 | 32 | Network identifier |
| Source Node ID | 1 | 8 | Sender (1–232) |
| Frame Control | 1 | 8 | Type, routed, ACK request |
| Length | 1 | 8 | Remaining frame bytes |
| Dest Node ID | 1 | 8 | Receiver (1–232) |
| Command Class | 1 | 8 | Feature category (e.g., 0x25 = Switch Binary) |
| Command | 1 | 8 | Action (e.g., 0x01 = SET) |
| Data | N | N×8 | Command parameters |
| Checksum | 1 | 8 | XOR of all preceding bytes |

### C Code Example

```c
/* Z-Wave Frame Builder (Application Layer) */
#include <stdint.h>
#include <string.h>

#define ZWAVE_CMD_CLASS_SWITCH_BINARY  0x25
#define ZWAVE_CMD_SET                  0x01
#define ZWAVE_CMD_GET                  0x02
#define ZWAVE_CMD_REPORT               0x03

typedef struct {
    uint32_t home_id;
    uint8_t  src_node;
    uint8_t  frame_ctrl;
    uint8_t  length;
    uint8_t  dst_node;
    uint8_t  cmd_class;
    uint8_t  command;
    uint8_t  data[32];
    uint8_t  data_len;
} zwave_frame_t;

/* Calculate Z-Wave checksum (XOR of all bytes) */
uint8_t zwave_checksum(const uint8_t *data, uint8_t len) {
    uint8_t cs = 0xFF;
    for (uint8_t i = 0; i < len; i++) {
        cs ^= data[i];
    }
    return cs;
}

/* Build Switch Binary SET command */
int zwave_switch_set(uint8_t *buffer, uint32_t home_id,
                     uint8_t src, uint8_t dst, uint8_t value) {
    int offset = 0;

    /* Home ID */
    buffer[offset++] = (home_id >> 24) & 0xFF;
    buffer[offset++] = (home_id >> 16) & 0xFF;
    buffer[offset++] = (home_id >> 8)  & 0xFF;
    buffer[offset++] = home_id & 0xFF;

    buffer[offset++] = src;   /* Source Node */
    buffer[offset++] = 0x41;  /* Frame Control: singlecast + ACK req */
    buffer[offset++] = 0x04;  /* Length of remaining */
    buffer[offset++] = dst;   /* Destination Node */

    /* Command */
    buffer[offset++] = ZWAVE_CMD_CLASS_SWITCH_BINARY;
    buffer[offset++] = ZWAVE_CMD_SET;
    buffer[offset++] = value; /* 0x00=OFF, 0xFF=ON */

    /* Checksum */
    buffer[offset] = zwave_checksum(buffer, offset);
    offset++;

    return offset;
}
```

---

## 3.5 LoRaWAN

### Overview

- **OSI Layer:** MAC layer over LoRa PHY
- **Modulation:** Chirp Spread Spectrum (CSS)
- **Range:** Up to 15+ km (rural), 2–5 km (urban)
- **Bandwidth:** Very low (0.3–50 kbps)
- **Device Classes:** A (lowest power), B (beacon), C (always listening)

### Block Diagram

```
 ┌─────────┐  ┌─────────┐  ┌─────────┐
 │End Dev. │  │End Dev. │  │End Dev. │
 │Class A  │  │ Class B │  │ Class C │
 └────┬────┘  └────┬────┘  └────┬────┘
      │LoRa RF     │LoRa RF     │LoRa RF
  ┌───┴────────────┴────────────┴───┐
  │         LoRa Gateway            │
  │      (Concentrator Module)      │
  └──────────────┬──────────────────┘
                 │ IP Backhaul (Ethernet/4G)
  ┌──────────────┴──────────────────┐
  │       Network Server            │
  │  (ADR, Dedup, MAC Commands)     │
  └──────────────┬──────────────────┘
                 │
  ┌──────────────┴──────────────────┐
  │      Application Server         │
  └─────────────────────────────────┘
```

### LoRaWAN Frame (Uplink)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| **PHY Layer:** | | | |
| Preamble | 8 symbols | — | Synchronization chirps |
| PHDR | 2+ | — | Payload length + CRC config |
| **MAC Layer:** | | | |
| MHDR (MAC Header) | 1 | 8 | MType (3) + RFU (3) + Major (2) |
| DevAddr | 4 | 32 | Device address |
| FCtrl | 1 | 8 | ADR, ACK, FOptsLen |
| FCnt | 2 | 16 | Frame counter |
| FOpts | 0–15 | 0–120 | MAC commands (piggybacked) |
| FPort | 1 | 8 | 0=MAC, 1–223=Application |
| FRMPayload | N | N×8 | Encrypted application data |
| MIC | 4 | 32 | Message Integrity Code (AES-CMAC) |

### C Code Example

```c
/* LoRaWAN Uplink Frame Builder */
#include <stdint.h>
#include <string.h>

#define LORAWAN_MTYPE_UNCONFIRMED_UP  0x40
#define LORAWAN_MTYPE_CONFIRMED_UP    0x80
#define LORAWAN_MAJOR_R1              0x00

typedef struct __attribute__((packed)) {
    uint8_t  mhdr;
    uint32_t dev_addr;
    uint8_t  fctrl;
    uint16_t fcnt;
    uint8_t  fport;
    uint8_t  payload[64];
    uint8_t  payload_len;
} lorawan_frame_t;

/* Build LoRaWAN uplink frame (before encryption) */
int lorawan_build_uplink(uint8_t *buffer, uint32_t dev_addr,
                         uint16_t frame_counter, uint8_t port,
                         const uint8_t *data, uint8_t data_len,
                         uint8_t confirmed) {
    int offset = 0;

    /* MHDR: Message Type + Major version */
    buffer[offset++] = (confirmed ? LORAWAN_MTYPE_CONFIRMED_UP
                                  : LORAWAN_MTYPE_UNCONFIRMED_UP)
                       | LORAWAN_MAJOR_R1;

    /* Device Address (little-endian) */
    buffer[offset++] = dev_addr & 0xFF;
    buffer[offset++] = (dev_addr >> 8) & 0xFF;
    buffer[offset++] = (dev_addr >> 16) & 0xFF;
    buffer[offset++] = (dev_addr >> 24) & 0xFF;

    /* Frame Control */
    buffer[offset++] = 0x00;  /* ADR=0, no FOpts */

    /* Frame Counter (little-endian) */
    buffer[offset++] = frame_counter & 0xFF;
    buffer[offset++] = (frame_counter >> 8) & 0xFF;

    /* FPort */
    buffer[offset++] = port;

    /* Payload (to be encrypted with AppSKey) */
    memcpy(&buffer[offset], data, data_len);
    offset += data_len;

    /* MIC would be calculated here using NwkSKey (AES-CMAC) */
    /* Placeholder: 4 zero bytes */
    memset(&buffer[offset], 0, 4);
    offset += 4;

    return offset;
}
```

---

## 3.6 BLE (Bluetooth Low Energy)

### Overview

- **OSI Layer:** Complete controller + host stack
- **Frequency:** 2.4 GHz ISM band (40 channels: 3 advertising + 37 data)
- **Hopping:** Adaptive Frequency Hopping (AFH)
- **Data Model:** GATT — Services → Characteristics → Descriptors
- **Power:** Ultra-low (radio off most of the time)

### Block Diagram

```
 ┌──────────────────────────────┐
 │         BLE Central          │
 │        (Smartphone)          │
 │  ┌────────────────────────┐  │
 │  │ GATT Client            │  │
 │  │  Read/Write Chars.     │  │
 │  └────────────────────────┘  │
 └──────────────┬───────────────┘
                │ 2.4 GHz (AFH)
 ┌──────────────┴───────────────┐
 │       BLE Peripheral         │
 │       (Heart Rate Sensor)    │
 │  ┌────────────────────────┐  │
 │  │ GATT Server            │  │
 │  │  Heart Rate Service    │  │
 │  │   └─ HR Measurement    │  │
 │  │   └─ Body Location     │  │
 │  └────────────────────────┘  │
 └──────────────────────────────┘
```

### BLE Advertising PDU (on channels 37, 38, 39)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Preamble | 1 | 8 | 0xAA (10101010) |
| Access Address | 4 | 32 | 0x8E89BED6 (advertising) |
| PDU Header | 2 | 16 | Type (4), RFU (2), TxAdd (1), RxAdd (1), Length (8) |
| AdvA (Advertiser Address) | 6 | 48 | Device MAC |
| AdvData | 0–31 | 0–248 | AD structures (type + data) |
| CRC | 3 | 24 | CRC-24 |

### BLE Data Channel PDU

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Preamble | 1 | 8 | 0xAA or 0x55 |
| Access Address | 4 | 32 | Connection-specific (random) |
| PDU Header | 2 | 16 | LLID (2), NESN (1), SN (1), MD (1), Length (8) |
| L2CAP + ATT Payload | 0–251 | — | GATT operations |
| CRC | 3 | 24 | CRC-24 |
| **Max total** | **~261** | — | **Per data PDU** |

### C Code Example

```c
/* BLE GATT Characteristic Notification (Conceptual) */
#include <stdint.h>
#include <string.h>

/* BLE AD Type definitions */
#define BLE_AD_TYPE_FLAGS          0x01
#define BLE_AD_TYPE_COMPLETE_NAME  0x09
#define BLE_AD_TYPE_TX_POWER       0x0A

/* GATT ATT opcodes */
#define ATT_READ_REQ               0x0A
#define ATT_READ_RSP               0x0B
#define ATT_WRITE_REQ              0x12
#define ATT_HANDLE_VALUE_NTF       0x1B

typedef struct {
    uint16_t handle;
    uint8_t  properties;  /* Read, Write, Notify, Indicate */
    uint8_t  value[20];
    uint8_t  value_len;
} ble_characteristic_t;

/* Build BLE advertising data */
int ble_build_adv_data(uint8_t *buffer, const char *name, uint8_t flags) {
    int offset = 0;

    /* Flags AD structure */
    buffer[offset++] = 0x02;            /* Length */
    buffer[offset++] = BLE_AD_TYPE_FLAGS;
    buffer[offset++] = flags;           /* 0x06 = General Discoverable */

    /* Complete Local Name */
    uint8_t name_len = strlen(name);
    buffer[offset++] = name_len + 1;
    buffer[offset++] = BLE_AD_TYPE_COMPLETE_NAME;
    memcpy(&buffer[offset], name, name_len);
    offset += name_len;

    return offset;
}

/* Build ATT Handle Value Notification */
int ble_build_notification(uint8_t *buffer, uint16_t handle,
                           const uint8_t *value, uint8_t len) {
    int offset = 0;

    /* ATT opcode */
    buffer[offset++] = ATT_HANDLE_VALUE_NTF;

    /* Attribute handle (little-endian) */
    buffer[offset++] = handle & 0xFF;
    buffer[offset++] = (handle >> 8) & 0xFF;

    /* Attribute value */
    memcpy(&buffer[offset], value, len);
    offset += len;

    return offset;
}

/* Example: Notify heart rate measurement */
void ble_notify_heart_rate(uint16_t hr_handle, uint8_t bpm) {
    uint8_t pdu[20];
    uint8_t hr_data[2] = {0x00, bpm};  /* Flags=0, HR value */
    int len = ble_build_notification(pdu, hr_handle, hr_data, 2);
    /* Send pdu via BLE stack */
    (void)len;
}
```

---

# Module 4: Web & Application Protocols

---

## 4.1 HTTP / HTTPS

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP (HTTP/1.1, HTTP/2), QUIC/UDP (HTTP/3)
- **Security:** TLS (HTTPS) — certificate + symmetric key exchange
- **Methods:** GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS

### Block Diagram

```
 ┌──────────────┐                          ┌──────────────┐
 │   Browser    │   TLS Handshake          │  Web Server  │
 │   (Client)   │─────────────────────────►│  (Nginx)     │
 │              │◄─────────────────────────│              │
 │  GET /index  │   Certificate + Keys     │              │
 │              │═════════════════════════►│   /index.html│
 │              │◄═════════════════════════│   /api/data  │
 │              │   Encrypted HTTP Data    │              │
 └──────────────┘                          └──────────────┘
```

### HTTP/1.1 Request Format

```
GET /api/data HTTP/1.1\r\n
Host: example.com\r\n
User-Agent: MyApp/1.0\r\n
Accept: application/json\r\n
Connection: keep-alive\r\n
\r\n
```

### HTTP/2 Binary Frame

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Length | 3 | 24 | Payload length |
| Type | 1 | 8 | DATA=0, HEADERS=1, SETTINGS=4, etc. |
| Flags | 1 | 8 | END_STREAM, END_HEADERS, PADDED, etc. |
| Reserved | 1 bit | 1 | Must be 0 |
| Stream ID | 31 bits | 31 | Identifies multiplexed stream |
| Payload | N | N×8 | Frame-type-specific data |

### C Code Example

```c
/* Simple HTTP/1.1 GET Client (POSIX sockets) */
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#include <unistd.h>

int http_get(const char *host, const char *path, char *response, int max_len) {
    int sock;
    struct hostent *server;
    struct sockaddr_in addr;

    /* Create socket */
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) return -1;

    /* Resolve host */
    server = gethostbyname(host);
    if (!server) { close(sock); return -1; }

    /* Connect */
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(80);
    memcpy(&addr.sin_addr.s_addr, server->h_addr, server->h_length);

    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock); return -1;
    }

    /* Build HTTP request */
    char request[512];
    snprintf(request, sizeof(request),
             "GET %s HTTP/1.1\r\n"
             "Host: %s\r\n"
             "Connection: close\r\n"
             "\r\n", path, host);

    /* Send */
    send(sock, request, strlen(request), 0);

    /* Receive */
    int total = 0, n;
    while ((n = recv(sock, response + total, max_len - total - 1, 0)) > 0)
        total += n;
    response[total] = '\0';

    close(sock);
    return total;
}

int main(void) {
    char response[8192];
    int len = http_get("httpbin.org", "/get", response, sizeof(response));
    if (len > 0) printf("Response (%d bytes):\n%s\n", len, response);
    return 0;
}
```

---

## 4.2 WebSocket

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP (upgraded from HTTP/1.1)
- **Handshake:** HTTP Upgrade → 101 Switching Protocols
- **Framing:** 2–14 byte overhead, full-duplex

### Block Diagram

```
 ┌──────────┐                        ┌──────────┐
 │  Client  │  HTTP Upgrade Request  │  Server  │
 │ (Browser)├───────────────────────►│          │
 │          │  101 Switching Proto.  │          │
 │          │◄───────────────────────┤          │
 │          │                        │          │
 │          │◄══════════════════════►│          │
 │          │  Full-Duplex Frames    │          │
 │          │  (Persistent TCP conn) │          │
 └──────────┘                        └──────────┘
```

### WebSocket Frame

| Field | Bits | Description |
|-------|------|-------------|
| FIN | 1 | Final fragment of message |
| RSV1–RSV3 | 3 | Reserved (for extensions) |
| Opcode | 4 | 0x1=Text, 0x2=Binary, 0x8=Close, 0x9=Ping, 0xA=Pong |
| MASK | 1 | 1 = payload is masked (client→server) |
| Payload Length | 7 | 0–125, or 126 (next 2 bytes), or 127 (next 8 bytes) |
| Extended Length | 0/16/64 | If length=126 or 127 |
| Masking Key | 0 or 32 | 4-byte XOR key (if MASK=1) |
| Payload Data | N×8 | Application data |

### C Code Example

```c
/* WebSocket Frame Builder */
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#define WS_OPCODE_TEXT   0x01
#define WS_OPCODE_BIN    0x02
#define WS_OPCODE_CLOSE  0x08
#define WS_OPCODE_PING   0x09
#define WS_OPCODE_PONG   0x0A

/* Build a WebSocket frame (client-side, masked) */
int ws_build_frame(uint8_t *buffer, uint8_t opcode,
                   const uint8_t *payload, uint64_t len, int mask) {
    int offset = 0;

    /* Byte 0: FIN + Opcode */
    buffer[offset++] = 0x80 | (opcode & 0x0F);  /* FIN=1 */

    /* Byte 1: MASK + Payload Length */
    uint8_t mask_bit = mask ? 0x80 : 0x00;

    if (len <= 125) {
        buffer[offset++] = mask_bit | (uint8_t)len;
    } else if (len <= 65535) {
        buffer[offset++] = mask_bit | 126;
        buffer[offset++] = (len >> 8) & 0xFF;
        buffer[offset++] = len & 0xFF;
    } else {
        buffer[offset++] = mask_bit | 127;
        for (int i = 7; i >= 0; i--)
            buffer[offset++] = (len >> (i * 8)) & 0xFF;
    }

    /* Masking key (client→server frames must be masked) */
    uint8_t masking_key[4] = {0};
    if (mask) {
        for (int i = 0; i < 4; i++)
            masking_key[i] = (uint8_t)(rand() & 0xFF);
        memcpy(&buffer[offset], masking_key, 4);
        offset += 4;
    }

    /* Payload (XOR with mask if applicable) */
    for (uint64_t i = 0; i < len; i++) {
        buffer[offset++] = mask ? payload[i] ^ masking_key[i % 4]
                                : payload[i];
    }
    return offset;
}
```

---

## 4.3 gRPC

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** HTTP/2 (mandatory)
- **Serialization:** Protocol Buffers (Protobuf) — binary
- **Streaming:** Unary, Client, Server, Bidirectional

### Block Diagram

```
 ┌────────────────┐                    ┌────────────────┐
 │  gRPC Client   │     HTTP/2         │  gRPC Server   │
 │                │═══════════════════►│                │
 │  Stub (auto-   │  Protobuf binary   │  Service Impl  │
 │  generated)    │◄═══════════════════│                │
 │                │                    │                │
 │ .proto schema  │  Multiplexed       │ .proto schema  │
 │ defines API    │  streams           │ defines API    │
 └────────────────┘                    └────────────────┘
```

### gRPC over HTTP/2 Frame

| Field | Bytes | Description |
|-------|-------|-------------|
| **HTTP/2 Frame Header:** | | |
| Length | 3 | Payload size |
| Type | 1 | HEADERS or DATA |
| Flags | 1 | END_STREAM, END_HEADERS |
| Stream ID | 4 | Identifies RPC call |
| **gRPC Length-Prefixed Message:** | | |
| Compressed Flag | 1 | 0 = uncompressed |
| Message Length | 4 | Protobuf message size |
| Protobuf Message | N | Serialized request/response |

### C Code Example

```c
/* gRPC Length-Prefixed Message Encoder/Decoder */
#include <stdint.h>
#include <string.h>
#include <arpa/inet.h>

/* gRPC wire format: 1-byte compressed flag + 4-byte length + protobuf */
int grpc_encode_message(uint8_t *buffer, const uint8_t *protobuf_msg,
                        uint32_t msg_len, uint8_t compressed) {
    int offset = 0;

    /* Compressed flag */
    buffer[offset++] = compressed ? 1 : 0;

    /* Message length (big-endian) */
    uint32_t net_len = htonl(msg_len);
    memcpy(&buffer[offset], &net_len, 4);
    offset += 4;

    /* Protobuf serialized message */
    memcpy(&buffer[offset], protobuf_msg, msg_len);
    offset += msg_len;

    return offset;  /* Total: 5 + msg_len */
}

/* Decode gRPC length-prefixed message */
int grpc_decode_message(const uint8_t *buffer, uint8_t *compressed,
                        uint32_t *msg_len, const uint8_t **msg_data) {
    *compressed = buffer[0];
    uint32_t net_len;
    memcpy(&net_len, &buffer[1], 4);
    *msg_len = ntohl(net_len);
    *msg_data = &buffer[5];
    return 5 + *msg_len;
}

/* Simple Protobuf varint encoder (field 1, type varint) */
int protobuf_encode_int32(uint8_t *buf, int field_num, int32_t value) {
    int offset = 0;
    buf[offset++] = (field_num << 3) | 0;  /* Wire type 0 = varint */
    uint32_t uval = (uint32_t)value;
    while (uval > 0x7F) {
        buf[offset++] = (uval & 0x7F) | 0x80;
        uval >>= 7;
    }
    buf[offset++] = uval & 0x7F;
    return offset;
}
```

---

## 4.4 GraphQL

### Overview

- **OSI Layer:** Layer 7 (Application query language)
- **Transport:** HTTP POST to a single endpoint (e.g., `/graphql`)
- **Key Concept:** Client specifies exact data shape; server resolves via Schema + Resolvers

### Block Diagram

```
 ┌─────────────────┐                  ┌─────────────────┐
 │   GraphQL       │  POST /graphql   │  GraphQL Server │
 │   Client        ├─────────────────►│                 │
 │                 │  { query: "..." }│  ┌───────────┐  │
 │  Requests exact │                  │  │  Schema   │  │
 │  fields needed  │  JSON Response   │  │ (Types +  │  │
 │                 │◄─────────────────┤  │ Resolvers)│  │
 │  No over-fetch  │  Exact shape     │  └───────────┘  │
 └─────────────────┘                  └─────────────────┘
```

### GraphQL Request/Response Format

**Request (HTTP POST body):**
```json
{
  "query": "query { user(id: 1) { name email posts { title } } }",
  "variables": { "id": 1 }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": [{ "title": "Hello World" }]
    }
  }
}
```

### C Code Example

```c
/* GraphQL HTTP POST Request Builder */
#include <stdio.h>
#include <string.h>

/* Build a GraphQL query as HTTP POST body */
int graphql_build_request(char *buffer, int max_len,
                          const char *host, const char *query,
                          const char *variables) {
    /* JSON body */
    char body[2048];
    if (variables) {
        snprintf(body, sizeof(body),
                 "{\"query\":\"%s\",\"variables\":%s}", query, variables);
    } else {
        snprintf(body, sizeof(body), "{\"query\":\"%s\"}", query);
    }

    /* HTTP POST request */
    int len = snprintf(buffer, max_len,
        "POST /graphql HTTP/1.1\r\n"
        "Host: %s\r\n"
        "Content-Type: application/json\r\n"
        "Content-Length: %zu\r\n"
        "\r\n"
        "%s", host, strlen(body), body);

    return len;
}

/* Example usage */
int main(void) {
    char request[4096];
    const char *query = "{ user(id: 1) { name email } }";

    int len = graphql_build_request(request, sizeof(request),
                                    "api.example.com", query, NULL);
    printf("GraphQL Request (%d bytes):\n%s\n", len, request);
    return 0;
}
```

---

# Module 5: IT, Networking & Infrastructure

---

## 5.1 TCP/IP

### Overview

- **OSI Layer:** Layer 4 (TCP) + Layer 3 (IP)
- **Connection:** 3-Way Handshake (SYN → SYN-ACK → ACK)
- **Reliability:** Sequence numbers, ACKs, sliding window, retransmission
- **Congestion:** Slow start, congestion avoidance, fast retransmit

### Block Diagram

```
 ┌────────────┐                         ┌────────────┐
 │   Client   │                         │   Server   │
 └─────┬──────┘                         └─────┬──────┘
       │  SYN (seq=x)                         │
       ├─────────────────────────────────────►│
       │  SYN-ACK (seq=y, ack=x+1)            │
       │◄─────────────────────────────────────┤
       │  ACK (ack=y+1)                       │
       ├─────────────────────────────────────►│
       │         Connection Established       │
       │◄════════════════════════════════════►│
       │         Data Transfer (Reliable)     │
       │  FIN                                 │
       ├─────────────────────────────────────►│
       │  FIN-ACK                             │
       │◄─────────────────────────────────────┤
       │  ACK                                 │
       ├─────────────────────────────────────►│
```

### TCP Header (20 bytes minimum)

| Field | Bits | Description |
|-------|------|-------------|
| Source Port | 16 | Sender port number |
| Destination Port | 16 | Receiver port number |
| Sequence Number | 32 | Byte position in stream |
| Acknowledgment Number | 32 | Next expected byte from sender |
| Data Offset | 4 | Header length in 32-bit words |
| Reserved | 3 | Must be zero |
| Flags | 9 | NS, CWR, ECE, URG, ACK, PSH, RST, SYN, FIN |
| Window Size | 16 | Receive buffer space (bytes) |
| Checksum | 16 | Header + data integrity check |
| Urgent Pointer | 16 | Offset to urgent data |
| Options | 0–320 | MSS, Window Scale, SACK, Timestamps |
| **Minimum Total** | **160** | **20 bytes without options** |

### IPv4 Header (20 bytes minimum)

| Field | Bits | Description |
|-------|------|-------------|
| Version | 4 | 4 = IPv4 |
| IHL (Header Length) | 4 | In 32-bit words (min 5) |
| DSCP | 6 | Differentiated Services |
| ECN | 2 | Explicit Congestion Notification |
| Total Length | 16 | Entire packet size (bytes) |
| Identification | 16 | Fragment group identifier |
| Flags | 3 | DF (Don't Fragment), MF (More Fragments) |
| Fragment Offset | 13 | Position in reassembly |
| TTL | 8 | Hop limit |
| Protocol | 8 | 6=TCP, 17=UDP, 1=ICMP |
| Header Checksum | 16 | Header-only integrity |
| Source IP | 32 | Sender address |
| Destination IP | 32 | Receiver address |

### C Code Example

```c
/* TCP Client with 3-Way Handshake (POSIX) */
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int tcp_connect_send(const char *server_ip, uint16_t port,
                     const char *message) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return -1; }

    struct sockaddr_in addr;
    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(port);
    addr.sin_addr.s_addr = inet_addr(server_ip);

    /* TCP 3-way handshake happens here internally */
    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("connect"); close(sock); return -1;
    }
    printf("TCP connected to %s:%d\n", server_ip, port);

    /* Send data */
    send(sock, message, strlen(message), 0);

    /* Receive response */
    char buffer[4096];
    int n = recv(sock, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        buffer[n] = '\0';
        printf("Received: %s\n", buffer);
    }

    close(sock);  /* FIN handshake */
    return 0;
}

/* Raw TCP header structure for educational purposes */
typedef struct __attribute__((packed)) {
    uint16_t src_port;
    uint16_t dst_port;
    uint32_t seq_num;
    uint32_t ack_num;
    uint8_t  data_offset;  /* Upper 4 bits: header len in 32-bit words */
    uint8_t  flags;        /* URG|ACK|PSH|RST|SYN|FIN */
    uint16_t window;
    uint16_t checksum;
    uint16_t urgent_ptr;
} tcp_header_t;

#define TCP_FIN  0x01
#define TCP_SYN  0x02
#define TCP_RST  0x04
#define TCP_PSH  0x08
#define TCP_ACK  0x10
#define TCP_URG  0x20
```

---

## 5.2 DNS (Domain Name System)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** UDP port 53 (queries), TCP port 53 (zone transfers / large responses)
- **Resolution:** Recursive (resolver finds answer) or Iterative (step-by-step referrals)

### Block Diagram

```
 ┌──────────┐  Query   ┌───────────────┐
 │  Client  ├─────────►│ Recursive     │
 │ (Stub)   │◄─────────┤ Resolver      │
 └──────────┘  Answer  └───────┬───────┘
                               │ Iterative
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌──────────┐    ┌──────────┐     ┌─────────────┐
       │Root (.)  │───►│TLD (.com)│────►│Authoritative│
       │Name Srvr │    │Name Srvr │     │ Name Server │
       └──────────┘    └──────────┘     │example.com  │
                                        └─────────────┘
```

### DNS Message Format

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| **Header (12 bytes):** | | | |
| Transaction ID | 2 | 16 | Query/response matching |
| Flags | 2 | 16 | QR(1), Opcode(4), AA(1), TC(1), RD(1), RA(1), Z(3), RCODE(4) |
| QDCOUNT | 2 | 16 | Number of questions |
| ANCOUNT | 2 | 16 | Number of answers |
| NSCOUNT | 2 | 16 | Number of authority records |
| ARCOUNT | 2 | 16 | Number of additional records |
| **Question Section:** | | | |
| QNAME | Variable | — | Domain name (label-encoded) |
| QTYPE | 2 | 16 | A=1, AAAA=28, MX=15, CNAME=5 |
| QCLASS | 2 | 16 | IN=1 (Internet) |
| **Answer Section:** | | | |
| NAME | Variable | — | Domain (possibly compressed) |
| TYPE | 2 | 16 | Record type |
| CLASS | 2 | 16 | IN=1 |
| TTL | 4 | 32 | Cache duration (seconds) |
| RDLENGTH | 2 | 16 | RDATA length |
| RDATA | Variable | — | Record data (IP, CNAME, etc.) |

### C Code Example

```c
/* DNS Query Builder and Parser */
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <arpa/inet.h>

typedef struct __attribute__((packed)) {
    uint16_t id;
    uint16_t flags;
    uint16_t qdcount;
    uint16_t ancount;
    uint16_t nscount;
    uint16_t arcount;
} dns_header_t;

/* Encode domain name (e.g., "example.com" → "\x07example\x03com\x00") */
int dns_encode_name(uint8_t *buf, const char *domain) {
    int offset = 0;
    char copy[256];
    strncpy(copy, domain, sizeof(copy) - 1);

    char *label = strtok(copy, ".");
    while (label) {
        uint8_t len = strlen(label);
        buf[offset++] = len;
        memcpy(&buf[offset], label, len);
        offset += len;
        label = strtok(NULL, ".");
    }
    buf[offset++] = 0x00;  /* Root terminator */
    return offset;
}

/* Build DNS A-record query */
int dns_build_query(uint8_t *buffer, const char *domain, uint16_t txid) {
    int offset = 0;

    /* Header */
    dns_header_t *hdr = (dns_header_t *)buffer;
    hdr->id      = htons(txid);
    hdr->flags   = htons(0x0100);  /* RD=1 (Recursion Desired) */
    hdr->qdcount = htons(1);
    hdr->ancount = 0;
    hdr->nscount = 0;
    hdr->arcount = 0;
    offset += sizeof(dns_header_t);

    /* Question: QNAME */
    offset += dns_encode_name(&buffer[offset], domain);

    /* QTYPE = A (1) */
    buffer[offset++] = 0x00;
    buffer[offset++] = 0x01;

    /* QCLASS = IN (1) */
    buffer[offset++] = 0x00;
    buffer[offset++] = 0x01;

    return offset;
}

/* Parse A-record from DNS response */
void dns_parse_a_record(const uint8_t *response, int len) {
    const dns_header_t *hdr = (const dns_header_t *)response;
    int answers = ntohs(hdr->ancount);
    printf("DNS Response: %d answer(s)\n", answers);

    /* Skip header + question section for parsing answers */
    /* (Simplified: real parser must handle name compression) */
}
```

---

## 5.3 DHCP (Dynamic Host Configuration Protocol)

### Overview

- **OSI Layer:** Layer 7 (Application) over UDP
- **Ports:** Server = 67, Client = 68
- **Process:** DORA — Discover → Offer → Request → Acknowledge

### Block Diagram

```
 ┌──────────┐                         ┌──────────────┐
 │  Client  │  DHCPDISCOVER (bcast)   │  DHCP Server │
 │ (New PC) ├────────────────────────►│              │
 │          │  DHCPOFFER (IP offer)   │  IP Pool:    │
 │          │◄────────────────────────┤  .100-.200   │
 │          │  DHCPREQUEST (bcast)    │              │
 │          ├────────────────────────►│              │
 │          │  DHCPACK (confirmed)    │              │
 │          │◄────────────────────────┤              │
 │ IP: .150 │                         └──────────────┘
 └──────────┘
```

### DHCP Message Format

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Op | 1 | 8 | 1=BOOTREQUEST, 2=BOOTREPLY |
| HType | 1 | 8 | 1 = Ethernet |
| HLen | 1 | 8 | 6 (MAC address length) |
| Hops | 1 | 8 | Relay agent hop count |
| XID | 4 | 32 | Transaction ID |
| Secs | 2 | 16 | Seconds since client started |
| Flags | 2 | 16 | Bit 0 = Broadcast flag |
| CIAddr | 4 | 32 | Client IP (if already known) |
| YIAddr | 4 | 32 | "Your" IP (offered by server) |
| SIAddr | 4 | 32 | Server IP |
| GIAddr | 4 | 32 | Relay agent IP |
| CHAddr | 16 | 128 | Client hardware address (MAC) |
| SName | 64 | 512 | Server hostname |
| File | 128 | 1024 | Boot filename |
| Options | Variable | — | DHCP options (magic cookie 0x63825363) |
| **Fixed portion** | **236** | **1888** | **Before options** |

### C Code Example

```c
/* DHCP Discover Packet Builder */
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#define DHCP_MAGIC_COOKIE  0x63825363
#define DHCP_OPT_MSG_TYPE  53
#define DHCP_DISCOVER      1
#define DHCP_REQUEST       3

typedef struct __attribute__((packed)) {
    uint8_t  op;
    uint8_t  htype;
    uint8_t  hlen;
    uint8_t  hops;
    uint32_t xid;
    uint16_t secs;
    uint16_t flags;
    uint32_t ciaddr;
    uint32_t yiaddr;
    uint32_t siaddr;
    uint32_t giaddr;
    uint8_t  chaddr[16];
    uint8_t  sname[64];
    uint8_t  file[128];
    uint32_t magic_cookie;
    uint8_t  options[312];
} dhcp_packet_t;

/* Build DHCP Discover */
int dhcp_build_discover(dhcp_packet_t *pkt, const uint8_t *mac) {
    memset(pkt, 0, sizeof(*pkt));

    pkt->op    = 1;             /* BOOTREQUEST */
    pkt->htype = 1;             /* Ethernet */
    pkt->hlen  = 6;             /* MAC length */
    pkt->xid   = htonl(rand());
    pkt->flags = htons(0x8000); /* Broadcast */
    memcpy(pkt->chaddr, mac, 6);
    pkt->magic_cookie = htonl(DHCP_MAGIC_COOKIE);

    /* Option 53: DHCP Message Type = Discover */
    int opt = 0;
    pkt->options[opt++] = DHCP_OPT_MSG_TYPE;
    pkt->options[opt++] = 1;           /* Length */
    pkt->options[opt++] = DHCP_DISCOVER;

    /* Option 55: Parameter Request List */
    pkt->options[opt++] = 55;
    pkt->options[opt++] = 4;     /* Length */
    pkt->options[opt++] = 1;     /* Subnet Mask */
    pkt->options[opt++] = 3;     /* Router */
    pkt->options[opt++] = 6;     /* DNS Server */
    pkt->options[opt++] = 15;    /* Domain Name */

    /* End option */
    pkt->options[opt++] = 255;

    return sizeof(*pkt) - 312 + opt;
}
```

---

## 5.4 BGP (Border Gateway Protocol)

### Overview

- **OSI Layer:** Layer 7 (over TCP port 179)
- **Type:** Path-vector routing protocol
- **Purpose:** Inter-AS (Autonomous System) routing
- **Decision:** Weight → Local Pref → AS_PATH length → MED

### Block Diagram

```
  ┌─────────────┐    eBGP    ┌─────────────┐
  │   AS 64500  │◄══════════►│   AS 64501  │
  │   (ISP-A)   │ TCP:179    │   (ISP-B)   │
  │  ┌───────┐  │            │  ┌───────┐  │
  │  │BGP    │  │            │  │BGP    │  │
  │  │Speaker│  │            │  │Speaker│  │
  │  └───┬───┘  │            │  └───────┘  │
  │      │iBGP  │            │             │
  │  ┌───┴───┐  │            │             │
  │  │BGP    │  │            │             │
  │  │Peer   │  │            │             │
  │  └───────┘  │            │             │
  └─────────────┘            └─────────────┘
```

### BGP Message Header

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Marker | 16 | 128 | All 1s (0xFF × 16) for authentication |
| Length | 2 | 16 | Total message length (19–4096) |
| Type | 1 | 8 | 1=OPEN, 2=UPDATE, 3=NOTIFICATION, 4=KEEPALIVE |

### BGP UPDATE Message (after header)

| Field | Bytes | Description |
|-------|-------|-------------|
| Withdrawn Routes Length | 2 | Length of withdrawn routes |
| Withdrawn Routes | Variable | Prefixes being removed |
| Path Attribute Length | 2 | Length of path attributes |
| Path Attributes | Variable | ORIGIN, AS_PATH, NEXT_HOP, etc. |
| NLRI (Network Layer Reachability) | Variable | Prefix + length being advertised |

### C Code Example

```c
/* BGP OPEN Message Builder */
#include <stdint.h>
#include <string.h>
#include <arpa/inet.h>

#define BGP_TYPE_OPEN         1
#define BGP_TYPE_UPDATE       2
#define BGP_TYPE_NOTIFICATION 3
#define BGP_TYPE_KEEPALIVE    4

typedef struct __attribute__((packed)) {
    uint8_t  marker[16];
    uint16_t length;
    uint8_t  type;
} bgp_header_t;

typedef struct __attribute__((packed)) {
    bgp_header_t header;
    uint8_t  version;       /* BGP version 4 */
    uint16_t my_as;         /* Local AS number */
    uint16_t hold_time;     /* Hold timer (seconds) */
    uint32_t bgp_id;        /* Router ID (IP format) */
    uint8_t  opt_param_len; /* Optional parameters length */
} bgp_open_t;

/* Build BGP OPEN message */
int bgp_build_open(uint8_t *buffer, uint16_t as_number,
                   uint32_t router_id, uint16_t hold_time) {
    bgp_open_t *open = (bgp_open_t *)buffer;

    /* Header */
    memset(open->header.marker, 0xFF, 16);
    open->header.type   = BGP_TYPE_OPEN;

    /* OPEN fields */
    open->version       = 4;
    open->my_as         = htons(as_number);
    open->hold_time     = htons(hold_time);
    open->bgp_id        = htonl(router_id);
    open->opt_param_len = 0;

    int total = sizeof(bgp_open_t);
    open->header.length = htons(total);

    return total;
}

/* Build BGP KEEPALIVE (19 bytes: header only) */
int bgp_build_keepalive(uint8_t *buffer) {
    bgp_header_t *hdr = (bgp_header_t *)buffer;
    memset(hdr->marker, 0xFF, 16);
    hdr->length = htons(19);
    hdr->type   = BGP_TYPE_KEEPALIVE;
    return 19;
}
```

---

## 5.5 SNMP (Simple Network Management Protocol)

### Overview

- **OSI Layer:** Layer 7 (Application) over UDP
- **Ports:** Agent = 161, Traps = 162
- **Components:** Manager, Agent, MIB (Management Information Base)
- **Operations:** GET, SET, GETNEXT, GETBULK, TRAP/INFORM

### Block Diagram

```
 ┌──────────────────┐         ┌──────────────────┐
 │  SNMP Manager    │  GET    │  Network Device  │
 │  (NMS Software)  ├────────►│  (SNMP Agent)    │
 │                  │◄────────┤                  │
 │  Polls devices   │Response │  ┌────────────┐  │
 │  Port 162 ◄──────┼─ TRAP ──┤  │    MIB     │  │
 │  (receives traps)│         │  │(OID Tree)  │  │
 └──────────────────┘         │  └────────────┘  │
                              └──────────────────┘
```

### SNMPv2c Message (BER/ASN.1 Encoded)

| Field | Description |
|-------|-------------|
| Sequence (TLV) | Wraps entire SNMP message |
| → Version | Integer: 0=v1, 1=v2c, 3=v3 |
| → Community | OctetString: "public" / "private" |
| → PDU Type | GetRequest=0xA0, GetResponse=0xA2, SetRequest=0xA3, Trap=0xA7 |
| → Request ID | Integer: matching ID |
| → Error Status | Integer: 0=noError |
| → Error Index | Integer: 0 |
| → Varbind List | Sequence of OID-Value pairs |

### C Code Example

```c
/* SNMP GET Request Builder (SNMPv2c, simplified BER) */
#include <stdint.h>
#include <string.h>

#define SNMP_VERSION_2C     1
#define SNMP_GET_REQUEST    0xA0
#define SNMP_GET_RESPONSE   0xA2
#define SNMP_SET_REQUEST    0xA3

/* ASN.1 BER: encode length */
int asn1_encode_length(uint8_t *buf, int length) {
    if (length < 128) {
        buf[0] = (uint8_t)length;
        return 1;
    } else {
        buf[0] = 0x81;
        buf[1] = (uint8_t)length;
        return 2;
    }
}

/* ASN.1 BER: encode integer */
int asn1_encode_int(uint8_t *buf, int32_t value) {
    int offset = 0;
    buf[offset++] = 0x02;  /* INTEGER tag */
    if (value <= 127 && value >= -128) {
        buf[offset++] = 1;
        buf[offset++] = (uint8_t)(value & 0xFF);
    } else {
        buf[offset++] = 4;
        buf[offset++] = (value >> 24) & 0xFF;
        buf[offset++] = (value >> 16) & 0xFF;
        buf[offset++] = (value >> 8)  & 0xFF;
        buf[offset++] = value & 0xFF;
    }
    return offset;
}

/* Encode OID (e.g., 1.3.6.1.2.1.1.1.0 = sysDescr) */
int asn1_encode_oid(uint8_t *buf, const uint8_t *oid, int oid_len) {
    int offset = 0;
    buf[offset++] = 0x06;  /* OID tag */
    buf[offset++] = (uint8_t)oid_len;
    memcpy(&buf[offset], oid, oid_len);
    offset += oid_len;
    return offset;
}

/* Build SNMP GET request for sysDescr (1.3.6.1.2.1.1.1.0) */
int snmp_build_get(uint8_t *buffer, const char *community, int32_t req_id) {
    /* sysDescr OID: 1.3.6.1.2.1.1.1.0 */
    uint8_t sys_descr_oid[] = {0x2B, 0x06, 0x01, 0x02, 0x01, 0x01, 0x01, 0x00};

    /* This is a simplified builder — production code uses a full BER library */
    int offset = 0;

    /* Outer SEQUENCE */
    buffer[offset++] = 0x30; /* SEQUENCE */
    int len_pos = offset;
    offset++;  /* Placeholder for length */

    /* Version */
    offset += asn1_encode_int(&buffer[offset], SNMP_VERSION_2C);

    /* Community string */
    buffer[offset++] = 0x04; /* OCTET STRING */
    uint8_t comm_len = strlen(community);
    buffer[offset++] = comm_len;
    memcpy(&buffer[offset], community, comm_len);
    offset += comm_len;

    /* GetRequest PDU */
    buffer[offset++] = SNMP_GET_REQUEST;
    int pdu_len_pos = offset;
    offset++;

    offset += asn1_encode_int(&buffer[offset], req_id);
    offset += asn1_encode_int(&buffer[offset], 0); /* Error Status */
    offset += asn1_encode_int(&buffer[offset], 0); /* Error Index */

    /* Varbind list */
    buffer[offset++] = 0x30;  /* SEQUENCE */
    int vbl_len_pos = offset;
    offset++;

    /* Single Varbind */
    buffer[offset++] = 0x30;
    int vb_len_pos = offset;
    offset++;

    offset += asn1_encode_oid(&buffer[offset], sys_descr_oid, sizeof(sys_descr_oid));
    buffer[offset++] = 0x05; buffer[offset++] = 0x00; /* NULL value */

    buffer[vb_len_pos]  = offset - vb_len_pos - 1;
    buffer[vbl_len_pos] = offset - vbl_len_pos - 1;
    buffer[pdu_len_pos] = offset - pdu_len_pos - 1;
    buffer[len_pos]     = offset - len_pos - 1;

    return offset;
}
```

---

# Module 6: File Transfer & Storage Protocols

---

## 6.1 FTP / SFTP

### Overview

- **FTP:** Dual-channel (control on port 21, data on dynamic port), unencrypted
- **SFTP:** Single encrypted channel over SSH (port 22)
- **Modes:** Active (server connects back) vs Passive (client connects)

### Block Diagram

```
FTP:
 ┌──────────┐  Control (TCP 21)  ┌──────────┐
 │  Client  ├───────────────────►│  Server  │
 │          │  Data (dynamic)    │          │
 │          │◄══════════════════►│          │
 └──────────┘                    └──────────┘

SFTP:
 ┌──────────┐  SSH Tunnel (TCP 22) ┌──────────┐
 │  Client  │═════════════════════►│  Server  │
 │          │  All commands +      │          │
 │          │  data encrypted      │          │
 └──────────┘                      └──────────┘
```

### FTP Command/Response Format

| Component | Format | Example |
|-----------|--------|---------|
| Command | `VERB argument\r\n` | `RETR file.txt\r\n` |
| Response | `code message\r\n` | `226 Transfer complete\r\n` |
| Common Codes | 150, 200, 226, 230, 331, 425, 530 | |

### C Code Example

```c
/* FTP Client — Login and Retrieve File */
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

/* Send FTP command and read response */
int ftp_command(int sock, const char *cmd, char *resp, int max) {
    if (cmd) {
        send(sock, cmd, strlen(cmd), 0);
    }
    int n = recv(sock, resp, max - 1, 0);
    if (n > 0) resp[n] = '\0';
    return atoi(resp);  /* Return status code */
}

/* FTP session example */
int ftp_download(const char *server, const char *user,
                 const char *pass, const char *filename) {
    char resp[1024], cmd[256];

    /* Connect control channel */
    int ctrl = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    addr.sin_family      = AF_INET;
    addr.sin_port        = htons(21);
    addr.sin_addr.s_addr = inet_addr(server);
    connect(ctrl, (struct sockaddr *)&addr, sizeof(addr));

    ftp_command(ctrl, NULL, resp, sizeof(resp));   /* 220 Welcome */

    snprintf(cmd, sizeof(cmd), "USER %s\r\n", user);
    ftp_command(ctrl, cmd, resp, sizeof(resp));     /* 331 */

    snprintf(cmd, sizeof(cmd), "PASS %s\r\n", pass);
    ftp_command(ctrl, cmd, resp, sizeof(resp));     /* 230 */

    /* Enter passive mode */
    ftp_command(ctrl, "PASV\r\n", resp, sizeof(resp));
    /* Parse IP and port from response for data connection */

    /* Request file */
    snprintf(cmd, sizeof(cmd), "RETR %s\r\n", filename);
    ftp_command(ctrl, cmd, resp, sizeof(resp));     /* 150 */

    /* Data would be read from the passive data connection */

    ftp_command(ctrl, "QUIT\r\n", resp, sizeof(resp));
    close(ctrl);
    return 0;
}
```

---

## 6.2 NFS (Network File System)

### Overview

- **OSI Layer:** Layer 7 (Application) over RPC
- **NFSv3:** Stateless, over UDP/TCP
- **NFSv4:** Stateful, TCP port 2049, Kerberos security

### Block Diagram

```
 ┌──────────────┐                  ┌──────────────┐
 │  NFS Client  │  MOUNT + RPC     │  NFS Server  │
 │              ├─────────────────►│              │
 │ /mnt/share ──┼──  NFS ops ─────►│ /export/data │
 │ (local mount)│◄─────────────────┤              │
 │              │  File data       │              │
 └──────────────┘                  └──────────────┘
```

### NFSv4 Compound Request

| Field | Description |
|-------|-------------|
| RPC Header | XID, Msg Type, RPC Version, Program, Procedure |
| NFS Minor Version | 0 or 1 |
| Operation Array | PUTROOTFH, LOOKUP, OPEN, READ, WRITE, CLOSE |
| Each Op | Opcode + arguments specific to operation |

### C Code Example

```c
/* NFS Mount and Read (using POSIX — kernel handles NFS) */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mount.h>

/* Mount NFS share (requires root) */
int nfs_mount_share(const char *server_path, const char *mount_point) {
    /* mount -t nfs server:/export /mnt/nfs */
    int ret = mount(server_path, mount_point, "nfs",
                    MS_RDONLY, "vers=4,proto=tcp");
    if (ret != 0) {
        perror("mount");
        return -1;
    }
    printf("NFS mounted: %s -> %s\n", server_path, mount_point);
    return 0;
}

/* Read file from NFS mount (transparent to application) */
int nfs_read_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) { perror("open"); return -1; }

    char buffer[4096];
    ssize_t n;
    while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
        write(STDOUT_FILENO, buffer, n);
    }
    close(fd);
    return 0;
}
```

---

## 6.3 SMB (Server Message Block)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP port 445 (direct), NetBIOS over TCP port 139 (legacy)
- **Versions:** SMB 1.0/CIFS (legacy), SMB 2.x, SMB 3.x (AES encryption)

### Block Diagram

```
 ┌──────────────┐                     ┌──────────────┐
 │  SMB Client  │  TCP Port 445       │  SMB Server  │
 │  (Windows)   ├────────────────────►│  (File Share)│
 │              │  Negotiate → Setup  │              │
 │              │  → Tree Connect     │  \\srv\share │
 │  \\srv\share │◄════════════════════│              │
 │              │  Read/Write files   │              │
 └──────────────┘                     └──────────────┘
```

### SMB2 Header

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Protocol ID | 4 | 32 | 0xFE 'S' 'M' 'B' |
| Structure Size | 2 | 16 | Always 64 |
| Credit Charge | 2 | 16 | Credits consumed |
| Status | 4 | 32 | NT status code |
| Command | 2 | 16 | NEGOTIATE=0, SESSION_SETUP=1, TREE_CONNECT=3, CREATE=5, READ=8, WRITE=9 |
| Credits | 2 | 16 | Credits requested/granted |
| Flags | 4 | 32 | Direction, signing, etc. |
| Next Command | 4 | 32 | Compound request offset |
| Message ID | 8 | 64 | Unique request identifier |
| Tree ID | 4 | 32 | Share connection handle |
| Session ID | 8 | 64 | Authentication session |
| Signature | 16 | 128 | Message authentication |
| **Total** | **64** | **512** | **Fixed header** |

### C Code Example

```c
/* SMB2 Negotiate Request (Conceptual) */
#include <stdint.h>
#include <string.h>

#define SMB2_MAGIC          0x424D53FE  /* 0xFE 'S' 'M' 'B' */
#define SMB2_CMD_NEGOTIATE  0x0000
#define SMB2_DIALECT_300    0x0300
#define SMB2_DIALECT_311    0x0311

typedef struct __attribute__((packed)) {
    uint32_t protocol_id;
    uint16_t structure_size;
    uint16_t credit_charge;
    uint32_t status;
    uint16_t command;
    uint16_t credits;
    uint32_t flags;
    uint32_t next_command;
    uint64_t message_id;
    uint32_t reserved;
    uint32_t tree_id;
    uint64_t session_id;
    uint8_t  signature[16];
} smb2_header_t;

typedef struct __attribute__((packed)) {
    uint16_t structure_size;   /* 36 */
    uint16_t dialect_count;
    uint16_t security_mode;    /* Signing enabled/required */
    uint16_t reserved;
    uint32_t capabilities;
    uint8_t  client_guid[16];
    uint64_t client_start_time;
    uint16_t dialects[];       /* Supported dialect versions */
} smb2_negotiate_req_t;

/* Build SMB2 Negotiate request */
int smb2_build_negotiate(uint8_t *buffer) {
    smb2_header_t *hdr = (smb2_header_t *)buffer;
    memset(hdr, 0, sizeof(*hdr));
    hdr->protocol_id    = SMB2_MAGIC;
    hdr->structure_size = 64;
    hdr->command        = SMB2_CMD_NEGOTIATE;
    hdr->credits        = 1;
    hdr->message_id     = 0;

    smb2_negotiate_req_t *neg = (smb2_negotiate_req_t *)(buffer + 64);
    neg->structure_size = 36;
    neg->dialect_count  = 2;
    neg->security_mode  = 0x01;  /* Signing enabled */
    neg->dialects[0]    = SMB2_DIALECT_300;
    neg->dialects[1]    = SMB2_DIALECT_311;

    return 64 + 36 + (2 * sizeof(uint16_t));
}
```

---

## 6.4 iSCSI

### Overview

- **OSI Layer:** Layer 4/7 (SCSI commands over TCP)
- **Transport:** TCP port 3260
- **Level:** Block-level storage (client mounts as raw disk)
- **Components:** Initiator (client) ↔ Target (storage)

### Block Diagram

```
 ┌───────────────┐                     ┌───────────────┐
 │  iSCSI        │  TCP Port 3260      │  iSCSI Target │
 │  Initiator    ├────────────────────►│  (Storage)    │
 │               │  SCSI Commands      │               │
 │  OS sees a    │  encapsulated in    │  LUN 0: 500GB │
 │  local disk   │  TCP/IP             │  LUN 1: 1TB   │
 │  /dev/sdb     │◄════════════════════│               │
 └───────────────┘  Block data         └───────────────┘
```

### iSCSI PDU (Protocol Data Unit)

| Field | Bytes | Bits | Description |
|-------|-------|------|-------------|
| Opcode | 1 | 8 | 0x01=SCSI Command, 0x05=SCSI Data-Out, 0x25=SCSI Response |
| Flags | 1 | 8 | Final, Read/Write, Attr bits |
| Reserved / AHS Length | 2 | 16 | Additional header segments |
| Data Segment Length | 3 | 24 | Payload length |
| LUN | 8 | 64 | Logical Unit Number |
| Initiator Task Tag | 4 | 32 | Command identifier |
| Expected Data Length | 4 | 32 | Data transfer size |
| CmdSN | 4 | 32 | Command sequence number |
| ExpStatSN | 4 | 32 | Expected status sequence |
| CDB | 16 | 128 | SCSI Command Descriptor Block |
| **BHS Total** | **48** | **384** | **Basic Header Segment** |

### C Code Example

```c
/* iSCSI Login Request Builder (Conceptual) */
#include <stdint.h>
#include <string.h>

#define ISCSI_OP_LOGIN_REQ    0x03
#define ISCSI_OP_SCSI_CMD     0x01
#define ISCSI_OP_SCSI_RSP     0x21

typedef struct __attribute__((packed)) {
    uint8_t  opcode;
    uint8_t  flags;
    uint8_t  version_max;
    uint8_t  version_min;
    uint8_t  total_ahs_len;
    uint8_t  data_seg_len[3];
    uint8_t  isid[6];          /* Initiator Session ID */
    uint16_t tsih;             /* Target Session ID Handle */
    uint32_t init_task_tag;
    uint16_t cid;              /* Connection ID */
    uint16_t reserved;
    uint32_t cmd_sn;
    uint32_t exp_stat_sn;
} iscsi_login_req_t;

/* Build iSCSI Login Request */
int iscsi_build_login(uint8_t *buffer, uint32_t task_tag) {
    iscsi_login_req_t *req = (iscsi_login_req_t *)buffer;
    memset(req, 0, sizeof(*req));

    req->opcode      = ISCSI_OP_LOGIN_REQ | 0x40; /* Immediate */
    req->flags       = 0x87;  /* Transit + CSG=Login + NSG=FullFeature */
    req->version_max = 0x00;
    req->version_min = 0x00;
    req->init_task_tag = htonl(task_tag);
    req->cmd_sn      = htonl(1);

    /* ISID: random session identifier */
    req->isid[0] = 0x40;  /* Type=OUI */
    req->isid[1] = 0x00;
    req->isid[2] = 0x00;
    req->isid[3] = 0x01;

    /* Key-value pairs in data segment (e.g., "InitiatorName=...") */
    const char *kv = "InitiatorName=iqn.2024-01.com.example:client\0"
                     "TargetName=iqn.2024-01.com.example:storage\0";
    int kv_len = 88;  /* Includes null terminators */

    req->data_seg_len[0] = 0;
    req->data_seg_len[1] = (kv_len >> 8) & 0xFF;
    req->data_seg_len[2] = kv_len & 0xFF;

    memcpy(buffer + sizeof(*req), kv, kv_len);

    return sizeof(*req) + kv_len;
}
```

---

# Module 7: Communication & Messaging Protocols

---

## 7.1 Email Protocols (SMTP / IMAP / POP3)

### Overview

- **SMTP:** Push protocol for sending mail (port 25/587)
- **POP3:** Download-and-delete retrieval (port 110/995)
- **IMAP:** Server-side synchronization (port 143/993)

### Block Diagram

```
 ┌──────────┐  SMTP (587)  ┌──────────────┐  SMTP (25)   ┌──────────────┐
 │  Sender  ├─────────────►│ Sender's MTA ├─────────────►│Receiver's MTA│
 │  (MUA)   │              │ (Mail Server)│              │ (Mail Server)│
 └──────────┘              └──────────────┘              └──────┬───────┘
                                                                │
                           ┌────────────────────────────────────┘
                           │
 ┌──────────┐  IMAP (993)  │  ┌──────────────┐
 │ Receiver ├──────────────┼─►│  Mailbox     │
 │  (MUA)   │              │  │  (Server)    │
 │  Phone   │  POP3 (995)  │  │  INBOX/      │
 │  Laptop  ├──────────────┘  │  Sent/       │
 └──────────┘                 └──────────────┘
```

### SMTP Transaction Flow

| Step | Command | Response | Description |
|------|---------|----------|-------------|
| 1 | `EHLO client.com` | `250 OK` | Handshake |
| 2 | `MAIL FROM:<a@b.com>` | `250 OK` | Envelope sender |
| 3 | `RCPT TO:<c@d.com>` | `250 OK` | Envelope recipient |
| 4 | `DATA` | `354 Start` | Begin message body |
| 5 | (headers + body + `.`) | `250 OK` | Message delivered |
| 6 | `QUIT` | `221 Bye` | Close connection |

### C Code Example

```c
/* Minimal SMTP Client */
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int smtp_send(int sock, const char *cmd, char *resp, int max) {
    if (cmd) send(sock, cmd, strlen(cmd), 0);
    int n = recv(sock, resp, max - 1, 0);
    if (n > 0) resp[n] = '\0';
    return atoi(resp);
}

int send_email(const char *server, const char *from,
               const char *to, const char *subject,
               const char *body) {
    char resp[1024], cmd[1024];

    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(25),
    };
    addr.sin_addr.s_addr = inet_addr(server);
    connect(sock, (struct sockaddr *)&addr, sizeof(addr));

    smtp_send(sock, NULL, resp, sizeof(resp));                    /* 220 */
    smtp_send(sock, "EHLO client.local\r\n", resp, sizeof(resp));/* 250 */

    snprintf(cmd, sizeof(cmd), "MAIL FROM:<%s>\r\n", from);
    smtp_send(sock, cmd, resp, sizeof(resp));                     /* 250 */

    snprintf(cmd, sizeof(cmd), "RCPT TO:<%s>\r\n", to);
    smtp_send(sock, cmd, resp, sizeof(resp));                     /* 250 */

    smtp_send(sock, "DATA\r\n", resp, sizeof(resp));              /* 354 */

    snprintf(cmd, sizeof(cmd),
             "From: %s\r\nTo: %s\r\nSubject: %s\r\n\r\n%s\r\n.\r\n",
             from, to, subject, body);
    smtp_send(sock, cmd, resp, sizeof(resp));                     /* 250 */

    smtp_send(sock, "QUIT\r\n", resp, sizeof(resp));
    close(sock);
    return 0;
}
```

---

## 7.2 SIP (Session Initiation Protocol)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP/UDP port 5060 (SIP), UDP (RTP for media)
- **Purpose:** Signaling only — call setup, modification, teardown
- **Media:** Negotiated via SDP, carried by RTP

### Block Diagram

```
 ┌───────────┐                                    ┌───────────┐
 │  SIP UA   │  INVITE (SDP offer)                │  SIP UA   │
 │  (Caller) ├───────────────────────────────────►│  (Callee) │
 │           │  100 Trying                        │           │
 │           │◄───────────────────────────────────┤           │
 │           │  180 Ringing                       │           │
 │           │◄───────────────────────────────────┤           │
 │           │  200 OK (SDP answer)               │           │
 │           │◄───────────────────────────────────┤           │
 │           │  ACK                               │           │
 │           ├───────────────────────────────────►│           │
 │           │◄══════════════════════════════════►│           │
 │           │     RTP Media (Voice/Video)        │           │
 │           │  BYE                               │           │
 │           ├───────────────────────────────────►│           │
 │           │  200 OK                            │           │
 │           │◄───────────────────────────────────┤           │
 └───────────┘                                    └───────────┘
```

### SIP Message Format

```
INVITE sip:bob@example.com SIP/2.0
Via: SIP/2.0/UDP 192.168.1.10:5060
From: <sip:alice@example.com>;tag=1234
To: <sip:bob@example.com>
Call-ID: abc123@192.168.1.10
CSeq: 1 INVITE
Contact: <sip:alice@192.168.1.10>
Content-Type: application/sdp
Content-Length: ...

(SDP body follows)
```

### C Code Example

```c
/* SIP INVITE Message Builder */
#include <stdio.h>
#include <string.h>

/* Build SIP INVITE request with SDP */
int sip_build_invite(char *buffer, int max_len,
                     const char *from_uri, const char *to_uri,
                     const char *from_ip, uint16_t rtp_port,
                     const char *call_id) {
    /* SDP body */
    char sdp[512];
    int sdp_len = snprintf(sdp, sizeof(sdp),
        "v=0\r\n"
        "o=- 0 0 IN IP4 %s\r\n"
        "s=Call\r\n"
        "c=IN IP4 %s\r\n"
        "t=0 0\r\n"
        "m=audio %d RTP/AVP 0 8 101\r\n"
        "a=rtpmap:0 PCMU/8000\r\n"
        "a=rtpmap:8 PCMA/8000\r\n"
        "a=rtpmap:101 telephone-event/8000\r\n",
        from_ip, from_ip, rtp_port);

    /* SIP headers + SDP */
    int len = snprintf(buffer, max_len,
        "INVITE sip:%s SIP/2.0\r\n"
        "Via: SIP/2.0/UDP %s:5060;branch=z9hG4bK776asdhds\r\n"
        "Max-Forwards: 70\r\n"
        "From: <sip:%s>;tag=1928301774\r\n"
        "To: <sip:%s>\r\n"
        "Call-ID: %s\r\n"
        "CSeq: 1 INVITE\r\n"
        "Contact: <sip:%s@%s:5060>\r\n"
        "Content-Type: application/sdp\r\n"
        "Content-Length: %d\r\n"
        "\r\n"
        "%s",
        to_uri, from_ip, from_uri, to_uri,
        call_id, from_uri, from_ip, sdp_len, sdp);

    return len;
}
```

---

## 7.3 XMPP (Extensible Messaging and Presence Protocol)

### Overview

- **OSI Layer:** Layer 7 (Application)
- **Transport:** TCP port 5222 (client-server), 5269 (server-server)
- **Format:** XML stream with three stanza types
- **Addressing:** user@domain.com/resource (JID)

### Block Diagram

```
 ┌───────────┐  TCP 5222  ┌───────────────┐  TCP 5269  ┌───────────────┐
 │  Client A │◄══════════►│   Server A    │◄══════════►│   Server B    │
 │alice@a.com│            │   (a.com)     │            │   (b.com)     │
 └───────────┘            └───────────────┘            └───────┬───────┘
                                                               │
                                                       ┌───────┴───────┐
                                                       │   Client B    │
                                                       │  bob@b.com    │
                                                       └───────────────┘
 Stanzas: <message>, <presence>, <iq>
```

### XMPP Stanza Types

| Stanza | Purpose | Example Attributes |
|--------|---------|--------------------|
| `<message>` | Chat, groupchat | to, from, type, id |
| `<presence>` | Online/away/DND status | type (subscribe, unavailable) |
| `<iq>` | Info/Query (get/set/result) | type (get, set, result, error) |

### C Code Example

```c
/* XMPP Message Stanza Builder */
#include <stdio.h>
#include <string.h>

/* Build XMPP <message> stanza */
int xmpp_build_message(char *buffer, int max_len,
                       const char *from_jid, const char *to_jid,
                       const char *body, const char *msg_id) {
    return snprintf(buffer, max_len,
        "<message from='%s' to='%s' type='chat' id='%s'>"
        "<body>%s</body>"
        "</message>",
        from_jid, to_jid, msg_id, body);
}

/* Build XMPP <presence> stanza */
int xmpp_build_presence(char *buffer, int max_len,
                        const char *show, const char *status) {
    if (show && status) {
        return snprintf(buffer, max_len,
            "<presence>"
            "<show>%s</show>"           /* chat, away, dnd, xa */
            "<status>%s</status>"
            "</presence>", show, status);
    }
    return snprintf(buffer, max_len, "<presence/>");
}

/* Build XMPP <iq> roster query */
int xmpp_build_roster_get(char *buffer, int max_len, const char *iq_id) {
    return snprintf(buffer, max_len,
        "<iq type='get' id='%s'>"
        "<query xmlns='jabber:iq:roster'/>"
        "</iq>", iq_id);
}

/* Build XMPP stream opening */
int xmpp_build_stream_open(char *buffer, int max_len,
                           const char *domain) {
    return snprintf(buffer, max_len,
        "<?xml version='1.0'?>"
        "<stream:stream to='%s' "
        "xmlns='jabber:client' "
        "xmlns:stream='http://etherx.jabber.org/streams' "
        "version='1.0'>", domain);
}
```

---

# Appendix: Protocol Quick-Reference Table

| Protocol | OSI Layer | Speed / Range | Topology | Transport |
|----------|-----------|---------------|----------|-----------|
| CAN | 1–2 | 1 Mbps | Bus | Differential serial |
| LIN | 1–2 | 20 kbps | Bus (master/slave) | Single wire |
| FlexRay | 1–2 | 10 Mbps/ch | Dual bus/star | Differential |
| MOST | 1–7 | 25–150 Mbps | Ring | Optical fiber |
| Auto Ethernet | 1–2 | 100M–1Gbps | Star/Switch | Twisted pair |
| Modbus | 7 | 115.2 kbps (RTU) | Bus | RS-485 / Ethernet |
| PROFINET | 2–7 | 100 Mbps | Star | Ethernet |
| EtherCAT | 2 | 100 Mbps | Line/Ring | Ethernet |
| EtherNet/IP | 7 | 100 Mbps–1 Gbps | Star | TCP/UDP |
| BACnet | 1–7 | Variable | Bus/Star | MS/TP, IP |
| MQTT | 7 | N/A | Client/Broker | TCP |
| CoAP | 7 | N/A | Client/Server | UDP |
| Zigbee | 3–7 | 250 kbps | Mesh | IEEE 802.15.4 |
| Z-Wave | 1–7 | 100 kbps | Mesh | Sub-GHz RF |
| LoRaWAN | MAC | 0.3–50 kbps | Star-of-stars | CSS (sub-GHz) |
| BLE | 1–7 | 1–2 Mbps | Point-to-point/Star | 2.4 GHz |
| HTTP/HTTPS | 7 | N/A | Client/Server | TCP (/ QUIC) |
| WebSocket | 7 | N/A | Persistent duplex | TCP |
| gRPC | 7 | N/A | Client/Server | HTTP/2 |
| GraphQL | 7 | N/A | Client/Server | HTTP |
| TCP/IP | 3–4 | N/A | End-to-end | TCP over IP |
| DNS | 7 | N/A | Hierarchical | UDP / TCP |
| DHCP | 7 | N/A | Client/Server | UDP |
| BGP | 7 | N/A | AS mesh | TCP |
| SNMP | 7 | N/A | Manager/Agent | UDP |
| FTP/SFTP | 7 | N/A | Client/Server | TCP / SSH |
| NFS | 7 | N/A | Client/Server | RPC / TCP |
| SMB | 7 | N/A | Client/Server | TCP |
| iSCSI | 4–7 | N/A | Initiator/Target | TCP |
| SMTP | 7 | N/A | Relay chain | TCP |
| SIP | 7 | N/A | Client/Server/Proxy | TCP / UDP |
| XMPP | 7 | N/A | Federated C/S | TCP |

---

*Document Version 1.0 — Generated September 2026*
*Covers 30 protocols across 7 domains with architecture diagrams, frame structures, and C code examples.*
