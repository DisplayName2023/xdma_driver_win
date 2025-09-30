# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is Xilinx's sample Windows driver for the 'DMA/Bridge Subsystem for PCI Express v4.0' (XDMA) IP. The project provides a complete Windows kernel driver implementation along with sample applications and utilities for testing and development.

## Build Commands

### Visual Studio Build
- Open `XDMA.sln` in Visual Studio 2015 or later
- Build Solution from the Build menu
- Supports multiple configurations: Debug/Release for Win7/Win10, x86/x64

### Command Line Build
```bash
# Open Developer Command Prompt for VS2015 (or later)
msbuild /t:clean /t:build XDMA.sln
```

## Project Architecture

### Core Components

#### Driver Layer (`sys/`)
- **XDMA_Driver**: Main Windows kernel driver
- **driver.c/driver.h**: Core driver implementation with device initialization and cleanup
- **file_io.c/file_io.h**: Device node file operations (read/write/ioctl)
- **XDMA.inx**: Driver installation file template

#### Kernel Library (`libxdma/`)
- **Static kernel library** providing core XDMA functionality
- **device.c/device.h**: Device enumeration and management
- **dma_engine.c/dma_engine.h**: DMA transfer engine implementation
- **interrupt.c/interrupt.h**: Interrupt handling for DMA completions
- **xdma.h**: Main library header with core data structures
- **reg.h**: Hardware register definitions
- **pcie_common.h**: PCIe-specific definitions

#### Public API (`inc/`)
- **xdma_public.h**: Public API header for user applications
- Defines device node paths, IOCTLs, and data structures
- GUID definitions for device interface

### Device Node Architecture
The driver exposes multiple device nodes for different operations:
- **control**: Register access and configuration
- **user**: User memory space access
- **bypass**: Bypass mode access
- **h2c_0-3**: Host-to-Card DMA channels
- **c2h_0-3**: Card-to-Host DMA channels
- **event_0-15**: User event interrupts

### Sample Applications (`exe/`)

#### Core Examples
- **simple_dma**: Basic DMA transfer demonstrations
- **streaming_dma**: AXI-ST streaming mode examples
- **user_event**: Event interrupt handling examples

#### Utilities
- **xdma_test**: Comprehensive test suite for all DMA channels
- **xdma_info**: Hardware configuration reporting tool
- **xdma_rw**: General-purpose read/write utility for device nodes

### Build Output Structure
Compiled binaries are placed in `build/ARCH/`:
- `bin/`: Sample applications and utilities
- `libxdma/`: Static kernel library
- `XDMA_Driver/`: Driver installation package

## Development Notes

### Driver Configurations
- Multiple platform targets: Win7/Win10, x86/x64
- Debug and Release configurations available
- Poll mode can be enabled by modifying XDMA.inf registry settings

### Key Architectural Patterns
- **Layered approach**: Applications → Public API → Driver → libxdma → Hardware
- **Channel-based DMA**: Separate device nodes for each DMA channel
- **Interrupt vs Polling**: Configurable completion detection methods
- **Memory mapping**: Direct BAR access through user/control device nodes

### Testing
Use the provided sample applications for validation:
- Run `xdma_test.exe` for comprehensive channel testing
- Use `xdma_info.exe` to verify hardware configuration
- `xdma_rw.exe` for manual register/memory access testing