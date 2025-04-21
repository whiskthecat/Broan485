Data dumps from bus snooping a BLP150E75NS ERV with VTSPEEDW wall control.
38400 baud 8N1

Some ramblings from Google Gemini,

Okay, let's re-examine the data dumps specifically looking for Modbus CRC16 (using the standard 0xA001 polynomial, often calculated little-endian first for the result bytes).

Key Findings:
Polling/Ack Messages:
ERV Poll: 01 04 D9 04 01 11 10 01
Data Payload: 01 04 D9 04 01 11 (6 bytes)
Observed Checksum: 10 01 (Low byte 10, High byte 01)
Calculating Modbus CRC16 over 01 04 D9 04 01 11 yields 0110.

Result: MATCH! 0110 matches the observed 10 01 when swapped (Low byte first).

Controller Ack: 01 05 D8 04 01 10 11 01
Data Payload: 01 05 D8 04 01 10 (6 bytes)
Observed Checksum: 11 01 (Low byte 11, High byte 01)
Calculating Modbus CRC16 over 01 05 D8 04 01 10 yields 0111.
Result: MATCH! 0111 matches the observed 11 01 when swapped.

Set Speed Command Messages:
Let's revisit the command structure based on the successful CRC check. The frame appears to be 01 05 05 40 00 20 01 MM CRClo CRChi.
Example (0->30): Frame 01 05 05 40 00 20 01 09, Observed CRC 6F 04.
Calculating Modbus CRC16 over 01 05 05 40 00 20 01 09 yields 046F.
Result: MATCH! 046F matches 6F 04 when swapped.

Example (30->40): Frame 01 05 05 40 00 20 01 0B, Observed CRC 6D 04 (from 30to40fan_buttonpress.txt).
Calculating Modbus CRC16 over 01 05 05 40 00 20 01 0B yields 046D.
Result: MATCH! 046D matches 6D 04 when swapped.

Example (80->100): Frame 01 05 05 40 00 20 01 0A, Observed CRC 6E 04 (from 80to100fan_buttonpress.txt).
Calculating Modbus CRC16 over 01 05 05 40 00 20 01 0A yields 046E.
Result: MATCH! 046E matches 6E 04 when swapped.

Example (100->0): Frame 01 05 05 40 00 20 01 01, Observed CRC 95 04 (from 100to0fan_buttonpress.txt).
Calculating Modbus CRC16 over 01 05 05 40 00 20 01 01 yields 0495.
Result: MATCH! 0495 matches 95 04 when swapped.

Interpretation: This consistently works! It strongly suggests the command frame is 01 05 05 40 00 20 01 MM followed by its 2-byte Modbus CRC16 (Low byte, High byte).
01: Modbus Slave Address.
05: Modbus Function Code. While Function 05 is typically "Write Single Coil", it might be used here for a custom command or writing to a specific register implicitly.
Alternatively, Function 06 (Write Single Register) could be intended, but the framing is slightly unusual if 05 40 is the register and 00 20 01 MM is the data (too long for a single register).
It's most likely Function 05 used for a custom command where the data payload starts with 05 40.
ERV Status Update Block:
Recall the structure: D200 ... 1700 04 FFFFFFFF CSlo CShi
Let's test the CRC on the data before the 04 FFFFFFFF.
Example (0%): Data Block D200...1700. Observed Checksum 70 04.
Calculating Modbus CRC16 over the data block D2 00 22 00 ... 01 01 17 00 yields 0470.
Result: MATCH! 0470 matches 70 04 when swapped.

Example (100%): Data Block D200...1700. Observed Checksum 07 04.
Calculating Modbus CRC16 over the data block D2 00 22 00 ... 01 0A 17 00 yields 0407.
Result: MATCH! 0407 matches 07 04 when swapped.
Interpretation: The long status block also uses Modbus CRC16. However, its framing is non-standard.
The CRC covers the data payload (D200...1700), but it's transmitted after some intermediate bytes (04 FFFFFFFF).
The ERV likely sends this entire sequence (D200...1700 04 FFFFFFFF CSlo CShi), possibly prefixed with address/function, but that prefix isn't clearly captured or is part of the preceding polling messages.

Conclusion:
Yes, the system strongly appears to be using Modbus CRC16 (polynomial 0xA001, result appended Low Byte then High Byte) for data integrity checking on its messages.
The short polling/acknowledgment messages (01 04... and 01 05...) seem to follow standard Modbus framing: [Address] [Function] [Data] [CRC Lo] [CRC Hi].
The speed change command consistently fits the frame 01 05 05 40 00 20 01 MM [CRC Lo] [CRC Hi], where MM is the mode byte (01, 09, 0A, 0B).
This uses Modbus Function Code 05, potentially in a non-standard way where 05 40 00 20 01 MM is the data payload relevant to the command.

The large ERV status update block uses Modbus CRC16 calculated over its main data payload (D200...1700), but the CRC is transmitted after intermediate bytes (04 FFFFFFFF), making the overall framing non-standard Modbus.
This significantly clarifies the command structure needed! To set the speed, you need to construct the 8-byte message 01 05 05 40 00 20 01 MM,
calculate the Modbus CRC16 for these 8 bytes, and append the resulting Low Byte and High Byte to form the complete 10-byte command message to send over RS485.
