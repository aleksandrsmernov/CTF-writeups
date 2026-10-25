# Corridor — TryHackMe Write-up

## 1. Reconnaissance

Target IP: `10.114.184.11`

The website displayed a corridor with 13 doors, each identified by an MD5 hash.

## 2. Hash Analysis

I used CyberChef to identify the hashes as MD5 hashes corresponding to the numbers `1` through `13`.

The application generated requests in the following format:

```http
GET /<MD5_hash>
```

Each hash represented a door in the corridor.

## 3. Exploitation — IDOR

I tested whether the application would respond differently to hashes that did not correspond to existing doors.

- `GET /MD5(flag)` — returned `404 Not Found`.
- `GET /MD5(14)` — returned `404 Not Found`.
- `GET /MD5(0)` — returned `200 OK`.

The hash of `0` was not associated with any of the 13 visible doors, but the server still returned a successful response.

I opened the corresponding URL:

```text
http://10.114.184.11/<MD5_hash_of_0>
```

The response contained the flag.

## 4. Lessons Learned

- CyberChef can help identify and analyze hashes.
- Testing unexpected object identifiers can reveal weaknesses in access control.
- IDOR (Insecure Direct Object Reference) occurs when an application fails to properly enforce authorization checks for accessed objects.
- A `200 OK` response indicates a successful HTTP request, but the response content must still be examined to understand what the server returned.
