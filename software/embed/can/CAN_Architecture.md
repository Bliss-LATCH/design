# CAN Architecture Overview

**Project:** LATCH (Lightweight Actuated Tunnels for Crewed Habitation)
**Protocol Version:** 1.0 (2-4-5 Split)

This document defines the software-level protocol used on the LATCH CAN bus. While the standard CAN protocol defines *how* bits move on the wire, this architecture defines *what* those bits mean for our specific robot.

## The Challenge
We are operating on a **Standard 11-bit CAN 2.0A** network. We have 2048 available IDs (`0x000` to `0x7FF`). To ensure the Jetson (High-Level Controls) can effectively communicate with our distributed STM32 nodes (Low-Level Controls) without collision or confusion, we must strictly partition these bits.

## The "2-4-5" Architecture
For LATCH, we split the 11-bit Identifier into three distinct fields: **Priority**, **Source Device**, and **Message Type**.

### Bit Allocation Map
| Field | Bits | Bit Range | Purpose |
| :--- | :--- | :--- | :--- |
| **Priority** | 2 | `[10:9]` | Determines arbitration (Lower value = Higher Priority). |
| **Device ID** | 4 | `[8:5]` | Identifies the **SENDER** (Source) of the message. |
| **Message** | 5 | `[4:0]` | Identifies the command or data type (e.g., Set Position). |

---

## 1. Priority Field (Bits 10-9)
There are 4 priority levels. The CAN hardware arbitration ensures that lower numbers always win control of the bus.

| Value | Level | Description | Example Use Case |
| :--- | :--- | :--- | :--- |
| **0** | **CRITICAL** | Safety-critical commands that must be processed immediately. | E-Stop, Current Overload Protection. |
| **1** | **CONTROL** | High-speed, real-time control loop data. | "Set Position", "Get Encoder Value". |
| **2** | **STATUS** | Low-speed housekeeping data. | Battery Voltage, Board Temps. |
| **3** | **DEBUG** | Non-essential data. | Logging strings, Ping. |

## 2. Device ID Field (Bits 8-5)
This field identifies **WHO IS TALKING** (The Source).
*Note: We define the Source in the ID so the receiver always knows who sent the command.*

| ID | Device Name | Notes |
| :--- | :--- | :--- |
| `0x00` | **Broadcast / Reserved** | Reserved for global announcements. |
| `0x01` | **Jetson AGX** | Main Brain (High-Level Controls). |
| `0x02` | **STM32 Segment 1** | |
| `0x03` | **STM32 Segment 2** | |
| `0x04` | **STM32 Segment 3** | |
| `0x05` | **STM32 Segment 4** | |
| `0x06` | **STM32 Segment 5** | |
| `0x07` | **STM32 Segment 6** | |
| `0x08-0x0F` | *Reserved* | Available for future expansion. |

## 3. Message Type Field (Bits 4-0)
We have 32 available slots (`0x00` to `0x1F`) for specific commands.

* `0x00`: **E-STOP** (Must be Priority 0)
* `0x01`: **Heartbeat**
* `0x02`: **Set Position Target**
* `0x03`: **Get Current Position**
* `0x04`: **Set Velocity Target**
* *(Add more commands as the controls team defines them)*
