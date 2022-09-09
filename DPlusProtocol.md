# DPlus Protocol

The DPlus protocol is used to route DStar data from a repeater to a reflector or sometimes between reflectors. It works with REF and XRF type reflectors. The protocol goes over UDP with the server listening on port 20001 and the client sending from any port.

## High Level

1. client sends connect request
2. server says "go ahead"
3. client says "here's my call, can I connect?"
4. server says yay / nay
5. server and client start exchanging ping / pong
6. client sends header. no ack, so send a few times
7. client starts sending frames
8. every time the packet counter reaches 0x14 (20), send header a few times
9. client sends end of transmission
10. server sends a c0 packet?!
11. go back to 5.
12. disconnect


## Packets

Packets in the protocol are structured as:

| Byte | Description |
|------|-------------|
| 0    | Length      |
| 1    | Type        |
| ...  | Payload     |

The length includes the length packet. For example, the ping packet {0x03, 0x60, 0x00 } has 0x03 at byte zero because it's 3 bytes long.

The protocol defines the following packet types:

| Id   | Type     | Direction     | Description                 |
|------|----------|---------------|-----------------------------|
| 0x00 | Connect  | Client -> Srv | Request session from server |
| 0xC0 | Login    | Client -> Srv | Send ask server for auth    |
| 0xC0 | LoginAck | Srv -> Client | Successful / Failed Login   |
| 0xC0 | Eot Ack  | Srv -> Client | Received transmission (?)   |
| 0x60 | Ping     | Client -> Srv | Are you still there?        |
| 0x60 | Pong     | Srv -> Client | Yes, still there            |
| 0x80 | Header   | Client -> Srv | DStar routing info          |
| 0x80 | Frame    | Client -> Srv | Voice and Data              |
| 0x80 | Eot      | Client -> Srv | End of transmission         |

### Connect

Sent by the client to initiate a connection. The server will echo back the same packet to indicate it's available and ready to accept a connection.

Example:

    0000  05 00 18 00 01                                    .....

The last byte indicates if this is a connect or a disconnect packet.

Example disconnect:

Example:

    0000  05 00 18 00 00                                    .....

### Login

Sent by the client to request authorization for sending transmissions.

Example:

    0000  1c c0 04 00 41 49 36 56 57 00 00 00 00 00 00 00   ....AI6VW.......
    0010  00 00 00 00 44 56 30 31 39 39 39 34               ....DV019994

    0    |  1   |   2  |    3
    0x1C | 0xC0 | 0x04 | 0x00

    4      5      6      7      8      9      10     11
    0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00

    12     13     14     15     16     17     18     19
    0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00

    20     21     22     23     24     25     26     27
    0x44 | 0x56 | 0x30 | 0x31 | 0x39 | 0x39 | 0x39 | 0x34

    Byte:
    0       Length of frame (28 bytes)
    1       Type is login
    4..11   mycall (8 bytes)
    12..19  no idea
    20..27  'DV019994'

### Login Ack/Nak

Sent by the server to indicate if the client may connect.

Example:

    0      1      2      3      4     5     6     7
    0x08 | 0xC0 | 0x04 | 0x00 | 'O' | 'K' | 'R' | 'W' 

    0      1      2      3      4     5     6     7
    0x08 | 0xC0 | 0x04 | 0x00 | 'B' | 'U' | 'S' | 'Y' 

    Bytes:
    0       Length
    4..7    Response type

### Ping / Pong

The client will periodically send ping requests. The server will respond by echoing back the packet.

    0      1      2
    0x03 | 0x60 | 0x00

### Header

First package in a voice / data transmission. Contains routing information.
Server verifies mycall and rpt2 before the transmission goes on the air.
The header carries a randomly generated stream id which associates subsequent
header or voice frames with the same stream. Once the stream is closed, the
client will generate a new stream id. A second header for the same session
will be inored but the connection timeout timer will be reset.

Example:

    0000  3a 80 44 53 56 54 10 00 00 00 20 00 02 01 7d 37   :.DSVT.... ...}7
    0010  80 00 00 00 52 45 46 30 33 30 20 43 41 49 36 56   ....REF030 CAI6V
    0020  57 20 20 44 43 51 43 51 43 51 20 20 41 49 36 56   W  DCQCQCQ  AI6V
    0030  57 20 20 20 49 44 35 32 00 0b                     W   ID52..

    0      1      2     3     4     5     6      7      
    0x3A | 0x80 | 'D' | 'S' | 'V' | 'T' | 0x10 | 0x00

    8      9      10     11     12     13     14     15     16
    0x00 | 0x00 | 0x20 | 0x00 | 0x02 | 0x01 | 0x00 | 0x00 | 0x80

    [ some dstar routing stuff ]

    56     57
    0x00 | 0x0B

| Bytes | description |
|-------|-------------|
| 0     | Length (58 bytes) |
| 2     | 0x80 |
| 3..5  | 'D' 'S' 'V' 'T' |
| 6     | 0x10 |
| 7     | 0x00 |
| 8     | 0x00 |
| 9     | 0x00 |
| 10    | 0x20 |
| 11    | 0x00 |
| 12    | 0x02 |
| 13    | 0x01 |
| 14    | session id byte 1 |
| 15    | session id byte 2 |
| 16    | 0x80 |
| 17    | 0x00 Flag 1 |
| 18    | 0x00 Flag 2 |
| 19    | 0x00 Flag 3 |
| 20..27 | 'REF030 C' Rpt2    |
| 28..35 | 'AI6VW  D' Rpt1    |
| 36..43 | 'CQCQCQ  ' ur call |
| 44..51 | 'AI6VW   ' my call |
| 52..55 | 'ID52'     suffix  |
| 56     | 0x00 crc byte 1    |
| 57     | 0x0b crc byte 2    |

The check sum can be simply piped through from the radio, no need to calculate it.

### Frame

Used to transmit AMBE voice and slow data.

Example:

    0000  1d 80 44 53 56 54 20 00 00 00 20 00 02 01 7d 37   ..DSVT ... ...}7
    0010  01 5e a5 06 52 15 b0 46 20 b6 25 4f 93            .^..R..F .%O.

| Byte | Description |
|------|-------------|
| 0    | Len (29 bytes) |
| 1    | 0x80 |
| 2..5 | 'D' 'S' 'V' 'T' |
| 6    | 0x20 |
| 7    | 0x00 Flag 1 |
| 8    | 0x00 Flag 2 |
| 8    | 0x00 Flag 3 |
| 9    | 0x20 |
| 10   | 0x00 |
| 11   | 0x02 |
| 12   | 0x01 |
| 13   | session id byte 1 |
| 14   | session id byte 2 |
| 15   | packet id |
| 16..18 | 12 bytes = 9 bytes voice + 3 bytes data |

### Frame End of Transmission

Sent to indicate this is the last frame in a transmission. Note that this is the same "empty voice, last frame" special frame payload we say in the Icom serial protocol.

Example:

    0000  20 80 44 53 56 54 20 00 00 00 20 00 02 01 7d 37    .DSVT ... ...}7
    0010  52 9e 8d 32 88 26 1a 3f 61 e8 55 55 55 55 c8 7a   R..2.&.?a.UUUU.z

| Byte | Description |
|------|-------------|
| 0    | Len (32 bytes) |
| 1    | 0x80 |
| 2..5 | 'D' 'S' 'V' 'T' |
| 6    | 0x20 |
| 7    | 0x00 |
| 8    | 0x00 |
| 9    | 0x00 |
| 10    | 0x20 |
| 11   | 0x00 |
| 12   | 0x02 |
| 13   | 0x01 |
| 14   | session id byte 1 |
| 15   | session id byte 2 |
| 16   | packet id |
| 17..28  | 12 bytes = 9 bytes voice + 3 bytes data  |
| 29 | ?? |
| 30 | ?? |
| 31 | ?? |

### Ack End of Transmission

Example:

    0000  0a c0 0b 00 7d 37 3e 00 00 00                     ....}7>...

| Byte | Description |
|------|-------------|
| 0    | Len (10 bytes) |
| 1    | 0xC0           |
| 2    | 0x0b           |
| 3    | 0x00           |
| 4..5 | session id     |
| 6    | 0x3E           |
| 7..9 | 0x00           |
