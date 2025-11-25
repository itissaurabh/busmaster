# BusMaster Quick Reference Guide

A quick reference for developers working with the BusMaster codebase.

---

## Key Files at a Glance

| Purpose | File Path |
|---------|-----------|
| Application Entry | `Sources/BUSMASTER/Application/BUSMASTER.cpp` |
| Main Window | `Sources/BUSMASTER/Application/MainFrm.cpp` |
| CAN DIL Base Interface | `Sources/Kernel/BusmasterDriverInterface/Include/BaseDIL_CAN.h` |
| CAN Controller Base Class | `Sources/Kernel/BusmasterDriverInterface/Include/BaseDIL_CAN_Controller.h` |
| CAN Message Structures | `Sources/BUSMASTER/Include/Struct_CAN_.h` |
| Vector Adapter (Example) | `Sources/BUSMASTER/CAN_Vector_XL/CAN_Vector_XL.cpp` |
| Plugin Manager | `Sources/BUSMASTER/Application/BusmasterPluginManager.cpp` |

---

## Core Data Structures

### STCAN_MSG - CAN Message

```cpp
typedef struct sTCAN_MSG
{
    unsigned int  m_unMsgID;      // Message ID (11 or 29 bit)
    unsigned char m_ucEXTENDED;   // 1 = Extended (29-bit) ID
    unsigned char m_ucRTR;        // 1 = Remote Request
    unsigned char m_ucDataLen;    // Data length (0-8, or 0-64 for CAN FD)
    unsigned char m_ucChannel;    // Channel number (1-based!)
    unsigned char m_ucData[64];   // Data bytes
    bool m_bCANFD;                // true = CAN FD frame
} STCAN_MSG;
```

### STCANDATA - CAN Data with Metadata

```cpp
typedef struct sTCANDATA
{
    unsigned char m_ucDataType;   // TX_FLAG, RX_FLAG, ERR_FLAG
    LARGE_INTEGER m_lTickCount;   // Timestamp
    union {
        STCAN_MSG   m_sCANMsg;    // CAN message
        SERROR_INFO m_sErrInfo;   // Error info
    } m_uDataInfo;
} STCANDATA;
```

### Message Type Flags

```cpp
#define TX_FLAG     0x01  // Transmitted message
#define RX_FLAG     0x02  // Received message
#define ERR_FLAG    0x04  // Error frame
#define INTR_FLAG   0x08  // Interrupt

#define IS_TX_MESSAGE(a)   (a & TX_FLAG)
#define IS_RX_MESSAGE(a)   (a & RX_FLAG)
#define IS_ERR_MESSAGE(a)  (a & ERR_FLAG)
```

---

## DIL Interface Methods

### Driver Management

```cpp
// Get list of available drivers
DWORD DILC_GetDILList(bool bAvailable, DILLIST* List);

// Select a driver by ID
HRESULT DILC_SelectDriver(DWORD dwDriverID, HWND hWndParent);

// Get currently selected driver
DWORD DILC_GetSelectedDriver(void);
```

### Hardware Management

```cpp
// List available hardware
HRESULT DILC_ListHwInterfaces(INTERFACE_HW_LIST& sSelHwInterface,
                              INT& nCount,
                              PSCONTROLLER_DETAILS InitData);

// Select hardware interfaces
HRESULT DILC_SelectHwInterfaces(const INTERFACE_HW_LIST& sSelHwInterface,
                                INT nCount);

// Deselect hardware
HRESULT DILC_DeselectHwInterfaces(void);

// Set configuration (baud rate, filters, etc.)
HRESULT DILC_SetConfigData(PSCONTROLLER_DETAILS pInitData, int Length);
```

### Communication

```cpp
// Start hardware communication
HRESULT DILC_StartHardware(void);

// Stop hardware communication
HRESULT DILC_StopHardware(void);

// Send a CAN message
HRESULT DILC_SendMsg(DWORD dwClientID, const STCAN_MSG& sCanTxMsg);
```

### Client Management

```cpp
// Register/unregister client
HRESULT DILC_RegisterClient(BOOL bRegister, DWORD& ClientID, char* pacClientName);

// Add/remove message buffer
HRESULT DILC_ManageMsgBuf(BYTE bAction, DWORD ClientID, CBaseCANBufFSE* pBufObj);

// Buffer actions
#define MSGBUF_ADD     0x01  // Add buffer to client
#define MSGBUF_CLEAR   0x02  // Clear all buffers
```

---

## Controller Base Class Methods

All hardware adapters must implement:

```cpp
class CBaseDIL_CAN_Controller
{
    // Lifecycle
    virtual HRESULT CAN_PerformInitOperations(void) = 0;
    virtual HRESULT CAN_PerformClosureOperations(void) = 0;
    virtual HRESULT CAN_LoadDriverLibrary(void) = 0;
    virtual HRESULT CAN_UnloadDriverLibrary(void) = 0;

    // Hardware
    virtual HRESULT CAN_ListHwInterfaces(...) = 0;
    virtual HRESULT CAN_SelectHwInterface(...) = 0;
    virtual HRESULT CAN_DeselectHwInterface(void) = 0;
    virtual HRESULT CAN_SetHardwareChannel(...) = 0;

    // Configuration
    virtual HRESULT CAN_SetConfigData(...) = 0;
    virtual HRESULT CAN_SetAppParams(HWND hWndOwner) = 0;

    // Communication
    virtual HRESULT CAN_StartHardware(void) = 0;
    virtual HRESULT CAN_StopHardware(void) = 0;
    virtual HRESULT CAN_SendMsg(DWORD dwClientID, const STCAN_MSG& sCanTxMsg) = 0;

    // Clients
    virtual HRESULT CAN_RegisterClient(...) = 0;
    virtual HRESULT CAN_ManageMsgBuf(...) = 0;

    // Status
    virtual HRESULT CAN_GetCurrStatus(STATUSMSG& StatusData) = 0;
    virtual HRESULT CAN_GetCntrlStatus(...) = 0;
    virtual HRESULT CAN_GetControllerParams(...) = 0;
    virtual HRESULT CAN_SetControllerParams(...) = 0;
    virtual HRESULT CAN_GetErrorCount(...) = 0;
    virtual HRESULT CAN_GetTimeModeMapping(...) = 0;
    virtual HRESULT CAN_GetLastErrorString(std::string& acErrorStr) = 0;
};
```

---

## Creating a Custom Adapter - Checklist

```
[ ] Create new DLL project (MFC DLL)
[ ] Include required headers from BusMaster
[ ] Create class inheriting from CBaseDIL_CAN_Controller
[ ] Implement all virtual methods
[ ] Create .def file with GetIDIL_CAN_Controller export
[ ] Implement factory function GetIDIL_CAN_Controller()
[ ] Implement client registration and buffer management
[ ] Implement receive thread for message distribution
[ ] Test all lifecycle states
[ ] Handle errors gracefully with descriptive messages
```

---

## Factory Function Template

```cpp
// Required export for all adapters
USAGEMODE HRESULT GetIDIL_CAN_Controller(void** ppvInterface)
{
    HRESULT hResult = S_OK;
    if (nullptr == g_pouDIL_CAN_YourDevice)
    {
        g_pouDIL_CAN_YourDevice = new CDIL_CAN_YourDevice();
        if (nullptr == g_pouDIL_CAN_YourDevice)
        {
            hResult = S_FALSE;
        }
    }
    *ppvInterface = (void*)g_pouDIL_CAN_YourDevice;
    return hResult;
}
```

---

## Common Return Codes

```cpp
#define S_OK                      0x00000000  // Success
#define S_FALSE                   0x00000001  // Failed
#define ERR_CLIENT_EXISTS         0x80000001  // Client already registered
#define ERR_NO_CLIENT_EXIST       0x80000002  // Client not found
#define ERR_NO_MORE_CLIENT_ALLOWED 0x80000003 // Max clients reached
```

---

## Controller Status Values

```cpp
enum defCONTROLLER
{
    defCONTROLLER_ACTIVE  = 1,  // Normal operation
    defCONTROLLER_PASSIVE = 2,  // Error passive state
    defCONTROLLER_BUSOFF  = 3   // Bus off state
};
```

---

## Error States

```cpp
enum eERROR_STATE
{
    ERROR_ACTIVE = 0,       // Normal operation
    ERROR_WARNING_LIMIT,    // Warning limit reached
    ERROR_PASSIVE,          // Error passive
    ERROR_BUS_OFF,          // Bus off
    ERROR_FRAME,            // Error frame received
    ERROR_INVALID           // Invalid state
};
```

---

## Controller Parameters (ECONTR_PARAM)

```cpp
enum ECONTR_PARAM
{
    NUMBER_HW,              // Number of hardware channels
    NUMBER_CONNECTED_HW,    // Number of connected channels
    DRIVER_STATUS,          // Driver connection status
    HW_MODE,                // Hardware mode
    CON_TEST,               // Connection test
    CNTR_STATUS,            // Controller status
    // ... more parameters
};
```

---

## Directory Structure for Adapters

```
CAN_YourDevice/
├── CAN_YourDevice.cpp         # Main implementation
├── CAN_YourDevice.h           # Class declaration
├── CAN_YourDevice_Extern.h    # Export definitions
├── CAN_YourDevice_stdafx.h    # Precompiled header
├── CAN_YourDevice_stdafx.cpp  # Precompiled header source
├── CAN_YourDevice.def         # Module definition (exports)
├── CAN_YourDevice.rc          # Resources
├── resource.h                 # Resource IDs
└── EXTERNAL/                  # Vendor SDK headers/libs
    ├── YourDeviceSDK.h
    └── YourDeviceSDK.lib
```

---

## Build Configuration

### Required Preprocessor Definitions

```
WIN32
_WINDOWS
_USRDLL
CAN_YOURDEVICE_EXPORTS
UNICODE
_UNICODE
```

### Required Libraries

```
user32.lib
kernel32.lib
advapi32.lib
// Your vendor SDK library
YourDeviceSDK.lib
```

---

## Tips and Best Practices

1. **Channel Numbering**: BusMaster uses 1-based channels, SDKs typically use 0-based
2. **Thread Safety**: Use critical sections for client/buffer access
3. **Timestamps**: Use `QueryPerformanceCounter()` for consistent timestamps
4. **Error Messages**: Always provide meaningful error strings
5. **Clean Shutdown**: Always clean up in `CAN_PerformClosureOperations()`
6. **Reference Adapter**: Use `CAN_STUB` as a reference implementation

---

## Useful Macros

```cpp
// Check if connected
if (!m_bIsConnected)
{
    m_strLastError = "Not connected";
    return S_FALSE;
}

// Convert channel numbers
int sdkChannel = busChannel - 1;  // BusMaster to SDK (1-based to 0-based)
int busChannel = sdkChannel + 1;  // SDK to BusMaster (0-based to 1-based)

// Timestamp handling
LARGE_INTEGER tickCount;
QueryPerformanceCounter(&tickCount);
sCanData.m_lTickCount = tickCount;
```

---

## Related Documents

- [01-architecture-overview.md](01-architecture-overview.md) - Architecture overview
- [02-code-walkthrough.md](02-code-walkthrough.md) - Detailed code walkthrough
- [03-creating-custom-adapter.md](03-creating-custom-adapter.md) - Complete adapter implementation guide
