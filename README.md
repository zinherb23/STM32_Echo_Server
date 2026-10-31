# STM32 + ENC28J60 Ethernet with FreeRTOS and lwIP

A TCP/IP networking stack running on an STM32F401RE Nucleo board with a low-cost
ENC28J60 SPI Ethernet module. It uses FreeRTOS and lwIP in full RTOS mode (`NO_SYS=0`)
and has an interrupt-driven driver, link detection and a TCP echo server.
It's meant for embedded developers who want a small, working reference port
for an MCU without a built-in Ethernet MAC.

## Demo

![demo](docs/demo.gif)

<!-- Suggested content: split screen with (1) ping replies at 1–2 ms, (2) PuTTY Raw
     session on port 7 echoing text, (3) pressing B1 → "button pressed 3" appears,
     (4) unplugging the cable → "link down" in the UART log, replug → "link up". -->

| Ping | TCP echo + button event |
|---|---|
| ![ping](docs/ping.png) | ![echo](docs/echo.png) |

## Features

- **Interrupt-driven RX:** the ENC28J60 INT pin (PA0, EXTI0) wakes a dedicated RX task instead of polling
- **ICMP ping** with replies in 1–2 ms
- **TCP echo server on port 7**, one client at a time. A second client is rejected with `busy`
- **Link up/down detection.** An open TCP connection survives the cable being unplugged and plugged back in
- **Push-button events:** pressing B1 sends `button pressed <count>` to the connected client
- **Thread-safe logging:** `printf` over UART guarded by a mutex, with a panic handler

## Hardware and wiring

| ENC28J60 | STM32F401RE (Nucleo) | Notes |
|---|---|---|
| SCK  | PC10 (SPI3_SCK)  | GPIO speed **Medium** (see Challenges) |
| MISO | PC11 (SPI3_MISO) | |
| MOSI | PC12 (SPI3_MOSI) | |
| CS   | PA4  | GPIO output, idle high |
| RESET| PB0  | GPIO output, idle high |
| INT  | PA0  | EXTI falling edge, pull-up |
| VCC / GND | 3.3V / GND | |

Other pins: B1 = PC13 (button), LD2 = PA5 (LED).

## Tech stack

- **MCU:** STM32F401RE (Cortex-M4F, 84 MHz), STM32 HAL, STM32CubeMX / CubeIDE
- **RTOS:** FreeRTOS Kernel (V11.x), `heap_4`, 32 KB heap
- **Network stack:** lwIP 2.2.1 (ST's stm32-mw-lwip) with the FreeRTOS `sys_arch` port
- **Ethernet:** Microchip ENC28J60 (rev B7) over SPI at 5.25 MHz
- **Toolchain:** arm-none-eabi-gcc 13.2

## Architecture

```
 PA0 EXTI ──notify──► [enc_rx  prio 5] ──pbuf──► [tcpip  prio 4] ──► [echo    prio 3]
                       ENC28J60 driver            lwIP core           netconn server
                       (SPI3)                                          ▲
 PC13 EXTI ─notify──► [button  prio 2] ────────────────────────────────┘
```

| Task | Priority | Stack (words) | Role |
|---|---|---|---|
| enc_rx | 5 | 512 | Drain ENC28J60 RX buffer, hand frames to lwIP |
| tcpip  | 4 | 1024 | lwIP core thread |
| echo   | 3 | 768 | TCP echo server, port 7 |
| button | 2 | 384 | Debounced B1 events → client |

### Design decisions

- **`NO_SYS=0` (full RTOS mode) instead of bare-metal lwIP.** Application code can use the blocking netconn API in its own tasks, and the protocol stack is kept in the tcpip thread. The cost is more RAM, and every call into lwIP has to respect core locking (`LWIP_FREERTOS_CHECK_CORE_LOCKING` is enabled during development to catch mistakes).
- **The ISR does no SPI work.** The EXTI handler only sends a task notification. All SPI traffic happens in task context, so the bus never has to be shared with an interrupt and ISR latency stays low.
- **RX task priority is above tcpip.** Frames leave the ENC28J60's 8 KB buffer quickly, which avoids overflow under bursts.
- **Generated code is kept separate from my code.** Application code lives in its own files (`app.c`, `enc.c`, `ethernetif.c`, `net.c`, `echo.c`, `button.c`) or inside `USER CODE` blocks, so regenerating from CubeMX never overwrites it. SVC/PendSV/SysTick generation is turned off because FreeRTOS owns those handlers, and the HAL timebase runs on TIM5.
- **The repo is self-contained.** FreeRTOS and lwIP are vendored under `ThirdParty/`, replacing the Eclipse linked resources the project used before, so a fresh clone builds without any sibling folders.

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
3. Set the IP and MAC in `Core/Inc/app_config.h` to match your LAN (default: `<192.168.x.x>`).
4. Build (Debug), connect the Nucleo over USB and click Run.
5. Open a serial terminal at `<115200>` 8N1 to see the log.
6. Test:
   ```bash
   ping <board-ip>
   nc <board-ip> 7        # or PuTTY → Connection type: Raw, port 7
   ```
   Type something and it's echoed back. Press B1 and you'll see `button pressed 1`.

## Challenges and what I learned

- **Corrupted SPI reads caused by GPIO slew rate, not timing.** One byte of the MAC address (MAADR6) consistently read `0x57` instead of `0xAB`. I wrote a run-time sweep (`enc28j60_spi_sweep`) that tried every combination of SPI prescaler and GPIO speed. It showed 0 errors at Low/Medium for every prescaler and failures at High/Very High. The cause was signal integrity: fast edges ringing on the jumper wires. The fix was setting the SPI pins' output speed to Medium. *Lesson: with flying wires, "faster" GPIO settings can make things worse, and measuring beats guessing.*
- **Keeping the diagnostics.** The diagnostic code stays in the driver but only runs when init fails, so the cost at normal runtime is zero.
- **Interrupt priorities with FreeRTOS on Cortex-M.** ISRs that call FreeRTOS APIs must sit at or below `configMAX_SYSCALL_INTERRUPT_PRIORITY` (EXTI0 = 6, EXTI15_10 = 7, TIM5 timebase = 15).
- **lwIP thread safety.** Learned the difference between the raw, netconn and socket APIs and which threads may call each.
- **Building in milestones.** I worked in four verifiable steps (clock + RTOS + log → ping → TCP echo → button events) and took a backup at each one, so every regression had a small search space.

## Roadmap

- [ ] Trim stack and heap sizes using the measured high-water marks (about 21 KB of heap is still free)
- [ ] Production build profile: turn off development checks (stack-overflow hook, core-locking check, heartbeat)
- [ ] DHCP client instead of a static IP
- [ ] An application protocol on top (e.g. MQTT or a small HTTP status page)
- [ ] Measure throughput with iperf
- [ ] Move to a custom PCB to remove the flying-wire signal-integrity limits
