# Embedded Systems Handbook

Practical notes on embedded firmware for real-time and safety-critical systems. Each chapter answers the same questions for its topic: **what it is, when to choose it over the alternatives, and what goes wrong**.

The notes are written from general engineering knowledge and public standards. They describe no specific product, customer or employer.

## Chapters

| # | Chapter | Covers |
|---|---|---|
| 01 | [Embedded C](chapters/01-embedded-c.md) | `volatile`, `const`, `static`, padding, pointers, memory layout, undefined behaviour, MISRA C, endianness |
| 02 | [RTOS & Concurrency](chapters/02-rtos-and-concurrency.md) | choosing primitives, ISR rules, priority inversion, deadlock, stack sizing, watchdogs |
| 03 | [Serial Protocols](chapters/03-serial-protocols.md) | SPI, I²C, UART, RS-422/485 — trade-offs and failure modes |
| 04 | [MIL-STD-1553](chapters/04-mil-std-1553.md) | bus architecture, word formats, transfers, building a Remote Terminal on an MCU |
| 05 | [Sensors & Navigation](chapters/05-sensors-and-navigation.md) | IMU sensors, sensor fusion, complementary vs Kalman, filters, calibration, AHRS/INS, IMU + GNSS |
| 06 | [Software Architecture](chapters/06-software-architecture.md) | layering, portability, state machines, fault management, patterns, V-model |
| 07 | [DO-178C](chapters/07-do-178c.md) | software levels, processes, traceability, verification, MC/DC, SOI audits |
| 08 | [Bare-metal, Hardware & Debug](chapters/08-bare-metal-and-debug.md) | super loop, registers, NVIC, DMA, watchdog, bootloader, clocks, SWD/JTAG, HardFault |
| 09 | [Git & Code Review](chapters/09-git-and-code-review.md) | PR workflow, merge/rebase scenarios, hotfixes, change control in DO-178C projects |
| 10 | [Build System & Cross-Compile](chapters/10-build-system-and-cross-compile.md) | build chain, Make vs CMake, toolchain files, linker script, `.map` file, bootloader offsets |
| 11 | [CI/CD & Docker](chapters/11-ci-cd-and-docker.md) | firmware pipeline, traceability, pinned toolchains, hardware-in-the-loop CI, CI for IDE + SVN projects |
| 12 | [Python & Bash Automation](chapters/12-python-and-bash-automation.md) | talking to firmware over UART, exit codes, command interpreters, data capture |
| 13 | [Unit Test, Static Analysis & Coverage](chapters/13-unit-test-static-analysis-coverage.md) | testable design, Unity, cppcheck, branch coverage, on-target test gating |
| 14 | [Real-time Metrics](chapters/14-real-time-metrics.md) | latency, jitter, WCET, schedulability, measuring with GPIO/DWT |
| 15 | [Atomics & Memory Barriers](chapters/15-atomics-and-memory-barriers.md) | lost updates, critical sections, LDREX/STREX, DMB/DSB/ISB |
| 16 | [Board Bring-up](chapters/16-board-bring-up.md) | bring-up ladder, typical symptoms, checklist, power-on self-test |
| 17 | [GDB & Fault Analysis](chapters/17-gdb-and-fault-analysis.md) | GDB chain, watchpoints, HardFault decoding, reset causes, watchdog manager |
| 18 | [Firmware Update & Secure Boot](chapters/18-firmware-update-and-secure-boot.md) | A/B updates, integrity vs authenticity, rollback, redundant boot records |
| 19 | [Motor Control](chapters/19-motor-control.md) | PWM, PI with anti-windup, cascaded loops, 6-step vs FOC |

## How to read it

```mermaid
flowchart LR
    C[01 Embedded C] --> R[02 RTOS]
    C --> B[08 Bare-metal & debug]
    R --> P[03 Serial protocols] --> M[04 MIL-STD-1553]
    R --> S[05 Sensors & navigation] --> MC[19 Motor control]
    R --> A[06 Architecture] --> D[07 DO-178C]
    R --> RT[14 Real-time metrics] --> AT[15 Atomics]
    B --> BU[16 Bring-up] --> G[17 GDB & faults] --> FU[18 Firmware update]
    A --> GIT[09 Git] --> BLD[10 Build] --> CI[11 CI/CD]
    BLD --> PY[12 Automation] --> CI
    BLD --> T[13 Testing] --> CI
```

Chapters stand alone; the graph shows which ones build on which. Most chapters end with a **Quick answers** list for revision.

## Conventions

- RTOS examples use FreeRTOS / CMSIS-RTOS v2 names; the reasoning applies to any preemptive RTOS.
- Code samples are C99 for 32-bit Arm Cortex-M unless stated otherwise.
- Diagrams are SVG files in [`assets/diagrams`](assets/diagrams) with light and dark variants; a few flow charts are Mermaid. Both render directly on GitHub.
- Callouts use GitHub alerts: **Tip** = the fix or the result, **Warning** = a diagnosis or a pitfall, **Important** = a rule, **Note** = background.

## Related

- [px4-sitl-learning](https://github.com/batuuhnk/px4-sitl-learning) — PX4 flight stack studied in simulation
- [cpp-temelleri](https://github.com/batuuhnk/cpp-temelleri) — C++ concepts rebuilt side by side with their C equivalents

## License

Text and diagrams: [CC BY 4.0](LICENSE). Code snippets may be reused freely.
