# SR50 DiTech K-line protocol cheat sheet

**Your setup:** Aprilia SR50 DiTech, FT232R KKL cable, MKT100 and our Python diagnostic tool. This sheet describes the **observed exchange on your scooter / MKT100 captures**. It is a practical introduction to **KWP2000 (ISO 14230) over K-line**, not a claim that every DiTech byte is defined by the standard.

## 1. The four layers

| Layer | What it means here | What to look for |
| --- | --- | --- |
| Electrical | Single-wire K-line on the scooter; the KKL interface handles vehicle voltage. | Do not wire a bare 3.3/5 V UART pin directly to the scooter K-line. |
| Serial/timing | About **10,400 baud, 8N1**; the tester begins with a roughly **25 ms LOW + 25 ms HIGH** fast init. | The Python tool uses the FTDI break signal. A plain serial terminal does not perform our whole handshake. |
| Frame | Startup uses a longer addressed frame; later observed requests and replies use compact length/payload/checksum frames. | A KKL adapter also echoes outgoing bytes back to RX. Echo is **not** an ECU reply. |
| Diagnostic service | A service ID (SID) asks for a function; a positive response usually uses SID + `0x40`. | `27` → `67` for security, `21` → `61` for a local-ID read. |

ISO 14230-2 covers the K-line data link; ISO 14230-3 covers KWP2000 diagnostic services. The exact security calculation and the meanings of DiTech local IDs are ECU/MKT-specific.

## 2. Hex and the checksum

Each pair of hex digits is one byte. `0D` is hexadecimal thirteen, not the characters `0` and `D` sent separately.

The compact frames we captured are:

```text
[number of payload bytes] [payload bytes...] [checksum]
checksum = sum of all preceding bytes, modulo 256
```

Example: read local ID `0D`:

```text
02 21 0D 30
│  │  │  └─ checksum: (02 + 21 + 0D) & FF = 30
│  │  └──── local identifier
│  └─────── service 21: read by local identifier
└────────── 2 payload bytes (21 and 0D)
```

The first startup frame is **different**: `81 10 F1 81 03`. In the usual KWP interpretation, it contains a format byte `81`, ECU target `10`, tester source `F1`, StartCommunication service `81`, and checksum `03`. Its five bytes also add up according to the same checksum rule. Do not parse it as a compact frame with `81` payload bytes.

## 3. What a real login looked like

`TX` is tester → ECU. `RX` is ECU → tester **after removing the KKL echo**.

| Step | TX | RX / interpretation |
| --- | --- | --- |
| Fast init | K-line LOW ~25 ms, HIGH ~25 ms | Wakes the link; no serial frame yet. |
| Start communication | `81 10 F1 81 03` | `03 C1 D5 8F 28`: positive `C1`, two returned key bytes. |
| Ask for seed | `02 27 03 2C` | `06 67 03 53 5A 53 5D CD`: seed is `53 5A 53 5D`. |
| Send calculated key | `06 27 04 F6 46 45 FD AF` | `02 67 04 6D`: key accepted. |
| Enter session `82` | `02 10 82 94` | `01 50 51`: session accepted. |
| Read ECU ID | `02 1A 86 A2` | `0A 5A 08 53 52 35 30 30 33 01 03 DD`: contains ASCII `SR5003`. |
| Read local ID `0D` | `02 21 0D 30` | `05 61 0D 02 2F 28 CC`: positive read, data `02 2F 28`. |

These are example frames from the successful run; **the seed and key change**. Do not replay the example key against a different seed.

For this MKT100 version we recovered and checked the key calculation:

```python
seed_number = int.from_bytes(seed_4_bytes, "big")
key_number = ((seed_number * 0x41FA3583) & 0xFFFFFFFF) ^ 0x960AAC6A
key_4_bytes = key_number.to_bytes(4, "big")
```

It maps example seed `53 5A 53 5D` to `F6 46 45 FD`; the ECU returned a positive `67 04` to that key. This is **our observed MKT100 calculation**, not a generic ISO 14230 formula.

## 4. Reading a reply

Take the actual `0D` reply:

```text
05 61 0D 02 2F 28 CC
│  │  │  └── data: 02 2F 28
│  │  └───── echoed identifier: 0D
│  └──────── positive response to service 21: 61
└─────────── five payload bytes: 61 0D 02 2F 28
```

Its checksum is `CC`, because `(05 + 61 + 0D + 02 + 2F + 28) & FF = CC`. For this **confirmed** local ID, `02` indicates a two-byte value, `2F 28` is `0x2F28 = 12072`, and our tool displays **12.072 V**. That observation does **not** mean that every other `21` identifier is a voltage or uses the same scaling.

Common response patterns in these captures:

| Pattern | Meaning |
| --- | --- |
| `67 03 [4 seed bytes]` | Positive SecurityAccess seed response. |
| `67 04` | Positive key acceptance. |
| `50` | Positive response to session service `10`. |
| `5A ...` | Positive ECU identification response to `1A`. |
| `61 [ID] [data...]` | Positive response to local-ID service `21`. |
| `57 [count] ...` | Positive response to captured DTC read service `17`. |
| `7F [requested SID] [NRC]` | Negative response; the final byte is the reason code. |

Example negative response: `03 7F 17 31 CA` = service `17` rejected with `31` (request out of range). The `03` is payload length and `CA` is checksum. A negative response is still a real ECU response; a **no reply** is a different problem.

## 5. The DTC scan we captured

MKT100 reads DTCs by sending service `17` for these groups in order:

| Group | Full request |
| --- | --- |
| `10` | `02 17 10 29` |
| `11` | `02 17 11 2A` |
| `20` | `02 17 20 39` |
| `30` | `02 17 30 49` |
| `40` | `02 17 40 59` |
| `50` | `02 17 50 69` |
| `60` | `02 17 60 79` |
| `70` | `02 17 70 89` |

The parser expects a positive payload of `57 COUNT [DTC high, DTC low, status] × COUNT`. For example `02 57 00 59` means **zero codes in that group**; it does not by itself prove that all eight groups are clear. Our Python script shows the two DTC bytes as an SAE-style `P/C/B/U` code and preserves the raw bytes and status in CSV. **The precise status-bit meanings have not been confirmed** on this ECU.

The observed scan can end with `7F 17 31` on group `70`. Save all groups' responses before deciding whether the ECU has stored faults. An absent reply or bad checksum must not be counted as “no codes.”

**Clearing DTCs:** MKT100 has a CLEAR DTC button, but we have **not yet captured and verified its request and ECU acknowledgment**. The Python tool v0.9 reads DTCs; it does not clear them. For now, save/read first, clear with MKT100, then read again. We need one capture of that button press before adding a trustworthy clear command to our tool.

## 6. Following a trace without getting fooled

1. Find the tester's outgoing frame (`TX`).
2. Remove the identical bytes echoed back by the KKL adapter (`ECHO`). USBPcap sees USB traffic; it is not a direct scope trace of the K-line voltage.
3. The remaining bytes are the ECU frame (`ECU RX`). Check `length + 2` against its total bytes, then verify checksum.
4. Read the payload's first byte as SID. Match a positive SID to the request (`+40` hex), or parse `7F original-SID NRC` for a rejection.
5. Keep requests close together. We saw a local-ID read fail after an idle pause of roughly five seconds; **we have not measured the exact session timeout**.

## 7. Three quick exercises

1. What is the full compact request to read local ID `00`? **Answer:** `02 21 00 23`.
2. In `02 67 04 6D`, which byte says the key was accepted? **Answer:** `67 04` is the positive SecurityAccess/key response; `6D` is only the checksum.
3. Does `02 57 00 59` mean all scooter faults were cleared? **Answer:** No. It reports zero codes for **one group**; inspect the rest of the scan.

## References and scope

- [ISO 14230-2:2016](https://www.iso.org/standard/69115.html): K-line data-link layer overview.
- [ISO 14230-3:1999](https://www.iso.org/standard/23921.html): KWP2000 application-layer/service overview.
- [ELM327 datasheet](https://www.elmelectronics.com/wp-content/uploads/2016/07/ELM327DS.pdf): useful background on K-line initialization, with the caveat that generic OBD commands do not automatically implement this DiTech diagnostic session.
- The byte examples and MKT100 key formula above come from **our captures and your successful scooter run**, not from the ISO abstracts. Verify a response on your own ECU before treating a local ID or DTC status as understood.
