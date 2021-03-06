# Arduino - Raspi UART Protocol

Boards connected via USB UART.
# Generic command format
All commands start with the byte `0xAA` (SYN Byte), followed by a packet number byte, the command byte and eventual information data. This is then followed by the checksum start byte `0x0F`, and from the checksum byte. At the end the stop byte `0xC3` is present. The checksum is a CRC-8 Dallas/Maxim of the data frame (setting the checksum byte as `0x00`).
Example command `0x0A`, packet number `0x12`, with data `0x01 0x02`:
`AA 12 0A 01 02 0F ?? C3`
The checksum of this command is calculated from a CRC-8 Dallas/Maxim of `AA 12 0A 01 02 0F 00 C3`, which results `0x65`. The resulting packet is `AA 12 0A 01 02 0F 65 C3`.


# Controller Enable/Disable
|CB|Section|Information  |
|--|--|--|
|`0xE0`|Enable|Enables the Arduino to control the motors|
|`0xE1`|Disable|Gives stop command to the driver|
|`0xE2`|Disable|Sets DC direction pins to ground|
|`0xE3`|Disable|Sets DC enable pins to ground|
|`0xED`|Disable|Cuts power to all servos|
|`0xEE`|Disable|Cuts power to both DC motors|
|`0xEF`|Disable|Cuts power to all motors|
The first command enables the Arduino to control motors (sent on program startup).The second sends a stop command for all motors. The third stops the DC motors by setting the 2 directions pins to ground (the motor is still powered). The fourth pulls the enable pins low, so the motors don't recieve power, but are ready to resume rotation on enable command. The fifth command pulls the driver enable pin to ground, cutting power to the driver and all the servos. The sixth sets the direction and enable pins to ground, immediately halting rotation. The seventh command is a shorthand for the previous two.

These commands have a special structure, since they don't need to be numbered or error checked. Their structure is `[SYN] 0xFF [CB] [CB] 0xC3`, without checksums. The packet number is always 0xFF, to cover the special case. If the 2 command bytes aren't the same, a "Retrasmit 0xFF" is requested. An example packet is `0xAA 0xFF 0xEF 0xEF 0xC3`, cutting power to all motors.

# Servo control
|CB|Section|Information  |
|--|--|--|
|`0xC0`|Set PWM %|Sets PWM Pulse length|
|`0xC1`|Set PWM|Sets PWM Registers manually|
|`0xC2`|Set PWM % mul|Same as first, but for multiple channels|
|`0xC3`|Set PWM mul|Same as second, but for multiple channels|
|`0xCF`|Set Protection|Sets min and max pulse length for servos|

The servos operate on a 12 bit counter. The register ON and OFF values tell when to set the output high and low in the pulse duration.
### Set PWM %
Payload `0xC0 [2 bytes]`. The first 4 bits represent the channel number, the other 12 bits are the OFF register time. The ON register gets automatically set at 0. Example payload:
`C0 3C 00`. The 2 bytes in binary are `0011  1100 0000 0000`, or `3 2048` in decimal if split. The maximum value of the register is 4096, so this payload will make channel 3 (the fourth channel) run at 50% duty cycle.

### Set PWM % mul
Payload `0xC2 [Channels] [2 bytes] ...`.  The first byte represents the number of channels following. The 2 byte parts function exactly as `Set PWM %`.

### Set PWM
Payload `0xC1 [Channel] [3 bytes]`. The first byte represents the channel to operate on. The 3 byte sequence is split into two 12 bit sequences, the first being the ON and the second being the OFF register. Example payload: `C1 0C 60 AE 0A`. The first byte is 12 in decimal, so we operate on channel 12. The 3 bytes are `0110 0000 1010  1110 0000 1010` in binary, or `1546 3594` in decimal if split. This payload will make channel 12 run at 50% duty cycle with a phase shift.

### Set PWM mul
Payload `0xC3 [Channels] [Channel] [3 bytes] ...`. The first byte represents the number of channels following. The channel and 3 byte sequence operate exactly as `Set PWM`.

### Set Protection
Payload `0xCF [3 bytes]`. The 3 byte sequence, split into two 12 byte sequences, represents the minimum and maximum pulse width respectively. If the total pulse width recieved from a packet isn't between these two values (inclusive) no action is performed.

# DC Motor control
|CB|Section|Information  |
|--|--|--|
|`0xD0`|Set PWM 1|Sets PWM Pulse length for motor 1|
|`0xD1`|Set PWM 2|Sets PWM Pulse length for motor 2|
|`0xD2/D3`|Set direction 1|Set direction of motor 1|
|`0xD4/D5`|Set direction 2|Set direction of motor 2|
|`0xDE`|Stop 1|Sets direction pins to ground, stopping motor 1|
|`0xDF`|Stop 2|Sets direction pins to ground, stopping motor 2|

### Set PWM 1/2
Payload `0xD0 [Byte]`/`0xD1 [Byte]`. Sets the Arduino PWM output of motor 1/2 to the byte value. The arduino has 8 bits of PWM resolution.

### Set direction 1/2
|CB|Motor|Direction|
|--|--|--|
|`0xD2`|1|Forwards|
|`0xD3`|1|Backwards|
|`0xD4`|2|Forwards|
|`0xD5`|2|Backwards|

### Stop 1/2
These commands stop the motors by pulling both direction pins to ground, stopping the movement but still leaving the motor powered.

# Error handling
The packet number is sequentially incremented, starting from `0x00` up to `0xFE`. The packet number `0xFF` is reserved for Controller enable/disable packets. In case of a missed packet or of a failing checksum a retrasmit is required with a NACK followed by the packet number. When the packet number `0xFE` is reached, the Raspberry PI sends a NEXT packet, to indicate a packet number reset, and waits for an OK response from the Arduino, then it will refill its buffer with another 255 packets. If there's an error (for example asking to retrasmit an invalid packet) the Arduino will be reset. These packets operate exactly as the Controller Enable/Disable packets, so they don't have checksums and have the special packet number `0xFF`.
|CB|Packet|
|--|--|
|`0xF0`|NEXT|
|`0xF9`|OK|
|`0xFF`|NACK|
