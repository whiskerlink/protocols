# WhiskerLink: Testo 115i BLE Protocol Notes

Reverse-engineered interoperability notes, October 2026.

**Status:** working implementation proven on a real Testo 115i.

This document describes the subset of the Testo 115i Bluetooth protocol needed to:

- discover the probe
- connect over BLE
- start live measurement streaming
- decode clamp temperature
- decode battery level
- query firmware version
- stop the session cleanly

It is based on Bluetooth HCI captures of the official Testo app plus a Linux/Bleak proof of concept that successfully connects directly to the probe and receives live measurements.

---

## 1. Scope and confidence

| Feature | Status | Notes |
|---|---|---|
| Advertisement / model detection | Verified | Probe advertises a human-readable local name containing model and serial. |
| GATT write / notify transport | Verified | FFF1 write, FFF2 notify for the Testo protocol stream. |
| Session startup | Verified | Captured startup sequence works in the Linux POC. |
| Temperature decoding | Verified | `LineTemperature` is float32 little-endian in degrees Celsius. |
| Battery decoding | Verified | `BatteryLevel` is float32 little-endian percentage. |
| Firmware query | Verified | `FirmwareVersion` query returns an ASCII version string. |
| CRC algorithm | Verified | CRC-16/MODBUS, transmitted little-endian. |
| Sequence counter | Verified by family behavior | Startup commands consume increasing sequence values. |
| `TempCompensationActive` query | Verified as a query name | Exact semantic purpose beyond the obvious name is not fully explored. |
| Generic meaning of every opcode | Unknown | Only observed behaviors are assigned meanings here. |
| Full protocol / all Smart Probes | Not claimed | This document is specifically about observed 115i behavior. |

---

## 2. Device discovery

The working probe advertises as:

```text
T115i SN:47672465
```

A practical scanner can identify model and serial with something like:

```regex
^T(?P<model>115i)\s+SN:(?P<serial>\d+)$
```

The official Testo app showed the shorter identifier:

```text
465
```

The BLE advertisement contains the full serial:

```text
47672465
```

The fact that the app's ID matches the last three digits is observable, but no separate short-ID field has been proven.

Do not hardcode that relationship as protocol truth unless separately confirmed.

---

## 3. BLE / GATT transport

The observed 115i GATT layout includes:

```text
Service:
00005a90-0000-1000-8000-00805f9b34fb

Characteristics:
5a93  read
5a94  read
5a91  read, notify
5a92  read, notify
```

Standard Device Information service:

```text
180a
```

Observed proprietary FFF0 service:

```text
fff0
```

Important protocol characteristics:

| UUID | Properties | Observed role |
|---|---|---|
| `0000fe63-0000-1000-8000-00805f9b34fb` | write-without-response, notify | Present, not required for the working POC path |
| `0000fff2-0000-1000-8000-00805f9b34fb` | notify | Testo responses and live data |
| `0000fff1-0000-1000-8000-00805f9b34fb` | write / write-without-response | Testo commands |
| `0000fe64-0000-1000-8000-00805f9b34fb` | write-without-response, notify | Present, not required for the working POC path |

The working Linux proof of concept uses:

```text
WRITE:
0000fff1-0000-1000-8000-00805f9b34fb

NOTIFY:
0000fff2-0000-1000-8000-00805f9b34fb
```

ATT handles may appear one higher in packet captures than characteristic declaration handles because the ATT value handle is distinct from the declaration handle.

Using UUIDs avoids depending on fixed handles.

---

## 4. Security / connection behavior

In the tested workflow, no app-layer authentication mechanism was observed.

The working path is:

```text
normal BLE connection
subscribe to notifications
send static startup frames
receive live data
```

No observed:

```text
pairing requirement
bonding requirement
challenge / response
secret key
per-device credential
```

This statement applies only to the tested 115i workflow.

It should not be generalized to all Testo Bluetooth products.

---

## 5. Command packet framing

Observed Testo commands use an 8-byte header followed by an optional payload.

The general format is:

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

The header payload-length field includes the 2-byte payload CRC.

Conceptually:

```text
[ opcode ]
[ sequence ]
[ payload length LE32 ]
[ header CRC LE16 ]
[ payload bytes ... ]
[ payload CRC LE16 ]
```

The same CRC framing was independently proven by generated `ZeroPressure` packets on the 549i and matches the 115i captured frames.

---

## 6. CRC-16/MODBUS

The CRC algorithm is CRC-16/MODBUS.

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

Parameters:

```text
width:      16
poly:       0x8005
reflected:  0xA001
init:       0xFFFF
refin:      true
refout:     true
xorout:     0x0000
```

CRC bytes are transmitted little-endian.

---

## 7. Observed 115i startup sequence

The working 115i startup sequence is:

### Command 1

```text
56 00 03 00 00 00 0c 69 02 3e 81
```

The exact vendor semantic meaning of opcode `0x56` is unknown.

Behavior suggests session/open/init activity.

### Command 2

```text
20 01 00 00 00 00 3a bb
```

The exact vendor semantic meaning of opcode `0x20` is unknown.

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

Observed response:

```text
07 02 0d 00 00 00 7a c0
07 00 00 00
32 2e 32 30 34 2e 31
a2 63
```

Decoded firmware string:

```text
2.204.1
```

### Command 4: TempCompensationActive query

```text
04 03 1c 00 00 00 42 0f
16 00 00 00
54 65 6d 70 43 6f 6d 70 65 6e 73 61 74 69
6f 6e 41 63 74 69 76 65
b5 4a
```

ASCII field name:

```text
TempCompensationActive
```

Observed response:

```text
07 03 03 00 00 00 45 e8
01
7e 80
```

The payload contains a one-byte value:

```text
01
```

This is consistent with a boolean true value, although no broader setting semantics were explored.

### Command 5: stream start

```text
11 04 00 00 00 00 f2 9a
```

Behavior strongly indicates opcode `0x11` starts the live stream.

After this sequence, the next command sequence would logically be:

```text
05
```

---

## 8. Startup responses

The captured startup exchange included the following responses.

### Response to command 1

```text
07 00 00 00 00 00 01 ac
```

### Response to command 2

Header:

```text
07 01 04 00 00 00 3d 5c
```

Payload:

```text
01 00
00 20
```

Exact semantics have not been established.

### FirmwareVersion response

Header:

```text
07 02 0d 00 00 00 7a c0
```

Payload:

```text
07 00 00 00
32 2e 32 30 34 2e 31
a2 63
```

Decoded:

```text
2.204.1
```

### TempCompensationActive response

```text
07 03 03 00 00 00 45 e8
01
7e 80
```

### Response to stream-start command

```text
07 04 00 00 00 00 f0 6c
```

---

## 9. Asynchronous measurement framing

Live measurements arrive over FFF2 notifications.

Observed async headers begin:

```text
10 80
```

Example:

```text
10 80 1b 00 00 00 05 b1
```

Interpretation:

```text
10 80       async measurement/event type
1b 00 00 00 payload length = 27 bytes
05 b1       header CRC
```

The declared number of bytes follows the 8-byte header.

The payload itself ends in a 2-byte CRC.

BLE notification boundaries are not guaranteed to align with complete Testo messages.

A client should therefore:

```text
read the declared payload length
buffer notifications
collect exactly that many bytes
then parse the complete payload
```

Do not assume one BLE notification equals one Testo logical packet.

---

## 10. Named-value payload structure

Observed live named values use this shape:

```text
uint32_le name_length
ASCII       name[name_length]
float32_le  value
...         optional metadata
uint16_le   CRC-16/MODBUS
```

The 115i currently exposes at least:

```text
LineTemperature
BatteryLevel
```

---

## 11. LineTemperature

The live temperature field is named:

```text
LineTemperature
```

Observed body shape:

```text
0f 00 00 00
"LineTemperature"
<float32 little-endian>
01 00
<CRC16>
```

The 4-byte value is an IEEE-754 float32 in degrees Celsius.

The captured values changed correctly when the physical clamp was warmed, which confirmed the float interpretation.

Example application conversion:

```python
fahrenheit = celsius * 9 / 5 + 32
```

The trailing:

```text
01 00
```

is observed metadata.

Its exact semantic meaning has not been established.

---

## 12. BatteryLevel

Battery data is streamed under:

```text
BatteryLevel
```

Observed body shape:

```text
0c 00 00 00
"BatteryLevel"
<float32 little-endian>
<CRC16>
```

Example value bytes:

```text
00 00 c8 42
```

decode to:

```text
100.0
```

So:

```text
BatteryLevel = 100.0 %
```

---

## 13. Session stop / close

A likely stop/close frame was captured:

```text
57 05 00 00 00 00 c1 fc
```

Behavior and placement strongly suggest opcode:

```text
0x57
```

is a session stop/close command.

The exact formal vendor command name is not known.

Because this is an empty-payload command, it can be generated from the generic header builder using the current sequence number.

---

## 14. Generic packet builder

A generic Testo packet builder suitable for the 115i family framing is:

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

For the current Linux POC, retaining the known-good captured startup frames is reasonable.

There is no requirement to synthesize every startup query until doing so provides a practical benefit.

---

## 15. Minimal 115i implementation recipe

A minimal independent client can:

1. Scan for:

   ```text
   T115i SN:<serial>
   ```

2. Connect normally over BLE.

3. Subscribe to:

   ```text
   0000fff2-0000-1000-8000-00805f9b34fb
   ```

4. Send the five known startup packets to:

   ```text
   0000fff1-0000-1000-8000-00805f9b34fb
   ```

5. Reassemble asynchronous `10 80` messages according to their declared payload length.

6. Decode:

   ```text
   LineTemperature
   ```

   as float32 little-endian Celsius.

7. Decode:

   ```text
   BatteryLevel
   ```

   as float32 little-endian percent.

8. Optionally parse the startup `FirmwareVersion` response.

9. On clean shutdown, send opcode `0x57` using the current sequence number.

---

## 16. 115i versus 549i startup difference

The two tested probes share the same first three startup commands:

```text
56 00 ...
20 01 ...
04 02 ... "FirmwareVersion" ...
```

After that they diverge.

### 115i

```text
04 03 ... "TempCompensationActive" ...
11 04 ... stream start
```

### 549i

```text
11 03 ... stream start
```

So the 115i consumes one additional command sequence number before streaming.

This is a useful sign that the devices share a common Testo protocol family while still exposing model-specific startup behavior.

---

## 17. Driver model

A clean WhiskerLink 115i driver only needs to expose:

```text
identity:
  model
  serial

measurements:
  LineTemperature -> degrees Celsius
  BatteryLevel    -> percent
```

A useful application-facing representation is:

```text
temperature_c
battery_percent
```

Unit conversion belongs above the probe driver:

```text
Celsius
Fahrenheit
display formatting
```

This keeps the BLE/protocol driver focused on what the instrument actually transmits.

---

## 18. Known unknowns

Still not established:

- formal vendor names for opcodes `0x56`, `0x20`, `0x11`, and `0x57`
- exact meaning of the response to opcode `0x20`
- exact semantics of the trailing `01 00` after `LineTemperature`
- whether `TempCompensationActive` can be changed, or is merely queried
- whether all 115i firmware revisions use the same startup sequence
- sequence wraparound behavior at `0xFF`
- any additional hidden settings or diagnostic commands
- whether the observed 5A90/5A9x service participates in other official workflows
- which protocol details are universal across the entire Smart Probe family

These should remain labeled as unknown until tested.

---

## 19. Proven milestone

The Linux WhiskerLink proof of concept successfully:

```text
discovers the 115i
connects without the Testo app
subscribes to FFF2
sends the captured startup sequence
receives live LineTemperature values
receives BatteryLevel values
decodes the measurements correctly
```

The probe's green connected state behaves as if it is communicating with the official client.

This proves that the tested 115i can be used directly by an independent BLE client with the protocol information documented here.

---

## 20. Project note

These notes are intended as an independent interoperability reference.

They document behavior observed from hardware and traffic captures and distinguish verified behavior from inference.

No proprietary Testo source code or confidential vendor documentation is required for the working implementation described here.
