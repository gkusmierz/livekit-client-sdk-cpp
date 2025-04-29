# LiveKit C++ Client SDK

This repository contains the C++ client SDK for LiveKit.

## Prerequisites

Before building the SDK, ensure you have the following installed:

1.  **C++ Compiler:** A modern C++ compiler supporting C++17 (e.g., GCC, Clang, MSVC).
2.  **CMake:** Version 3.16 or higher. ([https://cmake.org/download/](https://cmake.org/download/))
3.  **Rust Toolchain:** Install Rust and Cargo. ([https://rustup.rs/](https://rustup.rs/)) The build process uses Cargo to build the underlying Rust FFI library.
4.  **Protobuf Compiler and Libraries:** The `protoc` compiler and the Protobuf development libraries are required.
    *   On macOS (using Homebrew): `brew install protobuf`
    *   On Debian/Ubuntu: `sudo apt update && sudo apt install -y protobuf-compiler libprotobuf-dev`
    *   Other systems: Follow the official Protobuf installation guide. ([https://protobuf.dev/getting-started/cpp-quickstart/](https://protobuf.dev/getting-started/cpp-quickstart/))
5.  **Abseil Libraries:** Abseil is a dependency of Protobuf and is needed for linking.
    *   On macOS (using Homebrew): `brew install abseil`
    *   On Debian/Ubuntu: `sudo apt update && sudo apt install -y libabsl-dev`
    *   Other systems: Follow the official Abseil installation guide or ensure your Protobuf installation includes it correctly.

## Building

1.  **Clone the repository:**
    ```bash
    git clone <repository-url>
    cd livekit-client-sdk-cpp
    ```

2.  **Configure with CMake:** Create a build directory and run CMake. Building examples is optional but recommended.
    ```bash
    mkdir build
    cd build
    cmake .. -DLK_BUILD_EXAMPLES=ON
    ```
    *   This step will also trigger `cargo build` for the Rust FFI component.

3.  **Build the project:**
    ```bash
    cmake --build .
    ```
    *   This will compile the C++ code, link against the Rust FFI library, and build the examples if enabled.

## Running Examples

The compiled examples (e.g., `SimpleRoom`) will be located in the `build/examples/` directory.

```bash
./examples/SimpleRoom
```

**Note:** The examples require a valid LiveKit server URL and a JWT token with appropriate permissions to connect successfully. A segmentation fault might occur if invalid connection details are used or if there are issues during the connection process. Ensure you provide valid credentials when running the examples. You might need to modify the example source code to include your specific URL and token.
