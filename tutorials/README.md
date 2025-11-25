# BusMaster Developer Tutorials

This folder contains comprehensive tutorials for understanding and modifying the BusMaster codebase.

## Tutorial Index

| Document | Description |
|----------|-------------|
| [01-architecture-overview.md](01-architecture-overview.md) | High-level architecture, directory structure, and component overview |
| [02-code-walkthrough.md](02-code-walkthrough.md) | Detailed code walkthrough with key classes, interfaces, and data structures |
| [03-creating-custom-adapter.md](03-creating-custom-adapter.md) | Step-by-step guide to create a custom CAN hardware adapter (like PCAN) |
| [04-quick-reference.md](04-quick-reference.md) | Quick reference guide with code snippets and checklists |

## Getting Started

If you're new to the BusMaster codebase, we recommend reading the tutorials in order:

1. **Start with Architecture Overview** - Understand the overall structure and component relationships
2. **Read the Code Walkthrough** - Learn about key classes, data structures, and code flow
3. **Follow the Custom Adapter Guide** - When ready to integrate your own hardware
4. **Use Quick Reference** - Keep handy while developing

## Common Use Cases

### I want to understand how BusMaster works
Start with [01-architecture-overview.md](01-architecture-overview.md) and then [02-code-walkthrough.md](02-code-walkthrough.md).

### I want to connect my custom CAN device
Read [03-creating-custom-adapter.md](03-creating-custom-adapter.md) for a complete guide with example code.

### I need a quick reminder about the API
Check [04-quick-reference.md](04-quick-reference.md) for quick lookup of structures, methods, and return codes.

## Key Concepts

### DIL (Device Interface Layer)
The hardware abstraction layer that allows BusMaster to work with different CAN adapters through a common interface.

### Hardware Adapter
A DLL that implements the `CBaseDIL_CAN_Controller` interface to communicate with specific CAN hardware (PCAN, Vector, Kvaser, etc.).

### Client Registration
Applications register as "clients" to receive CAN messages. Each client has message buffers that receive copies of all messages.

### Message Flow
```
Hardware → Adapter DLL → DIL Interface → Client Buffers → UI/Logging
```

## Contributing

When adding to these tutorials:
- Keep examples practical and code-focused
- Update the README when adding new documents
- Cross-reference related documents
- Include file paths when referencing source code
