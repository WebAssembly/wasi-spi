# WASI SPI

A proposed [WebAssembly System Interface](https://github.com/WebAssembly/WASI) API for Serial Peripheral Interface (SPI) communication.

### Current Phase

Phase 1

### Champions

- [@merlijn-sebrechts](https://github.com/merlijn-sebrechts)

### Phase 4 Advancement Criteria

TODO before entering Phase 2.


### Introduction

WASI SPI is a capability-based API that allows WebAssembly guests to interact with Serial Peripheral Interface (SPI) devices. Heavily inspired by the Rust embedded-hal ecosystem, this proposal prioritizes guest portability by delegating hardware-specific configuration (like baud rates, clock phases, and pin mappings) not covered by embedded-hal to the host environment. Guests simply request a logical device by name and perform standard reads, writes, transfers, and transactions.

### Goals

- Device Communication: Enable WebAssembly guests to exchange data with SPI peripherals (e.g., sensors, displays, flash memory) exposed by the host environment.
- Guest Portability: Wasm modules remain hardware-agnostic. A guest compiled once should run on any host (from embedded microcontrollers to Linux SBCs) without needing recompilation or awareness of the underlying bus speeds and pin mappings.

### Non-goals

- Manual Bus Configuration: Allowing the guest to dynamically configure hardware-specific parameters (like baud rates, CPOL, CPHA, or pinouts) or manually assert or de-assert the CS line.

### API walk-through

#### Reading and parsing data from a configured sensor

```Rust
use bindings::wasi::spi::spi;

fn read_temperature() -> Result<f32, spi::Error> {
    let sensor = spi::open("temperature-sensor")?;
    let data = sensor.read(2)?;
    Ok((data as f32) * 0.01)
}
```

#### Executing a CS-enforced transaction

```Rust
use bindings::wasi::spi::spi::{self, Operation, OperationResult};

fn read_display_controller_id() -> Result<u8, spi::Error> {
    let display = spi::open("spi-display")?;
    
    // The host guarantees CS stays low for the entire sequence.
    let results = display.transaction(&[
        Operation::Write(vec![0x04]), // 'Read Display ID' command
        Operation::DelayNs(1500),     // Hardware requires 1.5us delay to prepare data
        Operation::Read(1),           // Read the 1-byte ID
    ])?;
    
    // Extract and return the data from the final Read operation
    if let Some(OperationResult::Read(id_bytes)) = results.last() {
        return Ok(id_bytes[0]);
    }
    
    Err(spi::Error::Other("Unexpected transaction result".into()))
}
```

### Detailed design discussion

#### Alignment with `embedded-hal SpiDevice`

This proposal is heavily modeled after the SPI traits defined in Rust's [embedded-hal 1.0](https://docs.rs/embedded-hal/latest/embedded_hal/spi/index.html). The `embedded-hal` ecosystem defines hardware-agnostic abstractions for embedded systems, making it the good blueprint for a portable Wasm interface. 

Many concepts map 1-to-1 from `embedded-hal` to this WIT proposal:

* **Base Operations:** The discrete `read`, `write`, and `transfer` functions mirror the `embedded_hal::spi::SpiDevice` trait.

* **Error Handling:** The `error` variant in WIT directly maps to `embedded_hal::spi::ErrorKind` (e.g. `overrun`, `mode-fault`, `frame-format`, `chip-select-fault`).

* **Transactions:** The concept of grouping `Operation`s (Reads, Writes, Transfers, and Delays) into a single hardware-enforced transaction is preserved to guarantee Chip Select (CS) integrity.

#### Divergences and the Component Model ABI

While the conceptual model matches `embedded-hal`, the Wasm Component Model's canonical ABI necessitates strict **pass-by-value** semantics, forcing a divergence from Rust's native memory management.

**1. Mutable References vs. Pass-by-Value**

In native Rust, `embedded-hal` operations are zero-allocation and zero-copy. The caller provides mutable references to buffers (`&mut [u8]`), and the hardware driver fills them in-place:

```rust
// embedded-hal (Rust)
fn read(&mut self, buf: &mut [u8]) -> Result<(), Error>;
```

Because Wasm should not share mutable pointers to guest memory across the host boundary, this WIT interface must allocate and return data by value:

```wit
// wasi:spi (WIT)
read: func(len: u64) -> result<list<u8>, error>;
```

**2. Transaction Results**

In `embedded-hal`, a transaction takes an array of mutable `Operation` enums. Data read during the transaction is written directly into the buffers provided inside those enums. 

To achieve this in Wasm, the `transaction` function takes a `list<operation>` and returns a newly allocated `list<operation-result>` containing the fetched data. While this incurs allocation overhead on the host, it is a trade-off to preserve the pass-by-value semantics of wasm.

**3. Omission of `TransferInPlace`**

`embedded-hal` includes an `Operation::TransferInPlace(&mut [u8])` variant, which uses a single buffer for both transmitting and receiving to save memory. Without shared mutable memory over the Wasm boundary, implementing this in WIT would just result in a standard `transfer` under the hood. It was omitted to keep the API surface minimal.

### Considered alternatives

[This section is not required if you already covered considered alternatives in the design discussion above.]

#### [Alternative 1]

[Describe an alternative which was considered, and why you decided against it.]

#### [Alternative 2]

[etc.]

### Stakeholder Interest & Feedback

TODO before entering Phase 3.

[This should include a list of implementers who have expressed interest in implementing the proposal]

### References & acknowledgements

Many thanks for valuable feedback and advice from:

- [@merlijn-sebrechts](https://github.com/merlijn-sebrechts)
- [@Michielvk](https://github.com/Michielvk)
- [@Zelzahn](https://github.com/Zelzahn)

This work has been partially supported by the ELASTIC project, which received funding from the Smart Networks and Services Joint Undertaking (SNS JU) under the European Union’s Horizon Europe research and innovation programme under Grant Agreement No 101139067. Views and opinions expressed are however those of the author(s) only and do not necessarily reflect those of the European Union. Neither the European Union nor the granting authority can be held responsible for them. This funding supported individual contributor organisations, not the W3C or this community group as a whole.
