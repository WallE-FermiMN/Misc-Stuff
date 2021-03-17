
# Comm Protocol Version 2
## Format
The protocol is split in multiple levels:
- Level 1: Physical layer
- Level 2: Handles packet framing and error detection
- Level 3: Handles commands

For brevity, level 1 will not be mentioned in this reference.
## Level 2: Framing
All packets start with the byte `0x00` and end with `0xCC`. In case a `0x00` byte is present, the bytes `0xFF` (DLE) and `0xEE` (Null escape) will replace it. If `0xFF` is present in the message, it will be replaced by the sequence `0xFF 0xDD`. If the byte `0xCC` is present it's replaced by `0xFF 0xBB`. If bytes other than `0xEE`, `0xDD` or `0xBB` follow `0xFF` the frame must be discarded. To ensure wrong packets don't affect the system, at the end of the data payload a CRC8 Dallas/Maxim byte is present. To check if the data is correct, a CRC8 of the payload + checksum bytes (After DLE removal) must result 0. This protocol operates as UDP, so errors will be discarded silently.
Example 1:
If I need to send `01 02 03 04 05`, the bytes `00 01 02 03 04 05 {2A} CC` (`{2A}` is the  CRC8)
Example 2:
`0A 3D 00 4E 5F FF 0D 7B` will become `00 0A 3D FF EE 4E 5F FF DD 0D 7B {0A} CC`
Example 3:
`00 00 FF FF CC CC` will become `00 FF EE FF EE FF DD FF DD FF BB FF BB {6F} CC`.

## Level 3: Commands
### Introduction
All command payloads are formed by a command byte. After the command byte variable data follows. The protocol mainly transfers integers, which are represented in rust notation in the following commands (u for unsigned and i for signed, followed by the number of bits). Regarding the time, it's a u32 representing the number of milliseconds from startup. If the time is set to 0 in any command, the action must be computed immediately.
### Servos: Ease PWM
Command byte and data: `D2 [u32] [u8] [u16]` (8 bytes total)

Data: a u32, representing the time T when the easing will complete, a u8 to indicate the channel to execute the easing on (only lower 4 bits used), and a u16 indicating the  value to ease to (only lower 12 bits used).

### DC: Ease Speed + direction
Command byte and data: `87 [u32] [i16] [i16]` (9 bytes total)

Data: a u32, representing the time T when the easing will complete, and two i16, representing the direction (+ is forwards and - is backwards) and speed of the two motors (first value is the left motor, second value is the right motor).

### Arduino: ClockSync
Command byte and data: `4B [u32]` (5 bytes total)
Data: a single u32, representing the number of milliseconds which have passed after the Startup command. The Arduino must compare the current millisecond timer with this value and adjust the clock skew accordingly.

### Arduino: Startup
Command byte and data: `CB` (1 byte total)

Data: No data, this command enables the Arduino to control the motors, and resets the internal millisecond clock to 0. If a startup was already recieved, the timer must still be reset.

## Packet scheme
![Packet scheme](https://i.imgur.com/E5Y0aXi.png)
(Warning: The image is old, check the first chapter to see the updated protocol)
