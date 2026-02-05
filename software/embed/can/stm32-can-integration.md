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

Here is also a good write up on the lower-level deep dive into the CAN protocol:  
[Understanding the CAN Bus](https://www.wevolver.com/article/understanding-can-bus-a-comprehensive-guide)

---

# CAN Integration for STM32CubeMX

## Enable the CAN Communication Protocol

For the version of **STM32CubeIDE** we are using *(2.0.0)*, the **STM32CubeMX** tool is not built in, so we will need to open the **IOC** for our project and enable the **CAN1** interface under the communication tab. 

## Setting the Baud Rate

Once **CAN1** is enabled, the next step is configuring the CAN bit timing to achieve the desired baud rate. In **STM32CubeMX**, the CAN baud rate is **not set directly** as a single value; instead, it is derived from several timing parameters that divide the CAN peripheral clock.

Navigate to:

**CAN1 → Parameter Settings → Bit Timing Parameters**

### CAN Bit Timing Overview

For our LATCH we are looking to implement a 500MBs can bus. Below is an example of how to achieve this and the formula used.

![bitrate formula](imgs/bitrate_formula.png)

Assuming a CAN peripheral clock of **48 MHz**:

- Prescaler = 6  
- BS1 = 13  
- BS2 = 2  

![bitrate example](imgs/bitrate_example.png)

Once these parameters are set, save the IOC file and regenerate the project to apply the CAN configuration.


# Integrating CAN-API into Your STM32CubeIDE Project

Now that the IOC is setup, we need to add the custom CAN api written by your awesome lead that will help reduce the complexity of our can structure.

The first step is to copy the CAN folder where the *hpp* and *cpp* files live. You can download the zip containing the folder [here](https://drive.google.com/file/d/1Oi9-cgGh-wmnKVrizlkShOQthr1hAkUJ/view?usp=sharing).

After downloading this place the CAN folder in the parent directory of your STM32CubeMX project along side ```Core```, ```Drivers```, ```Debug```, and so on. Should look like the directory layout depicted below.

![can_dir_locatioin](imgs/can_dir_loc.png)

Next you need to add the *Src* and *Inc* directories to your build configurations to get everything to build. 

Navigate to:  
**Project → Properties → C/C++ Build → Settings → Include paths** and add ```../CAN/Inc```.

![can_inc](imgs/can_inc.png)

Then select apply and then if you are prompted, Rebuild Index as well.

Now navigate to:  
**Project → Properties → C/C++ General → Paths and Symbols → Source Location** and select ```add folder``` then select the ```CAN``` folder.

![can_src](imgs/can_src.png)

Now apply and close and you should be able to build and use the CAN API now!! Just add the header to your ```main.c``` private includes.`

# CAN-API Usage

The custom CAN API simplifies the process of filtering, initializing, and communicating over the CAN bus by wrapping the STM32 HAL functions into a device-centric structure.

---

## 1. Define Device and Data Buffers
In your `main.c` (within the **Private variables** section), define your `CANDevice_t` instance and the buffers needed for transmission and filtering. Below is an example of what this should look like.s

```c
/* Private variables ---------------------------------------------------------*/
/* USER CODE BEGIN PV */
CANDevice_t canDV;
uint16_t rxIdList[ID_LIST_LEN] = {0x110, 0x120}; // IDs we want to receive
uint8_t txData[8] = {0x12, 0x0f, 0x23, 0x33, 0x23};
uint8_t txLen = 5;
/* USER CODE END PV */
```

## 2. Initialize the Device
In the ```main()``` function, after the peripheral initialization calls, initialize the CAN device. This links your device struct to the hardware handle, starts the CAN peripheral, and enables RX interrupts.

**Note:** You must call ```can_config_filter``` to specify which IDs the hardware should accept.

```c
/* Initialize all configured peripherals */
MX_GPIO_Init();
MX_CAN1_Init();

/* USER CODE BEGIN 2 */
// 1. Link hardware handle and start CAN/Interrupts
device_can_init(&canDV, &hcan1);

// 2. Configure hardware filters for specific IDs
can_config_filter(&canDV, rxIdList, ID_LIST_LEN);

// 3. Link your custom callback function
link_rx_callback(&canDV, can_rx_callback);
/* USER CODE END 2 */
```

## 3. Handling Received Data
To process incoming messages, define the callback function you linked in the previous step. This function is automatically triggered by the internal ```HAL_CAN_RxFifo0MsgPendingCallback``` override whenever a filtered message arrives.

```c
/* USER CODE BEGIN 4 */
void can_rx_callback(CANDevice_t *device) {
    // The received data is automatically stored in device->rxBuffer
    uint8_t *data = device->rxBuffer;
    
    // The message header (ID, DLC, etc.) is available in device->rxHeader
    uint32_t receivedID = device->rxHeader.StdId;

    // The msgReceived flag is set to true automatically
    if(device->msgReceived) {
        // Handle data...
        device->msgReceived = false; // Reset flag after processing
    }
}
/* USER CODE END 4 */
```

## 4. Transmitting Data

Use the wrapper function ```transmit_can_data``` to send messages. This function handles the configuration of the ```TxHeader``` (Standard ID, Data Frame) and adds the message to the TX mailbox.

```c
/* Infinite loop */
while (1)
{
    // Transmit data to a specific ID using the device wrapper
    transmit_can_data(&canDV, 0x120, txData, txLen);
    
    HAL_Delay(500); // Send every 500ms
}
```

## API Reference

| Function | Description |
| :--- | :--- |
| **`device_can_init`** | Links the HAL handle, clears the buffer, starts the CAN peripheral, and activates RX notifications. |
| **`can_config_filter`** | Configures filter banks in `IDLIST` mode (up to 4 IDs per bank) using 16-bit scaling. |
| **`link_rx_callback`** | Assigns a user-defined function to be called immediately when a message is received. |
| **`transmit_can_data`** | Simplifies transmission by setting up the `TxHeader` and calling `HAL_CAN_AddTxMessage`. |