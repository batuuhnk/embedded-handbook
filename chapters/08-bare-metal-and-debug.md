# 08 · Bare-metal, Hardware & Debug

> What sits below the RTOS: super loops, registers, interrupts, peripherals, clocks, bootloaders — and the tools to find out what went wrong when they misbehave.

**Contents:** [Bare-metal vs RTOS](#bare-metal-vs-rtos) · [Register-level programming](#register-level-programming) · [Interrupts and the NVIC](#interrupts-and-the-nvic) · [Peripherals](#peripherals) · [Clock tree](#clock-tree-and-pll) · [Debug interfaces](#debug-interfaces) · [HardFault](#hardfault-debugging) · [Finding communication faults](#finding-communication-faults) · [Testing](#testing) · [Quick answers](#quick-answers)

---

## Bare-metal vs RTOS

| | Bare-metal | RTOS |
|---|---|---|
| Structure | super loop + ISRs | tasks + scheduler |
| Timing owned by | the programmer (timers, flags) | the scheduler (pre-emptive) |
| Determinism | very high (direct control) | high, plus kernel overhead |
| Complexity | simple systems | complex, multi-activity systems |
| Memory | minimal | +5–20 KB flash, +1–5 KB RAM |
| Interrupt handling | direct, via flags | ISR → semaphore/notification → task |

**Bare-metal when:** one or few activities · very tight memory (< 16 KB flash, < 4 KB RAM) · ultra-low latency · simple sensor reading or motor control.
**RTOS when:** several independent activities (sensor + comms + control) · different periods · timeouts and blocking I/O · complex systems such as navigation or attitude estimation.

### The super-loop pattern

```c
volatile uint8_t flag_10ms, flag_100ms, flag_1s;

int main(void) {
    system_init(); gpio_init(); timer_init(); uart_init(); adc_init();
    while (1) {
        if (flag_10ms)  { flag_10ms = 0;  sensor_read(); apply_filter(); }
        if (flag_100ms) { flag_100ms = 0; uart_send_data(); }
        if (flag_1s)    { flag_1s = 0;    led_toggle(); watchdog_feed(); }
    }
}

void TIM2_IRQHandler(void) {             /* 1 ms */
    static uint16_t cnt = 0;
    cnt++;
    if (cnt % 10   == 0) flag_10ms  = 1;
    if (cnt % 100  == 0) flag_100ms = 1;
    if (cnt % 1000 == 0) { flag_1s = 1; cnt = 0; }
}
```

---

## Register-level programming

HAL libraries are convenient, but on performance-critical paths, on parts without a vendor HAL, or when the HAL hides a bug, you go to the registers.

### GPIO (STM32F4, PA5 as output)

```c
RCC->AHB1ENR   |=  (1U << 0);      /* 1. clock for GPIOA        */
GPIOA->MODER   &= ~(3U << 10);     /* 2. clear PA5 mode         */
GPIOA->MODER   |=  (1U << 10);     /*    01 = output            */
GPIOA->OTYPER  &= ~(1U << 5);      /* 3. push-pull              */
GPIOA->OSPEEDR &= ~(3U << 10);
GPIOA->OSPEEDR |=  (2U << 10);     /* 4. high speed             */
GPIOA->PUPDR   &= ~(3U << 10);     /* 5. no pull                */

GPIOA->BSRR = (1U << 5);           /* set   (atomic) */
GPIOA->BSRR = (1U << 21);          /* reset (atomic) */
```

> [!TIP]
> **Why BSRR:** there is no read-modify-write, so an interrupt in between cannot corrupt the port — the set/reset is atomic.

### I²C start-up (STM32F1/F4 style)

```c
RCC->APB1ENR |= RCC_APB1ENR_I2C1EN;
I2C1->CR1 &= ~I2C_CR1_PE;            /* disable              */
I2C1->CR2  = 36;                     /* APB1 = 36 MHz        */
I2C1->CCR  = 180;                    /* 100 kHz standard mode */
I2C1->TRISE = 37;                    /* max rise time        */
I2C1->CR1 |= I2C_CR1_PE;             /* enable               */

I2C1->CR1 |= I2C_CR1_START;          /* START                */
while (!(I2C1->SR1 & I2C_SR1_SB));
I2C1->DR = (slave_addr << 1) | 0;    /* address + write      */
while (!(I2C1->SR1 & I2C_SR1_ADDR));
(void)I2C1->SR2;                     /* reading SR2 clears ADDR */
```

> [!WARNING]
> Every `while` above waits forever if the bus is stuck. Production code bounds each wait with a timeout and recovers the bus (toggle SCL, reset the peripheral).

---

## Interrupts and the NVIC

- The NVIC manages interrupts on Cortex-M; **priority 0 is the highest** (typically 0–15).
- Priority is split into **pre-emption priority + sub-priority** according to the grouping setting.
- **Nesting:** a higher-priority ISR pre-empts a lower one.

**ISR rules:** keep it short (set a flag, leave) · no blocking (delay, `printf`, `malloc`) · shared variables `volatile` · critical sections with `__disable_irq()` / `taskENTER_CRITICAL()` · `FromISR` APIs under an RTOS. More in [chapter 02](02-rtos-and-concurrency.md#isr-rules).

### DMA + interrupt

```c
void DMA1_Stream5_IRQHandler(void) {
    if (DMA1->HISR & DMA_HISR_TCIF5) {           /* transfer complete */
        DMA1->HIFCR |= DMA_HIFCR_CTCIF5;         /* clear the flag    */
        BaseType_t woken = pdFALSE;
        xSemaphoreGiveFromISR(xUartRxSem, &woken);
        portYIELD_FROM_ISR(woken);
    }
}
```

---

## Peripherals

### GPIO / UART / SPI / I²C in one line each

- **GPIO:** digital in/out (LEDs, buttons, simple signals).
- **UART:** asynchronous, TX/RX, no clock line → both sides must agree on baud. Debug consoles, GNSS, modules; RS-232/485 are physical variants.
- **SPI:** synchronous, full-duplex, MOSI/MISO/SCK/CS, fast (tens of MHz), one CS per slave. IMUs, SPI flash, ADCs.
- **I²C:** synchronous, half-duplex, SDA/SCL, addressed, slower (100 k / 400 k / 1 M), saves pins. Slow sensors, EEPROM, RTC.

Detailed comparison → [chapter 03](03-serial-protocols.md).

### DMA

Moves data peripheral ↔ memory (or memory ↔ memory) without the CPU; the CPU does other work meanwhile.
Uses: ADC → buffer (continuous sampling) · UART RX → buffer · SPI block reads · buffer copies.

```c
DMA1_Stream5->CR  &= ~DMA_SxCR_EN;            /* disable                  */
DMA1_Stream5->PAR  = (uint32_t)&USART1->DR;   /* source: UART data reg    */
DMA1_Stream5->M0AR = (uint32_t)rx_buf;        /* destination: RAM         */
DMA1_Stream5->NDTR = BUFFER_SIZE;             /* number of items          */
DMA1_Stream5->CR  |=  DMA_SxCR_EN;            /* start — IRQ on completion */
```

A typical acquisition framework: several sensors read concurrently by DMA; the transfer-complete ISR wakes the processing task with a semaphore; the CPU computes the filter instead of waiting.

### Watchdog

Resets the system if the software stops saying "I'm alive".

| | IWDG | WWDG |
|---|---|---|
| Clock | independent LSI | main clock |
| Can be stopped once started | **no** | — |
| Refresh rule | before the timeout | inside a **window**: too early also resets |
| Catches | hangs | hangs *and* runaway fast loops |

```c
IWDG->KR  = 0x5555;        /* enable register access */
IWDG->PR  = IWDG_PR_PR_2;  /* prescaler /64          */
IWDG->RLR = 1000;          /* reload                 */
IWDG->KR  = 0xCCCC;        /* start                  */

void watchdog_feed(void) { IWDG->KR = 0xAAAA; }
```

Under an RTOS → the watchdog-manager pattern ([chapter 02](02-rtos-and-concurrency.md#watchdog-in-an-rtos), [roadmap 09](../roadmap/09-gdb-and-fault-analysis.md#41-watchdog-manager-which-task-is-stuck)).

### Bootloader

A small program that runs before the application and handles firmware updates.

1. Reset → the bootloader starts (start of flash, `0x08000000`).
2. Update condition? (pin, UART command, flag)
3. None → jump to the application.
4. Yes → receive over UART/SPI → write to flash → verify → jump.

```c
typedef void (*AppEntry_t)(void);

void jump_to_application(uint32_t app_addr) {
    __disable_irq();                                             /* interrupts off          */
    SysTick->CTRL = 0;                                           /* de-init peripherals     */
    SCB->VTOR = app_addr;                                        /* app's vector table      */
    __set_MSP(*(uint32_t *)app_addr);                            /* word 0: stack pointer   */
    AppEntry_t app = (AppEntry_t)(*(uint32_t *)(app_addr + 4));  /* word 1: reset handler   */
    app();
}
```

**Safety:** verify with a CRC/checksum · recover from a power loss during the write (dual bank / golden image) · keep the bootloader small and simple · put bootloader and application in separate flash regions in the linker script. Full treatment in [roadmap 10](../roadmap/10-firmware-update-and-secure-boot.md).

### ADC

- **Resolution:** 12 bits → 4096 levels; with a 3.3 V reference ≈ 0.8 mV per step. `voltage = adc / 4096.0f * 3.3f`.
- **Sample rate:** Nyquist — at least twice the highest frequency (1 kHz signal → > 2 kHz; 5–10× in practice).
- Watch for: **acquisition time** (charging the internal capacitor) · continuous sampling with **DMA** · **oversampling** for extra effective resolution · an **anti-aliasing** low-pass filter in front of the ADC.

### PWM

Average voltage set by the on-time (duty) at a fixed frequency. 50 % duty on a 3.3 V system → 1.65 V average.

```c
TIM3->PSC  = 72 - 1;      /* 72 MHz / 72 = 1 MHz timer clock */
TIM3->ARR  = 1000 - 1;    /* 1 MHz / 1000 = 1 kHz PWM        */
TIM3->CCR1 = 500;         /* 50 % duty                       */
TIM3->CCMR1 |= TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_2;   /* PWM mode 1 */
TIM3->CCER  |= TIM_CCER_CC1E;
TIM3->CR1   |= TIM_CR1_CEN;
```

`f_pwm = f_timer / ((PSC+1)(ARR+1))`, duty = `CCR / (ARR+1)`. Uses: LEDs, motor speed, servos, a cheap DAC. Motor control in depth → [roadmap 11](../roadmap/11-motor-control.md).

---

## Clock tree and PLL

| Source | What | Typical |
|---|---|---|
| HSE | external crystal, stable | 8 / 25 MHz |
| HSI | internal RC, starts fast, less stable | 8 / 16 MHz |
| LSE | 32.768 kHz crystal | RTC |
| LSI | ~32 kHz internal | IWDG |

- **PLL:** `SYSCLK = HSE × N / (M × P)` → `8 × 336 / (8 × 2) = 168 MHz`.
- Distribution: SYSCLK → AHB → **APB1** (slower: I²C, UART, some timers) / **APB2** (faster: SPI1, ADC, some timers).

> [!IMPORTANT]
> A wrong clock breaks UART baud rates, SPI speed and timer periods all at once. It is **the first thing to verify** on a new board ([roadmap 08](../roadmap/08-board-bring-up.md)).

**MPU:** isolates memory regions so a task cannot write outside its own. **SysTick conflicts:** when both the HAL and the RTOS want SysTick, move the HAL time base to a spare timer (e.g. TIM6).

---

## Debug interfaces

| | JTAG | SWD |
|---|---|---|
| Pins | TCK, TMS, TDI, TDO (+ nTRST) | SWDIO, SWCLK |
| Daisy chain | yes (several chips) | no |
| Trace | separate pins | printf-style trace via **SWO** |
| Used for | FPGAs, multi-chip chains, boundary scan | Cortex-M (STM32, TI) |

SWD is the default on Cortex-M: two pins (a PCB advantage), full debug, SWO trace (not on Cortex-M0/M0+). JTAG is for multi-chip chains. Probes: **ST-LINK** (built into dev boards), **J-Link** (faster, more features). Using GDB without the IDE → [roadmap 09](../roadmap/09-gdb-and-fault-analysis.md).

---

## HardFault debugging

Typical causes: NULL pointer · unaligned access (packed struct) · stack overflow · jump to an invalid address (Thumb bit 0) · MPU violation.

1. **Capture the stack frame.** On a fault the CPU pushes R0–R3, R12, LR, PC and xPSR. Bit 2 of LR (EXC_RETURN) says which stack (MSP/PSP) was in use.
2. **PC** = the faulting instruction's address → function via the **map file / `addr2line`**.
3. **LR** = the caller's return address → the call chain.
4. **CFSR** = the cause: MemManage / BusFault / UsageFault — `IMPRECISERR` (bus, usually a pointer), `INVSTATE` (Thumb bit), `UNDEFINSTR`, `UNALIGNED`, `DIVBYZERO`. Address from `BFAR` / `MMFAR`.

```c
void HardFault_Handler(void) {
    __asm volatile (
        "TST LR, #4     \n"
        "ITE EQ         \n"
        "MRSEQ R0, MSP  \n"
        "MRSNE R0, PSP  \n"
        "B hard_fault_handler_c \n"
    );
}

void hard_fault_handler_c(uint32_t *stack) {
    volatile uint32_t lr  = stack[5];
    volatile uint32_t pc  = stack[6];
    volatile uint32_t psr = stack[7];
    error_log(pc, lr, SCB->CFSR);     /* write to EEPROM / send over UART */
    NVIC_SystemReset();               /* or enter a safe state */
}
```

> [!TIP]
> Saving PC, LR and CFSR to non-volatile memory turns a returned field unit into a debuggable one — the fault cause can be read after the fact.

---

## Finding communication faults

1. **Logic analyzer:** frame structure, ACK/NACK, start/stop, baud match (protocol decoder).
2. **Oscilloscope:** voltage levels, rise/fall times, glitches; weak I²C pull-ups show as rounded edges.
3. **Software:** timeouts, CRC/checksum, retries, error counters.
4. **GPIO toggling** to time the start and end of transfers.

> [!NOTE]
> **Example.** Intermittent SPI errors on a test rig → the logic analyzer showed a timing mismatch between clock and data → lowering the SPI clock fixed it → root cause: a long PCB trace degraded signal integrity at the higher speed.

Other tools: SWO/UART logging · **SEGGER SystemView** (RTOS timeline).

---

## Testing

### Unit tests in embedded code

1. **On the host, without hardware:** platform-independent code (algorithms, filters, parsers, state machines) against a mock/stub HAL.

```c
int mock_spi_read(uint8_t reg, uint8_t *buf, uint16_t len) {
    memcpy(buf, test_data, len);   /* predefined test data */
    return 0;
}

void test_sensor_parse(void) {
    uint8_t raw[] = {0x00, 0x10, 0xFF, 0xF0};
    TEST_ASSERT_EQUAL(expected_value, parse_accel_data(raw));
}
```

2. **On the target:** results reported over UART/SWO; drivers verified on real hardware.

In the V-model: unit → integration → system. Tools: Unity, CMock, Ceedling. Full workflow → [roadmap 05](../roadmap/05-unit-test-static-analysis-coverage.md).

### SIL and HIL

| | SIL (software-in-the-loop) | HIL (hardware-in-the-loop) |
|---|---|---|
| What runs | the software, on a PC, against a simulated plant | the real ECU/board, against simulated or real signals |
| Finds | logic and algorithm errors, early and cheap | timing, driver and integration errors |
| Cost | low | rig + hardware |

### Code review checklist

- **Reliability:** NULL checks · buffer bounds · ISR-shared variables `volatile` · enough critical sections · error paths handled
- **RTOS:** `FromISR` in ISRs · mutex/semaphore in the right place · deadlock risk · stack sizes
- **Quality:** single responsibility · `#define`/`enum` instead of magic numbers · clear names · comments that explain *why*
- **Portability:** endianness assumptions · `packed` / bit-fields used deliberately · layering respected

---

## Quick answers

- **Bare-metal or RTOS?** Number of activities, memory, latency. Few activities / very tight / ultra-low latency → bare-metal. Many independent activities, different periods, timeouts → RTOS.
- **GPIO without the HAL?** Enable the clock in RCC → MODER → OTYPER → OSPEEDR → PUPDR → atomic set/reset with BSRR.
- **What is DMA for?** Moving data without the CPU, so the CPU can do other work: ADC, UART RX, SPI block reads.
- **IWDG vs WWDG?** IWDG has its own clock and cannot be stopped; WWDG must be refreshed inside a window.
- **Jumping from a bootloader to the application?** Disable interrupts, de-init peripherals, set VTOR, MSP = word 0, PC = word 1. Verify the image with a CRC and plan for recovery.
- **ADC resolution and sample rate?** 12 bits = 4096 levels; Nyquist ≥ 2×; anti-aliasing, oversampling, DMA.
- **Clock tree and PLL?** Clock sources (HSE/HSI/LSE/LSI) and their distribution; the PLL multiplies the input to produce a high SYSCLK.
- **HardFault?** Stack frame → PC, LR; cause from CFSR; address from BFAR/MMFAR; function from the map file.
- **JTAG vs SWD?** SWD is 2 pins + SWO, for Cortex-M; JTAG has more pins, daisy-chaining and boundary scan.
- **Finding a communication fault?** Logic analyzer → oscilloscope → software (timeout, CRC, retry, counters) → GPIO timing.

[← Previous: DO-178C](07-do-178c.md) · [Back to index](../README.md) · [Roadmap →](../roadmap/README.md)
