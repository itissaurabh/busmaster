# Creating a Custom CAN Adapter for BusMaster

This guide explains how to create a custom CAN adapter DLL to connect your own CAN hardware (like a custom PCAN-style device) to BusMaster.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Implementing the Interface](#implementing-the-interface)
5. [Complete Implementation Example](#complete-implementation-example)
6. [Registering Your Adapter](#registering-your-adapter)
7. [Testing Your Adapter](#testing-your-adapter)
8. [Common Pitfalls](#common-pitfalls)

---

## Overview

### How BusMaster Discovers Adapters

BusMaster uses a plugin-style architecture for hardware adapters:

1. Each adapter is a separate DLL (e.g., `CAN_MyDevice.dll`)
2. The DLL exports a factory function: `GetIDIL_CAN_Controller()`
3. BusMaster loads the DLL and calls this function to get the adapter instance
4. All communication goes through the `CBaseDIL_CAN_Controller` interface

### What You Need to Implement

Your adapter DLL must:

1. **Inherit** from `CBaseDIL_CAN_Controller`
2. **Implement** all pure virtual methods (~20 methods)
3. **Export** the `GetIDIL_CAN_Controller()` factory function
4. **Manage** client registration and message distribution

---

## Prerequisites

### Development Environment

- Visual Studio 2015 or later (for MFC support)
- Windows SDK
- BusMaster source code (for header files)

### Required Header Files

Copy these headers to your project or reference them from BusMaster:

```
Sources/Kernel/BusmasterDriverInterface/Include/
├── BaseDIL_CAN_Controller.h   # Base class to inherit
├── CANDriverDefines.h         # Type definitions
├── CAN_Error_Defs.h           # Error definitions
└── Error.h                    # Error codes

Sources/BUSMASTER/Include/
├── Struct_CAN_.h              # STCAN_MSG, STCANDATA structures
└── BaseDefs.h                 # Base definitions
```

### Your Hardware SDK

You'll need your hardware vendor's SDK/API for:
- Device enumeration
- Opening/closing device handles
- Sending CAN messages
- Receiving CAN messages
- Error handling

---

## Project Setup

### Step 1: Create a New DLL Project

1. Create a new MFC DLL project in Visual Studio
2. Name it `CAN_YourDevice` (following the BusMaster naming convention)

### Step 2: Project Structure

```
CAN_YourDevice/
├── CAN_YourDevice.cpp         # Main implementation
├── CAN_YourDevice.h           # Class declaration
├── CAN_YourDevice_Extern.h    # Export definitions
├── CAN_YourDevice_stdafx.h    # Precompiled header
├── CAN_YourDevice.def         # Module definition (exports)
├── resource.h                 # Resource definitions
└── CAN_YourDevice.rc          # Resources (dialogs, strings)
```

### Step 3: Module Definition File (CAN_YourDevice.def)

```def
; CAN_YourDevice.def : Declares the module parameters for the DLL.

LIBRARY      "CAN_YourDevice"

EXPORTS
    GetIDIL_CAN_Controller
```

### Step 4: Export Header (CAN_YourDevice_Extern.h)

```cpp
#pragma once

#if defined USAGE_EXPORT
#define USAGEMODE __declspec(dllexport)
#else
#define USAGEMODE __declspec(dllimport)
#endif

#ifdef __cplusplus
extern "C" {
#endif

USAGEMODE HRESULT GetIDIL_CAN_Controller(void** ppvInterface);

#ifdef __cplusplus
}
#endif
```

---

## Implementing the Interface

### Step 1: Class Declaration

```cpp
// CAN_YourDevice.h

#pragma once

#include "BaseDIL_CAN_Controller.h"
#include "Struct_CAN_.h"

class CDIL_CAN_YourDevice : public CBaseDIL_CAN_Controller
{
public:
    CDIL_CAN_YourDevice();
    virtual ~CDIL_CAN_YourDevice();

    // =========== LIFECYCLE METHODS ===========

    // Called when adapter is selected
    HRESULT CAN_PerformInitOperations(void) override;

    // Called when adapter is deselected
    HRESULT CAN_PerformClosureOperations(void) override;

    // Load your hardware SDK DLL
    HRESULT CAN_LoadDriverLibrary(void) override;

    // Unload your hardware SDK DLL
    HRESULT CAN_UnloadDriverLibrary(void) override;


    // =========== HARDWARE DISCOVERY ===========

    // List available hardware devices
    HRESULT CAN_ListHwInterfaces(
        INTERFACE_HW_LIST& sSelHwInterface,
        INT& nCount,
        PSCONTROLLER_DETAILS InitData) override;

    // User has selected specific hardware
    HRESULT CAN_SelectHwInterface(
        const INTERFACE_HW_LIST& sSelHwInterface,
        INT nCount) override;

    // User has deselected hardware
    HRESULT CAN_DeselectHwInterface(void) override;

    // Hardware channel configuration
    HRESULT CAN_SetHardwareChannel(
        PSCONTROLLER_DETAILS pControllerDetails,
        DWORD dwDriverId,
        bool bIsHardwareListed,
        unsigned int unChannelCount) override;


    // =========== CONFIGURATION ===========

    // Apply configuration (baud rate, filters, etc.)
    HRESULT CAN_SetConfigData(
        PSCONTROLLER_DETAILS InitData,
        int Length) override;

    // Set application window handle (for dialogs)
    HRESULT CAN_SetAppParams(HWND hWndOwner) override;


    // =========== COMMUNICATION CONTROL ===========

    // Start CAN communication
    HRESULT CAN_StartHardware(void) override;

    // Stop CAN communication
    HRESULT CAN_StopHardware(void) override;

    // Send a CAN message
    HRESULT CAN_SendMsg(
        DWORD dwClientID,
        const STCAN_MSG& sCanTxMsg) override;


    // =========== CLIENT MANAGEMENT ===========

    // Register/unregister a client
    HRESULT CAN_RegisterClient(
        BOOL bRegister,
        DWORD& ClientID,
        char* pacClientName) override;

    // Add/remove message buffer for a client
    HRESULT CAN_ManageMsgBuf(
        BYTE bAction,
        DWORD ClientID,
        CBaseCANBufFSE* pBufObj) override;


    // =========== STATUS AND DIAGNOSTICS ===========

    // Get current controller status
    HRESULT CAN_GetCurrStatus(STATUSMSG& StatusData) override;

    // Get controller status with event notification
    HRESULT CAN_GetCntrlStatus(
        const HANDLE& hEvent,
        UINT& unCntrlStatus) override;

    // Get controller parameters
    HRESULT CAN_GetControllerParams(
        LONG& lParam,
        UINT nChannel,
        ECONTR_PARAM eContrParam) override;

    // Set controller parameters
    HRESULT CAN_SetControllerParams(
        int nValue,
        ECONTR_PARAM eContrparam) override;

    // Get error counts
    HRESULT CAN_GetErrorCount(
        SERROR_CNT& sErrorCnt,
        UINT nChannel,
        ECONTR_PARAM eContrParam) override;

    // Get time mapping for timestamps
    HRESULT CAN_GetTimeModeMapping(
        SYSTEMTIME& CurrSysTime,
        UINT64& TimeStamp,
        LARGE_INTEGER& QueryTickCount) override;

    // Get last error description
    HRESULT CAN_GetLastErrorString(std::string& acErrorStr) override;

private:
    // Your private members here
    bool m_bIsConnected;
    HWND m_hOwnerWnd;
    std::string m_strLastError;

    // Hardware handle (your SDK type)
    // HANDLE m_hDevice;

    // Receive thread
    HANDLE m_hReceiveThread;
    bool m_bStopReceiveThread;

    // Client management
    static const int MAX_CLIENTS = 16;
    static const int MAX_BUFFERS_PER_CLIENT = 16;

    struct ClientInfo {
        DWORD dwClientID;
        char szClientName[MAX_PATH];
        CBaseCANBufFSE* pBuffers[MAX_BUFFERS_PER_CLIENT];
        int nBufferCount;
        bool bActive;
    };

    ClientInfo m_Clients[MAX_CLIENTS];
    int m_nClientCount;
    DWORD m_dwNextClientID;

    // Configuration
    SCONTROLLER_DETAILS m_ControllerDetails[defNO_OF_CHANNELS];
    int m_nChannelCount;

    // Timestamps
    SYSTEMTIME m_StartSysTime;
    UINT64 m_StartTimestamp;
    LARGE_INTEGER m_QueryTickCount;

    // Helper methods
    DWORD GetAvailableClientSlot();
    bool RemoveClient(DWORD dwClientID);
    void DistributeMessage(const STCANDATA& sCanData);
    static DWORD WINAPI ReceiveThreadProc(LPVOID pVoid);
};
```

---

## Complete Implementation Example

### Main Implementation File (CAN_YourDevice.cpp)

```cpp
// CAN_YourDevice.cpp

#include "CAN_YourDevice_stdafx.h"
#include "CAN_YourDevice.h"

// Include your hardware SDK
// #include "YourDeviceSDK.h"

#define USAGE_EXPORT
#include "CAN_YourDevice_Extern.h"

// Global instance
static CDIL_CAN_YourDevice* g_pouDIL_CAN_YourDevice = nullptr;

//=============================================================================
// FACTORY FUNCTION - This is what BusMaster calls to get your adapter
//=============================================================================
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

//=============================================================================
// CONSTRUCTOR / DESTRUCTOR
//=============================================================================
CDIL_CAN_YourDevice::CDIL_CAN_YourDevice()
    : m_bIsConnected(false)
    , m_hOwnerWnd(nullptr)
    , m_hReceiveThread(nullptr)
    , m_bStopReceiveThread(false)
    , m_nClientCount(0)
    , m_dwNextClientID(1)
    , m_nChannelCount(0)
{
    memset(m_Clients, 0, sizeof(m_Clients));
    memset(m_ControllerDetails, 0, sizeof(m_ControllerDetails));
}

CDIL_CAN_YourDevice::~CDIL_CAN_YourDevice()
{
    CAN_PerformClosureOperations();
}

//=============================================================================
// LIFECYCLE METHODS
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_PerformInitOperations(void)
{
    // Initialize your adapter
    // - Load any required resources
    // - Initialize timestamps

    GetSystemTime(&m_StartSysTime);
    QueryPerformanceCounter(&m_QueryTickCount);
    m_StartTimestamp = 0;

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_PerformClosureOperations(void)
{
    // Clean up
    CAN_StopHardware();
    CAN_DeselectHwInterface();
    CAN_UnloadDriverLibrary();

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_LoadDriverLibrary(void)
{
    // Load your hardware SDK DLL
    // Example:
    //
    // m_hSDKDll = LoadLibrary("YourDeviceSDK.dll");
    // if (m_hSDKDll == nullptr)
    // {
    //     m_strLastError = "Failed to load YourDeviceSDK.dll";
    //     return S_FALSE;
    // }
    //
    // // Get function pointers
    // pfnOpenDevice = (OpenDeviceFunc)GetProcAddress(m_hSDKDll, "OpenDevice");
    // ...

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_UnloadDriverLibrary(void)
{
    // Unload your hardware SDK DLL
    // FreeLibrary(m_hSDKDll);
    // m_hSDKDll = nullptr;

    return S_OK;
}

//=============================================================================
// HARDWARE DISCOVERY
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_ListHwInterfaces(
    INTERFACE_HW_LIST& sSelHwInterface,
    INT& nCount,
    PSCONTROLLER_DETAILS InitData)
{
    // Enumerate available hardware devices
    // Fill in sSelHwInterface with discovered devices

    // Example: Discovering devices
    //
    // int nDevices = YourSDK_GetDeviceCount();
    //
    // for (int i = 0; i < nDevices && i < defNO_OF_CHANNELS; i++)
    // {
    //     char szSerial[64];
    //     YourSDK_GetDeviceSerial(i, szSerial, sizeof(szSerial));
    //
    //     sSelHwInterface[i].m_dwIdInterface = i;
    //     sSelHwInterface[i].m_dwVendor = 0xYOUR_VENDOR_ID;
    //     sprintf(sSelHwInterface[i].m_acDescription,
    //             "YourDevice Channel %d (SN: %s)", i + 1, szSerial);
    //     strcpy(sSelHwInterface[i].m_acDeviceName, "YourDevice");
    // }
    //
    // nCount = nDevices;

    // For testing, return a simulated device
    nCount = 1;
    sSelHwInterface[0].m_dwIdInterface = 0;
    sSelHwInterface[0].m_dwVendor = 0x1234;
    strcpy(sSelHwInterface[0].m_acDescription, "YourDevice CAN Channel 1");
    strcpy(sSelHwInterface[0].m_acDeviceName, "YourDevice");

    // Copy initial configuration
    if (InitData != nullptr)
    {
        for (int i = 0; i < nCount; i++)
        {
            m_ControllerDetails[i] = InitData[i];
        }
    }

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_SelectHwInterface(
    const INTERFACE_HW_LIST& sSelHwInterface,
    INT nCount)
{
    // User has selected hardware - open device handles

    m_nChannelCount = nCount;

    // Example:
    //
    // for (int i = 0; i < nCount; i++)
    // {
    //     int deviceIndex = sSelHwInterface[i].m_dwIdInterface;
    //     m_hDevice[i] = YourSDK_OpenDevice(deviceIndex);
    //     if (m_hDevice[i] == INVALID_HANDLE)
    //     {
    //         m_strLastError = "Failed to open device";
    //         return S_FALSE;
    //     }
    // }

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_DeselectHwInterface(void)
{
    // Close device handles

    // Example:
    //
    // for (int i = 0; i < m_nChannelCount; i++)
    // {
    //     if (m_hDevice[i] != INVALID_HANDLE)
    //     {
    //         YourSDK_CloseDevice(m_hDevice[i]);
    //         m_hDevice[i] = INVALID_HANDLE;
    //     }
    // }

    m_nChannelCount = 0;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_SetHardwareChannel(
    PSCONTROLLER_DETAILS pControllerDetails,
    DWORD dwDriverId,
    bool bIsHardwareListed,
    unsigned int unChannelCount)
{
    if (pControllerDetails != nullptr)
    {
        for (unsigned int i = 0; i < unChannelCount; i++)
        {
            m_ControllerDetails[i] = pControllerDetails[i];
        }
    }
    m_nChannelCount = unChannelCount;
    return S_OK;
}

//=============================================================================
// CONFIGURATION
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_SetConfigData(
    PSCONTROLLER_DETAILS InitData,
    int Length)
{
    // Apply configuration to hardware

    for (int i = 0; i < Length; i++)
    {
        m_ControllerDetails[i] = InitData[i];

        // Get baud rate
        UINT nBaudRate = atoi(m_ControllerDetails[i].m_omStrBaudrate.c_str());

        // Example: Set baud rate on hardware
        //
        // if (m_hDevice[i] != INVALID_HANDLE)
        // {
        //     YourSDK_SetBaudRate(m_hDevice[i], nBaudRate);
        // }
    }

    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_SetAppParams(HWND hWndOwner)
{
    m_hOwnerWnd = hWndOwner;
    return S_OK;
}

//=============================================================================
// COMMUNICATION CONTROL
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_StartHardware(void)
{
    if (m_bIsConnected)
    {
        return S_OK;  // Already connected
    }

    // Start communication on hardware
    //
    // Example:
    // for (int i = 0; i < m_nChannelCount; i++)
    // {
    //     YourSDK_StartCAN(m_hDevice[i]);
    // }

    // Record start time for timestamps
    GetSystemTime(&m_StartSysTime);
    QueryPerformanceCounter(&m_QueryTickCount);
    m_StartTimestamp = 0;

    // Start receive thread
    m_bStopReceiveThread = false;
    m_hReceiveThread = CreateThread(nullptr, 0, ReceiveThreadProc,
                                    this, 0, nullptr);
    if (m_hReceiveThread == nullptr)
    {
        m_strLastError = "Failed to create receive thread";
        return S_FALSE;
    }

    m_bIsConnected = true;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_StopHardware(void)
{
    if (!m_bIsConnected)
    {
        return S_OK;  // Already stopped
    }

    // Stop receive thread
    m_bStopReceiveThread = true;
    if (m_hReceiveThread != nullptr)
    {
        WaitForSingleObject(m_hReceiveThread, 1000);
        CloseHandle(m_hReceiveThread);
        m_hReceiveThread = nullptr;
    }

    // Stop communication on hardware
    //
    // Example:
    // for (int i = 0; i < m_nChannelCount; i++)
    // {
    //     YourSDK_StopCAN(m_hDevice[i]);
    // }

    m_bIsConnected = false;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_SendMsg(
    DWORD dwClientID,
    const STCAN_MSG& sCanTxMsg)
{
    if (!m_bIsConnected)
    {
        m_strLastError = "Not connected";
        return S_FALSE;
    }

    // Send message via hardware
    //
    // Example:
    //
    // YourSDK_CANMessage msg;
    // msg.id = sCanTxMsg.m_unMsgID;
    // msg.dlc = sCanTxMsg.m_ucDataLen;
    // msg.extended = sCanTxMsg.m_ucEXTENDED;
    // msg.rtr = sCanTxMsg.m_ucRTR;
    // memcpy(msg.data, sCanTxMsg.m_ucData, msg.dlc);
    //
    // int channel = sCanTxMsg.m_ucChannel - 1;  // Channel is 1-based
    // int result = YourSDK_SendMessage(m_hDevice[channel], &msg);
    //
    // if (result != YOURSDK_SUCCESS)
    // {
    //     m_strLastError = "Failed to send message";
    //     return S_FALSE;
    // }

    // Create TX echo for distribution to clients
    STCANDATA sCanData;
    sCanData.m_ucDataType = TX_FLAG;
    sCanData.m_uDataInfo.m_sCANMsg = sCanTxMsg;
    QueryPerformanceCounter(&sCanData.m_lTickCount);

    // Distribute TX message to all registered clients
    DistributeMessage(sCanData);

    return S_OK;
}

//=============================================================================
// CLIENT MANAGEMENT
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_RegisterClient(
    BOOL bRegister,
    DWORD& ClientID,
    char* pacClientName)
{
    if (bRegister)
    {
        // Register new client
        DWORD slot = GetAvailableClientSlot();
        if (slot == (DWORD)-1)
        {
            m_strLastError = "No more client slots available";
            return ERR_NO_MORE_CLIENT_ALLOWED;
        }

        // Check if client already exists
        for (int i = 0; i < MAX_CLIENTS; i++)
        {
            if (m_Clients[i].bActive &&
                strcmp(m_Clients[i].szClientName, pacClientName) == 0)
            {
                ClientID = m_Clients[i].dwClientID;
                return ERR_CLIENT_EXISTS;
            }
        }

        // Add new client
        m_Clients[slot].dwClientID = m_dwNextClientID++;
        strcpy(m_Clients[slot].szClientName, pacClientName);
        m_Clients[slot].nBufferCount = 0;
        m_Clients[slot].bActive = true;
        m_nClientCount++;

        ClientID = m_Clients[slot].dwClientID;
        return S_OK;
    }
    else
    {
        // Unregister client
        if (RemoveClient(ClientID))
        {
            return S_OK;
        }
        return ERR_NO_CLIENT_EXIST;
    }
}

HRESULT CDIL_CAN_YourDevice::CAN_ManageMsgBuf(
    BYTE bAction,
    DWORD ClientID,
    CBaseCANBufFSE* pBufObj)
{
    // Find client
    int clientIndex = -1;
    for (int i = 0; i < MAX_CLIENTS; i++)
    {
        if (m_Clients[i].bActive && m_Clients[i].dwClientID == ClientID)
        {
            clientIndex = i;
            break;
        }
    }

    if (clientIndex == -1)
    {
        return ERR_NO_CLIENT_EXIST;
    }

    if (bAction == MSGBUF_ADD)
    {
        // Add buffer
        if (m_Clients[clientIndex].nBufferCount >= MAX_BUFFERS_PER_CLIENT)
        {
            return S_FALSE;
        }

        m_Clients[clientIndex].pBuffers[m_Clients[clientIndex].nBufferCount++]
            = pBufObj;
    }
    else if (bAction == MSGBUF_CLEAR)
    {
        // Clear all buffers
        m_Clients[clientIndex].nBufferCount = 0;
        memset(m_Clients[clientIndex].pBuffers, 0,
               sizeof(m_Clients[clientIndex].pBuffers));
    }

    return S_OK;
}

//=============================================================================
// STATUS AND DIAGNOSTICS
//=============================================================================
HRESULT CDIL_CAN_YourDevice::CAN_GetCurrStatus(STATUSMSG& StatusData)
{
    StatusData.wControllerStatus = m_bIsConnected ?
        defCONTROLLER_ACTIVE : 0;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_GetCntrlStatus(
    const HANDLE& hEvent,
    UINT& unCntrlStatus)
{
    unCntrlStatus = m_bIsConnected ?
        defCONTROLLER_ACTIVE : 0;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_GetControllerParams(
    LONG& lParam,
    UINT nChannel,
    ECONTR_PARAM eContrParam)
{
    switch (eContrParam)
    {
    case NUMBER_HW:
        lParam = m_nChannelCount;
        break;
    case NUMBER_CONNECTED_HW:
        lParam = m_bIsConnected ? m_nChannelCount : 0;
        break;
    case DRIVER_STATUS:
        lParam = m_bIsConnected ? 1 : 0;
        break;
    case HW_MODE:
        lParam = defCONTROLLER_ACTIVE;
        break;
    default:
        lParam = 0;
        break;
    }
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_SetControllerParams(
    int nValue,
    ECONTR_PARAM eContrparam)
{
    // Handle parameter changes
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_GetErrorCount(
    SERROR_CNT& sErrorCnt,
    UINT nChannel,
    ECONTR_PARAM eContrParam)
{
    // Get error counts from hardware
    //
    // Example:
    // YourSDK_GetErrorCounts(m_hDevice[nChannel],
    //                        &sErrorCnt.m_ucTxErrCount,
    //                        &sErrorCnt.m_ucRxErrCount);

    sErrorCnt.m_ucTxErrCount = 0;
    sErrorCnt.m_ucRxErrCount = 0;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_GetTimeModeMapping(
    SYSTEMTIME& CurrSysTime,
    UINT64& TimeStamp,
    LARGE_INTEGER& QueryTickCount)
{
    CurrSysTime = m_StartSysTime;
    TimeStamp = m_StartTimestamp;
    QueryTickCount = m_QueryTickCount;
    return S_OK;
}

HRESULT CDIL_CAN_YourDevice::CAN_GetLastErrorString(std::string& acErrorStr)
{
    acErrorStr = m_strLastError;
    return S_OK;
}

//=============================================================================
// HELPER METHODS
//=============================================================================
DWORD CDIL_CAN_YourDevice::GetAvailableClientSlot()
{
    for (int i = 0; i < MAX_CLIENTS; i++)
    {
        if (!m_Clients[i].bActive)
        {
            return i;
        }
    }
    return (DWORD)-1;
}

bool CDIL_CAN_YourDevice::RemoveClient(DWORD dwClientID)
{
    for (int i = 0; i < MAX_CLIENTS; i++)
    {
        if (m_Clients[i].bActive && m_Clients[i].dwClientID == dwClientID)
        {
            m_Clients[i].bActive = false;
            m_Clients[i].nBufferCount = 0;
            m_nClientCount--;
            return true;
        }
    }
    return false;
}

void CDIL_CAN_YourDevice::DistributeMessage(const STCANDATA& sCanData)
{
    // Send message to all registered client buffers
    for (int i = 0; i < MAX_CLIENTS; i++)
    {
        if (m_Clients[i].bActive)
        {
            for (int j = 0; j < m_Clients[i].nBufferCount; j++)
            {
                if (m_Clients[i].pBuffers[j] != nullptr)
                {
                    m_Clients[i].pBuffers[j]->WriteIntoBuffer(&sCanData);
                }
            }
        }
    }
}

//=============================================================================
// RECEIVE THREAD
//=============================================================================
DWORD WINAPI CDIL_CAN_YourDevice::ReceiveThreadProc(LPVOID pVoid)
{
    CDIL_CAN_YourDevice* pThis = (CDIL_CAN_YourDevice*)pVoid;

    while (!pThis->m_bStopReceiveThread)
    {
        // Poll for received messages
        //
        // Example:
        //
        // for (int ch = 0; ch < pThis->m_nChannelCount; ch++)
        // {
        //     YourSDK_CANMessage msg;
        //     while (YourSDK_ReceiveMessage(pThis->m_hDevice[ch], &msg)
        //            == YOURSDK_SUCCESS)
        //     {
        //         // Convert to STCANDATA
        //         STCANDATA sCanData;
        //         sCanData.m_ucDataType = RX_FLAG;
        //         sCanData.m_uDataInfo.m_sCANMsg.m_unMsgID = msg.id;
        //         sCanData.m_uDataInfo.m_sCANMsg.m_ucDataLen = msg.dlc;
        //         sCanData.m_uDataInfo.m_sCANMsg.m_ucEXTENDED = msg.extended;
        //         sCanData.m_uDataInfo.m_sCANMsg.m_ucRTR = msg.rtr;
        //         sCanData.m_uDataInfo.m_sCANMsg.m_ucChannel = ch + 1;
        //         memcpy(sCanData.m_uDataInfo.m_sCANMsg.m_ucData,
        //                msg.data, msg.dlc);
        //         QueryPerformanceCounter(&sCanData.m_lTickCount);
        //
        //         // Distribute to clients
        //         pThis->DistributeMessage(sCanData);
        //     }
        // }

        // Sleep to prevent busy-waiting
        Sleep(1);
    }

    return 0;
}
```

---

## Registering Your Adapter

### Method 1: Add to Driver List Enumeration

Modify the driver enumeration in the DIL manager to include your adapter.

In `Sources/BUSMASTER/DIL_Interface/`, find where drivers are enumerated and add your driver ID.

### Method 2: Dynamic Loading

BusMaster can also load adapters dynamically from a plugins directory. Place your DLL in the application directory.

### Driver ID Constants

Add your driver ID to the driver definitions:

```cpp
// In CANDriverDefines.h or similar

enum DRIVER_CAN
{
    DRIVER_CAN_STUB = 0,
    DRIVER_CAN_PEAK_USB,
    DRIVER_CAN_VECTOR_XL,
    DRIVER_CAN_KVASER,
    DRIVER_CAN_IXXAT_VCI,
    // ... other drivers ...
    DRIVER_CAN_YOUR_DEVICE,  // Add your driver
    DAL_CAN_TOTAL
};
```

---

## Testing Your Adapter

### Step 1: Build in Debug Mode

Build your DLL in debug mode for easier troubleshooting.

### Step 2: Use Logging

Add logging to track execution:

```cpp
#include <fstream>

void LogMessage(const char* format, ...)
{
    static std::ofstream logFile("CAN_YourDevice_Debug.log",
                                  std::ios::app);
    char buffer[1024];
    va_list args;
    va_start(args, format);
    vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    logFile << buffer << std::endl;
    logFile.flush();
}
```

### Step 3: Test Sequence

1. **Load Test**: Verify DLL loads without errors
2. **Enumeration Test**: Verify hardware is discovered
3. **Configuration Test**: Verify baud rate can be set
4. **Connect Test**: Verify communication starts
5. **TX Test**: Send a message and verify it goes out
6. **RX Test**: Receive messages from the bus
7. **Disconnect Test**: Verify clean shutdown

### Step 4: Use the STUB Adapter as Reference

The `CAN_STUB` adapter is a simulation adapter that doesn't require hardware. Use it as a reference for implementing your adapter.

---

## Common Pitfalls

### 1. Thread Safety

Message distribution must be thread-safe:

```cpp
// Use critical sections
CRITICAL_SECTION m_csClients;

void DistributeMessage(const STCANDATA& sCanData)
{
    EnterCriticalSection(&m_csClients);
    // ... distribute to clients ...
    LeaveCriticalSection(&m_csClients);
}
```

### 2. Proper Cleanup

Always clean up resources:

```cpp
HRESULT CAN_PerformClosureOperations()
{
    CAN_StopHardware();      // Stop receive thread
    CAN_DeselectHwInterface(); // Close device handles
    CAN_UnloadDriverLibrary(); // Unload SDK DLL
    return S_OK;
}
```

### 3. Channel Numbering

BusMaster uses 1-based channel numbers, but most SDKs use 0-based:

```cpp
// When sending/receiving
int sdkChannel = sCanTxMsg.m_ucChannel - 1;  // Convert to 0-based

// When receiving
sCanData.m_uDataInfo.m_sCANMsg.m_ucChannel = sdkChannel + 1;  // Convert to 1-based
```

### 4. Timestamp Handling

Use `QueryPerformanceCounter` for consistent timestamps:

```cpp
LARGE_INTEGER tickCount;
QueryPerformanceCounter(&tickCount);
sCanData.m_lTickCount = tickCount;
```

### 5. Error Handling

Always provide meaningful error messages:

```cpp
if (YourSDK_OpenDevice(...) == FAILED)
{
    m_strLastError = "Failed to open device: ";
    m_strLastError += YourSDK_GetLastErrorString();
    return S_FALSE;
}
```

---

## Next Steps

- [04-quick-reference.md](04-quick-reference.md) - Quick reference guide
- Look at existing adapters (`CAN_Vector_XL`, `CAN_STUB`) for more examples
- Test thoroughly with BusMaster before deployment
