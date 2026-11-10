# mcp2515-nv

A driver for the MCP2515, a stand-alone CAN 2.0B controller with an SPI
interface from Microchip, specified in the data sheet "MCP2515: Stand-Alone
CAN Controller with SPI Interface", DS20001801J, 2019
([microchip.com/en-us/product/MCP2515](https://www.microchip.com/en-us/product/MCP2515)).
The driver implements the `CanBus` trait of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) over an SPI
bus, a chip-select pin and an optional interrupt pin, so a board with no CAN
controller of its own gets a CAN bus on any SPI port.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, so the driver has run only against a double of the part, in the host
suite. The device program below builds for the nRF52840-DK, the
nRF54LM20-DK and the FRDM-MCXN947 and has not been flashed.

## The device program

`examples/can_echo.nv` is a node on a CAN bus behind an MCP2515 breakout. It
opens the part at 500 kbit/s, sends a data frame with the identifier 0x123
and a counter once a second, and prints every frame it receives over the
board's UART:

```
rx 291 4 1 0 0 0
```

```novo norun:needs-pkg
use embedded_hal.{ BoardSpi, BoardGpio, CanBus }
use dev_bus
use mcp2515
use mcp2515.{ Mcp2515 }

@tier(embedded)
fn main() [hw]
    // Chip select on pin 28, INT on pin 29, a 16 MHz crystal.
    let w = dev_bus.wiring_irq(dev_bus.wiring_spi(0, 28), 29)
    let can = Mcp2515.attach(BoardSpi { id: 0 }, BoardGpio { id: 0 }, w, 16000000)
    hal.timer.delay_ms(10)
    if can.open_with(500000, mcp2515.config_default()) != 0
        hal.uart.write("mcp2515: no answer\n")
        return
    loop
        if let Some(f) = can.receive()
            hal.uart.write(str.from_int(f.id))
            hal.uart.write("\n")
        hal.timer.delay_ms(10)
```

The full program, which reads the INT pin before it looks and sends the
counter, is `examples/can_echo.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a
scratch package for the boards below and for the emulated board, where it
runs the image and checks that the program reports that the part does not
answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which the program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 18,036 B | 6,070 B | 2,660 B |
| `nrf54lm20-dk` | 18,936 B | 6,178 B | 2,644 B |
| `frdm-mcxn947` | 20,288 B | 8,004 B | 2,668 B |

The numbers are from the build's memory report. Flash also holds the runtime,
the compiler's support library and the program's tables. The arena, 65,536 B
on the Nordic boards, is the runtime's heap and is reserved whether or not
the program uses it. The two Nordic board packages answer for SPI as absent
hardware, so on those images `open_with` answers -1 until a board package
names an SPI driver with `@hal(spi)`; the FRDM-MCXN947 board package drives
SPI. The driver's own code allocates nothing: every buffer is a fixed-size
`Vec` in the caller's frame.

### The effects

Every function of the driver that reaches the bus, the pins or the timer
declares `[hw]`, and nothing else. The driver performs no input or output of
its own and allocates nothing of its own. The module checks at every tier,
`embedded` included. The program above declares `[hw]` for the same reason.

## What it is

The MCP2515 implements CAN 2.0B: standard (11-bit) and extended (29-bit)
data and remote frames, at up to 1 Mbit/s, with three transmit buffers, two
receive buffers, six acceptance filters and two masks. It speaks SPI in mode
0,0 or 1,1 at up to 10 MHz. The driver uses mode 0 at 4 MHz.

| Instruction | Byte | What it does |
|---|---|---|
| RESET | 0xC0 | the registers to their power-up values, Configuration mode |
| READ | 0x03 | registers from an address, the address advancing |
| READ RX BUFFER | 0x90, 0x92, 0x94, 0x96 | a receive buffer from its identifier or its data; the buffer's flag clears when CS rises |
| WRITE | 0x02 | registers from an address |
| LOAD TX BUFFER | 0x40 to 0x45 | a transmit buffer from its identifier or its data |
| RTS | 0x81, 0x82, 0x84 | requests to send the buffers named |
| READ STATUS | 0xA0 | one byte: the receive flags and, for each transmit buffer, TXREQ and its flag |
| RX STATUS | 0xB0 | one byte: the buffers holding a message, the frame type, the filter that matched |
| BIT MODIFY | 0x05 | the bits of a mask in a register, on the registers that take it |

The bit rate is `osc / (2 x BRP' x N)` for a baud rate prescaler BRP' of 1 to
64 and N time quanta a bit: one for the synchronisation segment, 1 to 8 for
the propagation segment, 1 to 8 for phase segment 1 and 2 to 8 for phase
segment 2, so 5 to 25 in all. CNF1, CNF2 and CNF3 hold them.

## Install

```sh
novo pkg add mcp2515-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `mcp2515` | The bit-timing computation and table, the frame encoding, the SPI instructions over the trait `MccLink`, the modes, `open_code`, the transmit and receive functions, the masks and filters, the interrupt flags, the error counters, the bus and pin framing, the types `Mcp2515` and `Mcp2515Link`, and the `CanBus` implementation |

## How to choose an entry point

A program on a board uses `Mcp2515`, as the board's device table builds it,
with the crystal of the breakout:

```novo norun:fragment
    devices
        can  Mcp2515  from "mcp2515-nv"  on spi0  cs can_cs  irq can_int
```

A program that holds its own bus, or a test with a double of the part,
implements `MccLink` over its bus and calls the module's functions, which
take the link: `mcp2515.open_code(link, osc_hz, bitrate, config)`,
`mcp2515.send_frame(link, frame)`. The framing of the bus is `spi_ready` and
the link's `link_xfer`, built over `SpiBus` and `GpioOut`.

| Method of `Mcp2515` | What it does |
|---|---|
| `attach(bus, pins, w, osc_hz)` | the part with the chip select `w.cs` and, if the wiring has one, the INT pin `w.irq`; nothing is sent |
| `open(bitrate, mode)` | `CanBus`: Normal or Loopback mode with the default settings |
| `open_with(bitrate, config)` | opens with the settings of `Mcp2515Config` and answers a code, 0 for success |
| `send(frame)` | `CanBus`: queues the frame behind those queued |
| `send_priority(frame, prio)` | sends at a priority of 0 to 3 in a free buffer; answers the buffer or a negative code |
| `receive()`, `receive_by_pin()` | `CanBus`: the next frame, or `None`; the second reads the INT pin first |
| `tx_done()`, `bitrate()`, `errors()` | `CanBus`: no frame waiting; the rate read back from CNF1 to CNF3; the error counters and state |
| `mode()`, `set_mode(m)` | the mode in CANSTAT; a mode change, confirmed |
| `flags()`, `clear_flags(mask)`, `int_level()` | CANINTF; its clearing; the INT pin |
| `abort_all()` | aborts every frame waiting to be sent |

The module's functions add `bit_timing`, `bit_timing_known` and
`bit_timing_solve`, `filter_set`, `mask_set`, `rx_mode_set`, `config_enter`
and `config_leave`, `one_shot_set`, `irq_enable`, `eflg_read` and
`eflg_clear_overflow`, `tx_pending`, `rx_status_read` and `status_read`.

`Mcp2515Config` carries the mode (`MCP_MODE_NORMAL`, `MCP_MODE_LOOPBACK`,
`MCP_MODE_LISTEN`, `MCP_MODE_SLEEP`), one-shot, rollover, whether the masks
and filters are off, the sources of the INT pin, the CLKOUT divider and an
explicit bit timing.

The `CanBus` trait has two modes, and the part has four; Listen-Only, the
one-shot mode, the filters, the priorities and the interrupt flags have no
place in the trait, so they are the driver's own functions beside it.

## The rules a user needs

1. The crystal's frequency is an argument of `attach`: the part does not
   know it, and the bit rate is computed from it. A breakout has 8 MHz or
   16 MHz. A wrong value makes a wrong bit rate and no error.
2. `open_code` answers a code: 0 for success, -1 when the part does not
   answer or the wiring has no chip select, -2 for a mode outside 0 to 4,
   -3 for a bit rate the crystal cannot make, -4 when the bit timing did
   not read back as written, -5 when the mode did not take within 50 ms. An
   SPI bus no part drives reads 0xFF or 0, never the Configuration mode that
   a reset part reports, so a missing part is -1.
3. The bit timing is the table of the reference libraries where it has the
   crystal and the rate (8 MHz and 16 MHz, 5 to 1000 kbit/s), and the solver
   elsewhere. `bit_timing_solve` takes the smallest prescaler that gives the
   rate exactly with 5 to 25 time quanta, puts the sample point at the asked
   fraction of the bit (75 % by default), and sets three samples a bit and
   an SJW of 1. A rate no prescaler makes exactly is refused, never rounded.
4. An 8 MHz crystal cannot make 1 Mbit/s. It gives 4 time quanta a bit, and
   the part needs 5: PS2 must be at least 2 time quanta. Both reference
   libraries carry a row for it with a PS2 of 1; this driver refuses it.
5. The bit timing, the masks and the filters are writable in Configuration
   mode only. The part ignores a write in any other mode and says nothing,
   and it reads the masks and the filters as zero. `filter_set` and
   `mask_set` check the mode first and read the registers back.
6. A mode change takes effect when the bus is idle: after the pending frames
   and, to join a bus in Normal mode, after 11 recessive bits, which is
   1.1 ms at 10 kbit/s. `mode_set` polls CANSTAT every 100 microseconds for
   50 ms and answers whether the mode came.
7. After RESET the part is held for 128 oscillator cycles. The driver waits
   1 ms, confirms Configuration mode in CANSTAT, and repeats the RESET up to
   ten times for a crystal that is slow to start.
8. BIT MODIFY works on CANCTRL, CANINTE, CANINTF, EFLG, TXBnCTRL, RXBnCTRL,
   CNF1 to CNF3, BFPCTRL and TXRTSCTRL only. On any other register the part
   writes the whole byte. The driver uses it on those registers only.
9. `send` queues frames in the order they are sent. The part sends the
   highest priority first, and among equal priorities the highest buffer
   number first, so frames put in buffers 0 and 1 at one priority leave in
   the wrong order. `send` gives buffers 0, 1 and 2 the priorities 2, 1 and
   0 and puts a frame in the buffer after the highest one waiting; with
   buffer 2 waiting it answers that the queue is full, even if buffer 0 is
   free. `send_priority` takes the lowest free buffer at the priority given
   and makes no promise about order among equals.
10. A frame the bus does not acknowledge is sent again until it is, and its
    buffer stays waiting. `tx_done` stays false and `send` answers full. With
    one-shot mode set (`one_shot_set`, or `one_shot` in the config) a frame
    is tried once. `abort_all` clears the buffers.
11. `receive` reads RX STATUS and then the buffer with READ RX BUFFER, whose
    instruction frees the buffer and clears its flag when CS rises. The
    driver reads all 13 bytes of the buffer. RXB0 is read before RXB1.
12. With rollover on, a frame that finds RXB0 full goes to RXB1 whatever
    RXB1's filters say. Without it the frame is lost and the overflow flag
    in EFLG is set; the flag stays until `eflg_clear_overflow`.
13. `open_code` receives every frame by default (RXM 11, the masks and
    filters off). With `accept_all` false the masks and filters decide, and
    until they are written, the filters and masks are zero: a zero standard
    filter under a zero mask accepts every standard frame and no extended
    one, because with RXM 00 a filter applies to standard frames or to
    extended frames by its EXIDE bit.
14. For a standard frame the part also compares the first two data bytes
    with the EID15 to EID0 bits of the filter and the mask. `filter_set` and
    `mask_set` write those bits as zero, so the data are not compared.
15. The INT pin is open-drain and active low. It follows the sources in
    CANINTE (`irq_enable`), which are off by default, so the flags in CANINTF
    are polled. `receive_by_pin` makes no SPI transaction while the pin is
    high. A wiring with no INT pin polls.
16. Bus-off is the TXBO flag, reported by `errors` as `CAN_BUS_OFF`. The part
    returns to error-active by itself after 128 occurrences of 11 recessive
    bits; the driver does nothing to hasten it.
17. The CLKOUT pin is off after `open_code` unless the config asks. The part
    powers up with it running at the oscillator divided by 8.
18. Loopback mode returns the frames sent and puts nothing on the bus,
    which a bench with no other node runs. Listen-Only mode receives
    frames and sends nothing, not even an acknowledge or an error frame.

## Running on a microcontroller

Every function of the module builds at the embedded tier. The driver's own
code builds no list: its bytes move through devices-nv's `DevBytes`, a
32-byte buffer held in the caller's frame, and a frame is 14 bytes at most.
The registry shows the package on the host layer, the layer of the trait
packages it depends on. The frame type is embedded-hal-nv's `CanBusFrame`,
which holds up to 8 data bytes in one integer; `can-nv`'s frame types are
not used, since `CanBus` names `CanBusFrame`.

## What is not included

- The RX0BF and RX1BF pins and the TX0RTS to TX2RTS pins (BFPCTRL and
  TXRTSCTRL). The pins are the board's; the driver sets TXREQ by the RTS
  instruction and reads buffer flags from the registers. NOT IMPLEMENTED.
- The start-of-frame signal on CLKOUT (CNF3.SOF) and the wake-up filter
  (CNF3.WAKFIL). NOT IMPLEMENTED.
- Sleep mode as a state to leave: `set_mode(1)` puts the part to sleep and
  waits for CANSTAT, but wake-up by bus activity (the WAKIF flag) is left to
  the program, which clears the flag and sets a mode. NOT IMPLEMENTED.
- Data-byte filtering. A mask and filter that compare the first two data
  bytes of standard frames can be written with `reg_write`; there is no
  function for it. NOT IMPLEMENTED.
- A transmit queue, a receive ring and ISO-TP above the frames: those are
  [can-nv](https://novo-lang.org/packages/can-nv)'s.
- The MCP2518FD and MCP25625, which are other parts.
- An awaitable form of the INT pin. The driver reads the pin; a program that
  waits for it uses the board's edge trait.

## Reference drivers read

Each of these was read for its fixes, and none was ported. The fix is named
at its site in `src/mcp2515.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/can/can_mcp2515.c` | the mode change polled in CANSTAT after a BIT MODIFY of CANCTRL; the 128 microsecond oscillator start-up after the reset; the transmit buffer chosen by TXREQ; LOAD TX BUFFER followed by RTS |
| `mcp2515` crate 0.3.0 | the bit-timing table (copied from `coryjfowler/MCP_CAN_lib`); RESET followed by a wait; READ RX BUFFER for the receive; the registers' bit layouts |
| `autowp/arduino-mcp2515` | the bit-timing values for 8 MHz and 16 MHz, checked against the formula (the 8 MHz rate of 1 Mbit/s has a PS2 of 1 TQ; the 16 MHz rate of 500 kbit/s has a sample point of 56 %); the RESET wait of 10 ms |
| `coryjfowler/MCP_CAN_lib` | the source of the Rust crate's table, which has a sample point of 75 % and three samples a bit |

## Related packages

- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `CanBus` trait, `CanBusFrame` and the SPI and GPIO traits and the board
  handles.
- [devices-nv](https://novo-lang.org/packages/devices-nv) names the wiring
  record a board's device table builds and the SPI transaction helper this
  driver uses.
- [can-nv](https://novo-lang.org/packages/can-nv) holds the frame queue,
  filters, error state machine, ISO-TP and the drivers for controllers on
  the chip.

## Tests

`novo test tests` runs the module over a double of the part that answers the
SPI instruction set. The double keeps a register file behind READ, WRITE,
BIT MODIFY, LOAD TX BUFFER, RTS, READ STATUS, RX STATUS, READ RX BUFFER and
RESET, follows the data sheet where the driver depends on it (the
Configuration-mode lock on the timing, masks and filters, the forced mask on
a register that does not take BIT MODIFY, the delay of a mode change, the
transmit priority and the masks, filters and rollover of the receive path),
and records every byte clocked out.

The suite checks the bit timing against the reference libraries' values (the
table decodes to its rate by the data sheet's formula, and the solver equals
the library wherever they choose the same number of time quanta), the exact
bytes of every transaction of `open_code`, the send, the receive, the mode
change and the filter writes, every refusal, and loopback end to end.
`bash tests/coverage.sh` merges the suites' line coverage over `src/`.

## Licence

Apache-2.0.
