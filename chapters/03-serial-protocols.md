# 03 · Serial Protocols

> How I choose between SPI, I²C, UART and RS-422/485, based on topology, speed, distance and built-in error handling. MIL-STD-1553 has its [own chapter](04-mil-std-1553.md).

**Contents:** [SPI](#spi) · [I²C](#ic) · [UART](#uart) · [RS-422 / RS-485](#rs-422--rs-485) · [Comparison](#comparison) · [Typical trade-off questions](#typical-trade-off-questions)

---

## SPI

**Shape:** master–slave, synchronous, full-duplex. Four wires: SCK, MOSI, MISO and one CS per slave.

- The master drives the clock, so there is no baud-rate matching.
- Clock rates in the **MHz to tens of MHz** range.
- **No built-in error detection or acknowledge.** Integrity checks (CRC, read-back) are up to the device or the application.

**I choose it for** fast, short-distance links on the same board: sensors, ADCs, external flash, protocol controller ICs.
**I avoid it when** many devices share the link (one CS pin each) or the signals must leave the board.

---

## I²C

**Shape:** multi-drop, addressed (7 or 10-bit), two open-drain wires (SDA, SCL) with pull-ups.

- Speeds: 100 kHz (Standard), 400 kHz (Fast), 1 MHz (Fast-mode Plus), up to 3.4 MHz (High-speed).
- **ACK/NACK on every byte** gives a basic "is the device there?" check.
- Multi-master arbitration exists, but most designs use a single master.

**I choose it for** many low-rate devices on two pins (temperature sensors, EEPROMs, configuration chips).
**I avoid it for** high data rates or tight timing. Clock stretching, bus lock-ups and pull-up/capacitance limits all add uncertainty. A driver should always include **bus recovery** (toggling SCL to release a stuck slave).

---

## UART

**Shape:** asynchronous, point-to-point, TX/RX only. Each frame is start bit + data + optional parity + stop bit(s).

- Both ends must agree on baud rate. Clock error accumulates across a frame, so oscillator tolerance matters.
- Error detection is at most a parity bit. Real integrity comes from the protocol on top (length + CRC).

**I choose it for** simple 1:1 links: GNSS modules, debug consoles, and **bootloaders**. A UART bootloader needs almost no hardware and works with any host.
**I avoid it for** long cables or noisy environments. There I use the same framing over RS-422/485.

> [!TIP]
> **Practical tip:** receive with DMA + idle-line interrupt into a ring buffer, and parse frames in a task. That gives no per-byte interrupts and no lost bytes under load.

---

## RS-422 / RS-485

Both carry UART-style framing over a **differential pair**. The receiver reads the voltage difference between two wires, which gives strong noise immunity and long reach (up to about 1200 m at low data rates).

| | RS-422 | RS-485 |
|---|---|---|
| Topology | point-to-point; one driver, up to 10 receivers | true multi-drop, typically 32 unit loads |
| Duplex | full (separate TX and RX pairs) | usually half (one pair, take turns) |
| Arbitration | not needed | **application protocol must handle it** (addressing, master polling, driver-enable timing) |

**RS-485 when** several units share one cable (one controller polling field units). **RS-422 when** two points need continuous full-duplex traffic.

**Why not plain UART:** single-ended UART levels are only good for short, quiet links. Vehicles, aircraft and industrial sites need differential signalling.

> [!WARNING]
> **RS-485 gotcha:** the driver-enable (DE) line must be released *after the last stop bit has left the shift register* (transmission complete, not just TX-buffer empty). Otherwise the final byte gets cut off.

---

## Comparison

| Protocol | Topology | Typical speed | Distance | Built-in error handling | My default use |
|---|---|---|---|---|---|
| **SPI** | master–slave, CS per device | MHz – tens of MHz | on-board | none | fast on-board peripherals |
| **I²C** | multi-drop, addressed | 100 kHz – 1 MHz | on-board | ACK/NACK | many slow devices, few pins |
| **UART** | point-to-point | ≤ ~1 Mbit/s | short | parity (optional) | consoles, GNSS, bootloaders |
| **RS-422** | point-to-point (1 → 10 RX) | up to ~10 Mbit/s | long | none | robust full-duplex 1:1 links |
| **RS-485** | multi-drop | up to ~10 Mbit/s | long | none | polled field buses |
| **MIL-STD-1553** | command/response, dual-redundant | 1 Mbit/s | aircraft-scale | strong ([chapter 04](04-mil-std-1553.md)) | avionics data buses |

---

## Typical trade-off questions

**SPI or I²C?**
SPI when speed matters and pins are available. I²C when there are many devices and pin or wiring count matters more than throughput.

**Why RS-485 instead of UART?**
Distance and noise immunity. Differential signalling survives long cables and electrically noisy platforms.

[← Previous: RTOS & Concurrency](02-rtos-and-concurrency.md) · [Back to index](../README.md) · [Next: MIL-STD-1553 →](04-mil-std-1553.md)
