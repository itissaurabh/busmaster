# BusMaster Architecture Overview

This document provides a comprehensive overview of the BusMaster architecture to help developers understand, modify, and extend the codebase.

## Table of Contents

1. [Introduction](#introduction)
2. [High-Level Architecture](#high-level-architecture)
3. [Directory Structure](#directory-structure)
4. [Core Components](#core-components)
5. [Data Flow](#data-flow)
6. [Technology Stack](#technology-stack)

---

## Introduction

BusMaster is an open-source tool for development, monitoring, and analysis of CAN (Controller Area Network), LIN (Local Interconnect Network), and J1939 bus systems. It is primarily designed for automotive and industrial applications.

### Key Features

- Multi-protocol support (CAN, CAN FD, LIN, J1939)
- Support for multiple hardware adapters (PCAN, Vector, Kvaser, IXXAT, etc.)
- Message logging and replay
- Signal watch and interpretation
- Node simulation with scripting
- Transmission window for sending messages
- Database (DBC/DBF) support for message interpretation

---

## High-Level Architecture

```
+------------------------------------------------------------------+
|                        BusMaster Application                       |
|  +-------------------------------------------------------------+  |
|  |                     User Interface (MFC)                     |  |
|  |  +------------+ +------------+ +------------+ +------------+ |  |
|  |  | Message    | | Signal     | | TX Window  | | Node Sim   | |  |
|  |  | Window     | | Watch      | |            | | Engine     | |  |
|  |  +------------+ +------------+ +------------+ +------------+ |  |
|  +-------------------------------------------------------------+  |
|                              |                                     |
|  +-------------------------------------------------------------+  |
|  |              DIL Interface (Device Interface Layer)          |  |
|  |                    Hardware Abstraction Layer                 |  |
|  +-------------------------------------------------------------+  |
|                              |                                     |
|  +-------------------------------------------------------------+  |
|  |                   Hardware Adapter DLLs                       |  |
|  |  +---------+ +---------+ +---------+ +---------+ +---------+ |  |
|  |  | PCAN    | | Vector  | | Kvaser  | | IXXAT   | | STUB    | |  |
|  |  | USB     | | XL      | | CAN     | | VCI     | | (Sim)   | |  |
|  |  +---------+ +---------+ +---------+ +---------+ +---------+ |  |
|  +-------------------------------------------------------------+  |
+------------------------------------------------------------------+
                              |
               +--------------+--------------+
               |              |              |
          +--------+     +--------+     +--------+
          | PCAN   |     | Vector |     | Other  |
          | Device |     | Device |     | Device |
          +--------+     +--------+     +--------+
```

---

## Directory Structure

```
busmaster/
├── Sources/
│   ├── BUSMASTER/                    # Main application source code
│   │   ├── Application/              # Main application entry point and core UI
│   │   │   ├── BUSMASTER.cpp         # Application entry point
│   │   │   ├── MainFrm.cpp           # Main window implementation
│   │   │   └── BusmasterPluginManager.cpp  # Plugin system
│   │   │
│   │   ├── DIL_Interface/            # Device Interface Layer manager
│   │   │   ├── HardwareListingCAN.cpp # Hardware enumeration UI
│   │   │   └── DILC_Dummy.cpp        # Dummy DIL for testing
│   │   │
│   │   ├── CAN_Vector_XL/            # Vector XL adapter implementation
│   │   ├── CAN_PEAK_USB/             # PEAK USB adapter implementation
│   │   ├── CAN_Kvaser_CAN/           # Kvaser adapter implementation
│   │   ├── CAN_IXXAT_VCI/            # IXXAT VCI adapter implementation
│   │   ├── CAN_ICS_neoVI/            # IntrepidCS neoVI adapter
│   │   ├── CAN_ETAS_BOA/             # ETAS BOA adapter
│   │   ├── CAN_STUB/                 # Simulation stub (no hardware)
│   │   ├── CAN_ISOLAR_EVE_VCAN/      # ISOLAR EVE Virtual CAN
│   │   ├── CAN_VSCOM/                # VSCOM adapter
│   │   ├── CAN_MHS/                  # MHS adapter
│   │   ├── CAN_NSI/                  # NSI adapter
│   │   ├── CAN_iVIEW/                # iVIEW adapter
│   │   │
│   │   ├── LIN_Vector_XL/            # LIN protocol adapters
│   │   ├── LIN_PEAK_USB/
│   │   ├── LIN_Kvaser/
│   │   ├── LIN_ETAS_BOA/
│   │   ├── LIN_ISOLAR_EVE_VLIN/
│   │   │
│   │   ├── FrameProcessor/           # Message logging and processing
│   │   ├── Filter/                   # Message filtering
│   │   ├── Replay/                   # Log file replay
│   │   ├── SignalWatch/              # Signal monitoring UI
│   │   ├── SignalDefiner/            # Signal definition editor
│   │   ├── TXWindow/                 # Message transmission UI
│   │   ├── NodeSimEx/                # Node simulation engine
│   │   ├── ProjectConfiguration/     # Configuration management
│   │   ├── BusEmulation/             # Virtual bus for testing
│   │   │
│   │   ├── Include/                  # Common header files
│   │   │   ├── Struct_CAN_.h         # CAN message structures
│   │   │   └── BaseDefs.h            # Base definitions
│   │   │
│   │   ├── CommonClass/              # Shared utility classes
│   │   │   └── MsgContainerBase.h    # Message buffer base class
│   │   │
│   │   └── DataTypes/                # Data structure definitions
│   │
│   └── Kernel/                       # Core driver infrastructure
│       ├── BusmasterKernel/          # Main kernel DLL
│       ├── BusmasterDriverInterface/ # Base DIL interfaces
│       │   └── Include/
│       │       ├── BaseDIL_CAN.h     # CAN DIL interface
│       │       ├── BaseDIL_CAN_Controller.h  # CAN controller base class
│       │       └── CANDriverDefines.h # CAN driver definitions
│       ├── BusmasterDBNetwork/       # Database network interface
│       ├── ProtocolDefinitions/      # Protocol-specific definitions
│       └── Utilities/                # Utility functions
│
├── Documents/                        # Documentation
├── Tests/                            # Test files
└── tutorials/                        # Tutorial documentation (this folder)
```

---

## Core Components

### 1. Application Layer (`Sources/BUSMASTER/Application/`)

The main application layer handles:
- Application initialization and lifecycle
- User interface (MFC-based Windows GUI)
- Plugin management
- Configuration persistence

**Key Files:**
- `BUSMASTER.cpp:76` - `CCANMonitorApp::InitInstance()` - Application entry point
- `MainFrm.cpp` - `CMainFrame` class - Main window with all UI elements
- `BusmasterPluginManager.cpp` - Plugin loading and management

### 2. DIL Interface Layer (`Sources/BUSMASTER/DIL_Interface/`)

The Device Interface Layer (DIL) provides hardware abstraction:
- Enumerates available hardware
- Manages hardware selection
- Routes messages between application and hardware adapters

### 3. Hardware Adapters (`Sources/BUSMASTER/CAN_*/`)

Each hardware adapter is a separate DLL that:
- Implements the `CBaseDIL_CAN_Controller` interface
- Communicates with vendor-specific hardware APIs
- Handles message reception and transmission

### 4. Kernel Layer (`Sources/Kernel/`)

The kernel provides:
- Base interfaces for all DIL implementations
- Common data structures
- Protocol definitions
- Utility functions

---

## Data Flow

### Message Reception Flow

```
Hardware Device
      │
      ▼
[Vendor SDK/API]
      │
      ▼
[Hardware Adapter DLL] ──────────────────────┐
      │                                       │
      │  Converts to STCAN_MSG structure      │
      ▼                                       │
CBaseDIL_CAN_Controller                       │
      │                                       │
      │  Routes to registered clients         │
      ▼                                       │
CBaseCANBufFSE* (Message Buffer)              │
      │                                       │
      │  Thread-safe circular buffer          │
      ▼                                       │
┌─────┴─────────────────────────┐             │
│  Message Consumers            │             │
├───────────────────────────────┤             │
│ • Message Window (UI display) │             │
│ • FrameProcessor (logging)    │             │
│ • SignalWatch (signal decode) │             │
│ • Node Simulation (scripts)   │             │
└───────────────────────────────┘             │
```

### Message Transmission Flow

```
┌─────────────────────────────────────────┐
│ TX Window / Node Simulation / Scripts   │
└─────────────────────┬───────────────────┘
                      │
                      ▼
              DILC_SendMsg()
                      │
                      ▼
          CBaseDIL_CAN interface
                      │
                      ▼
    [Hardware Adapter DLL::CAN_SendMsg()]
                      │
                      ▼
            [Vendor SDK/API]
                      │
                      ▼
              Hardware Device
```

---

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | C++ (Visual C++) |
| UI Framework | MFC (Microsoft Foundation Classes) |
| Build System | Visual Studio Solution / MSBuild |
| XML Processing | libxml2 |
| Lexer/Parser | Flex/Bison (for DBC files) |
| Hardware APIs | Vendor-specific SDKs |

### Dependencies

- **Windows SDK** - Core Windows functionality
- **MFC** - User interface
- **libxml2** - XML configuration file parsing
- **Vendor SDKs** - PCAN, Vector, Kvaser, IXXAT, etc.

---

## Next Steps

- [02-code-walkthrough.md](02-code-walkthrough.md) - Detailed code walkthrough
- [03-creating-custom-adapter.md](03-creating-custom-adapter.md) - How to create a custom CAN adapter
- [04-quick-reference.md](04-quick-reference.md) - Quick reference guide
