# PCAN-Clone Adapter Interface Specification

This document defines the exact interface that a PCAN-clone (or any custom CAN hardware) must implement to be compatible with BusMaster. If you're building your own CAN device similar to PCAN, this is the specification you need to follow.

## Table of Contents

1. [Overview](#overview)
2. [Architecture Options](#architecture-options)
3. [Option 1: PCAN-Basic API Compatibility](#option-1-pcan-basic-api-compatibility)
4. [Option 2: BusMaster DIL Interface](#option-2-busmaster-dil-interface)
5. [Required Data Structures](#required-data-structures)
6. [Message Format Specification](#message-format-specification)
7. [Error Handling](#error-handling)
8. [Timing and Timestamps](#timing-and-timestamps)
9. [Hardware Detection](#hardware-detection)
10. [Configuration Parameters](#configuration-parameters)
11. [Complete Interface Checklist](#complete-interface-checklist)

---

## Overview

To connect a custom CAN device to BusMaster, you have two options:

1. **PCAN-Basic API Compatible** - Make your device's SDK/DLL compatible with PCAN-Basic API, allowing it to work with the existing PCAN adapter in BusMaster
2. **Custom BusMaster DIL** - Create a custom DIL (Device Interface Layer) adapter DLL for BusMaster

This document covers both approaches in detail.

---

## Architecture Options

### Option 1: PCAN-Basic API Emulation

```
Your Hardware → Your SDK DLL (PCAN-Basic API compatible) → BusMaster CAN_PEAK_USB
```

**Pros:**
- No changes to BusMaster needed
- Drop-in replacement for PCAN hardware
- Simpler integration

**Cons:**
- Must exactly match PCAN-Basic API
- Less flexibility in implementation

### Option 2: Custom DIL Adapter

```
Your Hardware → Your SDK DLL → Your BusMaster DIL DLL → BusMaster
```

**Pros:**
- Full control over implementation
- Can optimize for your hardware
- More flexibility

**Cons:**
- More work to implement
- Need to modify BusMaster driver list

---

## Option 1: PCAN-Basic API Compatibility

If you want your device to be a drop-in replacement for PCAN, your SDK DLL must export the following functions with exact signatures.

### Required DLL Exports (PCANBasic.dll compatible)

Your DLL must be named `PCANBasic.dll` or you must redirect the existing PCAN adapter to use your DLL.

#### Core Functions

```cpp
//=============================================================================
// CAN_Initialize - Initialize a CAN channel
//=============================================================================
TPCANStatus __stdcall CAN_Initialize(
    TPCANHandle Channel,      // Channel handle (PCAN_USBBUS1, etc.)
    TPCANBaudrate Btr0Btr1,   // Baud rate code
    TPCANType HwType,         // Hardware type (optional, 0 for USB)
    DWORD IOPort,             // I/O port (optional, 0 for USB)
    WORD Interrupt            // Interrupt (optional, 0 for USB)
);

// Returns: PCAN_ERROR_OK on success, error code on failure

//=============================================================================
// CAN_Uninitialize - Uninitialize a CAN channel
//=============================================================================
TPCANStatus __stdcall CAN_Uninitialize(
    TPCANHandle Channel       // Channel to uninitialize
);

//=============================================================================
// CAN_Reset - Reset a CAN channel
//=============================================================================
TPCANStatus __stdcall CAN_Reset(
    TPCANHandle Channel       // Channel to reset
);

//=============================================================================
// CAN_GetStatus - Get current CAN status
//=============================================================================
TPCANStatus __stdcall CAN_GetStatus(
    TPCANHandle Channel       // Channel to query
);

//=============================================================================
// CAN_Read - Read a CAN message from receive queue
//=============================================================================
TPCANStatus __stdcall CAN_Read(
    TPCANHandle Channel,      // Channel to read from
    TPCANMsg* MessageBuffer,  // Buffer for message
    TPCANTimestamp* TimestampBuffer  // Buffer for timestamp (can be NULL)
);

// Returns: PCAN_ERROR_OK if message read
//          PCAN_ERROR_QRCVEMPTY if no message available

//=============================================================================
// CAN_Write - Send a CAN message
//=============================================================================
TPCANStatus __stdcall CAN_Write(
    TPCANHandle Channel,      // Channel to write to
    TPCANMsg* MessageBuffer   // Message to send
);

//=============================================================================
// CAN_FilterMessages - Set acceptance filter
//=============================================================================
TPCANStatus __stdcall CAN_FilterMessages(
    TPCANHandle Channel,      // Channel to configure
    DWORD FromID,             // Start of ID range
    DWORD ToID,               // End of ID range
    TPCANMode Mode            // PCAN_MODE_STANDARD or PCAN_MODE_EXTENDED
);

//=============================================================================
// CAN_GetValue - Get a parameter value
//=============================================================================
TPCANStatus __stdcall CAN_GetValue(
    TPCANHandle Channel,      // Channel (or PCAN_NONEBUS for global)
    TPCANParameter Parameter, // Parameter to get
    void* Buffer,             // Buffer for value
    DWORD BufferLength        // Buffer size
);

//=============================================================================
// CAN_SetValue - Set a parameter value
//=============================================================================
TPCANStatus __stdcall CAN_SetValue(
    TPCANHandle Channel,      // Channel (or PCAN_NONEBUS for global)
    TPCANParameter Parameter, // Parameter to set
    void* Buffer,             // Buffer with value
    DWORD BufferLength        // Buffer size
);

//=============================================================================
// CAN_GetErrorText - Get error description string
//=============================================================================
TPCANStatus __stdcall CAN_GetErrorText(
    TPCANStatus Error,        // Error code
    WORD Language,            // Language (0 = neutral)
    LPSTR Buffer              // Buffer for text (256 bytes min)
);
```

#### CAN FD Functions (Optional - for CAN FD support)

```cpp
//=============================================================================
// CAN_InitializeFD - Initialize a CAN FD channel
//=============================================================================
TPCANStatus __stdcall CAN_InitializeFD(
    TPCANHandle Channel,      // Channel handle
    TPCANBitrateFD BitrateFD  // Bit rate string (e.g., "f_clock=80000000,...")
);

//=============================================================================
// CAN_ReadFD - Read a CAN FD message
//=============================================================================
TPCANStatus __stdcall CAN_ReadFD(
    TPCANHandle Channel,
    TPCANMsgFD* MessageBuffer,
    TPCANTimestampFD* TimestampBuffer
);

//=============================================================================
// CAN_WriteFD - Write a CAN FD message
//=============================================================================
TPCANStatus __stdcall CAN_WriteFD(
    TPCANHandle Channel,
    TPCANMsgFD* MessageBuffer
);
```

### PCAN-Basic Data Types

```cpp
//=============================================================================
// Basic Types
//=============================================================================
typedef BYTE   TPCANHandle;      // Channel handle type
typedef DWORD  TPCANStatus;      // Status/error code type
typedef WORD   TPCANParameter;   // Parameter type
typedef WORD   TPCANBaudrate;    // Baud rate code type
typedef BYTE   TPCANType;        // Hardware type
typedef BYTE   TPCANMode;        // Filter mode
typedef BYTE   TPCANMessageType; // Message type flags

//=============================================================================
// Channel Handles
//=============================================================================
#define PCAN_NONEBUS        0x00  // No channel
#define PCAN_USBBUS1        0x51  // USB channel 1
#define PCAN_USBBUS2        0x52  // USB channel 2
#define PCAN_USBBUS3        0x53  // USB channel 3
#define PCAN_USBBUS4        0x54  // USB channel 4
#define PCAN_USBBUS5        0x55  // USB channel 5
#define PCAN_USBBUS6        0x56  // USB channel 6
#define PCAN_USBBUS7        0x57  // USB channel 7
#define PCAN_USBBUS8        0x58  // USB channel 8
// ... up to PCAN_USBBUS16 (0x60)

//=============================================================================
// Baud Rate Codes
//=============================================================================
#define PCAN_BAUD_1M        0x0014  // 1 MBit/s
#define PCAN_BAUD_800K      0x0016  // 800 kBit/s
#define PCAN_BAUD_500K      0x001C  // 500 kBit/s
#define PCAN_BAUD_250K      0x011C  // 250 kBit/s
#define PCAN_BAUD_125K      0x031C  // 125 kBit/s
#define PCAN_BAUD_100K      0x432F  // 100 kBit/s
#define PCAN_BAUD_95K       0xC34E  // 95.238 kBit/s
#define PCAN_BAUD_83K       0x852B  // 83.33 kBit/s
#define PCAN_BAUD_50K       0x472F  // 50 kBit/s
#define PCAN_BAUD_47K       0x1414  // 47.619 kBit/s
#define PCAN_BAUD_33K       0x8B2F  // 33.33 kBit/s
#define PCAN_BAUD_20K       0x532F  // 20 kBit/s
#define PCAN_BAUD_10K       0x672F  // 10 kBit/s
#define PCAN_BAUD_5K        0x7F7F  // 5 kBit/s

//=============================================================================
// Error/Status Codes
//=============================================================================
#define PCAN_ERROR_OK            0x00000  // No error
#define PCAN_ERROR_XMTFULL       0x00001  // Transmit buffer full
#define PCAN_ERROR_OVERRUN       0x00002  // CAN controller overrun
#define PCAN_ERROR_BUSLIGHT      0x00004  // Bus error (light)
#define PCAN_ERROR_BUSHEAVY      0x00008  // Bus error (heavy)
#define PCAN_ERROR_BUSWARNING    PCAN_ERROR_BUSHEAVY  // Alias
#define PCAN_ERROR_BUSPASSIVE    0x40000  // Bus passive
#define PCAN_ERROR_BUSOFF        0x00010  // Bus off
#define PCAN_ERROR_ANYBUSERR     (PCAN_ERROR_BUSWARNING | PCAN_ERROR_BUSLIGHT | \
                                  PCAN_ERROR_BUSHEAVY | PCAN_ERROR_BUSOFF | \
                                  PCAN_ERROR_BUSPASSIVE)
#define PCAN_ERROR_QRCVEMPTY     0x00020  // Receive queue empty
#define PCAN_ERROR_QOVERRUN      0x00040  // Receive queue overrun
#define PCAN_ERROR_QXMTFULL      0x00080  // Transmit queue full
#define PCAN_ERROR_REGTEST       0x00100  // Register test failed
#define PCAN_ERROR_NODRIVER      0x00200  // Driver not loaded
#define PCAN_ERROR_HWINUSE       0x00400  // Hardware already in use
#define PCAN_ERROR_NETINUSE      0x00800  // Network already in use
#define PCAN_ERROR_ILLHW         0x01400  // Invalid hardware handle
#define PCAN_ERROR_ILLNET        0x01800  // Invalid network handle
#define PCAN_ERROR_ILLCLIENT     0x01C00  // Invalid client handle
#define PCAN_ERROR_ILLHANDLE     (PCAN_ERROR_ILLHW | PCAN_ERROR_ILLNET | \
                                  PCAN_ERROR_ILLCLIENT)
#define PCAN_ERROR_RESOURCE      0x02000  // Resource error
#define PCAN_ERROR_ILLPARAMTYPE  0x04000  // Invalid parameter type
#define PCAN_ERROR_ILLPARAMVAL   0x08000  // Invalid parameter value
#define PCAN_ERROR_UNKNOWN       0x10000  // Unknown error
#define PCAN_ERROR_ILLDATA       0x20000  // Invalid data
#define PCAN_ERROR_CAUTION       0x2000000 // Operation succeeded with warning
#define PCAN_ERROR_INITIALIZE    0x4000000 // Channel not initialized
#define PCAN_ERROR_ILLOPERATION  0x8000000 // Invalid operation

//=============================================================================
// Message Type Flags
//=============================================================================
#define PCAN_MESSAGE_STANDARD    0x00  // Standard frame (11-bit ID)
#define PCAN_MESSAGE_RTR         0x01  // Remote request frame
#define PCAN_MESSAGE_EXTENDED    0x02  // Extended frame (29-bit ID)
#define PCAN_MESSAGE_FD          0x04  // FD frame
#define PCAN_MESSAGE_BRS         0x08  // FD bit rate switch
#define PCAN_MESSAGE_ESI         0x10  // FD error state indicator
#define PCAN_MESSAGE_STATUS      0x80  // Status message

//=============================================================================
// Filter Modes
//=============================================================================
#define PCAN_MODE_STANDARD       0x01  // Standard frames (11-bit)
#define PCAN_MODE_EXTENDED       0x02  // Extended frames (29-bit)

//=============================================================================
// Parameters for CAN_GetValue/CAN_SetValue
//=============================================================================
#define PCAN_DEVICE_NUMBER       0x01  // Device number
#define PCAN_5VOLTS_POWER        0x02  // 5V power on connector
#define PCAN_RECEIVE_EVENT       0x03  // Receive event handle
#define PCAN_MESSAGE_FILTER      0x04  // Message filter status
#define PCAN_API_VERSION         0x05  // API version string
#define PCAN_CHANNEL_VERSION     0x06  // Channel version string
#define PCAN_BUSOFF_AUTORESET    0x07  // Auto bus-off reset
#define PCAN_LISTEN_ONLY         0x08  // Listen-only mode
#define PCAN_LOG_LOCATION        0x09  // Log file location
#define PCAN_LOG_STATUS          0x0A  // Log status
#define PCAN_LOG_CONFIGURE       0x0B  // Log configuration
#define PCAN_LOG_TEXT            0x0C  // Log custom text
#define PCAN_CHANNEL_CONDITION   0x0D  // Channel condition
#define PCAN_HARDWARE_NAME       0x0E  // Hardware name
#define PCAN_RECEIVE_STATUS      0x0F  // Receive status
#define PCAN_CONTROLLER_NUMBER   0x10  // Controller number
#define PCAN_TRACE_LOCATION      0x11  // Trace file location
#define PCAN_TRACE_STATUS        0x12  // Trace status
#define PCAN_TRACE_SIZE          0x13  // Trace file size
#define PCAN_TRACE_CONFIGURE     0x14  // Trace configuration
#define PCAN_CHANNEL_IDENTIFYING 0x15  // Channel identifying (LED flash)
#define PCAN_CHANNEL_FEATURES    0x16  // Channel features
#define PCAN_BITRATE_ADAPTING    0x17  // Bit rate adapting
#define PCAN_BITRATE_INFO        0x18  // Bit rate info
#define PCAN_BITRATE_INFO_FD     0x19  // FD bit rate info
#define PCAN_BUSSPEED_NOMINAL    0x1A  // Nominal bus speed
#define PCAN_BUSSPEED_DATA       0x1B  // Data bus speed (FD)
#define PCAN_IP_ADDRESS          0x1C  // IP address (LAN)
#define PCAN_LAN_SERVICE_STATUS  0x1D  // LAN service status
#define PCAN_ALLOW_STATUS_FRAMES 0x1E  // Allow status frames
#define PCAN_ALLOW_RTR_FRAMES    0x1F  // Allow RTR frames
#define PCAN_ALLOW_ERROR_FRAMES  0x20  // Allow error frames
#define PCAN_INTERFRAME_DELAY    0x21  // Inter-frame delay
#define PCAN_ACCEPTANCE_FILTER_11BIT 0x22 // 11-bit acceptance filter
#define PCAN_ACCEPTANCE_FILTER_29BIT 0x23 // 29-bit acceptance filter

//=============================================================================
// Channel Conditions
//=============================================================================
#define PCAN_CHANNEL_UNAVAILABLE 0x00  // Channel not available
#define PCAN_CHANNEL_AVAILABLE   0x01  // Channel available
#define PCAN_CHANNEL_OCCUPIED    0x02  // Channel in use
#define PCAN_CHANNEL_PCANVIEW    (PCAN_CHANNEL_AVAILABLE | \
                                  PCAN_CHANNEL_OCCUPIED) // PCAN-View connection
```

### PCAN Message Structures

```cpp
//=============================================================================
// TPCANMsg - Standard CAN Message
//=============================================================================
typedef struct tagTPCANMsg
{
    DWORD ID;           // 11/29-bit message ID
    BYTE  MSGTYPE;      // Message type (see PCAN_MESSAGE_* flags)
    BYTE  LEN;          // Data length (0-8)
    BYTE  DATA[8];      // Data bytes
} TPCANMsg;

//=============================================================================
// TPCANTimestamp - Message Timestamp
//=============================================================================
typedef struct tagTPCANTimestamp
{
    DWORD millis;           // Milliseconds
    WORD  millis_overflow;  // Milliseconds overflow counter
    WORD  micros;           // Microseconds (0-999)
} TPCANTimestamp;

//=============================================================================
// TPCANMsgFD - CAN FD Message (for CAN FD support)
//=============================================================================
typedef struct tagTPCANMsgFD
{
    DWORD ID;           // 11/29-bit message ID
    BYTE  MSGTYPE;      // Message type (see PCAN_MESSAGE_* flags)
    BYTE  DLC;          // Data length code (0-15)
    BYTE  DATA[64];     // Data bytes (up to 64 for CAN FD)
} TPCANMsgFD;

//=============================================================================
// TPCANTimestampFD - CAN FD Timestamp (microseconds)
//=============================================================================
typedef UINT64 TPCANTimestampFD;
```

---

## Option 2: BusMaster DIL Interface

If you prefer to create a custom BusMaster adapter, implement the `CBaseDIL_CAN_Controller` interface. See [03-creating-custom-adapter.md](03-creating-custom-adapter.md) for the complete implementation guide.

### Minimum Required Methods

Your adapter DLL must implement these methods at minimum:

```cpp
class CYourAdapter : public CBaseDIL_CAN_Controller
{
public:
    // === MANDATORY - Lifecycle ===
    HRESULT CAN_PerformInitOperations(void);
    HRESULT CAN_PerformClosureOperations(void);
    HRESULT CAN_LoadDriverLibrary(void);
    HRESULT CAN_UnloadDriverLibrary(void);

    // === MANDATORY - Hardware Discovery ===
    HRESULT CAN_ListHwInterfaces(INTERFACE_HW_LIST& sSelHwInterface,
                                 INT& nCount, PSCONTROLLER_DETAILS InitData);
    HRESULT CAN_SelectHwInterface(const INTERFACE_HW_LIST& sSelHwInterface,
                                  INT nCount);
    HRESULT CAN_DeselectHwInterface(void);

    // === MANDATORY - Configuration ===
    HRESULT CAN_SetConfigData(PSCONTROLLER_DETAILS InitData, int Length);
    HRESULT CAN_SetAppParams(HWND hWndOwner);

    // === MANDATORY - Communication ===
    HRESULT CAN_StartHardware(void);
    HRESULT CAN_StopHardware(void);
    HRESULT CAN_SendMsg(DWORD dwClientID, const STCAN_MSG& sCanTxMsg);

    // === MANDATORY - Client Management ===
    HRESULT CAN_RegisterClient(BOOL bRegister, DWORD& ClientID,
                               char* pacClientName);
    HRESULT CAN_ManageMsgBuf(BYTE bAction, DWORD ClientID,
                             CBaseCANBufFSE* pBufObj);

    // === MANDATORY - Status ===
    HRESULT CAN_GetCurrStatus(STATUSMSG& StatusData);
    HRESULT CAN_GetCntrlStatus(const HANDLE& hEvent, UINT& unCntrlStatus);
    HRESULT CAN_GetControllerParams(LONG& lParam, UINT nChannel,
                                    ECONTR_PARAM eContrParam);
    HRESULT CAN_SetControllerParams(int nValue, ECONTR_PARAM eContrparam);
    HRESULT CAN_GetErrorCount(SERROR_CNT& sErrorCnt, UINT nChannel,
                              ECONTR_PARAM eContrParam);
    HRESULT CAN_GetTimeModeMapping(SYSTEMTIME& CurrSysTime, UINT64& TimeStamp,
                                   LARGE_INTEGER& QueryTickCount);
    HRESULT CAN_GetLastErrorString(std::string& acErrorStr);
    HRESULT CAN_SetHardwareChannel(PSCONTROLLER_DETAILS, DWORD dwDriverId,
                                   bool bIsHardwareListed, unsigned int unChannelCnt);
};

// Required DLL export
extern "C" __declspec(dllexport) HRESULT GetIDIL_CAN_Controller(void** ppvInterface);
```

---

## Required Data Structures

### BusMaster CAN Message (STCAN_MSG)

Your adapter must convert between your hardware's message format and this structure:

```cpp
typedef struct sTCAN_MSG
{
    unsigned int  m_unMsgID;      // Message ID (11-bit or 29-bit)
    unsigned char m_ucEXTENDED;   // 0 = Standard (11-bit), 1 = Extended (29-bit)
    unsigned char m_ucRTR;        // 0 = Data frame, 1 = Remote request
    unsigned char m_ucDataLen;    // Data length (0-8, or 0-64 for CAN FD)
    unsigned char m_ucChannel;    // Channel number (1-based!)
    unsigned char m_ucData[64];   // Data bytes
    bool m_bCANFD;                // true = CAN FD frame
} STCAN_MSG;
```

### Conversion Example (PCAN to BusMaster)

```cpp
void ConvertPCANToBusMaster(const TPCANMsg& pcanMsg, STCAN_MSG& busMsg,
                            int channel)
{
    busMsg.m_unMsgID = pcanMsg.ID;
    busMsg.m_ucEXTENDED = (pcanMsg.MSGTYPE & PCAN_MESSAGE_EXTENDED) ? 1 : 0;
    busMsg.m_ucRTR = (pcanMsg.MSGTYPE & PCAN_MESSAGE_RTR) ? 1 : 0;
    busMsg.m_ucDataLen = pcanMsg.LEN;
    busMsg.m_ucChannel = channel + 1;  // Convert 0-based to 1-based
    memcpy(busMsg.m_ucData, pcanMsg.DATA, pcanMsg.LEN);
    busMsg.m_bCANFD = false;
}

void ConvertBusMasterToPCAN(const STCAN_MSG& busMsg, TPCANMsg& pcanMsg)
{
    pcanMsg.ID = busMsg.m_unMsgID;
    pcanMsg.MSGTYPE = PCAN_MESSAGE_STANDARD;
    if (busMsg.m_ucEXTENDED)
        pcanMsg.MSGTYPE |= PCAN_MESSAGE_EXTENDED;
    if (busMsg.m_ucRTR)
        pcanMsg.MSGTYPE |= PCAN_MESSAGE_RTR;
    pcanMsg.LEN = busMsg.m_ucDataLen;
    memcpy(pcanMsg.DATA, busMsg.m_ucData, busMsg.m_ucDataLen);
}
```

---

## Message Format Specification

### Standard CAN Frame (11-bit ID)

```
+--------+-----+-----+----------+----------------------+
| ID     | RTR | DLC | Data     | Notes                |
| 11-bit | 1b  | 4b  | 0-8 bytes|                      |
+--------+-----+-----+----------+----------------------+

m_ucEXTENDED = 0
m_unMsgID = 0x000 to 0x7FF (11 bits)
```

### Extended CAN Frame (29-bit ID)

```
+--------+-----+-----+----------+----------------------+
| ID     | RTR | DLC | Data     | Notes                |
| 29-bit | 1b  | 4b  | 0-8 bytes|                      |
+--------+-----+-----+----------+----------------------+

m_ucEXTENDED = 1
m_unMsgID = 0x00000000 to 0x1FFFFFFF (29 bits)
```

### CAN FD Frame

```
+--------+-----+-----+----------+----------------------+
| ID     | BRS | DLC | Data     | Notes                |
| 11/29b | 1b  | 4b  | 0-64 bytes|                     |
+--------+-----+-----+----------+----------------------+

m_bCANFD = true
m_ucDataLen = 0, 1, 2, 3, 4, 5, 6, 7, 8, 12, 16, 20, 24, 32, 48, 64
```

### DLC to Data Length Mapping (CAN FD)

| DLC | Data Length |
|-----|-------------|
| 0-8 | 0-8 bytes   |
| 9   | 12 bytes    |
| 10  | 16 bytes    |
| 11  | 20 bytes    |
| 12  | 24 bytes    |
| 13  | 32 bytes    |
| 14  | 48 bytes    |
| 15  | 64 bytes    |

---

## Error Handling

### Error Frame Detection

Your adapter should detect and report these CAN errors:

```cpp
enum CAN_ERROR_TYPE
{
    ERROR_BUS = 0,              // Bus error
    ERROR_DEVICE_BUFF_OVERFLOW, // Device buffer overflow
    ERROR_DRIVER_BUFF_OVERFLOW, // Driver buffer overflow
    ERROR_UNKNOWN               // Unknown error
};

// Error info structure
struct sERROR_INFO
{
    unsigned char m_ucErrType;      // Error type (see above)
    unsigned char m_ucReg_ErrCap;   // Error capture register value
    unsigned char m_ucTxErrCount;   // TX error counter
    unsigned char m_ucRxErrCount;   // RX error counter
    unsigned char m_ucChannel;      // Channel number
    int m_nSubError;                // Sub-error code
};
```

### Controller States

```cpp
enum ControllerState
{
    defCONTROLLER_ACTIVE  = 1,  // Normal operation (error count < 96)
    defCONTROLLER_PASSIVE = 2,  // Error passive (error count >= 128)
    defCONTROLLER_BUSOFF  = 3   // Bus off (error count >= 256)
};
```

---

## Timing and Timestamps

### Timestamp Format

BusMaster uses Windows `QueryPerformanceCounter` for timestamps:

```cpp
LARGE_INTEGER m_lTickCount;  // From QueryPerformanceCounter()
```

### Time Mapping

Your adapter must provide a reference point for timestamp conversion:

```cpp
HRESULT CAN_GetTimeModeMapping(
    SYSTEMTIME& CurrSysTime,       // System time at reference
    UINT64& TimeStamp,             // Hardware timestamp at reference
    LARGE_INTEGER& QueryTickCount  // Performance counter at reference
);
```

### Implementation

```cpp
// At start of communication, record the reference point
HRESULT MyAdapter::CAN_StartHardware()
{
    // Record reference point
    GetSystemTime(&m_refSysTime);
    QueryPerformanceCounter(&m_refTickCount);
    m_refHwTimestamp = GetHardwareTimestamp();  // Your SDK call

    // ... start hardware
}

// When requested, return the reference
HRESULT MyAdapter::CAN_GetTimeModeMapping(
    SYSTEMTIME& CurrSysTime,
    UINT64& TimeStamp,
    LARGE_INTEGER& QueryTickCount)
{
    CurrSysTime = m_refSysTime;
    TimeStamp = m_refHwTimestamp;
    QueryTickCount = m_refTickCount;
    return S_OK;
}
```

---

## Hardware Detection

### Device Enumeration

Your adapter must enumerate available devices:

```cpp
HRESULT CAN_ListHwInterfaces(
    INTERFACE_HW_LIST& sSelHwInterface,  // Output: list of devices
    INT& nCount,                          // Output: number of devices
    PSCONTROLLER_DETAILS InitData)        // Input: initial config
{
    // Example: Enumerate your devices
    int numDevices = YourSDK_GetDeviceCount();

    for (int i = 0; i < numDevices && i < defNO_OF_CHANNELS; i++)
    {
        char serial[64];
        YourSDK_GetSerial(i, serial, sizeof(serial));

        sSelHwInterface[i].m_dwIdInterface = i;
        sSelHwInterface[i].m_dwVendor = YOUR_VENDOR_ID;
        sprintf(sSelHwInterface[i].m_acDescription,
                "YourDevice CAN%d (S/N: %s)", i + 1, serial);
        strcpy(sSelHwInterface[i].m_acDeviceName, "YourDevice");
    }

    nCount = numDevices;
    return S_OK;
}
```

### INTERFACE_HW Structure

```cpp
typedef struct tagINTERFACE_HW
{
    DWORD m_dwIdInterface;           // Interface ID
    DWORD m_dwVendor;                // Vendor ID
    BYTE  m_bytNetworkID;            // Network ID
    char  m_acDeviceName[MAX_CHAR];  // Device name
    char  m_acDescription[MAX_CHAR]; // Description
} INTERFACE_HW;

typedef INTERFACE_HW INTERFACE_HW_LIST[defNO_OF_CHANNELS];
```

---

## Configuration Parameters

### Bit Timing Parameters

Your adapter must support these configuration parameters:

```cpp
class sCONTROLLERDETAILS
{
    // Baud rate
    std::string m_omStrBaudrate;        // e.g., "500000"

    // Bit timing registers
    std::string m_omStrBTR0;            // BTR0 value (hex)
    std::string m_omStrBTR1;            // BTR1 value (hex)

    // Timing parameters
    std::string m_omStrClock;           // Clock frequency (MHz)
    std::string m_omStrSamplePercentage;// Sample point (%)
    std::string m_omStrSjw;             // SJW value

    // Controller mode
    unsigned short m_ucControllerMode;  // 1=Active, 2=Passive

    // CAN FD parameters (if supported)
    UINT32 m_unDataBitRate;             // Data phase bit rate
    UINT32 m_unDataSamplePoint;         // Data phase sample point
    UINT32 m_unDataSJW;                 // Data phase SJW
};
```

### Common Baud Rates

| Baud Rate | BTR0 | BTR1 | Clock | Sample% |
|-----------|------|------|-------|---------|
| 1 Mbit/s  | 0x00 | 0x14 | 16MHz | 75%     |
| 500 kbit/s| 0x00 | 0x1C | 16MHz | 75%     |
| 250 kbit/s| 0x01 | 0x1C | 16MHz | 75%     |
| 125 kbit/s| 0x03 | 0x1C | 16MHz | 75%     |
| 100 kbit/s| 0x04 | 0x2F | 16MHz | 75%     |
| 50 kbit/s | 0x04 | 0x7F | 16MHz | 75%     |

---

## Complete Interface Checklist

### For PCAN-Basic API Compatibility

```
[ ] CAN_Initialize() - Initialize channel with baud rate
[ ] CAN_Uninitialize() - Clean up channel
[ ] CAN_Reset() - Reset channel
[ ] CAN_GetStatus() - Get current status
[ ] CAN_Read() - Read message from queue
[ ] CAN_Write() - Send message
[ ] CAN_FilterMessages() - Set acceptance filter
[ ] CAN_GetValue() - Get parameter value
[ ] CAN_SetValue() - Set parameter value
[ ] CAN_GetErrorText() - Get error description
[ ] TPCANMsg structure - Standard message format
[ ] TPCANTimestamp structure - Timestamp format
[ ] All error codes defined
[ ] All channel handles defined
[ ] All baud rate codes defined
[ ] All parameter IDs defined

Optional (CAN FD):
[ ] CAN_InitializeFD() - Initialize CAN FD channel
[ ] CAN_ReadFD() - Read CAN FD message
[ ] CAN_WriteFD() - Write CAN FD message
[ ] TPCANMsgFD structure - FD message format
```

### For Custom BusMaster DIL

```
[ ] GetIDIL_CAN_Controller() export function
[ ] CAN_PerformInitOperations()
[ ] CAN_PerformClosureOperations()
[ ] CAN_LoadDriverLibrary()
[ ] CAN_UnloadDriverLibrary()
[ ] CAN_ListHwInterfaces()
[ ] CAN_SelectHwInterface()
[ ] CAN_DeselectHwInterface()
[ ] CAN_SetConfigData()
[ ] CAN_SetAppParams()
[ ] CAN_StartHardware()
[ ] CAN_StopHardware()
[ ] CAN_SendMsg()
[ ] CAN_RegisterClient()
[ ] CAN_ManageMsgBuf()
[ ] CAN_GetCurrStatus()
[ ] CAN_GetCntrlStatus()
[ ] CAN_GetControllerParams()
[ ] CAN_SetControllerParams()
[ ] CAN_GetErrorCount()
[ ] CAN_GetTimeModeMapping()
[ ] CAN_GetLastErrorString()
[ ] CAN_SetHardwareChannel()
[ ] Receive thread for message distribution
[ ] Client registration system
[ ] Thread-safe buffer management
```

---

## Testing Your Implementation

### Test Sequence

1. **Basic Connectivity**
   - Device enumeration
   - Channel open/close
   - Baud rate configuration

2. **Message Transmission**
   - Send standard frame
   - Send extended frame
   - Send RTR frame
   - Verify on oscilloscope or another device

3. **Message Reception**
   - Receive standard frame
   - Receive extended frame
   - Verify timestamps
   - Test buffer overflow handling

4. **Error Handling**
   - Disconnect bus → Bus off detection
   - Short to ground → Error detection
   - High bus load → Overflow handling

5. **Multi-client**
   - Register multiple clients
   - Verify message distribution
   - Test client removal

---

## Related Documents

- [01-architecture-overview.md](01-architecture-overview.md) - Architecture overview
- [02-code-walkthrough.md](02-code-walkthrough.md) - Code walkthrough
- [03-creating-custom-adapter.md](03-creating-custom-adapter.md) - Custom adapter implementation
- [04-quick-reference.md](04-quick-reference.md) - Quick reference
