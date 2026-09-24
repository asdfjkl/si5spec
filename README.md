# Scid 5 File Format Specification Version

Scid is a popular free chess database program. It stores games in it's
own binary format. Whereas Scid is open source, there is no single clear
file format specification, and one has to resort to deep study of it's
C++ source code to understand the format. This document attempts to
provide a technical specification of the format in order to facilitate
the implementation of the format by programs other than direct forks of
the SCID source base. It is not endorsed by the authors of SCID, and 
has no relation to the SCID project. Consider it informal notes.

In Scid 5, a chess database is stored across three companion files:

• .si5: Index file (fixed 56-byte record per game)                                               
• .sn5: Namebase file (names of players, events, sites, rounds)                                  
• .sg5: Game data file (moves, variations, comments, extra tags)

Unlike older formats (such as .si4), the .si5 file has no header. .sx5 database files are
identified by the file extension. Every game in the database is  
represented by exactly 56 bytes (INDEX_ENTRY_SIZE = 56), stored sequentially:

    Record File Offset = game\_index × 56                                                          
                                                                                                   
                  file\_size                                                                       
    Total Games = ──────────                                                                       
                      56        


Strings: In general, all strings that appear are UTF-8, sometimes null-terminated. (FEN-Strings are by definition ASCII only).


## Overview of the .si5 File Format

---

### Record Layout: Offset, Length, and Content

Each 56-byte record consists of 12 32-bit little-endian integers (48 bytes) followed by 8 bytes of Home Pawn signature data:

| Byte Offset | Byte Length | Bit Range | Field Name | Type / Encoding | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 – 3 | 4 bytes | bits 28..31 | Comment Rating | 4-bit unsigned | Comment count rating code (0..15 mapped non-linearly to ~0..50+ comments) |
| 0 – 3 | 4 bytes | bits 0..27 | White Player ID | 28-bit unsigned | Reference to player string in .sn5 (up to 268M unique players) |
| 4 – 7 | 4 bytes | bits 28..31 | Variation Rating | 4-bit unsigned | Alternative move sequence (RAV) count rating code (0..15) |
| 4 – 7 | 4 bytes | bits 0..27 | Black Player ID | 28-bit unsigned | Reference to player string in .sn5 |
| 8 – 11 | 4 bytes | bits 28..31 | NAG Rating | 4-bit unsigned | Numeric count rating code (0..15) |
| 8 – 11 | 4 bytes | bits 0..27 | Event ID | 28-bit unsigned | Reference to event string in .sn5 |
| 12 – 15 | 4 bytes | bits 0..31 | Site ID | 32-bit unsigned | Reference to site string in .sn5 |
| 16 – 19 | 4 bytes | bit 31 | Variant Flag | 1 bit | 0 = Standard Chess, 1 = Chess960 (Fischer Random) |
| 16 – 19 | 4 bytes | bits 0..30 | Round ID | 31-bit unsigned | Reference to round string in .sn5 |
| 20 – 23 | 4 bytes | bits 20..31 | White Elo | 12-bit unsigned | White rating value (0 ≤ Elo ≤ 4000) |
| 20 – 23 | 4 bytes | bits 0..19 | Game Date | 20-bit packed | Bits 0..4: Day (0..31), Bits 5..8: Month (0..12), Bits 9..19: Year (0..2047) |
| 24 – 27 | 4 bytes | bits 20..31 | Black Elo | 12-bit unsigned | Black rating value (0 ≤ Elo ≤ 4000) |
| 24 – 27 | 4 bytes | bits 0..19 | Event Date | 20-bit packed | Same format as Game Date |
| 28 – 31 | 4 bytes | bits 22..31 | Half Moves | 10-bit unsigned | Ply count (0 ≤ ply ≤ 1023) |
| 28 – 31 | 4 bytes | bits 0..21 | Flags | 22-bit bitmask | 22 game flags (e.g. custom start pos, delete, tactics, user flags) |
| 32 – 35 | 4 bytes | bits 15..31 | Data Size | 17-bit unsigned | Game data length in .sg5 (up to 128 KB) |
| 32 – 35 | 4 bytes | bits 0..14 | Offset High | 15-bit unsigned | Bits 32..46 of the game's byte offset in .sg5 |
| 36 – 39 | 4 bytes | bits 0..31 | Offset Low | 32-bit unsigned | Bits 0..31 of the game's byte offset in .sg5 (Combined 47-bit offset ≤ 128 TB) |
| 40 – 43 | 4 bytes | bits 24..31 | Stored Line | 8-bit unsigned | Opening line classification code for quick lookup |
| 40 – 43 | 4 bytes | bits 0..23 | Final MatSig | 24-bit bitfield | Final position material signature (finalMatSig) |
| 44 – 47 | 4 bytes | bits 24..31 | Home Pawn Count | 8-bit unsigned | Number of pawns that moved from initial squares (0..16) |
| 44 – 47 | 4 bytes | bits 21..23 | White Rating Type | 3-bit unsigned | 0: Elo, 1: Rating, 2: Rapid, 3: ICCF, 4: USCF, 5: DWZ, 6: ECF |
| 44 – 47 | 4 bytes | bits 18..20 | Black Rating Type | 3-bit unsigned | Same 3-bit enum as White Rating Type |
| 44 – 47 | 4 bytes | bits 16..17 | Result | 2-bit unsigned | 0: * (unknown), 1: 1-0, 2: 0-1, 3: 1/2-1/2 |
| 44 – 47 | 4 bytes | bits 0..15 | ECO Code | 16-bit unsigned | Encoded ECO classification (A00–E99 + subcodes) |
| 48 – 55 | 8 bytes | 64 bits | Home Pawn Data | 8 raw bytes | Up to 16 4-bit nibbles encoding which pawns left their starting ranks |

#### Detailed Field Specifications

- **Ratings Approximation Table**: Values 0..15 map to `[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20, 30, 40, 50]`.
- **Date Encoding (`date.h:40-76`)**:
  ```text
  date = (Year ≪ 9) | (Month ≪ 5) | Day
  ```
  A value of 0 for year, month, or day denotes an unknown value (`????.??.??`).
- **Flags Bitmask (`indexentry.h:258-285`)**:
  0: StartPos, 1: Promo, 2: UnderPromo, 3: Delete, 4: WhiteOpening, 5: BlackOpening, 6: Middlegame, 7: Endgame, 8: Novelty, 9: Pawn, 10: Tactics, 11: Kingside, 12: Queenside, 13: Brilliancy, 14: Blunder, 15: User, 16-21: Custom1..6.
- **Final Material Signature (`matsig.h:40-64`)**:
  Bits 0..3: BP (0-8), 4..5: BN (0-3), 6..7: BB (0-3), 8..9: BR (0-3), 10..11: BQ (0-3), 12..15: WP (0-8), 16..17: WN (0-3), 18..19: WB (0-3), 20..21: WR (0-3), 22..23: WQ (0-3).
- **ECO Code Decoding (`misc.cpp:78-103`)**:
  `code == 0` is empty. Otherwise, `c = code - 1`. Base code is `c // 131` (letter = `'A' + base // 100`, digits `(base % 100) // 10` and `base % 10`). Remainder `c % 131` encodes subcodes `a..z` and numbers `1..4`.

### What is HomePawnData?

In chess, pawns can only move forward and can never move backward or respawn on their starting square. Once a pawn leaves rank 2 (for White) or rank 7 (for Black)—either by moving or being captured on its starting square—it can never return to that square for the remainder of the game.

Scid leverages this irreversible property to implement fast search filters (such as Exact Position Search, Pawn Structure Search, and Duplicate Game Detection). By inspecting just the 56-byte index record (.si5) in memory, Scid can instantly discard games that could not possibly contain a target position—without decompressing or parsing the moves from the .sg5 game file.

---

### 1. The 16 Home Pawn Squares & Nibble Mapping

Each of the 16 home pawn starting squares is assigned a unique 4-bit index (0 … 15, or 0x0 … 0xF), corresponding to its bit position in the 16-bit pawn bitmask (HPSig, defined in `position.cpp:541-560` and `matsig.h:263-270`):

| Side | Square | Bit Mask | Nibble Value (Hex / Dec) |
| :--- | :---: | :---: | :---: |
| Black | h7 | 0x0001 (bit 0) | 0x0 (0) |
| Black | g7 | 0x0002 (bit 1) | 0x1 (1) |
| Black | f7 | 0x0004 (bit 2) | 0x2 (2) |
| Black | e7 | 0x0008 (bit 3) | 0x3 (3) |
| Black | d7 | 0x0010 (bit 4) | 0x4 (4) |
| Black | c7 | 0x0020 (bit 5) | 0x5 (5) |
| Black | b7 | 0x0040 (bit 6) | 0x6 (6) |
| Black | a7 | 0x0080 (bit 7) | 0x7 (7) |
| White | h2 | 0x0100 (bit 8) | 0x8 (8) |
| White | g2 | 0x0200 (bit 9) | 0x9 (9) |
| White | f2 | 0x0400 (bit 10) | 0xA (10) |
| White | e2 | 0x0800 (bit 11) | 0xB (11) |
| White | d2 | 0x1000 (bit 12) | 0xC (12) |
| White | c2 | 0x2000 (bit 13) | 0xD (13) |
| White | b2 | 0x4000 (bit 14) | 0xE (14) |
| White | a2 | 0x8000 (bit 15) | 0xF (15) |

---

### 2. How HomePawnData is Constructed

During game encoding in `game.cpp:2822-2852` (`mainlineInfo()`):

1. **Initial state**: All 16 pawns are home. The initial bitmask is `HPSIG_StdStart = 0xFFFF`.
2. **Move iteration**: Whenever a move causes a pawn on rank 2 or 7 to leave its home square (either by advancing or by being captured on that square):
   - Exactly one bit drops from 1 to 0 in `pos.GetHPSig()`.
   - Scid identifies the index of the moved pawn using the bit position:
     ```text
     idxMovedPawn = ctz(hpOld - hpNew)
     ```
   - This 4-bit nibble is appended to an ordered sequence called the change list.
3. **Bit packing**:
   - Two 4-bit nibbles are packed into each byte (the earlier move goes into the high nibble, and the later move goes into the low nibble):
     ```text
     Byte k: [ Nibble 2k (high 4 bits) | Nibble 2k+1 (low 4 bits) ]
     ```
   - Since there are at most 16 pawns, at most 16 departures can ever occur:
     ```text
     16 departures × 4 bits = 64 bits = 8 bytes
     ```

---

### 3. Physical Storage in .si5

The home pawn data is stored across two places in the 56-byte record:

1. **`home_pawn_count` (Offset 44, bits 24..31)**:
   An 8-bit unsigned integer (0 … 16) specifying how many pawns left their home squares.
2. **`HomePawnData` bytes (Offset 48..55, 8 bytes)**:
   The packed nibbles representing the exact chronological sequence of pawn departures. Any unused nibbles past `home_pawn_count` are padded with `0x0`.

---

### 4. Step-by-Step Concrete Example

Consider Game 1 from the database (`read_si5.py`):

- `home_pawn_count` = 11
- Raw 8 bytes: `e3 b4 5c 8d 6a 20 00 00`

Unpacking the first 11 nibbles:

```text
Byte 0: 0xe3 -> High nibble: 0xE (14) = b2 | Low nibble: 0x3 (3) = e7
Byte 1: 0xb4 -> High nibble: 0xB (11) = e2 | Low nibble: 0x4 (4) = d7
Byte 2: 0x5c -> High nibble: 0x5 (5)  = c7 | Low nibble: 0xC (12) = d2
Byte 3: 0x8d -> High nibble: 0x8 (8)  = h2 | Low nibble: 0xD (13) = c2
Byte 4: 0x6a -> High nibble: 0x6 (6)  = b7 | Low nibble: 0xA (10) = f2
Byte 5: 0x20 -> High nibble: 0x2 (2)  = f7 | Low nibble: 0x0 (padding)
Bytes 6-7: 00 00 (padding)
```

Correlating with the actual PGN moves:

1. 1. b4 → pawn leaves b2 (0xE)
2. 1... e5 → pawn leaves e7 (0x3)
3. 4. e3 → pawn leaves e2 (0xB)
4. 6... d5 → pawn leaves d7 (0x4)
5. 7... c5 → pawn leaves c7 (0x5)
6. 9. d3 → pawn leaves d2 (0xC)
7. 10. h3 → pawn leaves h2 (0x8)
8. 12. c4 → pawn leaves c2 (0xD)
9. 14... b6 → pawn leaves b7 (0x6)
10. 20. f4 → pawn leaves f2 (0xA)
11. 28... f5 → pawn leaves f7 (0x2)

Decoded sequence:

```text
𝐛𝟐 → 𝐞𝟕 → 𝐞𝟐 → 𝐝𝟕 → 𝐜𝟕 → 𝐝𝟐 → 𝐡𝟐 → 𝐜𝟐 → 𝐛𝟕 → 𝐟𝟐 → 𝐟𝟕
```

---

### 5. How Scid Uses HomePawnData

The core search functions are implemented in `matsig.cpp:82-196`:

#### A. Position Search Optimization (matsig.cpp:97-129)

When searching for a position (e.g. after 1. d4 d5 2. c4), the target position has an `hpSig` mask where c2, d2, and d7 are missing, but other pawns are expected on their home squares.
As Scid steps through the game's `changeList`, it checks:

```cpp
if ((hpCurrent & hpSig) != hpSig) {
    return false; // A pawn required on its home square has already departed!
}
```

If a pawn that the search query requires on its starting square has already moved, the game can never match at any future ply. Scid immediately prunes the entire game in nanoseconds.

#### B. Duplicate and Truncated Game Detection (matsig.cpp:140-169)

Checks if one game's pawn change list is an exact prefix of another game's list. If they diverge, the two games cannot be identical or truncated versions of each other.

#### C. Final State Reconstruction (matsig.cpp:177-196)

Subtracts each shifted bit (1 ≪ change) from 0xFFFF to reconstitute the exact 16-bit home pawn structure at the end of the game.



## Overview of the .sn5 File Format

The .sn5 (Namebase) file stores all unique strings referenced by the games in the database—specifically Players, Events, Sites, and Rounds, as well as general Database Info tags.

Key structural principles:

- **No File Header**: The file starts immediately at byte offset 0 with the first record.
- **Variable-Length Records**: Records follow sequentially one after another with no padding or delimiters.
- **Append-Only**: When games are edited or imported, new unique names are appended to the end of .sn5.
- **Sequential Independent IDs**: IDs start from 0 and are incremented independently per name type (i.e. the first player in the file gets Player ID 0, the first event gets Event ID 0, etc.).

---

### Record Table

| Offset | Field Name | Length | Format | Content Description |
| :--- | :--- | :--- | :--- | :--- |
| +0 | Varint Header | 1..5 B | LEB128 | `(Length << 3)` \| `Type` |
| +V | String Payload | L B | Raw text | Characters (not null-term.) |

- Where `V` is the byte length of the varint header (usually 1 or 2 bytes).
- Where `L` is the length of the string payload in bytes (0 ≤ L ≤ 255).
- Total record size in bytes = `V + L`.

---

### 1. Name Types Table (Bits 0..2 of Varint)

The lowest 3 bits of the decoded varint identify the category of the string (see `namebase.h:34-41`):

| Type Code | Constant Name | Referenced in .si5 … | ID Counter | Content / Format |
| :---: | :--- | :--- | :--- | :--- |
| 0 | `NAME_PLAYER` | White Player ID (Word 0)<br>Black Player ID (Word 1) | Player ID (0, 1, 2...) | Player name (e.g. "Ivanchuk, Vassily", "Giri, Anish") |
| 1 | `NAME_EVENT` | Event ID (Word 2) | Event ID (0, 1, 2...) | Tournament/event name (e.g. "26. Leon Masters g20") |
| 2 | `NAME_SITE` | Site ID (Word 3) | Site ID (0, 1, 2...) | City & country code (e.g. "Leon ESP", "Havana CUB") |
| 3 | `NAME_ROUND` | Round ID (Word 4) | Round ID (0, 1, 2...) | Round identifier (e.g. "1", "9.2", "Final") |
| 4 | `NAME_INFO` | (Database-level metadata) | (None) | Key-value database property (type, description, autoload, flag1..flag6) |

---

### 2. The Varint Header (LEB128)

The integer `V = (length × 8) + type` is encoded in variable-length 7-bit chunks (little-endian order):

- **Bit 7 (MSB, 0x80)**: Continuation bit (1 = more bytes follow; 0 = last byte of varint).
- **Bits 0..6**: 7 data bits of the integer.

#### Byte Length Thresholds:

- **1 byte**: For `V < 128` ⟹ String Length < 16 characters.
- **2 bytes**: For `128 ≤ V < 16384` ⟹ `16 ≤ String Length ≤ 2047` characters (covers almost all standard chess names).

---

### 3. Concrete Walkthrough from Real Database Data

Here is the actual hex dump from the start of an .sn5 file:

```text
Offset    Hex Dump                                           ASCII
0x0000:   a1 01 32 36 2e 20 4c 65 6f 6e 20 4d 61 73 74 65   ..26. Leon Maste
0x0010:   72 73 20 67 32 30 42 4c 65 6f 6e 20 45 53 50 0b   rs g20BLeon ESP.
0x0020:   34 88 01 49 76 61 6e 63 68 75 6b 2c 20 56 61 73   4..Ivanchuk, Vas
0x0030:   73 69 6c 79 58 47 69 72 69 2c 20 41 6e 69 73 68   silyXGiri, Anish
```

Let's dissect each record step-by-step:

#### Record 0: File Offset 0x0000 (Event ID 0)

- **Varint bytes**: `a1 01` (2 bytes)
   - Byte 0: `0xA1` (`10100001` → data: `0x21 = 33`, continuation = 1)
   - Byte 1: `0x01` (`00000001` → data: `1 << 7 = 128`, continuation = 0)
   - Combined value `V = 33 + 128 = 161`
   - Type: `161 & 7 = 1` (`NAME_EVENT`)
   - Length: `161 ≫ 3 = 20` bytes
- **String payload** (20 bytes, offset `0x0002` to `0x0015`):
  `"26. Leon Masters g20"`
- **Assigned ID**: Event ID 0

#### Record 1: File Offset 0x0016 (Site ID 0)

- **Varint byte**: `42` (1 byte)
   - Byte 0: `0x42` (`01000010` → continuation = 0)
   - Combined value `V = 66`
   - Type: `66 & 7 = 2` (`NAME_SITE`)
   - Length: `66 ≫ 3 = 8` bytes
- **String payload** (8 bytes, offset `0x0017` to `0x001E`):
  `"Leon ESP"`
- **Assigned ID**: Site ID 0

#### Record 2: File Offset 0x001F (Round ID 0)

- **Varint byte**: `0b` (1 byte)
   - Byte 0: `0x0B` (`00001011` → continuation = 0)
   - Combined value `V = 11`
   - Type: `11 & 7 = 3` (`NAME_ROUND`)
   - Length: `11 ≫ 3 = 1` byte
- **String payload** (1 byte, offset `0x0020`):
  `"4"`
- **Assigned ID**: Round ID 0

#### Record 3: File Offset 0x0021 (Player ID 0)

- **Varint bytes**: `88 01` (2 bytes)
   - Byte 0: `0x88` (`10001000` → data: `0x08 = 8`, continuation = 1)
   - Byte 1: `0x01` (`00000001` → data: `1 << 7 = 128`, continuation = 0)
   - Combined value `V = 8 + 128 = 136`
   - Type: `136 & 7 = 0` (`NAME_PLAYER`)
   - Length: `136 ≫ 3 = 17` bytes
- **String payload** (17 bytes, offset `0x0023` to `0x0033`):
  `"Ivanchuk, Vassily"`
- **Assigned ID**: Player ID 0

#### Record 4: File Offset 0x0034 (Player ID 1)

- **Varint byte**: `58` (1 byte)
   - Byte 0: `0x58` (`01011000` → continuation = 0)
   - Combined value `V = 88`
   - Type: `88 & 7 = 0` (`NAME_PLAYER`)
   - Length: `88 ≫ 3 = 11` bytes
- **String payload** (11 bytes, offset `0x0035` to `0x003F`):
  `"Giri, Anish"`
- **Assigned ID**: Player ID 1

---

### 4. Special Record: NAME_INFO (type = 4)

When Scid stores database metadata (description, autoload options, or custom flag names), it appends a record of `type = 4`. The payload starts with the property key, followed immediately by its value (see `codec_scid5.h:579-593`):

- `"descriptionMy Favorite World Championship Games"`
- `"typeStandard"`
- `"flag1Tactical"`

These records do not receive a game ID; they are parsed on database load to populate database settings.


## Overview of the .sg5 File Format (.sg5 game file)

### High-Level Record Architecture

Each game in .sg5 is stored as a variable-length binary blob (up to 128 KB).
Its starting byte offset and length are stored in the game's 56-byte .si5 index record.

| Section | Content Summary |
| :--- | :--- |
| **1. Extra Tags** | Non-standard PGN tags (omits 7 standard tags)<br>Ends with `0x00` delimiter byte |
| **2. Start Board** | 1-byte flags (custom start, promo, underpromo)<br>Optional null-terminated FEN string |
| **3. Move Stream** | Move opcodes, RAV variations, NAGs, comment marks<br>Ends with `0x0F` (`ENCODE_END_GAME`) delimiter |
| **4. Comments** | Null-terminated text strings in PGN order |

---

### Record Layout Table

| Section | Relative Offset | Length | Format / Type | Description |
| :--- | :--- | :--- | :--- | :--- |
| 1. Extra Tags | +0 | Variable | Tag-Value pairs | Non-STR tags; ends with `0x00` |
| 2. Start Board | After Tags | 1 B (+ FEN) | Bitflags (+ str) | Start pos & promotion flags |
| 3. Move Stream | After Board | Variable | Byte opcodes | Moves & marks; ends with `0x0F` |
| 4. Comments | After 0x0F | Variable | Text + \0 | Null-terminated comment strings |

---

### Section 1: Extra Tags

Standard tags (Event, Site, Date, Round, White, Black, Result) are omitted because they are already stored in .si5 and .sn5. Only supplementary tags (e.g. Annotator, Opening, PlyCount) are placed here.

Each tag is stored as:

1. **Tag Name**:
   - Common Tag: Encoded as a single byte (241 … 250).
   - Custom Tag: 1-byte length (1 … 240) followed by ASCII name.
2. **Tag Value**:
   - 1-byte length (0 … 255) followed by ASCII value.
3. **Delimiter**:
   - The section ends with a single `0x00` byte.
   - (If a game has no extra tags, this section is simply `0x00`).

#### Common Tag 1-Byte Codes (Values 241..250)

| Byte Code | Tag Name | Byte Code | Tag Name |
| :---: | :--- | :---: | :--- |
| `0xF1` (241) | WhiteCountry | `0xF6` (246) | Opening |
| `0xF2` (242) | BlackCountry | `0xF7` (247) | Variation |
| `0xF3` (243) | Annotator | `0xF8` (248) | Setup |
| `0xF4` (244) | PlyCount | `0xF9` (249) | Source |
| `0xF5` (245) | EventDate | `0xFA` (250) | SetUp |

---

### Section 2: Start Board

Immediately follows the `0x00` tag delimiter.

1. **Flag Byte (1 byte)**:
   - Bit 0 (`0x01`): Has custom starting position (FEN).
   - Bit 1 (`0x02`): Game contains pawn-to-queen promotion(s).
   - Bit 2 (`0x04`): Game contains underpromotion(s) (to R, B, or N).
2. **Optional FEN String**:
   - If Bit 0 == 1: Followed by the FEN string, terminated by `\0`.
   - If Bit 0 == 0: No FEN string follows (section is just 1 byte).

---

### Section 3: Move Stream (High-Level Container)

Immediately follows the Start Board section.

- Contains move opcodes, NAG annotations, and sub-variation markers.
- **Comment Marker (`0x0C` / 12)**: Placed in the move stream at the exact move where a comment belongs in the PGN.
- **Delimiter (`0x0F` / 15)**: The move section strictly terminates with the opcode `ENCODE_END_GAME` (`0x0F`).

---

### Section 4: Comments

Immediately follows the `0x0F` end-of-game marker.

- Each comment is stored as a null-terminated string (`text + '\0'`).
- Stored in the exact order the comments appear in the game.
- If a game has no comments, this section is 0 bytes (the blob ends right after `0x0F`).

---

### Concrete Byte Walkthrough (Game 5 from Database)

Here is a real blob from `res_database5.sg5` showing all 4 sections:

| Offset | Hex Bytes | Content & Meaning |
| :--- | :--- | :--- |
| **Section 1: Extra Tags** | | |
| `0x0000` | `f6 11 51 75 ... 65` | Tag 246 (Opening), Len 17:<br>&nbsp;&nbsp;`"Queen's pawn game"` |
| `0x0013` | `f3 0d 4d 61 ... 77` | Tag 243 (Annotator), Len 13:<br>&nbsp;&nbsp;`"Mark Crowther"` |
| `0x0022` | `00` | Tags Delimiter (End of Tags) |
| **Section 2: Start Board** | | |
| `0x0023` | `00` | Flags: 0 (standard start, no FEN) |
| **Section 3: Move Stream** | | |
| `0x0024` | `93 ... 0c ... 0f` | Move opcodes, comment markers<br>Ends with `0x0F` (End of Game) |
| **Section 4: Comments** | | |
| `0x0096` | `54 68 69 73 ... 2e 00` | Comment 1: `"...agreed drawn.\0"` |
| `0x00C2` | `22 49 74 27 ... 2e 00` | Comment 2: `"\"It's very...\0"` |

---

### 1. The Core 1-Byte Move Structure

Every move byte is divided into two 4-bit nibbles:

```text
       Bit 7   Bit 6   Bit 5   Bit 4 | Bit 3   Bit 2   Bit 1   Bit 0
     +-------------------------------+-------------------------------+
     |      Moving Piece Index       |      Move Value / Opcode      |
     |         (pieceNum)            |             (val)             |
     |      4 bits: 0 .. 15          |        4 bits: 0 .. 15        |
     +-------------------------------+-------------------------------+
```

- **High 4 bits (`pieceNum`, bits 4..7)**:
  Index of the moving piece (0 … 15) in the side's current dynamic piece list (`List[toMove]`).
- **Low 4 bits (`val`, bits 0..3)**:
  The destination square or move type (0 … 15), encoded differently depending on the piece type.

---

### 2. The Dynamic Piece List (List[toMove])

Each side maintains a list of its pieces currently on the board:

```text
Index 0      : KING (Always index 0!)
Index 1 .. 7 : Major / Minor pieces (Rooks, Knights, Bishops, Queen)
Index 8 .. 15: Pawns (8 pawns)
```

- **Standard Start Indices**:
   - White: `0:Ke1, 1:Ra1, 2:Nb1, 3:Bc1, 4:Qd1, 5:Bf1, 6:Ng1, 7:Rh1, 8..15:Pa2..Ph2`
   - Black: `0:Ke8, 1:Ra8, 2:Nb8, 3:Bc8, 4:Qd8, 5:Bf8, 6:Ng8, 7:Rh8, 8..15:Pa7..Ph7`
- **When a piece is captured**:
  The captured piece is removed, and the last piece in the list is swapped into its slot to keep the array compact.

---

### 3. King & Special Tokens (pieceNum == 0)

Because the King is strictly index 0, the high 4 bits for all King moves are always 0000. This means all King moves and special game tokens have a byte value in the range `0x00` to `0x0F`. All other pieces have `pieceNum >= 1`, so their bytes are always `≥ 0x10`.

A King only has 8 regular moves, 2 castling moves, and 1 null move (11 codes), leaving values 11 to 15 free for control tokens!

| Byte Value | Meaning / Action |
| :---: | :--- |
| `0x00` (0) | Null Move (King moves to its own square) |
| `0x01` (1) | King down-left (diff = -9) |
| `0x02` (2) | King down (diff = -8) |
| `0x03` (3) | King down-right (diff = -7) |
| `0x04` (4) | King left (diff = -1) |
| `0x05` (5) | King right (diff = +1) |
| `0x06` (6) | King up-left (diff = +7) |
| `0x07` (7) | King up (diff = +8) |
| `0x08` (8) | King up-right (diff = +9) |
| `0x09` (9) | Queenside Castle (O-O-O) |
| `0x0A` (10) | Kingside Castle (O-O) |
| `0x0B` (11) | NAG Prefix: Followed by 1 byte containing the NAG number |
| `0x0C` (12) | Comment Marker: Indicates a comment belongs at this move |
| `0x0D` (13) | Start Variation: Begins a sub-variation `(` |
| `0x0E` (14) | End Variation: Ends the sub-variation `)` |
| `0x0F` (15) | End of Game: Terminates the move section |

---

### 4. Encoding Other Pieces (Low 4 Bits val)

#### A. Pawns (pieceNum = 8..15)

Let `diff = |to - from|`:

- `15`: Advance 2 squares (`diff == 16`, e.g. `e2-e4`, `c7-c5`).
- `0, 1, 2`: Advance 1 square or capture (no promotion):
   - `0`: Capture left (`diff == 7`)
   - `1`: Advance 1 square (`diff == 8`)
   - `2`: Capture right (`diff == 9`)
- `3 .. 14`: Promotions:
  ```text
  val = base (0..2) + 3 × (promo_piece - 1)
  ```
   - `3, 4, 5`: Promote to Queen (capture-left, forward, capture-right)
   - `6, 7, 8`: Promote to Rook
   - `9, 10, 11`: Promote to Bishop
   - `12, 13, 14`: Promote to Knight

#### B. Knights (pieceNum = 1..7)

A knight has up to 8 moves, encoded based on square difference:

- `-17: 1`, `-15: 2`, `-10: 3`, `-6: 4`
- `+6: 5`, `+10: 6`, `+15: 7`, `+17: 8`

#### C. Rooks (pieceNum = 1..7)

- **Horizontal move (same rank)**: `val = File(to)` (0 … 7, where A=0 .. H=7).
- **Vertical move (same file)**: `val = 8 + Rank(to)` (8 … 15, where 1=8 .. 8=15).

#### D. Bishops (pieceNum = 1..7)

- **Main diagonal (up-right / down-left)**: `val = File(to)` (0 … 7).
- **Anti-diagonal (up-left / down-right)**: `val = 8 + File(to)` (8 … 15).

#### E. Queens: 1-Byte vs. 2-Byte Encoding

Because a queen has up to 27 reachable squares:

1. **Rook-like moves (1 byte)**:
   Encoded identically to Rook moves (horizontal: 0 … 7, vertical: 8 … 15).
2. **Diagonal moves (2 bytes)**:
   - **Byte 1**: `(pieceNum << 4) | File(from)`.
     (A rook-like move to the queen's own file is impossible, so this serves as an escape sequence meaning "diagonal queen move").
   - **Byte 2**: `to_square + 64` (values 64 … 127, avoiding any collision with special tokens 0 … 15).

---

### 5. Real-World Byte Walkthrough: 1. b4 e5 2. Bb2 Bxb4

Here is the exact byte stream from Game 1 of the database:

| Hex Byte | Binary | Piece Index (Hi) | Opcode (Lo) | Decoded Move |
| :---: | :---: | :--- | :--- | :--- |
| `0x9F` | `1001 1111` | 9 (White b2 pawn) | 15 (Push 2 sq) | 1. b4 |
| `0xCF` | `1100 1111` | 12 (Black e7 pawn) | 15 (Push 2 sq) | 1... e5 |
| `0x39` | `0011 1001` | 3 (White c1 bishop) | 9 (Anti-diag, B-file) | 2. Bb2 |
| `0x51` | `0101 0001` | 5 (Black f8 bishop) | 1 (Main diag, B-file) | 2... Bxb4 |

Every single move—including captures and piece disambiguation—is completely represented in 1 single byte!
