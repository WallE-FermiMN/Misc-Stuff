# Comm Protocol Version 2
## Format
The protocol is split in multiple levels:
- Level 1: Physical layer
- Level 2: Handles packet framing and error detection
- Level 3: Handles commands

For brevity, level 1 will not be mentioned in this reference.
## Level 2: Framing
All packets start and end with the byte `0x00`.  If more than one frame needs to needs to be sent in rapid succession, Back-To-Back frame support is achieved by removing the start byte from the frame. In case a `0x00` byte is present, the bytes `0xFF` (DLE) and `0xEE` (Null escape) will replace it. If `0xFF` is present in the message, it will be replaced by the sequence `0xFF 0xDD`. If bytes other than `0xEE` or `0xDD` follow `0xFF` the frame must be discarded. To ensure wrong packets don't affect the system, at the end of the data payload a CRC8 Dallas/Maxim byte is present. To check if the data is correct, a CRC8 of the payload + checksum bytes (After DLE removal) must result 0. This protocol operates as UDP, so errors will be discarded silently.
Example 1:
If I need to send `01 02 03 04 05`, the bytes `00 01 02 03 04 05 2A 00`  or `01 02 03 04 05 {2A} 00` will be sent. (`{2A}` is the  CRC8)
Example 2:
`0A 3D 00 4E 5F FF 0D 7B` will become `[00] 0A 3D FF EE 4E 5F FF DD 0D 7B {0A} 00`
Example 3:
`00 00 00 FF FF FF` will become `[00] FF EE FF EE FF EE FF DD FF DD FF DD {66} 00`.

## Level 3: Commands
### Introduction
All command payloads are formed by a command byte, being a SECDED (Extended hamming 7,4) code. If the CRC8 check fails, the command byte must be corrected (if it's wrong, if it's correct the packet must be discarded) and a new CRC8 check must be performed (this avoids losing a packet if the error is in the command byte). After the command byte variable data follows.
### Servos: Ease PWM
Command byte: `1` (`D2` with hamming code)
Data: 4 bytes, split into a byte and 2 12 bit sequences, the first byte is the channel to operate on, the first 12 bit sequence is the value to ease to, and the second 12 bit sequence is the time in milliseconds for the easing. If the millisecond time is 0, the value must be set directly.
Example: Ease in 142 milliseconds to the value 2351, on channel 4
If split, the final sequence is `04 92F 08E` in hex, or `00000100 100100101111 000010001110` in binary. corresponding to `4 2351 142`.
### Servos: Stop all
Command byte: `2` (`55` with hamming code)
Data: No data, this command stops immediately all servos, by setting the PWM register values to 0
### DC: Ease Speed + direction
Command byte: `3` (`87` with hamming code)
Data: 4 bytes, split into a 12 bit sequence, two 2 bit sequences and two bytes.
The 12 bits indicate the easing time, the first 2 bit indicates the values of the direction pins for motor 1, the second for motor 2, same for the 2 bytes that indicate the value to ease to.
Example: Ease in 582 milliseconds, for motor 1 set direction to forwards and value to 121, for motor 2 set direction to backward and value to 213. If split, the final sequence is `246 6 79 D5` in hex, or `001001000110 01 10 01111001 11010101` in binary, corresponding to `582 1(forwards) 2(backwards) 121 213`.
### DC: Stop all (PWM)
Command byte: `4` (`99` with hamming code)
Data: No data, this command stops immediately all servos, by setting the PWM output values to 0.
### DC: Stop all (Power)
Command byte: `5` (`4B` with hamming code)
Data: No data, this command stops immediately all servos, by cutting power to the driver.
### Arduino: Startup
Command byte: `6` (`CC` with hamming code)
Data: No data, this command enables the arduino to control the motors.
### Arduino: Shutdown
Command byte: `7` (`1E` with hamming code)
Data: No data, this command stops all motors gracefully (value to 0, ease time 500 ms). Control will restart once it recieves a startup command.
