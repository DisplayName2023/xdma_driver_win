# XDMA Windows Kernel Driver (sys/)

This directory contains the Windows kernel-mode driver implementation for Xilinx's XDMA (DMA/Bridge Subsystem for PCI Express) IP core. The driver provides a comprehensive interface between user applications and the XDMA hardware through Windows Driver Framework (WDF).

## Architecture Overview

The kernel driver consists of several key components:

- **Driver Core** (`driver.c/driver.h`): Main driver entry point and device management
- **File I/O Interface** (`file_io.c/file_io.h`): Device node operations and I/O request handling
- **Installation File** (`XDMA.inx`): Driver installation configuration
- **Tracing Support** (`trace.h`): Windows Performance Toolkit (WPP) tracing definitions

## Core Kernel Functions

### Driver Entry Point and Lifecycle (`driver.c`)

#### **DriverEntry()** - `driver.c:130`
**Purpose**: Main driver entry point called during driver loading
**MSDN Reference**: [DriverEntry](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nc-wdm-driver_initialize)
- Initializes WPP tracing for debugging
- Configures WDF driver object with device add callback
- Creates the main WDFDRIVER object
- Sets up driver unload callback

#### **DriverUnload()** - `driver.c:168`
**Purpose**: Called before driver removal to clean up resources
**MSDN Reference**: [DriverUnload](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nc-wdm-driver_unload)
- Performs WPP tracing cleanup
- Executes in pageable memory for efficiency

#### **EvtDeviceAdd()** - `driver.c:178`
**Purpose**: WDF callback when a new XDMA device is detected
**MSDN Reference**: [EVT_WDF_DRIVER_DEVICE_ADD](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdriver/nc-wdfdriver-evt_wdf_driver_device_add)
- Configures device for Direct I/O operations
- Sets up PnP power event callbacks
- Registers file object creation/cleanup callbacks
- Creates default I/O queue with parallel dispatch
- Establishes device interface for user applications

#### **EvtDevicePrepareHardware()** - `driver.c:266`
**Purpose**: Initializes hardware resources when device starts
**MSDN Reference**: [EVT_WDF_DEVICE_PREPARE_HARDWARE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_device_prepare_hardware)
- Opens XDMA device using libxdma
- Configures polling vs interrupt mode from registry
- Creates dedicated I/O queues for each DMA engine
- Initializes user event interrupt handling

#### **EvtDeviceReleaseHardware()** - `driver.c:323`
**Purpose**: Releases hardware resources when device stops
**MSDN Reference**: [EVT_WDF_DEVICE_RELEASE_HARDWARE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_device_release_hardware)
- Unmaps PCIe resources
- Closes XDMA device handle

#### **EngineCreateQueue()** - `driver.c:338`
**Purpose**: Creates dedicated I/O queue for each DMA engine
- Configures sequential dispatch for proper DMA ordering
- Sets appropriate read/write callbacks based on engine direction
- Associates engine context with queue for later operations

#### **GetPollModeParameter()** - `driver.c:100`
**Purpose**: Reads polling mode configuration from Windows registry
- Opens driver parameters registry key
- Retrieves POLL_MODE setting (0=interrupts, 1=polling)
- Used to configure DMA completion detection method

### Device Node Management (`file_io.c`)

#### **EvtDeviceFileCreate()** - `file_io.c:141`
**Purpose**: Handles creation of device node file objects
**MSDN Reference**: [EVT_WDF_DEVICE_FILE_CREATE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_device_file_create)
- Parses filename to determine device node type (control, user, bypass, DMA channels, events)
- Validates requested device node exists and is enabled
- Associates appropriate hardware resources (BARs, engines, events) with file context
- Configures interrupt vs polling mode for DMA engines

#### **EvtFileClose()** - `file_io.c:226`
**Purpose**: Cleanup when file handle is closed
**MSDN Reference**: [EVT_WDF_FILE_CLOSE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_file_close)

#### **EvtFileCleanup()** - `file_io.c:231`
**Purpose**: Final cleanup before file object destruction
**MSDN Reference**: [EVT_WDF_FILE_CLEANUP](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_file_cleanup)
- Tears down streaming DMA ring buffers for AXI-ST engines
- Unmaps and frees Memory Descriptor Lists (MDLs) for BAR mappings

### I/O Request Processing (`file_io.c`)

#### **EvtIoRead()** - `file_io.c:358`
**Purpose**: Handles read operations on device nodes
**MSDN Reference**: [EVT_WDF_IO_QUEUE_IO_READ](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfio/nc-wdfio-evt_wdf_io_queue_io_read)
- **Control/User/Bypass**: Direct BAR memory reads using `ReadBarToRequest()`
- **Events**: User interrupt waiting via `EvtReadUserEvent()`
- **C2H Channels**: Forwards to DMA read queue for hardware transfer

#### **EvtIoWrite()** - `file_io.c:408`
**Purpose**: Handles write operations on device nodes
**MSDN Reference**: [EVT_WDF_IO_QUEUE_IO_WRITE](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfio/nc-wdfio-evt_wdf_io_queue_io_write)
- **Control/User/Bypass**: Direct BAR memory writes using `WriteBarFromRequest()`
- **H2C Channels**: Forwards to DMA write queue for hardware transfer

#### **EvtIoDeviceControl()** - `file_io.c:774`
**Purpose**: Handles IOCTL (Device Control) operations
**MSDN Reference**: [EVT_WDF_IO_QUEUE_IO_DEVICE_CONTROL](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdfio/nc-wdfio-evt_wdf_io_queue_io_device_control)
- **BAR Operations**: Keyhole register writes, BAR mapping to user space
- **DMA Channel Control**: Performance monitoring, address mode configuration

### DMA Transfer Handlers (`file_io.c`)

#### **EvtIoReadDma()** - `file_io.c:853`
**Purpose**: Processes DMA read transfers (Card-to-Host)
**MSDN Reference**: [WDF DMA Operations](https://docs.microsoft.com/en-us/windows-hardware/drivers/wdf/handling-dma-operations-in-kmdf-drivers)
- Initializes WDF DMA transaction from I/O request
- Programs DMA engine via `XDMA_EngineProgramDma()`
- Executes transfer with proper synchronization

#### **EvtIoWriteDma()** - `file_io.c:816`
**Purpose**: Processes DMA write transfers (Host-to-Card)
- Configures DMA transaction for write-to-device direction
- Handles scatter-gather list generation for large transfers
- Manages transfer completion and error handling

#### **EvtIoReadEngineRing()** - `file_io.c:890`
**Purpose**: Handles streaming DMA reads using ring buffers
- Specifically for AXI-ST (streaming) configured engines
- Copies data from hardware ring buffer to user memory
- Implements timeout-based data availability waiting

### Memory and BAR Access Functions (`file_io.c`)

#### **ReadBarToRequest()** - `file_io.c:272`
**Purpose**: Reads from PCIe BAR memory into I/O request buffer
- Validates BAR access bounds via `ValidateBarParams()`
- Uses hardware-specific register access functions:
  - `READ_REGISTER_BUFFER_ULONG()` for 32-bit aligned access
  - `READ_REGISTER_BUFFER_USHORT()` for 16-bit aligned access
  - `READ_REGISTER_BUFFER_UCHAR()` for byte access

#### **WriteBarFromRequest()** - `file_io.c:314`
**Purpose**: Writes from I/O request buffer to PCIe BAR memory
- Performs similar alignment-based optimizations for writes
- Uses `WRITE_REGISTER_BUFFER_*()` functions for hardware access

### IOCTL Implementation Functions (`file_io.c`)

#### **IoctlGetPerf()** - `file_io.c:450`
**Purpose**: Retrieves DMA engine performance counters
- Returns `XDMA_PERF_DATA` structure with cycle counts
- Used for bandwidth and latency measurements

#### **IoctlSetAddrMode()`/`IoctlGetAddrMode()** - `file_io.c:474,498`
**Purpose**: Controls DMA addressing mode (incrementing vs fixed)
- Configures `XDMA_CTRL_NON_INCR_ADDR` register bit
- Enables keyhole/FIFO style transfers when non-incrementing

#### **IoctlMapBar()** - `file_io.c:530`
**Purpose**: Maps BAR memory into user process address space
**MSDN Reference**: [MmMapLockedPagesSpecifyCache](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-mmmaplockedpagesspecifycache)
- Creates Memory Descriptor List (MDL) for BAR region
- Maps to user mode with non-cached access
- Returns virtual address and length to application

#### **IoctlKeyholeWriteRegister()** - `file_io.c:605`
**Purpose**: Performs repeated writes to a single register address
- Useful for writing arrays to hardware FIFOs
- Writes multiple 32-bit values without address increment

### Event Handling (`file_io.c`)

#### **EvtReadUserEvent()** - `file_io.c:927`
**Purpose**: Waits for user interrupt events
- Implements 3-second timeout for event waiting
- Uses Windows kernel event objects (`KeWaitForSingleObject`)
- Returns boolean indicating event occurrence

#### **HandleUserEvent()** - `file_io.c:987`
**Purpose**: Interrupt handler callback for user events
- Called from libxdma interrupt context
- Signals waiting threads via `KePulseEvent()`

### Utility Functions (`file_io.c`)

#### **GetDevNodeType()** - `file_io.c:127`
**Purpose**: Converts filename string to device node type enumeration
- Maps filenames like "\\h2c_0" to `DEVNODE_TYPE_H2C`
- Determines channel/event index from filename

#### **ValidateBarParams()** - `file_io.c:251`
**Purpose**: Validates BAR access parameters for bounds checking
- Prevents access beyond BAR memory boundaries
- Ensures non-zero transfer lengths

## Device Node Types

The driver exposes multiple device node types, each with specific behaviors:

### **DEVNODE_TYPE_CONTROL**
- **Purpose**: Access to XDMA control registers
- **Operations**: Read/Write for configuration and status
- **BAR**: Uses configuration BAR (typically BAR0)

### **DEVNODE_TYPE_USER**
- **Purpose**: Access to user logic memory space
- **Operations**: Read/Write for application-specific registers
- **BAR**: Uses user BAR if available

### **DEVNODE_TYPE_BYPASS**
- **Purpose**: Access to bypass BAR for descriptor operations
- **Operations**: Read/Write for direct descriptor access
- **BAR**: Uses bypass BAR if available

### **DEVNODE_TYPE_H2C** (Host-to-Card)
- **Purpose**: DMA write operations from host to device
- **Operations**: Write operations trigger DMA transfers
- **Channels**: h2c_0 through h2c_3

### **DEVNODE_TYPE_C2H** (Card-to-Host)
- **Purpose**: DMA read operations from device to host
- **Operations**: Read operations trigger DMA transfers
- **Channels**: c2h_0 through c2h_3
- **Special**: AXI-ST engines use ring buffer mode

### **DEVNODE_TYPE_EVENTS**
- **Purpose**: User interrupt event notifications
- **Operations**: Read operations wait for interrupts
- **Events**: event_0 through event_15

## Driver Installation Configuration (`XDMA.inx`)

The `.inx` file defines driver installation parameters:

### **Supported Hardware IDs**
- **Xilinx Vendor ID**: 0x10ee
- **Device IDs**: 0x9011, 0x9012, 0x9014, 0x9018, 0x9021, 0x9022, 0x9024, 0x9028, 0x9031, 0x9032, 0x9034, 0x9038, 0x903f
- **Additional Series**: 0x8xxx and 0x7xxx device variants

### **MSI/MSI-X Interrupt Configuration**
- **Message Limit**: 32 interrupts maximum
- **Usage**: 16 for user events + 8 for DMA channels + extras for control
- **Registry Setting**: `MessageNumberLimit=32`

### **Driver Parameters**
- **POLL_MODE**: Registry parameter controlling interrupt vs polling mode
  - `0` = Interrupt-driven (default)
  - `1` = Hardware polling mode

## Tracing and Debugging (`trace.h`)

### **WPP Tracing Categories**
- **DBG_INIT** (0x01): Driver initialization and device setup
- **DBG_IRQ** (0x02): Interrupt handling events
- **DBG_DMA** (0x04): DMA operation tracing
- **DBG_DESC** (0x08): Descriptor ring operations
- **DBG_USER** (0x10): User application interactions
- **DBG_IO** (0x14): I/O request processing

### **Trace Functions**
- **TraceError()**: Critical errors requiring attention
- **TraceWarning()**: Warning conditions
- **TraceInfo()**: General informational messages
- **TraceVerbose()**: Detailed debugging information

### **Trace GUID**
- **Logger Name**: XdmaDrvTraceGuid
- **GUID**: `{7dd02079-3c3f-4dda-9384-c210c7cc490a}`

## Key Data Structures

### **DeviceContext** (`driver.h:65`)
```c
typedef struct DeviceContext_t {
    XDMA_DEVICE xdma;                               // Core XDMA device state
    WDFQUEUE engineQueue[2][XDMA_MAX_NUM_CHANNELS]; // Per-engine I/O queues
    KEVENT eventSignals[XDMA_MAX_USER_IRQ];         // User event signals
} DeviceContext;
```

### **FILE_CONTEXT** (`file_io.h:81`)
```c
typedef struct _FILE_CONTEXT {
    DEVNODE_TYPE devType;        // Type of device node
    union {
        void* bar;              // For BAR access nodes
        XDMA_EVENT* event;      // For event nodes
        XDMA_ENGINE* engine;    // For DMA channel nodes
    } u;
    WDFQUEUE queue;             // Associated I/O queue
    PMDL mdl;                   // For BAR memory mapping
    PVOID virtAddress;          // Mapped virtual address
} FILE_CONTEXT;
```

### **QUEUE_CONTEXT** (`file_io.h:95`)
```c
typedef struct _QUEUE_CONTEXT {
    XDMA_ENGINE* engine;        // Associated DMA engine
} QUEUE_CONTEXT;
```

## Error Handling and Status Codes

The driver uses standard Windows NTSTATUS codes:

- **STATUS_SUCCESS**: Operation completed successfully
- **STATUS_INVALID_PARAMETER**: Invalid input parameters
- **STATUS_INVALID_DEVICE_REQUEST**: Unsupported operation for device node
- **STATUS_INSUFFICIENT_RESOURCES**: Memory allocation failure
- **STATUS_TIMEOUT**: Operation timed out (user events)
- **STATUS_CANCELLED**: Operation was cancelled

## Performance Considerations

### **Direct I/O Mode**
- Configured via `WdfDeviceInitSetIoType(DeviceInit, WdfDeviceIoDirect)`
- Eliminates intermediate buffer copies for large transfers
- Requires page-aligned buffers for optimal performance

### **Parallel vs Sequential Queues**
- **Default Queue**: Parallel dispatch for multiple concurrent requests
- **Engine Queues**: Sequential dispatch to maintain DMA ordering
- **Synchronization**: WDF handles queue synchronization automatically

### **Polling vs Interrupt Mode**
- **Interrupt Mode**: Lower CPU usage, higher latency
- **Polling Mode**: Higher CPU usage, lower latency
- **Configuration**: Set via POLL_MODE registry parameter

## Integration with libxdma

The kernel driver is tightly integrated with the libxdma static library:

- **Device Operations**: `XDMA_DeviceOpen()`, `XDMA_DeviceClose()`
- **DMA Programming**: `XDMA_EngineProgramDma()`
- **Interrupt Handling**: `XDMA_UserIsrRegister()`
- **Performance Monitoring**: `EngineGetPerf()`, `EngineStartPerf()`

This architecture provides a clean separation between Windows-specific driver framework code and portable XDMA hardware abstraction.