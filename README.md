# RTL Design and Verification Foundation
## Project purpose
This project develops reusable RTL blocks from explicit specifications, with emphasis on interface contracts, backpressure, parameterization, verification, debugging, and synthesis.

Designs are implemented in SystemVerilog and verified using directed testing, assertions, reference models, randomized traffic, and seeded regression.

## Toolchain
- SystemVerilog
- Verilator
- Yosys
- GTKWave
- Bash

## Implemented Blocks
### Ready/Valid One-Entry Buffer
#### Architecture
`DATA_WIDTH`-parameterized one-entry buffer using a ready/valid interface.

A handshake occurs only when `valid && ready` is true. The buffer stores one payload and applies backpressure
when it cannot store additional data. Output data and validity are maintained while the downstream consumer is
stalled.
#### Verification
Verified:
- reset behavior
- basic enqueue/dequeue operation
- producer and consumer stalls
- backpressure
- payload stability during stalls
- data integrity across transactions
### Parameterized Synchronous FIFO
#### Architecture
Parameterized by `DATA_WIDTH` and `DEPTH`.

The FIFO uses:
- an internal data array
- independent read and write pointers
- an occupancy counter
- ready/valid input and output interfaces

Input transfers occur when `in_valid && in_ready`; output transfers occur when `out_valid && out_ready`.

Enqueue-only operations increment occupancy, dequeue-only operations decrement occupancy, and simultaneous enqueue/dequeue transactions leave occupancy unchanged while advancing their respective pointers.

The design supports both full and empty boundary conditions, backpressure, sustained simultaneous transfers, pointer wraparound, and reset after prior activity.

The FIFO uses combinational memory readout and registered write/read pointers. `out_valid` is asserted whenever occupancy is nonzero, and `in_ready` is asserted when the FIFO has available capacity or when a simultaneous dequeue frees space. There is no empty-state fall-through path; the first enqueued word becomes available after the accepting clock edge.
#### Verification
Verified:
- reset behavior
- reset after prior FIFO activity
- post-reset transactions
- basic enqueue/dequeue operation
- sustained simultaneous enqueue/dequeue
- producer and consumer stalls
- backpressure
- payload stability during stalls
- full-boundary behavior and pointer wraparound
- data integrity across transactions
- randomized producer/consumer traffic
- long-sequence reference queue scoreboard checking
- concurrent assertions
- seeded multi-run regression

Manual functional coverage tracks the following:
- reaching empty
- reaching full
- read pointer wraparound
- write pointer wraparound
- output stalls
- simultaneous transfers
- input transactions
- output transactions

Note that input transactions will not equal output transactions if the FIFO is reset while not empty.

Functional coverage is tracked using explicit testbench counters and flags rather than SystemVerilog covergroups.
#### Verification Validation
To verify that the testbench detects real design failures, the following bugs were injected.

A stuck write pointer caused data corruption and was detected by the scoreboard's expected output data check.

Incorrect occupancy counter modification on dequeue transaction was detected by the directed single-entry test since the FIFO did not return to an empty state when the expected queue was empty.

These bugs were removed after verification.
#### Synthesis
The FIFO was synthesized with Yosys.

For `DATA_WIDTH=32` and `DEPTH=4`, the design contains 128 bits of FIFO storage. Yosys synthesized this design into enabled flip-flops rather than using a memory primitive. Additional sequential logic implements FIFO state such as the read and write pointers as well as the occupancy counter, with combinational MUX and control logic supporting the state elements.  
## Verification Infrastructure
The project uses reusable verification infrastructure rather than relying exclusively on design-specific directed tests.
### Reference Model and Scoreboard
Testbench automatically maintains an independent model of expected DUT behavior. Expected transactions on the DUT update the reference model, and DUT outputs are compared against expected results.
### Assertions
SystemVerilog property assertions are used to continuously check interface and design invariants, including conditions such as data stability during stalls and legal FIFO boundary behavior.
### Randomized Testing
Random producers and consumers are told how many transactions to enact, but data sent is randomized and the producer/consumer decides whether or not to send/receive data on a given cycle. This generates varying transaction and stall patterns to cover interactions that directed tests may not target.
### Seeded Regression
The regression script executes the verification flow across multiple seeds, providing greater diversity of state and transaction sequences while preserving reproducibility of failures. The script stops upon the failure of any seed, and each seed's output is logged individually.
### Coverage
Coverage is used to distinguish successful assertions from actually verifying edge behaviors. The current environment uses explicit counters and flags to confirm that critical states and transitions occurred.
## Running the Flow
Run simulation and synthesis for an individual module:

```bash
./scripts/run_flow.sh sync_fifo 1
```

Run a multi-seed regression:

```bash
./scripts/run_regression.sh sync_fifo 10
```

The second argument specifies the random seed for `run_flow.sh` or the number of seeds for `run_regression.sh`. Both default to 1.
## Current Status
Completed:
- ready/valid one-entry buffer
- parameterized synchronous FIFO with directed and randomized verification, assertions, coverage tracking, multi-seed regression, bug injection, and synthesis inspection
