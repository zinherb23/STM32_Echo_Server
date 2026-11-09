# STM32_Echo_Server

在STM32F401RE與ENC28J60的驅動程式上移植lwIP與FreeRTOS，並將接收封包的方式改為中斷驅動。實現一個TCP Echo Server，回傳PC端傳送的訊息，同時偵測開發板按鍵，將按鍵事件即時傳送給 PC 端。

A TCP/IP networking stack running on an STM32F401RE Nucleo board with a low-cost
ENC28J60 SPI Ethernet module. It uses FreeRTOS and lwIP in full RTOS mode 
and has an interrupt-driven driver, link detection and a TCP echo server.
It's meant for an MCU without a built-in Ethernet MAC.

## Features

- **Interrupt-driven RX:** ENC28J60 INT pin (PA0, EXTI0) wakes a dedicated RX task instead of polling
- **Push-button events:** Pressing B1 sends `button pressed <count>` to the connected client
- **TCP echo server:** One client at a time.


## Hardware and wiring

| ENC28J60 | STM32F401RE (Nucleo) | Notes |
|---|---|---|
| SCK  | PC10 (SPI3_SCK)  | |
| MISO | PC11 (SPI3_MISO) | |
| MOSI | PC12 (SPI3_MOSI) | |
| CS   | PA4  | GPIO output, idle high |
| RESET| PB0  | GPIO output, idle high |
| INT  | PA0  | EXTI falling edge, pull-up |
| VCC / GND | 3.3V / GND | |

Other pins: B1 = PC13 (button), LD2 = PA5 (LED).

## Tech stack

- **MCU:** STM32F401RE
- **RTOS:** FreeRTOS Kernel (11.3.0)
- **Network stack:** lwIP (2.2.1)
- **Ethernet:** Microchip ENC28J60 (v7) 
- **Toolchain:** arm-none-eabi-gcc (13.2)

## Architecture

```
                          ┌──────────────── ENC28J60 (SPI3, 5.25 MHz) ───────────────┐
                          │                                                          │
 PA0 EXTI0 ──task notify──► [enc_rx   prio 5] ──tcpip_input()──► [tcpip   prio 4]    │
 (ENC INT, falling)          drain RX by EPKTCNT   (pbuf → mbox)    lwIP core:       │
                             link up/down          ◄─linkoutput()── ARP / IP / ICMP / TCP
                                                    (TX over SPI)        ▲
                                                                         │ socket calls
                                                                         │ (msg to tcpip thread)
 PC13 EXTI15_10 ─task notify─► [button  prio 2] ──FreeRTOS queue──► [echo    prio 3]
 (B1, falling)                  debounce, count     (press count)     BSD sockets + select()
                                                                      port 7, one client
```

| Task | Priority | Stack (words) | Role |
|---|---|---|---|
| enc_rx | 5 | 512 | Drain ENC28J60 RX buffer, hand frames to lwIP |
| tcpip  | 4 | 1024 | lwIP core thread |
| echo   | 3 | 768 | TCP echo server, port 7 |
| button | 2 | 384 | Debounced B1 events → client |

## Project structure

```
Core/          CubeMX-generated init + application code (app, enc, ethernetif, net, echo, button)
Port/lwip/     lwIP arch port (cc.h, sys_arch.h)
ThirdParty/
  FreeRTOS-Kernel/
  lwip/
Drivers/       STM32 HAL + CMSIS
```

## Getting started

1. Clone:
   ```bash
   git clone https://github.com/zinherb23/stm32-enc28j60-lwip-freertos.git
   ```
2. In **STM32CubeIDE**: File → Import → *Existing Projects into Workspace* → select the cloned folder.
3. Set the IP and MAC in `Core/Inc/app_config.h` to match your LAN (default: `192.168.1.200`).
4. Build (Debug), connect the Nucleo over USB and click Run.
5. Open a serial terminal at `115200` 8N1 to see the log.
6. Type something or press B1 and it's echoed back. 

