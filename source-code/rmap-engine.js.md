/\*\*  
 \* ECSS-E-ST-50-52C Remote Memory Access Protocol (RMAP) Core Engine  
 \* Implements packet assembly, dual CRC-8 calculations, instruction word decoding,  
 \* and memory transaction simulation for SpaceWire networks.  
 \*/

// RMAP Protocol ID  
const RMAP\_PROTOCOL\_ID \= 0x01;

// Packet Types (Instruction Byte Bits 5:4)  
const RMAP\_PACKET\_TYPES \= {  
  WRITE: 0x02,  
  READ: 0x01,  
  RMW: 0x03  
};

class RMAPEngine {  
  constructor(initiatorLogicalAddress \= 0xFE, key \= 0xD2) {  
    this.initiatorAddress \= initiatorLogicalAddress;  
    this.authKey \= key;  
    this.transactionId \= 0;  
  }

  getNextTransactionId() {  
    const tid \= this.transactionId;  
    this.transactionId \= (this.transactionId \+ 1\) & 0xFFFF;  
    return tid;  
  }

  /\*\*  
   \* ECSS-E-ST-50-52C CRC-8 Calculation  
   \* Polynomial: X^8 \+ X^2 \+ X \+ 1 (0x07)  
   \*/  
  static calculateCrc8(bytes) {  
    let crc \= 0x00;  
    for (let b of bytes) {  
      crc ^= b;  
      for (let i \= 0; i \< 8; i++) {  
        if ((crc & 0x80) \!== 0\) {  
          crc \= ((crc \<\< 1\) ^ 0x07) & 0xFF;  
        } else {  
          crc \= (crc \<\< 1\) & 0xFF;  
        }  
      }  
    }  
    return crc & 0xFF;  
  }

  /\*\*  
   \* Encodes the 1-byte RMAP Instruction field  
   \* Bit 7: Packet Type (1 \= Command, 0 \= Reply)  
   \* Bit 6: Write/Read (1 \= Write, 0 \= Read)  
   \* Bit 5: Verify Data (1 \= Verify, 0 \= Don't verify)  
   \* Bit 4: Reply Mode (1 \= Acknowledge / Reply requested)  
   \* Bit 3: Increment Address (1 \= Increment, 0 \= Non-incrementing FIFO)  
   \* Bits 1:0: Extended Address (2 bits)  
   \*/  
  static encodeInstruction(isWrite, isVerify, isReply, isIncrement, extAddr \= 0\) {  
    let inst \= 0x40; // Bit 6 \= Command  
    if (isWrite) inst \|= (1 \<\< 5);  
    if (isVerify) inst \|= (1 \<\< 4);  
    if (isReply) inst \|= (1 \<\< 3);  
    if (isIncrement) inst \|= (1 \<\< 2);  
    inst \|= (extAddr & 0x03);  
    return inst;  
  }

  /\*\*  
   \* Serializes an RMAP Write Command Packet  
   \*/  
  serializeWriteCommand(targetLogicalAddr, memAddress32, dataBytes, isVerify \= true, isIncrement \= true) {  
    const tid \= this.getNextTransactionId();  
    const instruction \= RMAPEngine.encodeInstruction(true, isVerify, true, isIncrement, 0);

    // Header bytes subject to Header CRC  
    const headerBytes \= \[  
      targetLogicalAddr & 0xFF,  
      RMAP\_PROTOCOL\_ID,  
      instruction,  
      this.authKey,  
      this.initiatorAddress,  
      (tid \>\> 8\) & 0xFF,  
      tid & 0xFF,  
      0x00, // Extended Address  
      (memAddress32 \>\> 24\) & 0xFF,  
      (memAddress32 \>\> 16\) & 0xFF,  
      (memAddress32 \>\> 8\) & 0xFF,  
      memAddress32 & 0xFF,  
      (dataBytes.length \>\> 16\) & 0xFF,  
      (dataBytes.length \>\> 8\) & 0xFF,  
      dataBytes.length & 0xFF  
    \];

    const headerCrc \= RMAPEngine.calculateCrc8(headerBytes);  
    const dataCrc \= RMAPEngine.calculateCrc8(dataBytes);

    const fullPacket \= \[  
      ...headerBytes,  
      headerCrc,  
      ...dataBytes,  
      dataCrc  
    \];

    return {  
      type: 'WRITE\_COMMAND',  
      targetAddress: '0x' \+ (targetLogicalAddr & 0xFF).toString(16).toUpperCase().padStart(2, '0'),  
      transactionId: tid,  
      memoryAddress: '0x' \+ (memAddress32 \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      dataLength: dataBytes.length,  
      headerCrc8: '0x' \+ headerCrc.toString(16).toUpperCase().padStart(2, '0'),  
      dataCrc8: '0x' \+ dataCrc.toString(16).toUpperCase().padStart(2, '0'),  
      totalBytes: fullPacket.length,  
      wireBytes: fullPacket,  
      hexDump: fullPacket.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }

  /\*\*  
   \* Serializes an RMAP Read Command Packet  
   \*/  
  serializeReadCommand(targetLogicalAddr, memAddress32, lengthBytes, isIncrement \= true) {  
    const tid \= this.getNextTransactionId();  
    const instruction \= RMAPEngine.encodeInstruction(false, false, true, isIncrement, 0);

    const headerBytes \= \[  
      targetLogicalAddr & 0xFF,  
      RMAP\_PROTOCOL\_ID,  
      instruction,  
      this.authKey,  
      this.initiatorAddress,  
      (tid \>\> 8\) & 0xFF,  
      tid & 0xFF,  
      0x00, // Extended Address  
      (memAddress32 \>\> 24\) & 0xFF,  
      (memAddress32 \>\> 16\) & 0xFF,  
      (memAddress32 \>\> 8\) & 0xFF,  
      memAddress32 & 0xFF,  
      (lengthBytes \>\> 16\) & 0xFF,  
      (lengthBytes \>\> 8\) & 0xFF,  
      lengthBytes & 0xFF  
    \];

    const headerCrc \= RMAPEngine.calculateCrc8(headerBytes);  
    const fullPacket \= \[...headerBytes, headerCrc\];

    return {  
      type: 'READ\_COMMAND',  
      targetAddress: '0x' \+ (targetLogicalAddr & 0xFF).toString(16).toUpperCase().padStart(2, '0'),  
      transactionId: tid,  
      memoryAddress: '0x' \+ (memAddress32 \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      readLength: lengthBytes,  
      headerCrc8: '0x' \+ headerCrc.toString(16).toUpperCase().padStart(2, '0'),  
      totalBytes: fullPacket.length,  
      wireBytes: fullPacket,  
      hexDump: fullPacket.map(b \=\> b.toString(16).padStart(2, '0').toUpperCase()).join(' ')  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    RMAPEngine,  
    RMAP\_PROTOCOL\_ID,  
    RMAP\_PACKET\_TYPES  
  };  
}  
