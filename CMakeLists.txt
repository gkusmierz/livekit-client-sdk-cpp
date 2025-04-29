cmake_minimum_required(VERSION 3.16)
project(livekit-client-sdk-cpp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(FFI_PROTO_PATH client-sdk-rust/livekit-ffi/protocol)
set(FFI_PROTO_FILES
    ${FFI_PROTO_PATH}/handle.proto
    ${FFI_PROTO_PATH}/ffi.proto
    ${FFI_PROTO_PATH}/participant.proto
    ${FFI_PROTO_PATH}/room.proto
    ${FFI_PROTO_PATH}/track.proto
    ${FFI_PROTO_PATH}/video_frame.proto
    ${FFI_PROTO_PATH}/audio_frame.proto
)
set(PROTO_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${PROTO_BINARY_DIR})

find_package(Protobuf REQUIRED)
find_package(absl REQUIRED) # Find Abseil separately

# livekit-proto
add_library(livekit_proto OBJECT ${FFI_PROTO_FILES})
# Link against protobuf and its Abseil dependencies (found separately)
target_link_libraries(livekit_proto PUBLIC
    protobuf::libprotobuf
    # Use standard Abseil targets now that we found the package
    absl::base
    absl::strings
    absl::log
    absl::synchronization
    # Add targets potentially containing CHECK macro implementations
    absl::check
    absl::raw_logging_internal # Might contain parts of log_internal
)
target_include_directories(livekit_proto PUBLIC "$<BUILD_INTERFACE:${PROTO_BINARY_DIR}>")
target_include_directories(livekit_proto PUBLIC ${Protobuf_INCLUDE_DIRS})

protobuf_generate(
    TARGET livekit_proto
    PROTOS ${FFI_PROTO_FILES} 
    APPEND_PATH ${FFI_PROTO_PATH}
    IMPORT_DIRS ${FFI_PROTO_PATH}
    PROTOC_OUT_DIR ${PROTO_BINARY_DIR}
)

# --- Build Rust FFI Library ---
find_program(CARGO_EXECUTABLE cargo REQUIRED)

set(RUST_PROJECT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/client-sdk-rust)
set(RUST_FFI_CRATE_DIR ${RUST_PROJECT_DIR}/livekit-ffi) # Adjust if Cargo.toml is elsewhere
set(RUST_TARGET_DIR ${RUST_PROJECT_DIR}/target) # Standard Cargo workspace target dir
set(RUST_PROFILE release) # Or debug
set(RUST_OUTPUT_DIR ${RUST_TARGET_DIR}/${RUST_PROFILE})

# Define the expected library name and path (adjust if needed based on Cargo.toml)
if(WIN32)
    set(LIVEKIT_FFI_LIB_NAME livekit_ffi.lib) # Assuming crate name is livekit-ffi -> livekit_ffi
else()
    set(LIVEKIT_FFI_LIB_NAME liblivekit_ffi.a) # Assuming staticlib output
endif()
set(LIVEKIT_FFI_LIB_PATH ${RUST_OUTPUT_DIR}/${LIVEKIT_FFI_LIB_NAME})

# Custom command to build the Rust FFI library
# Note: Assumes cargo build is run from the workspace root (RUST_PROJECT_DIR)
# and the livekit-ffi crate is part of that workspace.
add_custom_command(
    OUTPUT ${LIVEKIT_FFI_LIB_PATH}
    COMMAND ${CARGO_EXECUTABLE} build --release --package livekit-ffi # Build only the ffi package
    WORKING_DIRECTORY ${RUST_PROJECT_DIR}
    COMMENT "Building Rust FFI library (livekit_ffi)"
    VERBATIM
    DEPENDS ${RUST_FFI_CRATE_DIR}/src/lib.rs # Rebuild if source changes (adjust path if needed)
            ${RUST_FFI_CRATE_DIR}/Cargo.toml
)

# Custom target that executes the build command
add_custom_target(
    BuildRustFFI ALL # Build automatically as part of the default build
    DEPENDS ${LIVEKIT_FFI_LIB_PATH}
)

# Define an IMPORTED library target for livekit_ffi
add_library(livekit_ffi STATIC IMPORTED GLOBAL) # GLOBAL makes it visible in subdirs
set_target_properties(livekit_ffi PROPERTIES
    IMPORTED_LOCATION ${LIVEKIT_FFI_LIB_PATH}
)
# Ensure the imported target depends on the custom target that builds the lib
add_dependencies(livekit_ffi BuildRustFFI)
# --- End Build Rust FFI Library ---

# livekit
add_library(livekit 
    include/livekit/room.h
    include/livekit/ffi_client.h
    include/livekit/livekit.h
    src/ffi_client.cpp
    src/room.cpp
    ${PROTO_SRCS} 
    ${PROTO_HEADERS}
    ${PROTO_FILES}
)

# Include the auto-generated files from livekit-ffi (C headers)
target_include_directories(livekit PUBLIC client-sdk-rust/livekit-ffi/include/)
target_include_directories(livekit PUBLIC include/)

# Link against livekit-ffi (now an imported target) and livekit_proto
target_link_libraries(livekit PUBLIC livekit_ffi livekit_proto)

# Add platform-specific dependencies for livekit_ffi
if(APPLE)
    # Frameworks required by webrtc-sys/livekit_ffi on macOS
    target_link_libraries(livekit PUBLIC
        "-framework CoreFoundation"
        "-framework Foundation"
        "-framework Security"
        "-framework CoreAudio"
        "-framework AudioToolbox"
        "-framework CoreMedia"
        "-framework CoreVideo"
        "-framework VideoToolbox"
        "-framework CoreGraphics"
        "-framework AVFoundation" # Might be needed
    )
endif()

# Ensure the C++ library depends on the Rust build finishing
add_dependencies(livekit BuildRustFFI)

# Option to build examples
option(LK_BUILD_EXAMPLES "Build the example projects" ON)

# Conditionally add examples
if(LK_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
