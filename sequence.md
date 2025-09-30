# XDMA Driver Calling Sequences

This document describes the calling sequences for various operations in the XDMA Windows driver using Mermaid sequence diagrams.

## Driver Initialization Sequence

```mermaid
sequenceDiagram
    participant WM as Windows Manager
    participant DRV as DriverEntry
    participant WDF as WDF Framework
    participant DEV as EvtDeviceAdd
    participant HW as EvtDevicePrepareHardware
    participant XDMA as libxdma

    WM->>DRV: Driver Loading
    DRV->>DRV: WPP_INIT_TRACING()
    DRV->>WDF: WDF_DRIVER_CONFIG_INIT()
    DRV->>WDF: WdfDriverCreate()
    WDF->>DEV: Device Found
    DEV->>DEV: WdfDeviceInitSetIoType(Direct)
    DEV->>DEV: Setup PnP callbacks
    DEV->>DEV: Setup file object callbacks
    DEV->>WDF: WdfDeviceCreate()
    DEV->>WDF: WdfDeviceCreateDeviceInterface()
    DEV->>WDF: WdfIoQueueCreate(default queue)

    Note over WM,XDMA: Hardware Preparation
    WDF->>HW: Hardware detected
    HW->>XDMA: XDMA_DeviceOpen()
    XDMA-->>HW: Return channel counts
    HW->>HW: GetPollModeParameter()
    HW->>HW: Configure engines for poll/interrupt
    HW->>HW: EngineCreateQueue() for each engine
    HW->>XDMA: XDMA_UserIsrRegister() for events
```

## File Creation Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant FC as EvtDeviceFileCreate
    participant CTX as DeviceContext

    APP->>WIN: CreateFile("\\.\device\xdma\h2c_0")
    WIN->>DRV: IRP_MJ_CREATE
    DRV->>FC: EvtDeviceFileCreate()
    FC->>FC: WdfFileObjectGetFileName()
    FC->>FC: GetDevNodeType(fileName)

    alt H2C/C2H Channel
        FC->>CTX: Get engine from xdma.engines[ch][dir]
        FC->>FC: Validate engine.enabled
        FC->>FC: Setup ring buffer (if AXI-ST)
        FC->>FC: Configure interrupt/poll mode
        FC->>FC: Associate engine queue
    else Control/User/Bypass BAR
        FC->>CTX: Get BAR address from xdma.bar[]
        FC->>FC: Validate BAR exists
    else Event Node
        FC->>CTX: Get event from xdma.userEvents[]
    end

    FC->>WIN: WdfRequestComplete(STATUS_SUCCESS)
    WIN-->>APP: Handle returned
```

## DMA Write (H2C) Operation Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant IOW as EvtIoWrite
    participant DMA as EvtIoWriteDma
    participant XDMA as libxdma
    participant HW as Hardware

    APP->>WIN: WriteFile(h2c_handle, buffer, size)
    WIN->>DRV: IRP_MJ_WRITE
    DRV->>IOW: EvtIoWrite()
    IOW->>IOW: GetFileContext(WdfRequestGetFileObject)
    IOW->>IOW: Check devType == DEVNODE_TYPE_H2C
    IOW->>IOW: WdfRequestForwardToIoQueue(engine queue)

    Note over IOW,HW: Engine Queue Processing
    IOW->>DMA: EvtIoWriteDma()
    DMA->>DMA: GetQueueContext(wdfQueue)
    DMA->>DMA: WdfDmaTransactionInitializeUsingRequest()
    DMA->>DMA: WdfDmaTransactionExecute()
    DMA->>XDMA: XDMA_EngineProgramDma()

    XDMA->>HW: Program DMA descriptors
    XDMA->>HW: Start DMA engine
    HW->>HW: Transfer data

    alt Interrupt Mode
        HW->>XDMA: Generate interrupt
        XDMA->>DMA: DMA completion callback
    else Poll Mode
        XDMA->>XDMA: Poll for completion
    end

    XDMA->>DMA: Transfer complete
    DMA->>WIN: WdfRequestComplete()
    WIN-->>APP: WriteFile() returns
```

## DMA Read (C2H) Operation Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant IOR as EvtIoRead
    participant DMA as EvtIoReadDma/EvtIoReadEngineRing
    participant XDMA as libxdma
    participant HW as Hardware

    APP->>WIN: ReadFile(c2h_handle, buffer, size)
    WIN->>DRV: IRP_MJ_READ
    DRV->>IOR: EvtIoRead()
    IOR->>IOR: GetFileContext(WdfRequestGetFileObject)
    IOR->>IOR: Check devType == DEVNODE_TYPE_C2H
    IOR->>IOR: WdfRequestForwardToIoQueue(engine queue)

    alt Memory-Mapped (MM) Engine
        IOR->>DMA: EvtIoReadDma()
        DMA->>DMA: WdfDmaTransactionInitializeUsingRequest()
        DMA->>DMA: WdfDmaTransactionExecute()
        DMA->>XDMA: XDMA_EngineProgramDma()
        XDMA->>HW: Program DMA descriptors
        XDMA->>HW: Start DMA engine
        HW->>HW: Transfer data
        HW->>XDMA: Transfer complete
        XDMA->>DMA: Completion callback
        DMA->>WIN: WdfRequestComplete()
    else Streaming (ST) Engine
        IOR->>DMA: EvtIoReadEngineRing()
        DMA->>DMA: WdfRequestRetrieveOutputMemory()
        DMA->>XDMA: EngineRingCopyBytesToMemory()
        XDMA->>XDMA: Wait for data in ring buffer
        XDMA->>XDMA: Copy to user memory
        XDMA-->>DMA: Return bytes copied
        DMA->>WIN: WdfRequestCompleteWithInformation()
    end

    WIN-->>APP: ReadFile() returns
```

## BAR Access (Control/User) Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant IO as EvtIoRead/EvtIoWrite
    participant BAR as ReadBarToRequest/WriteBarFromRequest
    participant HW as Hardware BAR

    APP->>WIN: ReadFile/WriteFile(control_handle)
    WIN->>DRV: IRP_MJ_READ/WRITE
    DRV->>IO: EvtIoRead/EvtIoWrite()
    IO->>IO: GetFileContext(WdfRequestGetFileObject)
    IO->>IO: Check devType (CONTROL/USER/BYPASS)

    alt Read Operation
        IO->>BAR: ReadBarToRequest(request, file->u.bar)
        BAR->>BAR: WdfRequestGetParameters()
        BAR->>BAR: Calculate offset and length
        BAR->>BAR: WdfRequestRetrieveOutputMemory()
        BAR->>BAR: WdfMemoryGetBuffer()

        alt 32-bit aligned
            BAR->>HW: READ_REGISTER_BUFFER_ULONG()
        else 16-bit aligned
            BAR->>HW: READ_REGISTER_BUFFER_USHORT()
        else Byte access
            BAR->>HW: READ_REGISTER_BUFFER_UCHAR()
        end

        BAR-->>IO: Data read
        IO->>WIN: WdfRequestCompleteWithInformation()
    else Write Operation
        IO->>BAR: WriteBarFromRequest(request, file->u.bar)
        BAR->>BAR: WdfRequestRetrieveInputMemory()

        alt 32-bit aligned
            BAR->>HW: WRITE_REGISTER_BUFFER_ULONG()
        else 16-bit aligned
            BAR->>HW: WRITE_REGISTER_BUFFER_USHORT()
        else Byte access
            BAR->>HW: WRITE_REGISTER_BUFFER_UCHAR()
        end

        BAR-->>IO: Write complete
        IO->>WIN: WdfRequestCompleteWithInformation()
    end

    WIN-->>APP: Operation returns
```

## IOCTL Operations Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant IOCTL as EvtIoDeviceControl
    participant IMPL as IOCTL Implementation
    participant XDMA as libxdma

    APP->>WIN: DeviceIoControl(handle, IOCTL_code, ...)
    WIN->>DRV: IRP_MJ_DEVICE_CONTROL
    DRV->>IOCTL: EvtIoDeviceControl()
    IOCTL->>IOCTL: GetFileContext(WdfRequestGetFileObject)

    alt BAR-related IOCTLs
        IOCTL->>IMPL: IoctlBar(file, request, IoControlCode)

        alt IOCTL_WRITE_KEYHOLE_REGISTER
            IMPL->>IMPL: IoctlKeyholeWriteRegister()
            IMPL->>IMPL: WdfRequestRetrieveInputBuffer()
            IMPL->>IMPL: Validate XDMA_KEYHOLE_DATA
            loop For each data word
                IMPL->>IMPL: WRITE_REGISTER_ULONG(address, data)
            end
            IMPL->>WIN: WdfRequestComplete()
        else IOCTL_MAP_BAR
            IMPL->>IMPL: IoctlMapBar()
            IMPL->>IMPL: IoAllocateMdl()
            IMPL->>IMPL: MmBuildMdlForNonPagedPool()
            IMPL->>IMPL: MmMapLockedPagesSpecifyCache()
            IMPL->>IMPL: Return virtual address to user
            IMPL->>WIN: WdfRequestCompleteWithInformation()
        end

    else Channel-related IOCTLs
        IOCTL->>IMPL: IoctlChannel(file, request, IoControlCode)

        alt IOCTL_XDMA_PERF_START
            IMPL->>XDMA: EngineStartPerf(engine)
            IMPL->>WIN: WdfRequestComplete()
        else IOCTL_XDMA_PERF_GET
            IMPL->>IMPL: IoctlGetPerf(request, engine)
            IMPL->>XDMA: EngineGetPerf(engine, &perfData)
            IMPL->>IMPL: WdfMemoryCopyFromBuffer()
            IMPL->>WIN: WdfRequestCompleteWithInformation()
        else IOCTL_XDMA_ADDRMODE_SET/GET
            IMPL->>IMPL: IoctlSetAddrMode/IoctlGetAddrMode()
            IMPL->>IMPL: Configure XDMA_CTRL_NON_INCR_ADDR
            IMPL->>WIN: WdfRequestComplete()
        end
    end

    WIN-->>APP: DeviceIoControl() returns
```

## User Event Handling Sequence

```mermaid
sequenceDiagram
    participant APP as Application
    participant WIN as Windows I/O Manager
    participant DRV as Driver
    participant EVENT as EvtReadUserEvent
    participant ISR as HandleUserEvent
    participant HW as Hardware
    participant XDMA as libxdma

    Note over APP,XDMA: Setup Phase
    APP->>WIN: ReadFile(event_handle, &result, sizeof(BOOLEAN))
    WIN->>DRV: IRP_MJ_READ
    DRV->>EVENT: EvtReadUserEvent()
    EVENT->>EVENT: GetFileContext(WdfRequestGetFileObject)
    EVENT->>EVENT: Get KEVENT from file->u.event->userData
    EVENT->>EVENT: KeClearEvent(event)

    Note over EVENT,XDMA: Waiting Phase
    EVENT->>EVENT: KeWaitForSingleObject(event, 3sec timeout)

    Note over HW,XDMA: Hardware Event Occurs
    HW->>XDMA: User interrupt triggered
    XDMA->>ISR: HandleUserEvent(eventId, userData)
    ISR->>ISR: KePulseEvent(event)

    Note over EVENT,APP: Completion Phase
    EVENT->>EVENT: Wait completes (event or timeout)
    EVENT->>EVENT: WdfRequestRetrieveOutputMemory()
    EVENT->>EVENT: Set eventValue = TRUE/FALSE
    EVENT->>EVENT: WdfMemoryCopyFromBuffer()
    EVENT->>WIN: WdfRequestCompleteWithInformation()
    WIN-->>APP: ReadFile() returns with event status
```

## Driver Unload Sequence

```mermaid
sequenceDiagram
    participant WM as Windows Manager
    participant DRV as Driver
    participant DEV as Device Objects
    participant XDMA as libxdma
    participant HW as Hardware

    WM->>DRV: Driver unload request

    Note over WM,HW: Device Cleanup
    DRV->>DEV: EvtDeviceReleaseHardware()
    DEV->>XDMA: XDMA_DeviceClose()
    XDMA->>HW: Stop DMA engines
    XDMA->>HW: Disable interrupts
    XDMA->>HW: Unmap BAR memory
    XDMA-->>DEV: Cleanup complete

    Note over DRV,DEV: File Object Cleanup
    DEV->>DEV: EvtFileCleanup() for open files
    DEV->>DEV: Unmap MDLs
    DEV->>DEV: Free ring buffers

    Note over WM,DRV: Driver Cleanup
    DRV->>DRV: DriverUnload()
    DRV->>DRV: WPP_CLEANUP()
    DRV-->>WM: Driver unloaded
```

## Error Handling Sequences

### DMA Transfer Error Handling

```mermaid
sequenceDiagram
    participant APP as Application
    participant DMA as DMA Handler
    participant XDMA as libxdma
    participant WIN as Windows I/O Manager

    APP->>DMA: DMA operation request
    DMA->>DMA: WdfDmaTransactionInitializeUsingRequest()

    alt Initialization Success
        DMA->>DMA: WdfDmaTransactionExecute()
        DMA->>XDMA: XDMA_EngineProgramDma()

        alt DMA Programming Success
            XDMA->>DMA: Transfer in progress
            Note over DMA,XDMA: Normal completion path
        else DMA Programming Error
            XDMA-->>DMA: Error status
            DMA->>DMA: WdfDmaTransactionRelease()
            DMA->>WIN: WdfRequestComplete(error_status)
        end

    else Initialization Error
        DMA->>WIN: WdfRequestComplete(STATUS_INSUFFICIENT_RESOURCES)
    end

    WIN-->>APP: Return status
```

### File Creation Error Handling

```mermaid
sequenceDiagram
    participant APP as Application
    participant FC as EvtDeviceFileCreate
    participant WIN as Windows I/O Manager

    APP->>FC: File creation request
    FC->>FC: Parse filename

    alt Valid filename
        FC->>FC: GetDevNodeType()

        alt Valid device node
            FC->>FC: Validate hardware availability

            alt Hardware available
                FC->>FC: Setup file context
                FC->>WIN: WdfRequestComplete(STATUS_SUCCESS)
            else Hardware not available
                FC->>WIN: WdfRequestComplete(STATUS_INVALID_PARAMETER)
            end

        else Invalid device node
            FC->>WIN: WdfRequestComplete(STATUS_INVALID_PARAMETER)
        end

    else Invalid filename
        FC->>WIN: WdfRequestComplete(STATUS_INVALID_PARAMETER)
    end

    WIN-->>APP: Return creation status
```

## Performance Optimization Sequences

### Interrupt vs Polling Mode

```mermaid
sequenceDiagram
    participant CFG as Configuration
    participant ENG as Engine Setup
    participant DMA as DMA Operation
    participant HW as Hardware

    CFG->>ENG: GetPollModeParameter()
    CFG->>ENG: Configure engine mode

    alt Interrupt Mode (POLL_MODE=0)
        ENG->>ENG: EngineEnableInterrupt()
        ENG->>DMA: Start DMA transfer
        DMA->>HW: Transfer data
        HW->>ENG: Generate completion interrupt
        ENG->>ENG: ISR handles completion
        Note over ENG: Lower CPU usage, higher latency

    else Polling Mode (POLL_MODE=1)
        ENG->>ENG: EngineDisableInterrupt()
        ENG->>DMA: Start DMA transfer
        DMA->>HW: Transfer data
        loop Poll for completion
            ENG->>HW: Check status registers
        end
        ENG->>ENG: Detect completion
        Note over ENG: Higher CPU usage, lower latency
    end
```

This sequence documentation provides a comprehensive view of how the XDMA driver operates, showing the interaction between user applications, Windows I/O Manager, the driver components, and the underlying hardware through libxdma.