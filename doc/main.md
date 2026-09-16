# Portable, Transferable, Standardised Codeplug (PTSCD) proposal

This is the *Part 2* codeplug structure.
*Part 1* can be derived directly from it by splitting chunks into separate files.

## Chunks
The default byte order is little-endian, unless specified otherwise.

#### Data integrity
The default CRC type used is the standard CRC-32 *\[we may specify the details if required\]*.
Data integrity of every chunk (except `MAGIC`, whose fixed content is validated by exact match)
is guaranteed by its own CRC.
The input to CRC is everything in a given chunk, except for the CRC field itself.
Any padding uses zero-valued bytes or bits.

#### Size
`ChunkSize` holds byte count of the whole chunk, including its 4-character designator.
This applies to tagged, optional chunks (`KEYS`, `DIAL`, `CONT`, and any future additions).
Any future optional chunk type must begin with a 4-character tag followed by a `ChunkSize` field,
so that parsers which do not recognize it can skip it safely.

### MAGIC chunk (12 bytes)
  ```
  [ "PTSCD 1.0 p2" ]
  ```

### META chunk (56 bytes)
  ```
  META_CHUNK = [
    CreatedAt (4B, unix epoch),
    Author (32B, null-padded),
    ToolVersion (16B, null-padded),
    ChunkCRC (4B),
  ]
  ```

### CDAT chunk (**Channel DATa**, 24 bytes per entry)
Chunk structure:
  ```
  CDAT_CHUNK = [
    NumChannels (2B),
    NumChannels * CHANNEL_DATA,
    Pad (2B),
    ChunkCRC (4B),
  ]
  ```

The following structure is repeated for every channel entry:
  ```
  CHANNEL_DATA = [
    FreqRX (4B),
    FreqTX (4B),
    RFPower (1B),
    Mode (1B),
    Flags (2B),
    NameRef (2B),
    PerModeData (9B),
    Pad (1B),
  ]
  ```

Available modes:
| `Mode`   | Meaning                   |
|----------|---------------------------|
| 0x00     | FM                        |
| 0x01     | AM                        |
| 0x02     | LSB                       |
| 0x03     | USB                       |
| 0x04     | CW                        |
| 0x05     | RTTY                      |
| 0x06     | APRS                      |
| 0x07     | YSF                       |
| 0x08     | DMR                       |
| 0x09     | P25                       |
| 0x0A     | NXDN                      |
| 0x0B     | TETRA                     |
| 0x0C     | POCSAG                    |
| 0x0D     | dPMR                      |
| 0x0E     | PMR446                    |
| 0x0F     | D-Star                    |
| 0x10     | LoRa                      |
| 0x11     | M17                       |
| 0x12     | FreeDV                    |
| *other*  | *reserved*                |


Per-mode data (always 9 bytes, unused space is zero-padded):
| Mode         | Structure (field sizes in bytes)                  |
|--------------|---------------------------------------------------|
| CW           | BW (1)                                            |
| LSB/USB      | BW (1)                                            |
| FM           | CTCSS_RX (1), CTCSS_TX (1), BW (1)                |
| TETRA        | ISSI (3), GSSI (3), CallType (1)                  |
| M17          | CAN (1), DST base40 (6), EncrKey (1), SignKey (1) |
| DMR          | CC/TS (1), TG (3), EncrKey (1)                    |
| *other*      | *TBF*                                             |

Note:
  - The `CC/TS` field structure is `(CC<<1) | TS`.
  - `NameRef` points directly to a `NAME` chunk entry.
  - CTCSS frequencies are referenced by a look-up table ID. A value of 0 means that a particular CTCSS is unused.
  - `BW` byte defines the channel width: 0: 100Hz, 1: 2.7kHz, 2:6.25kHz, 3: 12.5kHz, 4: 25kHz, 5: 180kHz, and possibly other values for CW/SSB/LoRa/WBFM.
  - `CallType` is TETRA-specific.

Flags:
| Bit     | Meaning                                      |
|---------|----------------------------------------------|
| 0       | active                                       |
| 1       | skip scan                                    |
| 2       | inhibit TX                                   |
| 3       | bookmark/star/favorite                       |
| 4       | encrypted                                    |
| 5       | signed                                       |
| 6       | direct/repeater mode                         |
| *other* | *reserved*                                   |

Notes:
  - Frequencies are stored as `uint32_t`, unit: Hz.
  - Power is stored as `uint8_t` with 0.25dBm steps (`dec = enc \* 0.25dBm`). This covers 1mW up to over 2kW.
  - `EncrKey` and `SignKey` point to `KEYS` chunk entries for both encryption and authentication (private keys for signatures). Both fields can be left unused if the correcsponding flags are not set.
  - `NameRef` points to a `NAME` chunk entry.

### NAME  chunk (32 bytes per entry)
Chunk structure:
  ```
  NAME_CHUNK = [
    NumNames (2B),
    NumNames * NAME_DATA,
    Pad (2B),
    ChunkCRC (4B),
  ]
  ```

The following structure is repeated for every name entry:
  ```
  NAME_DATA = [
    Name (32B, zero-padded),
  ]
  ```

### KEYS chunk (36 bytes per entry)
`KEYS` chunk is present only if required. Maximum key length is 256 bits.

Chunk structure:
  ```
  KEYS_CHUNK = [
    "KEYS",
    ChunkSize (2B),
    NumKeys (1B),
    NumKeys * KEY_DATA,
    Pad (1B),
    ChunkCRC (4B),
  ]
  ```

The following structure is repeated for every key entry:
  ```
  KEY_DATA = [
    Algorithm (1B),
    KeyLen (1B),
    KeyData (32B, zero-padded),
    Pad (2B),
  ]
  ```

Encryption/authentication algorithms:
| Algorithm ID | Name                            |
|--------------|---------------------------------|
| 0x00         | Scrambler                       |
| 0x01         | AES                             |
| 0x02         | RC4                             |
| 0x03         | TEA-1                           |
| 0x04         | TEA-2                           |
| 0x05         | TEA-3                           |
| 0x06         | TEA-4                           |
| 0x07         | ECDSA (secp256r1 curve)         |
| *other*      | *reserved*                      |

Notes:
  - `KeyLength` is the key size in bits.

### DIAL chunk (8 bytes per entry)
This is a quick-dial list. The user selects an entry that configures the radio.
The `DIAL` chunk is present only if required/used.

Chunk structure:
  ```
  DIAL_CHUNK = [
    "DIAL",
    ChunkSize (4B),
    NumDial (2B),
    NumDial * DIAL_DATA,
    Pad (2B),
    ChunkCRC (4B),
  ]
  ```

The following structure is repeated for every contact entry:
  ```
  DIAL_DATA = [
    NAMEChunkLoc (2B),
    CDATAChunkLoc (2B),
    KEYSChunkLoc (1B),
    Pad (3B),
  ]
  ```

### CONT chunk (12 bytes per entry)
This is the contact list - a look-up table for mode-dependent indentifiers (IDs).
The `CONT` chunk is present only if required/used. The contents should be sorted by (`Kind`, `ID`).
That allows the look-up table to be searched using binary search.

Chunk structure:
  ```
  CONT_CHUNK = [
    "CONT",
    ChunkSize (4B),
    NumCont (2B),
    NumCont * CONT_DATA,
    Pad (2B),
    ChunkCRC (4B),
  ]
  ```

The following structure is repeated for every contact entry:
  ```
  CONT_DATA = [
    Kind (1B),
    ID (6B),
    NAMEChunkLoc (2B),
    Pad (3B),
  ]
  ```

Valid `Kind` fields are:
| `Kind`  | Meaning                   |
|---------|---------------------------|
| 0x00    | DMR talkgroup ID          |
| 0x01    | DMR private call ID       |
| 0x02    | M17 SRC ID                |
| 0x03    | M17 reflector ID          |
| 0x04    | TETRA ISSI                |
| 0x05    | TETRA GSSI                |
| *other* | *reserved*                |
