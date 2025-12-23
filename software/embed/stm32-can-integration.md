# What is CAN 

This document describes how to integrate **CAN (Controller Area Network)** into any of the STM32-based microcontroller nodes used in the LATCH system. These nodes act as distributed controllers for major subsystems such as actuation, sensing, power management, and diagnostics.

While the STM32 CAN peripheral is well documented, configuration details can be non-obvious when setting up a reliable multi-node network. The following blog post was a helpful reference during development:  
[Setting Up the CAN Bus on STM32](https://matthewtran.dev/2020/01/setting-up-the-can-bus-on-stm32)

However, this document is intended to be a **self-contained reference** specific to this project.

---

## Understanding CAN

Before diving into implementation details, it is important to understand the core concepts behind CAN.

**CAN (Controller Area Network)** is a robust, message-based communication protocol designed for distributed embedded systems. It uses a **multi-master architecture**, meaning there is no central master or slave device. Any node on the bus may transmit data when the bus is idle.

Instead of addressing messages to specific devices, CAN uses **message identifiers**. Each node decides which messages are relevant by filtering on these identifiers. This architecture makes CAN highly scalable and well-suited for fault-tolerant systems.

CAN also provides several hardware-level advantages:
- Differential signaling for noise immunity
- Automatic message arbitration
- Built-in error detection and handling
- Deterministic bus access

For a deeper protocol-level explanation, this video is a good resource:  
[CAN Bus Explained](https://www.youtube.com/watch?v=JZSCzRT9TTo)

---

## The CAN Bus

The **CAN bus** consists of two differential signal lines:
- **CAN_H**
- **CAN_L**

All nodes on the network connect in parallel to these two lines. The bus must be terminated with **120 Ω resistors** at both physical ends of the network to prevent signal reflections. Most CAN transceivers have these built in, so hopefully we wont have to worry about that.

Key points:
- The bus is linear (not star-topology)
- Stub lengths should be kept as short as possible
- All nodes share the same bus speed (bitrate)

---

## CAN Nodes

Each CAN node consists of:
1. A microcontroller with a CAN controller (e.g., STM32 FDCAN or bxCAN)
2. An external CAN transceiver (e.g., TJA1051, SN65HVD230)
3. Firmware responsible for transmitting and receiving CAN frames

The microcontroller handles framing, filtering, and error checking, while the transceiver converts logic-level signals to the differential CAN bus signals.

---

## The CAN Data Frame

Understanding CAN requires understanding the structure of the **data frame**, which is the fundamental unit of communication on the bus.

A standard CAN data frame consists of the following fields:

### 1. Start of Frame (SOF)
A single dominant bit that signals the beginning of a frame.

### 2. Arbitration Field
This field contains:
- **Identifier (11-bit or 29-bit)**  
- **RTR (Remote Transmission Request) bit**

The identifier defines the message type and its priority. Lower identifier values have **higher priority** during bus arbitration.

### 3. Control Field
Contains:
- Data Length Code (DLC), specifying 0–8 bytes of data
- Additional control bits depending on frame type

### 4. Data Field
The payload of the message (0–8 bytes for Classical CAN).

### 5. CRC Field
Used for error detection to ensure data integrity.

### 6. Acknowledge Field
Receiving nodes that correctly receive the frame assert a dominant bit to acknowledge receipt.

### 7. End of Frame (EOF)
Marks the end of the CAN frame.

---

## Message Arbitration

When multiple nodes attempt to transmit simultaneously, CAN uses **bitwise arbitration** on the identifier field.

- Dominant bits override recessive bits
- Nodes transmitting a recessive bit but seeing a dominant bit stop transmitting
- The node with the lowest identifier value wins arbitration without data loss

This ensures deterministic and collision-free bus access.

---

## Standard vs Extended Identifiers

CAN supports two identifier formats:
- **Standard ID**: 11-bit identifier
- **Extended ID**: 29-bit identifier

For this project:
- Standard IDs are preferred for simplicity and efficiency
- Extended IDs may be used when additional addressing or message categorization is required

---

## CAN Filtering

Each STM32 node configures **hardware acceptance filters** to determine which messages it receives.

Filtering allows:
- Reduced CPU load
- Isolation of subsystem-specific traffic
- Deterministic message handling

Filters can be configured to:
- Match exact identifiers
- Match ranges of identifiers
- Accept or reject extended IDs

---

## Error Handling and Fault Tolerance

CAN includes built-in error detection mechanisms:
- Bit monitoring
- Stuffing errors
- CRC errors
- Frame errors
- Acknowledgment errors

Nodes automatically track transmit and receive error counters. A node may enter:
- Error Active
- Error Passive
- Bus-Off state

Firmware should monitor these states and implement recovery logic when appropriate.

---

## Bitrate and Timing

All nodes must share the same CAN bitrate. Common bitrates include:
- 125 kbps
- 250 kbps
- 500 kbps
- 1 Mbps

Bit timing depends on:
- Peripheral clock frequency
- Prescaler value
- Time segments (BS1, BS2)
- Synchronization jump width (SJW)

These parameters must be consistent across all nodes.

---

## Next Steps

The following sections (to be added) will cover:
- STM32CubeMX CAN configuration
- HAL vs Low-Level driver usage
- Transmit and receive example code
- Recommended message ID map for LATCH
- Debugging and validation techniques

---

# CAN Integration for STM32CubeMX

## Enable the CAN Communication Protocol

For the version of **STM32CubeIDE** we are using *(2.0.0)*, the **STM32CubeMX** tool is not built in, so we will need to open the **IOC** for our project and enable the **CAN1** interface under the communication tab. 

## Setting the Baud Rate

Once **CAN1** is enabled, the next step is configuring the CAN bit timing to achieve the desired baud rate. In **STM32CubeMX**, the CAN baud rate is **not set directly** as a single value; instead, it is derived from several timing parameters that divide the CAN peripheral clock.

Navigate to:

**CAN1 → Parameter Settings → Bit Timing Parameters**

### CAN Bit Timing Overview

The CAN bit rate is defined by the following relationship:

![bitrate formula](/imgs/bitrate.png)

Where:
- ***f*<sub>Can** is the CAN peripheral clock frequency  
- **Prescaler** divides the CAN clock to generate the time quantum (TQ)  
- **BS1 (Time Segment 1)** determines the position of the sample point  
- **BS2 (Time Segment 2)** controls resynchronization timing  
- **SJW (Synchronization Jump Width)** specifies the maximum adjustment applied during resynchronization  

### Recommended Configuration

For reliable communication, the CAN sample point should typically fall between **75%–87.5%** of the bit time. A common and stable configuration is:

- **SJW** = 1 TQ  
- **BS1** = 13 TQ  
- **BS2** = 2 TQ  

The prescaler should then be adjusted based on the CAN clock frequency to achieve the desired baud rate (e.g., 500 kbps or 1 Mbps).

### Example (500 kbps)

Assuming a CAN peripheral clock of **48 MHz**:

- Prescaler = 6  
- BS1 = 13  
- BS2 = 2  

![bitrate example](/imgs/bitrate-example.png)

Once these parameters are set, save the IOC file and regenerate the project to apply the CAN configuration.


# Integrating CAN into Your STM32CubeIDE Project

This section walks through the **firmware-side integration** of CAN after it has been enabled and configured in **STM32CubeMX**. The goal is to bring the CAN peripheral online, transmit frames, and receive messages reliably across the LATCH network.

---

## Driver Selection: HAL vs Low-Level (LL)

STM32 provides two main APIs for CAN:
- **HAL (Hardware Abstraction Layer)**  
- **LL (Low-Level) drivers**

For this project, **HAL is recommended** because:
- It is easier to configure and maintain
- It integrates cleanly with CubeMX-generated code
- Performance is sufficient for LATCH bandwidth requirements

All examples below assume **HAL CAN (bxCAN or FDCAN)** usage.

---

## Initialization Flow

After code generation, CubeMX creates:
- `MX_CAN1_Init()` (or `MX_FDCAN1_Init()`)
- A global CAN handle (e.g., `CAN_HandleTypeDef hcan`)

### Required Startup Sequence

The following steps must occur **in order**:

1. Initialize the CAN peripheral
2. Configure acceptance filters
3. Start the CAN peripheral
4. Enable receive interrupts (if used)

This logic should be placed **after `HAL_Init()`** and **before entering the main loop**.

---

## Configuring CAN Filters

Filters determine which messages the node receives. Each node should only accept messages relevant to its subsystem.

### Example: Accept a Single Standard ID

```
CAN_FilterTypeDef filter = {0};

filter.FilterBank = 0;
filter.FilterMode = CAN_FILTERMODE_IDMASK;
filter.FilterScale = CAN_FILTERSCALE_32BIT;
filter.FilterIdHigh = (0x100 << 5);   // Standard ID << 5
filter.FilterIdLow = 0x0000;
filter.FilterMaskIdHigh = (0x7FF << 5); // Match all 11 bits
filter.FilterMaskIdLow = 0x0000;
filter.FilterFIFOAssignment = CAN_FILTER_FIFO0;
filter.FilterActivation = ENABLE;

HAL_CAN_ConfigFilter(&hcan, &filter);
```



