# ECSS-E-ST-50-52C Remote Memory Access Protocol (RMAP) Technical Architecture

## 1\. Executive Summary & Protocol Overview

The Remote Memory Access Protocol (RMAP), standardized under ECSS-E-ST-50-52C by the European Cooperation for Space Standardization (ECSS), is the primary transport protocol used across SpaceWire (ECSS-E-ST-50-12C) networks for space flight instrumentation and on-board computers.

### Key Architectural Concepts:

1. **Memory-Mapped Architecture**:  
   - Allows an Initiator (flight computer or instrument controller) to perform atomic read, write, and read-modify-write operations on a Target node's internal registers and memory without requiring application-level software intervention.  
2. **Dual CRC-8 Checksum Protection**:  
   - Polynomial \$X^8 \+ X^2 \+ X \+ 1\$ (\$0x07\$).  
   - Header CRC-8 protects routing, target address, instruction byte, authorization key, and data length fields.  
   - Separate Data CRC-8 protects user payload bytes, ensuring corrupt packets are rejected before affecting hardware memory.  
3. **Addressing & Routing**:  
   - **Path Addressing**: Sequences of SpaceWire router output port numbers guiding the packet through complex routing meshes.  
   - **Logical Addressing**: Single-byte logical address (values \$0x20\$ to \$0xFE\$) translated by SpaceWire router routing tables.  
4. **Safety & Hardware Authorization**:  
   - Authorization Key (`AuthKey`) prevents stray commands caused by single-event upsets (SEU) from triggering unauthorized hardware state changes.  
   - Non-incrementing address mode supports direct FIFO streaming to instruments.