# Changelog

## 0.1.0

The first release: a driver for Microchip's MCP2515 stand-alone CAN 2.0B
controller on an SPI bus.

- `Mcp2515`, the type a board's device table builds from the chip-select pin,
  the optional INT pin and the crystal's frequency, implementing
  embedded-hal-nv's `CanBus` (open, send, receive, tx_done, bitrate, errors),
  with its own methods for the priority send, the INT-pin receive, the mode,
  the interrupt flags and the abort.
- The bit timing computed from the crystal and the bit rate: a solver that
  takes the smallest baud rate prescaler giving the rate exactly with 5 to 25
  time quanta a bit and places the sample point, and a table of the reference
  libraries' values for 8 MHz and 16 MHz crystals at 5 to 1000 kbit/s that
  `bit_timing` uses first. The 1 Mbit/s row of the libraries for an 8 MHz
  crystal is refused: it has a PS2 of 1 time quantum, below the 2 the data
  sheet gives as the minimum.
- The open sequence: the bus and pins made ready, the RESET confirmed in
  CANSTAT and repeated for a slow crystal, the bit timing written and read
  back, the interrupt enables, the receive modes (every frame, with
  rollover), the one-shot and CLKOUT options and the mode, verified in
  CANSTAT within 50 ms.
- The modes Normal, Loopback, Listen-Only, Sleep and Configuration, and the
  one-shot mode.
- The three transmit buffers with priority: `send` queues frames so they
  leave in the order queued, which the part's rule for equal priorities does
  not do by itself, and `send_priority` takes an explicit priority of 0 to 3.
  Remote frames, standard and extended identifiers, and the abort of all
  pending frames.
- The two receive buffers with the masks, the six filters and the rollover;
  masks and filters are written in Configuration mode and read back, with the
  EID bits left zero so the first two data bytes of a standard frame are not
  compared. The receive reads RX STATUS and then READ RX BUFFER, and the
  receive by the INT pin as an option.
- The interrupt flags and enables, the error counters and the fault state
  (error-active, error-passive, bus-off), and the overflow flags.
- The suite runs over a double of the part and covers every line of `src/`.
- The device program `examples/can_echo.nv`, a node that sends a counter and
  prints the frames it receives. Built for the nRF52840-DK it is 18,476 B of
  flash, for the nRF54LM20-DK 19,516 B and for the FRDM-MCXN947 20,292 B;
  before the Nordic buses landed, with the reading path left out, the two DK
  images were 18,036 B and 18,936 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an MCP2515.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive SPI with SPIM2 and SPIM21, so their images carry the reading path.  A
  reply from the part over either bus is NOT VERIFIED.
- The `CanBus` trait of embedded-hal-nv 0.3.2 names two modes, no filters, no
  priority, no interrupt flags and no reason for a refused open: the
  driver's own functions beside the trait carry them.
- Not implemented: the RXnBF and TXnRTS pins, the start-of-frame signal and
  the wake-up filter, wake-up from sleep by bus activity, and filtering on
  the first two data bytes of standard frames.
