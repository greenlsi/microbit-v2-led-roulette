# microbit-v2-led-roulette
LED roulette in a Microbit V2 board. We use this repo as an example of setting an embedded Rust project from zero. We will use the [microbit V2 board](https://microbit.org/new-microbit/) in this example. However, you can adapt it for your favorite platform.

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
- `openocd`: Open Software project for developing embedded system. This is not specific for Rust, and there are a lot of auxiliary software to make our life easier (e.g., debugging on VS Code). However, it is complex to configure. Hopefully, this repo reduces the complexity significantly. If you go with this option, you need to install [OpenOCD])(https://openocd.org/pages/getting-openocd.html).
- `cargo-embed`: Rust tool for the development of embedded systems. This is way easier to use, but it is still under development, and some features are not supported yet (e.g., ITM communication or integration with VS Code). If you prefer this option, just run the following command in your terminal:
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
load
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
target = "thumbv7em-none-eabihf"  # Change this to match your target
```
- If we are using OpenOCD, we setup the OpenOCD session in `openocd.cfg`:
```shell
source [find interface/cmsis-dap.cfg]  # Change this to match your target
source [find target/nrf52.cfg]         # Change this to match your target
```
- If we are using `cargo-embed`, we setup the session in `Embed.toml`:
```shell
[default.general]
chip = "nrf52833_xxAA"   # Change this to match your target

[default.reset]
halt_afterwards = false   # If you are using RTT, keep this set to false

[default.rtt]
enabled = true          # for sending text to a debugger

[default.gdb]
enabled = false           # for debugging. RTT and GDB must be used separately
gdb_connection_string = "127.0.0.1:3333"
```

## Using VS Code for Debugging
If you are familiar with VS Code, you can use it to develop your projects. You need the following additional dependencies:
- VS Code
- The Rust and Cortex-Debug plugins

Now, let's define a launch task in `.vscode/launch.json`:
```json
{
    "version": "0.2.0",
    "showDevDebugOutput": true,
    "configurations": [
        {
            "name": "Debug (Blinky)",
            "type": "cortex-debug",
            "preLaunchTask": "build",
            "request": "launch",
            "servertype": "openocd",
            "interface": "swd",
            "cwd": "${workspaceRoot}",
            "executable": "${workspaceRoot}/target/thumbv7em-none-eabihf/debug/blink-led",  // change to match your project
            "device": "nrf52833_01AA",  // Change to match your target
            "svdFile": "${workspaceRoot}/nrf52833.svd",  // Change to match your target
            "configFiles": [
                "interface/cmsis-dap.cfg",  // Change to match your target
                "target/nrf52.cfg"  // Change to match your target
            ],
            "rttConfig": {
                "enabled": true,  // Disable this if you don't want to use RTT to debug
                "address": "auto",
                "decoders": [
                    {
                        "port": 0,
                        "type": "console",
                        "label": "RTT0"
                    }
                ]
            },
            "swoConfig": {
                "enabled": false,  // Disable this if you don't wat to use ITM to debug
                "cpuFrequency": 8000000,  // Change to match your target
                "swoFrequency": 2000000,  // Change to match your target
                "source": "probe",
                "decoders": [
                    {
                        "port": 0,
                        "type": "console",
                        "label": "ITM0",
                    }
                ]
            },
            "runToEntryPoint": "main"
        }
    ]
}
```

The `tasks.json` file ensures that your project is built again before launching a new debug session.
 
 ### Issues with RTT and OpenOCD
 OpenOCD looks once for an RTT channel. And it does it before the RTT channel is created, so... it fails. There is a workaround to fix this:
 - Set a breakpoint right <it>after</it> calling the `rtt_init_print!()` function (in `src/main.rs`, we call this function in line 39).
 - Launch a debug session and, once your program stops, open the `gdb-server` debug console and enter the following command:
 ```shell
 monitor rtt start
 ```
 - Hopefully, OpenOCD will now detect the RTT channel. Continue with the execution and you will be able to see your log messages while debugging.
