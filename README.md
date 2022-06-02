# microbit-v2-led-roulette
LED roulette in a Microbit V2 board. We use this repo as an example of
setting an embedded Rust project from zero.
We will use the [microbit V2 board](https://microbit.org/new-microbit/) in this example. However, you can
adapt it for your favorite platform.

## Requirements
You will need these tools:
- Rust 1.57 or greater. You can install it [here](https://www.rust-lang.org/tools/install).
- `gdb-multiarch` (or `arm-none-eabi-gdb`):
    - Click [here](https://docs.rust-embedded.org/discovery/microbit/03-setup/linux.html) for installing it on Linux.
    - Click [here](https://docs.rust-embedded.org/discovery/microbit/03-setup/macos.html) for installing it on MacOS.
    - Click [here](https://docs.rust-embedded.org/discovery/microbit/03-setup/windows.html) for installing it on Windows.
- `cargo-binutils`:
```shell
rustup component add llvm-tools-preview
cargo install cargo-binutils
```
For flashing/debugging our project, we have two options:
- `openocd`: Open Software project for developing embedded system. This is
not specific for Rust, and there are a lot of auxiliary software to make our
life easier (e.g., debugging on VS Code). However, it is complex to configure.
If you go with this option, you need to install [OpenOCD])(https://openocd.org/pages/getting-openocd.html).
- `cargo-embed`: Rust tool for the development of embedded systems. This is
way easier to use, but it is still under development, and some features are
not supported yet (e.g., ITM communication or integration with VS Code).
If you prefer this option, just run the following command in your terminal:
```shell
cargo install cargo-embed
```

## Creating a New Embedded Rust Project
- Generate a new Cargo project:
```shell
cargo new blink-led
cd blink-led
```
- Install the Rust compiler for your platform. THIS DEPENDS ON YOUR BOARD.
As we are using the microbit V2 board, we need the `thumbv7em-none-eabihf` target:
```shell
$ rustup target add thumbv7em-none-eabihf
```
- Configure the initial debug session. We do so in the `openocd.gdb` file:
```
target remote :3333
break main
continue
```
- Configure Cargo tu use the correct target and launch the debugger.
We do so in the `.cargo/config` file:
```shell
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "arm-none-eabi-gdb -q -x openocd.gdb"
rustflags = [
  "-C", "link-arg=-Tlink.x",
]

[build]
target = "thumbv7em-none-eabihf"
```
- If we are using OpenOCD, we setup the OpenOCD session in `openocd.cfg`:
```shell
source [find interface/cmsis-dap.cfg]  # Change this to match your target
source [find target/nrf52.cfg]         # Change this to match your target
```
- If we are using `cargo-embed`, we setup the session in `Embed.toml`:
```shell
[default.general]
chip = "nrf52833_xxAA"   # change this to match your target

[default.reset]
halt_afterwards = false   # If you are using RTT, keep this set to false

[default.rtt]
enabled = true          # for sending text to a debugger

[default.gdb]
enabled = false           # for debugging. RTT and GDB must be used separately
gdb_connection_string = "127.0.0.1:3333"
```
-
