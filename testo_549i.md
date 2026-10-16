# WhiskerLink: Testo 549i BLE Protocol Notes

Reverse-engineered interoperability notes, October 2026.

**Status:** working implementation proven on a real Testo 549i.

This document describes the subset of the Testo 549i Bluetooth protocol needed to:

- discover the probe
- connect over BLE
- start live measurement streaming
- decode pressure
- decode battery level
- issue an on-device `ZeroPressure` command

It is based on Bluetooth HCI captures of the official Testo app plus a Linux/Bleak proof of concept that successfully generates its own valid Testo command packets.

---

## 1. Scope and confidence

| Feature | Status | Notes |
|---|---|---|
| Advertisement / model detection | Verified | Probe advertises a human-readable local name containing model and serial. |
| GATT write / notify transport | Verified | FFF1 write, FFF2 notify for the Testo protocol stream. |
| Session startup | Verified | Captured startup sequence works in the Linux POC. |
| Pressure decoding | Verified | `DifferentialPressure` is float32 little-endian in pascals. |
| Battery decoding | Verified | `BatteryLevel` is float32 little-endian percentage. |
| `ZeroPressure` | Verified | Generated packets with calculated sequence and CRC are accepted and zero the probe. |
| CRC algorithm | Verified | CRC-16/MODBUS, transmitted little-endian. |
| Sequence counter | Verified | Increments by one for successive commands. |
| Generic meaning of every opcode | Unknown | Only observed behaviors are assigned meanings here. |
| Full protocol / all Smart Probes | Not claimed | This document is specifically about observed 549i behavior. |

---

## 2. Device discovery

The working probe advertises as:

```text
T549i SN:47245295
```

A practical scanner can identify the model and serial with something like:

```regex
^T(?P<model>549i)\s+SN:(?P<serial>\d+)$
```

The full serial number in the advertisement is useful as a stable human-facing identifier.

Do not assume any shorter ID shown by Testo software is separately transmitted unless captured.

---

## 3. BLE / GATT transport

The 549i uses the same proprietary FFFx service family observed on the Testo 115i.

The important characteristics for the working interoperability path are:

| UUID | Role | Use |
|---|---|---|
| `0000fff1-0000-1000-8000-00805f9b34fb` | Write | Commands from client to probe |
| `0000fff2-0000-1000-8000-00805f9b34fb` | Notify | Responses and live measurement/event data |

The Linux proof of concept:

1. connects normally over BLE
2. enables notifications on FFF2
3. waits briefly
4. writes startup commands to FFF1

No pairing, bonding, challenge/response, or per-device credential was observed in the tested workflow.

---

## 4. Command packet framing

Command packets have an 8-byte header followed by an optional payload.

Both the header and a non-empty payload are protected with CRC-16/MODBUS.

```text
Header, 8 bytes:

byte 0      command / opcode
byte 1      sequence number
bytes 2-5   payload length, uint32 little-endian
bytes 6-7   CRC-16/MODBUS of bytes 0-5, little-endian
```

For packets with a payload:

```text
payload data
CRC-16/MODBUS of payload data, little-endian
```

The header's payload-length field includes the 2-byte payload CRC.

So conceptually:

```text
[ opcode ]
[ sequence ]
[ payload length LE32 ]
[ header CRC LE16 ]
[ payload bytes ... ]
[ payload CRC LE16 ]
```

---

## 5. CRC-16/MODBUS

Both the command header CRC and the `ZeroPressure` payload CRC match CRC-16/MODBUS.

Useful implementation:

```python
def crc16_modbus(data: bytes) -> int:
    crc = 0xFFFF

    for byte in data:
        crc ^= byte

        for _ in range(8):
            if crc & 1:
                crc = (crc >> 1) ^ 0xA001
            else:
                crc >>= 1

    return crc
```

CRC parameters:

```text
width:      16
poly:       0x8005
reflected:  0xA001
init:       0xFFFF
refin:      true
refout:     true
xorout:     0x0000
```

The CRC is transmitted little-endian.

Example:

```text
05 04 13 00 00 00
```

CRC-16/MODBUS:

```text
0x0AF5
```

Wire order:

```text
f5 0a
```

---

## 6. Observed 549i startup sequence

The observed 549i startup consumes sequence numbers `00` through `03`.

The first three commands match the working 115i capture.

### Command 1

```text
56 00 03 00 00 00 0c 69 02 3e 81
```

The exact semantic meaning of opcode `0x56` is not known.

It appears to be part of session initialization.

### Command 2

```text
20 01 00 00 00 00 3a bb
```

The exact semantic meaning of opcode `0x20` is not known.

### Command 3: FirmwareVersion query

```text
04 02 15 00 00 00 7c 53
0f 00 00 00
46 69 72 6d 77 61 72 65 56 65 72 73 69 6f 6e
30 4f
```

ASCII field name:

```text
FirmwareVersion
```

The working probe reported firmware:

```text
2.004.17
```

### Command 4: stream start

```text
11 03 00 00 00 00 47 5a
```

The exact formal meaning of opcode `0x11` is not known, but behavior strongly indicates this starts the live stream.

Unlike the 115i capture, the 549i did not perform the observed `TempCompensationActive` query before streaming.

After startup, the next outgoing command sequence number is:

```text
04
```

---

## 7. Asynchronous measurement framing

Live values arrive via FFF2 notifications.

Observed async values begin with an 8-byte header:

```text
10 80 LL LL LL LL CC CC
```

Where:

```text
10 80       async measurement/event type
LL LL LL LL payload length, uint32 little-endian
CC CC       header CRC
```

The payload length tells the receiver how many bytes follow.

BLE notification boundaries are not protocol-message boundaries, so a client should reassemble messages by declared length instead of assuming one notification contains one complete value.

Observed named-value payload structure:

```text
uint32_le name_length
ASCII       name[name_length]
float32_le  value
...         optional extra metadata
uint16_le   CRC-16/MODBUS
```

---

## 8. Pressure measurement

The 549i streams pressure under the name:

```text
DifferentialPressure
```

The numeric value is:

```text
IEEE-754 float32
little-endian
unit: pascals
```

Convert pascals to psi with:

```python
psi = pascals / 6894.757293168
```

### Observed physical test

A controlled capture showed:

```text
small pre-zero offset
-> ZeroPressure
-> near 0 psi
-> R-134a bottle opened
-> about 49.9 psi
-> bottle closed / disconnected
-> back near 0 psi
```

The decoded BLE values tracked the official Testo app during this test.

That strongly confirms both the field interpretation and the units.

---

## 9. Battery measurement

Battery level is streamed under:

```text
BatteryLevel
```

The value is float32 little-endian percentage.

Example:

```text
00 00 c8 42
```

decodes to:

```text
100.0
```

So:

```text
BatteryLevel = 100.0 %
```

---

## 10. ZeroPressure command

Zeroing is performed on the probe itself.

It is not merely an offset maintained by the Testo phone app.

The Linux WhiskerLink POC generated its own valid `ZeroPressure` command and the real 549i accepted it and zeroed successfully.

### Logical payload

Before the payload CRC:

```text
0c 00 00 00
5a 65 72 6f 50 72 65 73 73 75 72 65
01
```

Decoded:

```text
uint32_le 12
ASCII "ZeroPressure"
01
```

### Payload CRC

CRC-16/MODBUS of:

```text
0c 00 00 00
5a 65 72 6f 50 72 65 73 73 75 72 65
01
```

is:

```text
0xE562
```

Transmitted little-endian:

```text
62 e5
```

### Complete ZeroPressure packet at sequence 0x04

```text
05 04 13 00 00 00 f5 0a
0c 00 00 00
5a 65 72 6f 50 72 65 73 73 75 72 65
01
62 e5
```

Interpretation:

```text
05                   opcode
04                   sequence
13 00 00 00          payload length = 19 bytes
f5 0a                header CRC

0c 00 00 00          string length = 12
"ZeroPressure"
01
62 e5                payload CRC
```

---

## 11. Sequence behavior

Repeated Zero presses showed that the second header byte is a command sequence counter.

The app sent five consecutive `ZeroPressure` commands with:

```text
04
05
06
07
08
```

The payload stayed identical.

Only the sequence byte and corresponding header CRC changed.

Observed examples:

```text
seq 04 -> header CRC f5 0a
seq 05 -> header CRC c8 ca
seq 06 -> header CRC 8c ca
seq 07 -> header CRC b1 0a
seq 08 -> header CRC e5 0b
```

Implementation rule:

```python
sequence = (sequence + 1) & 0xFF
```

No special behavior for wraparound has yet been tested.

---

## 12. Generic Testo packet builder

A working packet builder:

```python
def with_crc(data: bytes) -> bytes:
    return data + crc16_modbus(data).to_bytes(2, "little")


def build_testo_packet(
    command_type: int,
    sequence: int,
    payload: bytes = b"",
) -> bytes:
    if payload:
        framed_payload = with_crc(payload)
    else:
        framed_payload = b""

    header_without_crc = (
        bytes([command_type, sequence])
        + len(framed_payload).to_bytes(4, "little")
    )

    return with_crc(header_without_crc) + framed_payload
```

This builder has been used to generate a real `ZeroPressure` command accepted by the probe.

---

## 13. Minimal 549i implementation recipe

A minimal compatible client can do the following:

1. Scan for an advertisement matching:

   ```text
   T549i SN:<serial>
   ```

2. Connect with a normal BLE GATT client.

3. Subscribe to notifications on:

   ```text
   0000fff2-0000-1000-8000-00805f9b34fb
   ```

4. Send the four captured 549i startup frames to:

   ```text
   0000fff1-0000-1000-8000-00805f9b34fb
   ```

5. Set the next outgoing command sequence to:

   ```text
   0x04
   ```

6. Reassemble async `10 80` messages according to their declared payload length.

7. Decode:

   ```text
   DifferentialPressure
   ```

   as float32 little-endian pascals.

8. Convert pressure to psi if desired:

   ```python
   psi = pa / 6894.757293168
   ```

9. Decode:

   ```text
   BatteryLevel
   ```

   as float32 little-endian percentage.

10. To zero:

    - build the `ZeroPressure` payload
    - append its CRC
    - create opcode `0x05`
    - use the current sequence number
    - calculate the header CRC
    - send to FFF1
    - increment the sequence number

---

## 14. Stop / disconnect behavior

A stop/close command family using opcode:

```text
0x57
```

has been observed.

An earlier captured example from the 115i was:

```text
57 05 00 00 00 00 c1 fc
```

The current WhiskerLink Linux POC can build an empty-payload `0x57` command using the current sequence number.

The exact vendor semantic name of opcode `0x57` is not confirmed.

Treat it as observed session-stop behavior rather than a fully documented command name.

---

## 15. Known unknowns

Still not established:

- formal vendor names for opcodes `0x56`, `0x20`, `0x11`, `0x57`
- whether every 549i firmware revision uses exactly the same startup sequence
- whether sequence wraparound at `0xFF` needs special handling
- meaning of every optional byte following named async values
- existence and format of additional 549i settings or calibration commands
- how much of this exact framing is shared by every Testo Smart Probe model
- whether arbitrary property queries can be safely constructed using the same generic pattern

These should stay labeled as unknown until tested.

---

## 16. Interoperability observations

The tested 549i workflow showed:

```text
ordinary BLE connection
no observed pairing requirement
no observed bonding requirement
no observed challenge/response
no observed per-device secret
static startup structure
human-readable property names
CRC-protected binary framing
```

This does **not** imply every Testo Bluetooth product behaves the same way.

It only describes the tested 549i workflow.

---

## 17. CRC test vectors

These are useful for validating independent implementations.

### Header vector 1

Input:

```text
05 04 13 00 00 00
```

CRC:

```text
0x0AF5
```

Wire bytes:

```text
f5 0a
```

### Header vector 2

Input:

```text
05 05 13 00 00 00
```

CRC:

```text
0xCAC8
```

Wire bytes:

```text
c8 ca
```

### Header vector 3

Input:

```text
05 06 13 00 00 00
```

CRC:

```text
0xCA8C
```

Wire bytes:

```text
8c ca
```

### Header vector 4

Input:

```text
05 07 13 00 00 00
```

CRC:

```text
0x0AB1
```

Wire bytes:

```text
b1 0a
```

### Header vector 5

Input:

```text
05 08 13 00 00 00
```

CRC:

```text
0x0BE5
```

Wire bytes:

```text
e5 0b
```

### ZeroPressure payload

Input:

```text
0c 00 00 00
5a 65 72 6f 50 72 65 73 73 75 72 65
01
```

CRC:

```text
0xE562
```

Wire bytes:

```text
62 e5
```

---

## 18. Proven milestone

WhiskerLink does not merely replay a captured Zero packet.

It:

```text
builds the payload
calculates its CRC
chooses the current sequence number
builds the command header
calculates the header CRC
sends the packet over BLE
```

The real Testo 549i accepts the generated command and zeroes.

That is currently the strongest proof that this protocol description is sufficient for basic independent interoperability.

---

## 19. Suggested driver model

A 549i driver only needs to expose a small set of useful semantics:

```text
identity:
  model
  serial

measurements:
  DifferentialPressure -> pascals
  BatteryLevel         -> percent

actions:
  ZeroPressure
```

Unit conversion and refrigerant saturation calculations should live above the probe driver.

For example, WhiskerLink can keep the probe driver focused on:

```text
pressure_pa
battery_percent
zero()
```

while the application layer handles:

```text
psi / bar / kPa
selected refrigerant
dew / bubble saturation temperature
display formatting
```

This keeps the transport/protocol layer independent of HVAC presentation logic.

---

## 20. Project note

These notes are intended as an independent interoperability reference.

They document behavior observed from hardware and traffic captures and should distinguish verified behavior from inference.

No proprietary source code or confidential vendor documentation is required for the working implementation described here.
