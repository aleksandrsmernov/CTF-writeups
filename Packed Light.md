# TryHackMe — Packed Light

## 1. Network Traffic Analysis

Opened the provided `.pcap` file in Wireshark and started analyzing the HTTP traffic.

First, I identified the TCP three-way handshake used to establish the connection:

1. `SYN`
2. `SYN, ACK`
3. `ACK`

After the connection was established, HTTP requests and responses were exchanged.

## 2. Finding Suspicious Requests

I found a request to:

`http://byte-lotus-hotel.thm:8080/temp/updates.py`

This turned out to be a Python script that explained how the data was transmitted.

The script contained:
- `getkey()` — constructs the encryption key.
- `xor()` — performs XOR encryption.
- `sendltr()` — sends data through HTTP requests.
- `hotel_sess_state` — a cookie used to transmit encrypted data.

## 3. Extracting Cookies

I filtered the traffic by IP address, port, and TCP stream to isolate the relevant packets.

In the HTTP requests, I found the `hotel_sess_state` cookie containing encoded data.

Each keystroke was sent in a separate HTTP request, followed by an HTTP response from the server. The data I needed was in the request cookies, not in the response body.

I extracted the cookie values and combined them in their original order.

## 4. Base64 Decoding

The cookie values looked like Base64 strings. After decoding them, I obtained a sequence of bytes.

However, the result was not readable text because the data had also been encrypted using XOR.

It is important to distinguish between:
- **Base64** — an encoding format used to represent binary data as text.
- **XOR** — the operation used to encrypt the data.

## 5. Analyzing the XOR Encryption

I examined `updates.py` and found how the encryption key was constructed:

```python
p1 = "H0t3lSt@ff0Nly"
p2 = "K3epS3cr3t!"

return p1 + p2
```

The `sendltr()` function sent each keystroke in a separate HTTP request. Every time the XOR function was called, it started processing the key from the first byte.

Therefore, each individual character was encrypted using the first byte of the key: `H`.

To decrypt the data, I decoded each cookie from Base64 and XORed the resulting bytes with `H`.

## 6. What I Learned

During this room, I learned how to:

- Analyze TCP handshakes and HTTP traffic in Wireshark.
- Filter traffic by IP address, port, and TCP stream.
- Identify suspicious requests and extract HTTP cookies.
- Recognize Base64 encoding.
- Analyze a Python script to understand an encryption algorithm.
- Recover the original data using XOR.

**Key takeaway:** When analyzing network traffic, extracting the data is only part of the process. Finding the client-side script can reveal how the data is generated, encoded, and encrypted.
