TS3 PROTOCOL PAPER
==================

# 0. Naming Conventions
- `(Client -> Server)` denotes packets from client to server.
- `(Client <- Server)` denotes packets from server to client.
- All datatypes are sent in network order (Big Endian) unless otherwise
specified.
- Datatypes are declared with a prefixing `u` or `i` for unsigned and signed
and a number for the bitlength.  
For example `u8` would be the C equivalent of `uint8` or `unsigned char`
- Arrays are represented by the underlying datatype in square brackets,
additionally if the length is known it is added in the brackets, separated by
a semicolon. Eg: `[u8]`, `[i32; 16]`
- Array ranges (parts of an array) are specified in square brackets with the
included lower bound, two points (`..`) and the excluded upper bound. Eg: `[0..10]`


# 1. Low-Level Packets
## 1.1 Packet structure
- The packets are build in a fixed scheme,
though have differences depending in which direction.
- Every column here represents 1 byte.
- The entire packet size must be at max 500 bytes.

### 1.1.1 (Client -> Server)
    +--+--+--+--+--+--+--+--+--+--+--+--+--+---------//----------+
    |          MAC          | PId | CId |PT|        Data         |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+---------//----------+
    |                       \     Meta     |
    \                Header                /

| Name | Size        | Datatype | Explanation                     |
|------|-------------|----------|---------------------------------|
| MAC  | 8 bytes     | [u8]     | EAX Message Authentication Code |
| PId  | 2 bytes     | u16      | Packet Id                       |
| CId  | 2 bytes     | u16      | Client Id                       |
| PT   | 1 byte      | u8       | Packet Type + Flags             |
| Data | ≤487 bytes  | [u8]     | The packet payload              |

### 1.1.2 (Client <- Server)
    +--+--+--+--+--+--+--+--+--+--+--+------------//-------------+
    |          MAC          | PId |PT|           Data            |
    +--+--+--+--+--+--+--+--+--+--+--+------------//-------------+
    |                       \  Meta  |
    \             Header             /

| Name | Size        | Datatype | Explanation                     |
|------|-------------|----------|---------------------------------|
| MAC  | 8 bytes     | [u8]     | EAX Message Authentication Code |
| PId  | 2 bytes     | u16      | Packet Id                       |
| PT   | 1 byte      | u8       | Packet Type + Flags             |
| Data | ≤489 bytes  | [u8]     | The packet payload              |

## 1.2 Packet Types
- `0x00` Voice
- `0x01` VoiceWhisper
- `0x02` Command
- `0x03` CommandLow
- `0x04` Ping
- `0x05` Pong
- `0x06` Ack
- `0x07` AckLow
- `0x08` Init1

## 1.3 Packet Type + Flags byte
The final byte then looks like this

    MSB                   LSB
    +--+--+--+--+--+--+--+--+
    |UE|CP|NP|FR|   Type    |
    +--+--+--+--+--+--+--+--+

| Name | Size  | Hex  | Explanation     |
|------|-------|------|-----------------|
| UE   | 1 bit | 0x80 | Unencrypted     |
| CP   | 1 bit | 0x40 | Compressed      |
| NP   | 1 bit | 0x20 | Newprotocol     |
| FR   | 1 bit | 0x10 | Fragmented      |
| Type | 4 bit | 0-8  | The packet type |

## 1.4 Packet Compression
To reduce packet size, the data can be compressed.
When the data is compressed the `Compressed` flag must be set.
The algorithm "QuickLZ" with Level 1 is used for compression.

## 1.5 Packet Splitting
When the packet payload exceeds the maximum datablock size the data can be
split up across multiple packets.  
The current protocol only allows `Command` and `CommandLow` to be split and/or
compressed.  
When splitting occurs, the `Fragmented` flag must be set on the first and
the last packet.  
The `Unencrypted` and `Compressed` flag, if set in the original
packet, are only set on the first packet. (Though commands always have to be
encrypted)  
The `Newprotocol` flag has to be set on all commands and therefore also has to
be applied on all splitted command packets.  
The data can additionally be compressed *before* splitting.

*Example*:

The packet to split has the following flags:

    [__|CP|NP|__]  Packet Id: 42

it must be split into:

    [__|CP|NP|FR]  Packet Id: 42
    [__|__|NP|__]  Packet Id: 43
    [__|__|NP|FR]  Packet Id: 44


## 1.6 Packet Encryption
When a packet is not encrypted the `Unencrypted` flag is set. For encrypted
packets the flag gets cleared.
Packets get encrypted with EAX mode (AES_128_CTR with OMAC).
The en/decryption parameters are generated for each packet as follows

### 1.6.1 Inputs
| Name | Type     | Explanation                     |
|------|----------|---------------------------------|
| PT   | u8       | Packet Type                     |
| PId  | u16      | Packet Id                       |
| PGId | u32      | Packet GenerationId (see 1.9.2) |
| PD   | bool     | Packet Direction                |
| SIV  | [u8; 20] | Shared IV (see 3.2)             |

### 1.6.2 Generation pseudocode
The temporary variable will have a different length depending on the result
from the crypto init handshake. The old protocol SharedIV will be 20 bytes long,
since it is generated with sha1, while the new protocol SharedIV will have
64 bytes, since it is generated with sha512.

    let temporary: [u8; 26] OR [u8; 70]
    temporary[0]    = 0x30 if (Client <- Server)
                      0x31 if (Client -> Server)
    temporary[1]    = PT
    temporary[2..6] = (PGId in network order)[0..4]
    if SIV.length == 20
        temporary[6..26] = SIV[0..20]
    else
        temporary[6..70] = SIV[0..64]

    let keynonce: [u8; 32]
    keynonce        = sha256(temporary)

    key: [u8; 16]   = keynonce[ 0..16]
    nonce: [u8; 16] = keynonce[16..32]
    key[0]          = key[0] xor ((PId & 0xFF00) >> 8)
    key[1]          = key[1] xor ((PId & 0x00FF) >> 0)

### 1.6.3 Encryption
The data can now be encrypted with the `key` and `nonce` from (see 1.6.2) as the
EAX key and nonce and the packet `Meta` as defined in (see 1.1) as the EAX
header (sometimes called "Associated Text"). The resulting EAX mac (sometimes
called "Tag") will be stored in the `MAC` field as defined in (see 2.1).

### 1.6.4 Not encrypted packets
When a packet is not encrypted, no MAC can be generated by EAX. In this case
the SharedMac (see 3.2) will be used instead.

## 1.7 Packet Stack Wrap-up
This stack is a reference for the execution order of the set data operations.
For incoming packets the stack is executed bot to top, for outgoing packets
top to bot.

        Send                 Receive   
    +-----------+         +-----------+
    |   Data    | |     Λ |   Data    |
    +-----------+ |     | +-----------+
    | Compress  | |     | | Decompress|
    +-----------+ |     | +-----------+
    |   Split   | |     | |   Merge   |
    +-----------+ |     | +-----------+
    |  Encrypt  | V     | |  Decrypt  |
    +-----------+         +-----------+

## 1.8 Packet Types Data Structures
The following chapter describes the data structure for different packet types.

### 1.8.1.1 Voice (Client -> Server)
    +--+--+--+---------//---------+
    | VId |C |        Data        |
    +--+--+--+---------//---------+

| Name | Type | Explanation     |
|------|------|-----------------|
| VId  | u16  | Voice Packet Id |
| C    | u8   | Codec Type      |
| Data | var  | Voice Data      |

### 1.8.1.2 Voice (Client <- Server)
    +--+--+--+--+--+---------//---------+
    | VId | CId |C |        Data        |
    +--+--+--+--+--+---------//---------+

| Name | Type | Explanation     |
|------|------|-----------------|
| VId  | u16  | Voice Packet Id |
| CId  | u16  | Talking Client  |
| C    | u8   | Codec Type      |
| Data | var  | Voice Data      |

### 1.8.2.1 VoiceWhisper (Client -> Server)
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+---------//---------+
    | VId |C |N |M |           U*          |  T* |        Data        |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+---------//---------+

For direct user/channel targeting  
The `Newprotocol` Flag must be *unset*

| Name | Type  | Explanation                           |
|------|-------|---------------------------------------|
| VId  | u16   | Voice Packet Id                       |
| C    | u8    | Codec Type                            |
| N    | u8    | Count of ChannelIds to send to        |
| M    | u8    | Count of ClientIds to send to         |
| U    | [u64] | Targeted ChannelIds, repeated N times |
| T    | [u16] | Targeted ClientIds, repeated M times  |
| Data | var   | Voice Data                            |

OR

    +--+--+--+--+--+--+--+--+--+--+--+--+--+---------//---------+
    | VId |C |TY|TA|           U           |        Data        |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+---------//---------+

For targeting special groups  
The `Newprotocol` Flag must be *set*

| Name | Type  | Explanation                                             |
|------|-------|---------------------------------------------------------|
| VId  | u16   | Voice Packet Id                                         |
| C    | u8    | Codec Type                                              |
| TY   | u8    | GroupWhisperType (see below)                            |
| TA   | u8    | GroupWhisperTarget (see below)                          |
| U    | u64   | the targeted channelId or groupId (0 if not applicable) |
| Data | var   | Voice Data                                              |

```
enum GroupWhisperType : u8
{
	// Targets all users in the specified server group.
	ServerGroup      = 0 /* U = servergroup targetId */,
	// Targets all users in the specified channel group.
	ChannelGroup     = 1 /* U = channelgroup targetId */,
	// Targets all users with channel commander.
	ChannelCommander = 2, /* U = 0 (ignored) */,
	// Targets all users on the server.
	AllClients       = 3, /* U = 0 (ignored) */,
}

enum GroupWhisperTarget : u8
{
	AllChannels           = 0,
	CurrentChannel        = 1,
	ParentChannel         = 2,
	AllParentChannel      = 3,
	ChannelFamily         = 4,
	CompleteChannelFamily = 5,
	Subchannels           = 6,
}
```

### 1.8.2.2 VoiceWhisper (Client <- Server)
    +--+--+--+--+--+---------//---------+
    | VId | CId |C |        Data        |
    +--+--+--+--+--+---------//---------+

| Name | Type | Explanation     |
|------|------|-----------------|
| VId  | u16  | Voice Packet Id |
| CId  | u16  | Talking Client  |
| C    | u8   | Codec Type      |
| Data | var  | Voice Data      |

### 1.8.3-4 Command and CommandLow
The TeamSpeak3 Query like command string encoded in UTF-8

### 1.8.5 Ping
Empty.

### 1.8.6-8 Pong, Ack and AckLow
    +--+--+
    | PId |
    +--+--+

| Name | Type | Explanation                        |
|------|------|------------------------------------|
| PId  | u16  | The packet id that is acknowledged |

- In case of `Pong` a matching ping packet id is acknowledged.
- In case of `Ack` or `AckLow` a matching `Command` or `CommandLow` packet id
respectively is acknowledged.

### 1.8.9 Init1
(see 2.1)-(see 2.5)

## 1.9 Packet Ids and Generations
### 1.9.1 Packet Ids
Each packet type and packet direction must be maintained by an own packet id
counter.
This means the client has 9 different packet id counter for outgoing packets.

For each new packet the counter gets increased by 1. This also applies to
split packets.

The client must also maintain packet ids for incoming packets in case of
packets arriving out of order.

All Packet Ids start at 1 unless otherwise specified.

### 1.9.2 Generations
Packet Ids are stored as u16, this means they range from 0 up to 65535 included.

When the packet id overflows from 65535 to 0 at a packet,
the generation counter for this packet type gets increased by 1.

Note that the new generation id immediately applies to the 'overflowing' packet.

The generation id counter is solely used for encryption (see 1.6).

## 1.10 Packet Acknowledgement / Packet Loss
In order to reliably send packets over UDP some packet types must get
acknowledged when received (see 1.11).

The protocol uses selective repeat for lost packets. This means each packet has
its own timeout. Already acknowledged later packets must not be resent.
When a packet times out, the exact same packet should be resent until properly
acknowledged by the server.
If after 30 seconds no resent packet gets acknowledged the connection should be
closed.
Packet resend timeouts should be calculated with an exponential backoff to
prevent network congestion.

## 1.11 Wrap-up
| Type         | Acknowledged (by) | Resend | Encrypted | Splittable | Compressible |
|--------------|-------------------|--------|-----------|------------|--------------|
| Voice        | ✗                 | ✗      | Optional  | ✗          | ✗            |
| VoiceWhisper | ✗                 | ✗      | Optional  | ✗          | ✗            |
| Command      | ✓ (Ack)           | ✓      | ✓         | ✓          | ✓            |
| CommandLow   | ✓ (AckLow)        | ✓      | ✓         | ✓          | ✓            |
| Ping         | ✓ (Pong)          | ✗      | ✗         | ✗          | ✗            |
| Pong         | ✗                 | ✗      | ✗         | ✗          | ✗            |
| Ack          | ✗                 | ✓      | ✓         | ✗          | ✗            |
| AckLow       | ✗                 | ✓      | ✓         | ✗          | ✗            |
| Init1        | ✓ (next Init1)    | ✓      | ✗         | ✗          | ✗            |

# 2. The (Low-Level) Initiation/Handshake
A connection is started from the client by sending the first handshake
packet. The handshake process consists of 5 different init packets. This
includes the so called RSA puzzle to prevent DOS attacks.

The packet header values are set as following for all packets here:

| Parameter | Value                                                  |
|-----------|--------------------------------------------------------|
| MAC       | [u8]{ 0x54, 0x53, 0x33, 0x49, 0x4E, 0x49, 0x54, 0x31 } |
| key       | N/A                                                    |
| nonce     | N/A                                                    |
| Type      | Init1                                                  |
| Encrypted | ✗                                                      |
| Packet Id | u16: 101                                               |
| Client Id | u16: 0                                                 |

Init packets from the client contain a version field, which is the build
timestamp of the client. This is a unix timestamp subtracted with 1356998400.
The unix timestamp 1461588969 (date 2016-04-25) is encoded as
1461588969 - 1356998400 = 0x063bece9 = { 0x06, 0x3b, 0xec, 0xe9 }.

## 2.1 Packet 0 (Client -> Server)
    04 bytes : Version of the TeamSpeak client as timestamp
               Example: { 0x06, 0x3b, 0xec, 0xe9 }
    01 bytes : Init-packet step number
               Const: 0x00
    04 bytes : Current timestamp in unix format
    04 bytes : Random bytes := [A0]
    08 bytes : Zeros, reserved.

## 2.2 Packet 1 (Client <- Server)
    01 bytes : Init-packet step number
               Const: 0x01
    16 bytes : Server stuff := [A1]
    04 bytes : The bytes from [A0] in reversed order (not always) := [A0r]

This packets usually contains the bytes from [A0] in reversed order, except when
connecting from some networks for a yet unknown reason.

## 2.3 Packet 2 (Client -> Server)
    04 bytes : Version of the TeamSpeak client as timestamp
    01 bytes : Init-packet step number
               Const: 0x02
    16 bytes : The bytes from [A1]
    04 bytes : The bytes from [A0r]

## 2.4 Packet 3 (Client <- Server)
     01 bytes : Init-packet step number
                Const: 0x03
     64 bytes : 'x', an unsigned BigInteger
     64 bytes : 'n', an unsigned BigInteger
     04 bytes : 'level' a u32
    100 bytes : Server stuff := [A2]

Note:
- Sometimes, instead of sending an init with step 3, the server responds with
  an init that contains 127 as step number. In that case, the client has to
  restart the connection by sending packet 0 again.

## 2.5 Packet 4 (Client -> Server)
     04 bytes : Version of the TeamSpeak client as timestamp
     01 bytes : Init-packet step number
                Const: 0x04
     64 bytes : the received 'x'
     64 bytes : the received 'n'
     04 bytes : the received 'level'
    100 bytes : The bytes from [A2]
     64 bytes : 'y' which is the result of x ^ (2 ^ level) % n as an unsigned
                BigInteger. Padded from the lower side with '0x00' when shorter
                than 64 bytes.
                Example: { 0x00, 0x00, data ... data}
    var bytes : The clientinitiv command data as explained in (see 3.1)

Note:
- `^` in this context means 'power to'
- To calculate the power of such a high number use a language integrated
  function like `ModPow` or similar, when available.  
  If you don't have this function available you can multiply x iteratively
  and apply the modulo operation after each multiplication.

# 3. The (High-Level) Initiation/Handshake
In this phase the client and server exchange basic information and
agree on/calculate the symmetric AES encryption key with the ECDH
public/private key exchange technique.

Both the client and the server will need a EC public/private key. This key
is also the identity which the server uses to recognize a user again.
The curve used is 'prime256v1'.

All high level packets specified in this chapter are sent as `Command` Type
packets as explained in (see 2.8.3). Additionally the `Newprotocol` flag
(see 2.3) must be set on all `Command`, `CommandLow` and `Init1` packets.

All commands are specified in the `Messages.txt` file.

The packet header/encryption values for (see 3.1) and (see 3.2) are as following:

| Parameter | Value                                                                                                |
|-----------|------------------------------------------------------------------------------------------------------|
| MAC       | (Generated by EAX)                                                                                   |
| key       | [u8]{0x63, 0x3A, 0x5C, 0x77, 0x69, 0x6E, 0x64, 0x6F, 0x77, 0x73, 0x5C, 0x73, 0x79, 0x73, 0x74, 0x65} |
| nonce     | [u8]{0x6D, 0x5C, 0x66, 0x69, 0x72, 0x65, 0x77, 0x61, 0x6C, 0x6C, 0x33, 0x32, 0x2E, 0x63, 0x70, 0x6C} |
| Type      | Command                                                                                              |
| Encrypted | ✓                                                                                                    |
| Packet Id | u16: 0                                                                                               |
| Client Id | u16: 0                                                                                               |

The acknowledgement packets use the same parameters as the commands, except with
the Type `Ack`.

_(Maybe add a #3.0 Prelude for required cryptographic values, if yes move the
omega ASN.1 encoding here)_

## 3.1 clientinitiv (Client -> Server)
The first packet is sent (Client -> Server) although this is only sent for
legacy reasons since newer servers (at least 3.0.13.0?) use the data part
embedded in the last `Init1` packet from the low-level handshake (see 2.5).

    clientinitiv alpha={alpha} omega={omega} ot={ot} ip={ip}

- `alpha` is set to `base64(random[u8; 10])`  
  which are 10 random bytes for entropy.
- `omega` is set to `base64(publicKey[u8])`  
  omega is an ASN.1-DER encoded public key from the ECDH parameters as following:

  | Type       | Value          | Explanation                               |
  |------------|----------------|-------------------------------------------|
  | BIT STRING | 1bit, Value: 0 | LibTomCrypt uses 0 for a public key       |
  | INTEGER    | 32             | The LibTomCrypt used keysize              |
  | INTEGER    | publicKey.x    | The affine X-Coordinate of the public key |
  | INTEGER    | publicKey.y    | The affine Y-Coordinate of the public key |

- `ot` should always be `1`
- `ip` should be set to the final resolved ip address of the server you are
  actually connecting to.

## 3.2 initivexpand/initivexpand2 (Client <- Server)
Depending on the server version the server will send a different init request.
- TS3 server <3.1 will send `initivexpand`. Continue with (see 3.2.1)
- TS3 server ≥3.1 will send `initivexpand2`. Continue with (see 3.2.2)

If you want to support both protocol standards you don't need to check/know the
server version. The client just has to act accordingly depending on which
packet the server sends.

### 3.2.1 initivexpand (Client <- Server)
The server responds with this command.

    initivexpand alpha={alpha} beta={beta} omega={omega}

- `alpha` must have the same value as sent to the server in the previous step.
- `beta` is set to `base64(random[u8; 10])`  
  by the server.
- `omega` is set to `base64(publicKey[u8])`  
  with the public key from the server, encoded same as in (see 3.1)

With this information the client now must calculate the shared secret.

    let sharedSecret: ECPoint
    let x: [u8]
    let sharedData: [u8; 32]
    let SharedIV: [u8; 20]
    let SharedMac: [u8; 8]
    let ECDH(A, B)    := (A * B).Normalize

    sharedSecret         = ECDH(serverPublicKey, ownPrivateKey)
    x                    = sharedSecret.x.AsByteArray()
    if x.length < 32
        sharedData[ 0..(32-x.length)] = [0..0]
        sharedData[(32-x.length)..32] = x[0..x.length]
    elseif x.length == 32
        sharedData[0..32] = x[0..32]
    elseif x.length > 32
        sharedData[0..32] = x[(x.length-32)..x.length]
    SharedIV              = sha1(sharedData)
    SharedIV[0..10]       = SharedIV[0..10] xor alpha.decode64()
    SharedIV[10..20]      = SharedIV[10..20] xor beta.decode64()
    SharedMac[0..8]       = sha1(SharedIV)[0..8]

### 3.2.2 initivexpand2 (Client <- Server)
The server responds with this command.

    initivexpand2 l={l} beta={beta} omega={omega} ot={ot} proof={proof} tvd={tvd}

- `l` the server license (see 3.2.2.2)
- `beta` is set to `base64(random[u8; 54])`  
  by the server.
- `omega` is a `base64(publicKey[u8])`  
  with the public key from the server, encoded same as in (see 3.1)
- `ot` should always be `1`
- `proof` is a `base64(ecdh_sign(l))`  
- `tvd` (base64, unknown; only set on servers with a license)

#### 3.2.2.1 Verify integrity
This step **must** be done to verity the integrity of the connection.

The `proof` parameter is the sign of the `l` parameter (*not* base64 encoded).
The client can verify the `l` parameter with the public key of the server
which is sent in `omega`.

Both proofs which are exchanged (the one you received in (see 3.2.2), and the
one sent in (see 3.2.2.5)) use 'prime256v1 with sha256, DER encoded'.  
Note that the identity keys from client/server already should be keys
on 'prime256v1' as noted in (see 3).

#### 3.2.2.2 Parsing the license
The license has a small header continued by a list of blocks. Each block may
vary in length and must be parsed sequentially therefore.

The license header:

     01 bytes : License version
                Const: 0x01

The block base layout:

     01 bytes : Key type
                Const: 0x00 (public key)
     32 bytes : Block public key
     01 bytes : License block type
     04 bytes : Not valid before date
     04 bytes : Not valid after date
    var bytes : (Content from the block type)

There are currently 4 different `License block type`s used. 
- `00` Intermediate.  
  `content`:
    ```
     04 bytes : Unknown
    var bytes : A null terminated string, which describes the issuer of this certificate.
    ```
- `01` Website/`03` Code  
  `content`:
    ```
    var bytes : A null terminated string, which describes the issuer of this certificate.
    ```
- `02` TS3 Server  
  `content`:
    ```
     01 bytes : Server License Type
     04 bytes : Max clients allowed on the server
    var bytes : A null terminated string, which describes the issuer of this certificate.
    ```
- `08` TS5 Server  
  `content`:
    ```
     01 bytes : Server License Type
     01 bytes : Property count
    var bytes : Properties, see following description on how each property is encoded

    Per property:
     01 bytes : Length of following data
     01 bytes : Property Id
     01 bytes : Data Type
    var bytes : Content depending on data type, see below
    ```

    Data Types:
    - `00`: Null-terminated string
    - `01`/`03`: 4 byte data
    - `02`/`04`: 8 byte data

    Property Id:
    - `01` (type `01`): Unknown
    - `02` (type `00`): Issuer of the certificate
    - `03` (type `01`): Max clients allowed on the server, defaults to 32 if not present
- `32` Ephemeral  
  `content`: none

Both dates are stored in BigEndian, when read you must add `0x50e22700` to the
number and import as a unix timestamp.

Each `Not valid before` and `Not valid after` timespan must be within the range
of the parent license block.

The license must consist of `2 ≤ count ≤ 8` blocks.
The second to last block must be of type `Server`
and the last block must be of type `Ephemeral`.

#### 3.2.2.4 Calculating the shared secret
All elliptic curve operations for this step are done on the Curve25519.  
You might find more tools looking for Ed25519 but keep in mind that Ed25519
describes an EdDsa signing/verify operation and is not the curve itself.

To calculate the shared secret each license block now must be processed
sequentially the following way:

    next_key = public_key * clamp(sha512(block[1..])[0..32]) + parent

Where:
- `public_key` is the `Block public key` taken from the current license block.
  This array must be imported as a compressed Curve25519 EC point.
- `sha512(block[1..])[0..32]` is the sha512 of the current license block.  
  Note that the first byte (`Key type`) is skipped for the sha calculation.  
  For the result only the first 32 bytes are used.  
  This resulting array must be imported as a Curve25519 private key.
- `parent` which is the resulting `next_key` from the previous block.
  This is a compressed Curve25519 EC point.
- `clamp(num)` is a function describing `abs(num) mod B` where `B` is the base
  point of Curve25519. This function can usually be implemented conveniently
  on the number buffer like this:
  ```
  let buffer: [u8; 32]

  buffer[0]  &= 0xF8
  buffer[31] &= 0x3F
  buffer[31] |= 0x40
  ```

Since the first block has no predecessor.
A fixed 'root' key is used as `parent`.  
This key must be imported as a compressed Curve25519 EC point.

    [u8; 32] {0xcd, 0x0d, 0xe2, 0xae, 0xd4, 0x63, 0x45, 0x50, 0x9a, 0x7e, 0x3c,
              0xfd, 0x8f, 0x68, 0xb3, 0xdc, 0x75, 0x55, 0xb2, 0x9d, 0xcc, 0xec,
              0x73, 0xcd, 0x18, 0x75, 0x0f, 0x99, 0x38, 0x12, 0x40, 0x8a}

The last `next_key` is now used as the public key from the server
(see pseudocode below).

The client now has to create a temporary Curve25519 public/private keypair.
We will call them `client_public_key` and `client_private_key`.

Now the `SharedIV` and `SharedMac` which will be used in the encryption, just as
in the old protocol, can be calculated.

    let SharedIV: [u8; 64]
    let SharedMac: [u8; 8]
    let sharedData: [u8; 32]

    sharedData           = next_key * client_private_key
    SharedIV             = sha512(sharedData[0..32])
    SharedIV[ 0..10]     = SharedIV[ 0..10] xor alpha.decode64()
    SharedIV[10..64]     = SharedIV[10..64] xor  beta.decode64()
    SharedMac[0..8]      = sha1(SharedIV)[0..8]

#### 3.2.2.5 clientek (Client -> Server)

    clientek ek={ek} proof={proof}

- `ek` is `base64(client_public_key)`  
  which the ephemeral (temporary) key created in (see 3.2.2.4) by the client.
  This should obviously be the public key part only.
- `proof` is `base64(client_public_key + beta)`
  which is a sign of the client_public_key (the `ek`) concatenated with the
  `beta` parameter from the `initivexpand2` command.
  The sign must be done with the private key from the identity keypair.

The normal packet id counting starts with this packet. This means that
`clientek` already has the packet id `1` and the next command will continue
with `2`.

### 3.2.3 **Notes**:
- Only `SharedIV` and `SharedMac` are needed. The other values can (and should)
  be discarded.
- The crypto handshake is now completed. The normal encryption scheme (see 1.6)
  is from now on used.
- All `Command`, `CommandLow`, `Ack` and `AckLow` packets must get encrypted.
- `Voice` packets (and `VoiceWhisper` when wanted) should be encrypted when the
  channel encryption or server wide encryption flag is set.
- `Ping` and `Pong` must not be encrypted.

## 3.3 clientinit (Client -> Server)
    clientinit client_nickname client_version client_platform client_input_hardware client_output_hardware client_default_channel client_default_channel_password client_server_password client_meta_data client_version_sign client_key_offset client_nickname_phonetic client_default_token hwid

- `client_nickname` the desired nickname
- `client_version` the client version
- `client_platform` the client platform
- `client_input_hardware` whether an input device is available
- `client_output_hardware` whether an output device is available
- `client_default_channel` the default channel to join. This can be a channel
  path or `/<id>` (eg `/1`) for a channel id.
- `client_default_channel_password` the password for the join channel, prepared
  the following way `base64(sha1(password))`
- `client_server_password` the password to enter the server, prepared the
  following way `base64(sha1(password))`
- `client_meta_data` (can be left empty)
- `client_version_sign` a cryptographic sign to verify the genuinity of the
  client
- `client_key_offset` the number offset used to calculate the hashcash (see 4.1)
  value of the used identity 
- `client_nickname_phonetic` the phonetic nickname for text-to-speech
- `client_default_token` permission token to be used when connecting to a server
- `hwid` hardware identification string

**Notes**:
- Since client signs are only generated and distributed by TeamSpeak systems,
  this the recommended client triple, as it is the reference for this paper
  - Version: `3.0.19.3 [Build: 1466672534]`
  - Platform: `Windows`
  - Sign: `a1OYzvM18mrmfUQBUgxYBxYz2DUU6y5k3/mEL6FurzU0y97Bd1FL7+PRpcHyPkg4R+kKAFZ1nhyzbgkGphDWDg==`
- The `hwid` usually consists of two 32 char strings concatenated with `,` and
  looks like `87056c6e1268aaf5055abf8256415e0e,408978b6d98810cc03f0aa16a4c75600`
  but even empty strings are accepted.  
  On windows the hwid seems to be generated from a registry key
  (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProductId`).  
  On linux and macOS it seems to derive from the MAC address of the primary
  Ethernet/Wifi adapter.
- Parameters which are empty or not used must be declared but left without
  value and the `=` character

## 3.4 initserver (Client <- Server)
The server sends the `initserver` command.

Note:
- From this point on the client knows his client id, therefore it must be set
  in the header of each packet.
- Newer versions of the server send parts of the `clientinit` command in the `initserver` command.

## 3.5 Further notifications
The server will now send all needed information to display the entire
server properly. Those notifications are in no fixed order, although they
 are most of the time sent in the here declared order.

### 3.5.1 channellist and channellistfinished
The `channellist` notification type will be sent multiple times as needed to
transfer the entire server structure.

After the last `channellist` notification the server will send
`channellistfinished`

### 3.5.2 notifycliententerview
The `notifycliententerview` notification will be sent multiple times as needed
for each client currently connected.  
This is the same notification as when a new client connects.

# 4. Further concepts
## 4.1 Hashcash
To prevent client spamming (connecting to a server with many different clients)
the server requires a certain hashcash level on each identity. This level has a
exponentially growing calculation time with increasing level. This ensures that
a user wanting to spam a certain server needs to invest some time into
calculating the required level.

- The publicKey is a string encoded as in (see 3.1) the omega value.
- The key offset is a u64 number, which gets converted to a string when
concatenated.

The first step is to calculate a hash as following

    let data: [u8; 20] = sha1(publicKey + keyOffset)

The level can now be calculated by counting the continuous leading zero bits in
the data array.
The bytes in the array get counted from 0 to 20 and the bits in each byte
from least significant to most significant.

## 4.2 Uid
To calculate the uid of an identity the public key is required. Therefore you
can only calculate the uid of your own identity and the servers identity you
are connecting to.

The publicKey is a string encoded as in (see 3.1) the omega value.

The uid can be calculated as following

    let uid: string = base64(sha1(publicKey))

## 4.3 Ping/Pong
The server will regularly send ping packets to check if a client is still alive.
The client must answer them with the according pong packet.

The client should also send ping packets to the server to check for connection.
They will be answered with according pong packets.

Sending ping packets from the client side should not be started before the
crypto handshake has been completed (see 3.3)

## 4.4 Passwords
All passwords when sent are hashed and encoded with `base64(sha1(password))`

## 4.5 Importing/Deobfuscating Identities from the TeamSpeak3 Client
The TeamSpeak3 Client exports identities the following way:

    <key_offset> + 'V' + <obfuscaded_identity>

(For an explanation of the key offset (see 4.1))

```ts
const staticObfucationKey = b"b9dfaa7bee6ac57ac7b65f1094a1c155e747327bc2fe5d51c512023fe54a280201004e90ad1daaae1075d53b7d571c30e063b5a62a4a017bb394833aa0983e6e"
let ident = obfuscaded_identity.decode64()
let sha_part = ident[20..]
let idx = sha_part.indexof('\0') // where '\0' is the null-byte
let sha = sha1(sha_part[..idx])
ident[0..20] = ident[0..20] xor sha[0..20]
let xorlen = min(ident.length, 100)
ident[0..xorlen] = ident[0..xorlen] xor staticObfucationKey[0..xorlen]
```

## 4.6 Audio
When the Opus codec is used, Voice and VoiceWhisper packets are using a sampling
rate of 48 kHz. An voice packat without audio data signals the end of a stream.

## 4.7 Permissions
`permissionlist` requests the list of all permissions of the server.
`notifypermissionlist` returns the list of permissions and a grouping as a list
of `group_id_end`s. The end ids are excluding, so a `group_id_end=6` for the first
group means the first 6 permissions (`perms[0..6]`) are in this group.

## 4.8 Channel Subscription
Clients can subscribe channels, so they get notifications when someone enters or
leaves from a subscribed channel. If a new channel is subscribed, the server
usually sends a notification, except in some cases:
- If the client server groups or permissions change, it stays subscribed, even
  if it does not have the power anymore
- If the channel permissions change, the server sends a notification if the
  subscription status changed
- If a client enters a channel, it gets subscribed but there is no notification
- If a client leaves a channel, there is a notification that it unsubscribed
- If a new channel is created, clients are not automatically subscribed

## 4.? Differences between Query and Full Client
- notifyconnectioninforequest
- => setconnectioninfo
