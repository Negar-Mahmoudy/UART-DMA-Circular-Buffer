# UART DMA Circular Buffer Example (STM32)

## Overview
This project demonstrates how to use **UART1 with DMA** on an STM32 microcontroller to receive data efficiently.  
Instead of having the CPU handle every single incoming byte, the **DMA (Direct Memory Access)** controller transfers data directly from the UART peripheral into RAM.  
Received data is stored in a **circular buffer**, which makes the system always ready for new incoming bytes.

---

## How It Works
- At startup, DMA is configured to receive 1 byte over UART1.  
- When 1 byte is received, DMA writes it directly into the `buffer_rx` array.  
- After the transfer is complete, a **DMA interrupt** is triggered.  
- In the interrupt callback:
  1. A counter (`count`) is updated to point to the next buffer location (circular indexing: `0 → 1 → … → 7 → 0 …`).
  2. DMA is re-armed to receive the next byte at the new location.
- This creates a **circular buffer** where old data is overwritten by new data once the buffer is full.

---

## Explanation of `HAL_UART_RxCpltCallback`

The UART receive complete callback is implemented as follows:

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{	
    count = (count + 1) & 0x07;            // Update the buffer index (circular indexing)
    HAL_UART_Receive_DMA(&huart1, buffer_rx + count, 1); // Restart DMA for the next byte
}
```

---

# What Happens Here

1. Update the buffer index (count):

- count = (count + 1) & 0x07; increments the index and wraps it around using modulo 8.
- This creates a circular buffer, allowing new data to overwrite the oldest data if necessary.
  
2. Restart DMA for the next byte:
- After the previous byte has been transferred, the DMA controller is inactive.
- Calling HAL_UART_Receive_DMA again sets DMA to receive the next byte.
- This ensures continuous reception without losing any incoming bytes.

---

# Why This Design is Important
- Prevents data loss: DMA is restarted only after the previous byte is fully received.
- Circular buffer management: Automatically cycles through the buffer indices.
- Minimal CPU involvement: CPU only updates the counter and restarts DMA in a short callback.

---

# Why We Call HAL_UART_Receive_DMA in the Callback
- DMA transfers one byte at a time, and is inactive after completing a transfer.
- If we continuously call HAL_UART_Receive_DMA in the while(1) loop, DMA might be reset before completing the previous transfer, causing lost data.
- Restarting DMA only in the callback guarantees reliable circular reception.

---

# Why We Use the UART Callback Instead of DMA Callback

- In STM32 HAL, when using UART with DMA, there are two callbacks:
1. DMA Callback (HAL_DMA_XferCpltCallback)
- Triggered when DMA finishes a transfer.
- Requires manual management of UART and buffer.

2. UART Callback (HAL_UART_RxCpltCallback)
- Triggered by HAL after DMA finishes a UART transfer.
- HAL links DMA interrupt to this callback, simplifying code.

# How It Works in This Project
1. CPU calls HAL_UART_Receive_DMA(&huart1, buffer_rx+count, 1).
2. DMA transfers one byte from UART to RAM.
3. DMA triggers an interrupt when the transfer is complete.
4. HAL ISR handles the DMA interrupt and automatically calls HAL_UART_RxCpltCallback.
5. In this callback, the CPU updates count and restarts DMA for the next byte.

---

# Why Use a Circular Buffer?
- Continuous Reception: Always ready for new bytes.
- No Buffer Overflows: Old data is automatically replaced if buffer is full.
- Lightweight on CPU: CPU handles only short interrupts.
- Scalable: Ideal for high-speed serial data and streaming applications.

---

# CPU Usage
- During reception: 0% CPU (DMA handles transfers).
- On DMA transfer complete: CPU runs a short callback (updates counter and restarts DMA).
- Typical callback duration: a few microseconds → very low CPU load even at high baud rates.

---

#Example Applications
- GPS modules (continuous NMEA data streams)
- Bluetooth / Wi-Fi modules (HC-05, ESP8266, ESP32 in UART mode)
- Industrial sensors sending periodic measurements
- Data loggers needing uninterrupted data streams
- High baud-rate communication while CPU handles other tasks (motor control, DSP, etc.)
