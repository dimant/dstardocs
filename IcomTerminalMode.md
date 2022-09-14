# Icom Terminal Mode Protocol

The terminal mode protocol of the Icom ID52 is a serial protocol which is routed as RS232 over USB. Its purpose is to send voice and data transmissions to a computer application which then routes them over a network to a receiver gateway or reflector.

The serial port settings for the protocol are:

| Setting  | Value |
|----------|-------|
| BaudRate | 38400 |
| DataBits | 8     |
| Parity   | None  |
| StopBits | One   |
| Handshake| None  |
| RtsEnable| false |

## High Level

The protocol is binary with each packet having the following structure:

| Byte | Description |
|------|-------------|
| 0    | Length      |
| 1    | Type        |
| ...  | Payload     |
| Last | 0xFF        |

The length of the packet is measured starting at byte 1. For example, the ping packet: { 0x02, 0x02, 0xFF } has 0x02 at position 0. This indicates 2 bytes following the length byte.

The payload typically starts with one or more bytes with status flags, packet identification numbers, and ends with the payload itself. The structure of each payload depends on the packet type.

The protocol has two directions:

1. Voice and data sent from transceiver to computer
2. Voice and data sent from computer to transceiver

When the transceiver sends data, it is sent as fast as the transceiver can send it. There is no flow control. In the opposite direction, data is sliced into chunks and the computer must wait on an ACK packet before sending each following chunk.

When there are no ongoing transmissions, the computer sends periodic ping packets to the transceiver. If the transceiver replies with pong, we know that the serial connection is established. If the transceiver doesn't respond, we can attempt to reset the connection by spamming bytes of 0xFF on the line. Typically between 5 and 100 bytes are sufficient. If we still fail to receive a pong, we must assume that the connection is down and surface an error to the computer application.

## Packets

The protocol defines the following packet types:

| Id    | Type       | Direction     | Description |
|-------|------------|---------------|-------------|
| 0x02  | Ping       | Comp -> Radio | Keep alive  |
| 0x03  | Pong       | Radio -> Comp | I am alive  |
| 0x10  | Header     | Radio -> Comp | DStar routing information |
| 0x12  | Frame      | Radio -> Comp | AMBE voice and slow data  |
| 0x20  | Header     | Comp -> Radio | DStar routing information |
| 0x21  | Header Ack | Radio -> Comp | Radio received header     |
| 0x22  | Frame      | Comp -> Radio | AMBE voice and slow data  |
| 0x23  | Frame Ack  | Radio -> Comp | Radio received frame      |

### Ping
 
The ping packet is periodically sent from the computer to the radio when there is no ongoing transmission. This ensures that the connection from the computer to the transceiver is alive and functioning.

| Byte | Value | Description |
|------|-------|-------------|
| 0    | 0x02  | Length      |
| 1    | 0x02  | Type        |
| 2    | 0xFF  | End         |

### Pong

The transceiver replies to ping requests with the pong packet. The pong packet is also used to indicate that the transceiver is ready to receive frames after acknowledging a header packet.

| Byte | Value | Description |
|------|-------|-------------|
| 0    | 0x03  | Length      |
| 1    | 0x03  | Type        |
| 2    | 0x00  | Flag        |
| 3    | 0xFF  | End         |

The status byte may be:

- 0x00 indicating the transceiver is ready to send data to the computer
- 0x01 indicating the transceiver is ready to receive data from the computer

### Header From Transceiver

After keying down, the first packet the computer receives is a header. The packet carries status flags, routing information, and a checksum.

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x2c  | Length (44 bytes) |
| 1      | 0x10  | Type              |
| 2      | 0x00  | Flag 1            |
| 3      | 0x00  | Flag 2            |
| 4      | 0x00  | Flag 3            |
| 5..12  | 'AA1BBC C' | Rpt1 (8 bytes) |
| 13..20 | 'BB2DDE A' | Rpt2 (8 bytes) |
| 21..28 | 'CQCQCQ  ' | ur call (8 bytes) |
| 29..36 | 'YZ1AB   ' | my call (8 bytes) |
| 37..40 | 'ID52'     | suffix (4 bytes) |
| 41     | 0x00       | CRC high byte|
| 42     | 0x00       | CRC low byte |
| 43     | 0x00       | rx status    |
| 44     | 0xFF       | End          |

No idea what flag 1, 2, and 3 are used for. No idea how rx status is interpreted. It seems to be fine to just pipe the CRC through to the network without worrying about it.

### Frame From Transceiver

Immediately following the header packet are packets with voice and data payloads.

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x10  | Length (16 bytes) |
| 1      | 0x12  | Type              |
| 2      | 0x00  | Packet Id         |
| 3      | 0x00  | Sequence Id       |
| 4..12  |       | 9 Bytes AMBE data |
| 13..15 |       | 3 Bytes slow data |
| 16     | 0xFF  | End               |

The packet id goes from 0 to 20 (decimal) and then wraps back to 0. The sequence id remains the same for each key-down of the transceiver.

### Header To Transceiver

The header sent to the transceiver is similar to the one received from the transceiver. Note that the packet is only 41 bytes, there is no checksum and no rx status byte.

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x29  | Length (41 bytes) |
| 1      | 0x20  | Type              |
| 2      | 0x01  | Flag 1            |
| 3      | 0x00  | Flag 2            |
| 4      | 0x00  | Flag 3            |
| 5..12  | 'AA1BBC C' | Rpt1 (8 bytes) |
| 13..20 | 'BB2DDE A' | Rpt2 (8 bytes) |
| 21..28 | 'CQCQCQ  ' | ur call (8 bytes) |
| 29..36 | 'YZ1AB   ' | my call (8 bytes) |
| 37..40 | 'ID52'     | suffix (4 bytes) |
| 43     | 0xFF       | End          |


### Header To Transceiver Ack

Acknowledging the header is done with 2 packets. First, the header is acknowledged. In a second packet, the transceiver acknowledges it is ready to receive data frames.

Header Ack Packet:

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x03  | Length (3 bytes)  |
| 1      | 0x21  | Type              |
| 2      | 0x00  | Flag              |
| 3      | 0xFF  | End               |

I presume the flag is 0x00 for ack and 0x01 for nak.

Receiver ready is indicated with a pong packet:

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x03  | Length (3 bytes)  |
| 1      | 0x03  | Type              |
| 2      | 0x01  | Flag              |
| 3      | 0xFF  | End               |

The flag is 0x01 indicating the receiver is ready for a data frame.

### Frame To Transceiver

The data frame is similar to the one sent from the transceiver to the computer. 

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x10  | Length (16 bytes) |
| 1      | 0x22  | Type              |
| 2      |       | Sequence Id         |
| 3      |       | Frame Type (bitmask 0xC0 |
| 3      |       | Number Id (bitmask 0x1F |
| 4..12  |       | 9 Bytes AMBE data |
| 13..15 |       | 3 Bytes slow data |
| 16     | 0xFF  | End               |

### Frame To Transceiver Ack

The transceiver will respond with an ack packet for each data frame. 

| Byte   | Value | Description       |
|--------|-------|-------------------|
| 0      | 0x04  | Length (4 bytes)  |
| 1      | 0x23  | Type              |
| 2      | 0x00  | Packet Id         |
| 3      | 0x00  | Status            |
| 4      | 0x00  | End               |

The packet id indicates which data frame the transceiver is acknowledging. The status will be 0x00 if the transceiver is ready to receive more data. If the transceiver responds with 0x01 for the status byte, it means the transceiver is not ready for more data. Possibly because its input queue is full. In this case the computer must wait for a second ack packet with the byte set to 0x00.

The timing of sending data frames is critical. To prevent overfilling the receiver queue, a wait time of around 10ms between packets seems to work. It is also crucial that the transceiver keeps receiving data frames even when the data coming from the internet may not be available yet.

### Special Frames

Several special frames are used in the protocol.

The **end of transmission frame** is sent after the last data frame in a transmission:

        0x10 0x22 0x08 0x48 0x55 0xc8 0x7a 0x55 
        0x55 0x55 0x55 0x55 0x55 0x55 0x55 0x55
        0xFF

The **empty voice, empty data** frame is sent if we need to reply to a frame ack but there is no data in the network input buffer. The packet id and sequence id are determined in the usual way for the current transmission.

        0x10 0x22 0x00 0x00 0x9E 0x8D 0x32 0x88 
        0x26 0x1A 0x3F 0x61 0xE8 0x97 0xCB 0xE5
        0xFF

The **empty voice, data sync** frame is sent if the outgoing packet id is 0.

        0x10 0x22 0x00 0x00 0x9E 0x8D 0x32 0x88 
        0x26 0x1A 0x3F 0x61 0xE8 0x55 0x2D 0x16
        0xFF

The **empty voice, last frame** frame is sent if we are about to terminate the transmission because too many packets incoming from the network did not arrive in time. This frame is followed by the **end of transmission frame**.

        0x10 0x22 0x00 0x00 0x9E 0x8D 0x32 0x88 
        0x26 0x1A 0x3F 0x61 0xE8 0x55 0x55 0x55
        0xFF
