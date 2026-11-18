# CCT2019 — TryHackMe | Complete Challenge Write-up

![TryHackMe](https://img.shields.io/badge/TryHackMe-CCT2019-red) ![Category](https://img.shields.io/badge/Skills-PCAP%20%7C%20Forensics%20%7C%20Reverse%20Engineering%20%7C%20Crypto-blue) ![Status](https://img.shields.io/badge/Status-Completed-brightgreen) ![Spoilers](https://img.shields.io/badge/Spoilers-Full%20flags-orange)

> **Spoiler warning:** This repository documents the complete solutions and flags for the CCT2019 TryHackMe challenge.

## Overview

**CCT2019** originated from the U.S. Navy Cyber Competition Team 2019 Assessment and is available as a community TryHackMe challenge. It combines packet analysis, file recovery, reverse engineering, digital forensics, and classical cryptography. My goal in documenting the room is to organize the techniques into a reproducible study reference and connect offensive CTF skills to practical security analysis.

**Environment:** Authorized TryHackMe challenge only. **Reference:** The supplied `CCT2019-U.S.-Navy-Cyber-Competition-Assessment-main/README.md` write-up. This is my own reorganized study summary based on that source; the original PCAPs, executables, and screenshots were not supplied for independent validation.

## Tools and skills

| Area | Tools / concepts |
| --- | --- |
| Network forensics | Wireshark, tshark, PCAP/PCAPNG, TCP/ICMP, USB capture data |
| File extraction | xxd, binwalk, stegoveritas, steghide, unzip |
| Reverse engineering | Ghidra, dnSpy/ILSpy, .NET decompilation, XOR |
| Cryptography | Keyboard substitution, Rail Fence, Enigma, run-length encoding, Python |
| Investigation | Artifact recovery, protocol analysis, hypothesis testing |

## Task 1 — CCT2019 - pcap1

### Objective

Recover data from network captures and follow the evidence to the flag. The room warns that the **second PCAP contains 4,588 packets**, so complete file recovery is important.

### Investigation workflow

1. Open the provided capture in Wireshark and inspect USB traffic. A useful display filter from the reference is:

   ```wireshark
   usb.device_address == 7 && usb.capdata
   ```

2. Export the relevant payload data with `tshark` and reconstruct it:

   ```bash
   tshark -r pcap2.pcapng -Y 'usb.capdata and usb.device_address==7' -T fields -e usb.capdata > raw.hex
   xxd -r -p raw.hex output.bin
   ```

3. Inspect the reconstructed file using file-carving tools such as `binwalk` or `stegoveritas`. The reference describes recovering another capture, `pcap_chal.pcap`.
4. Inspect ICMP traffic, including the reference filter `frame.len < 98 && icmp`. A conversation points toward *The Net* and the cryptcat key `BER5348833`.
5. Examine the TCP stream on port `4444`, decrypt the cryptcat transfer, and recover an ELF executable.
6. Analyze the binary in Ghidra. The reference describes an IRC interaction with `irc.cct` and the `#flag` channel as the final stage.

### Flag

```text
CCT{h3's_a_pc@p_w1z@rd_th3re_h4s_g0t_to_6e_a_7w1st}
```

**What I learned:** Packet captures can carry embedded files and clues across several protocols. Accurate byte-for-byte recovery matters before attempting later analysis.

## Task 2 — CCT2019 - re3

### Objective

Analyze a .NET GUI executable with four sliders and recover a 32-character hexadecimal flag.

### Analysis

The reference describes four values that must satisfy the following constraints:

- Sum = `711`
- Product = `711000000`
- Strictly descending order

The documented values are `316`, `150`, `125`, and `120`.

The application uses a byte array and XOR-based transformation in the successful `goodBoy()` path. The following is a reproducible calculation from the reference:

```python
byteB = [20, 22, 100, 23, 21, 99, 100, 97, 99, 98, 21, 97, 100, 97, 16, 21,
         16, 23, 22, 17, 98, 21, 102, 16, 23, 18, 19, 101, 17, 99, 102, 18]
c = 177
second_slider = 150
print(''.join(chr(v ^ (c ^ second_slider)) for v in byteB))
```

### Flag

```text
31C02DCFDE2FCF727016E2A7054B6DA5
```

**What I learned:** Reverse engineering can convert a graphical puzzle into constraints that can be reasoned about and checked programmatically.

## Task 3 — CCT2019 - for1

### Objective

Investigate a JPEG file containing layered hidden artifacts, protected archives, and an Enigma-based clue.

### Investigation workflow

1. Inspect JPEG metadata using `exiftool`. The reference identifies Morse code that decodes to `JUSTAWARMUPRIGHT?`.
2. Carve the appended ZIP using `binwalk` or `stegoveritas`; the reference identifies `7212.zip` at offset `0x7212`.
3. Unlock the first archive using `justawarmupright?` and read the *Matrix*-themed hint.
4. Use `Z10N0101` with `steghide` to recover the next archive.
5. Extract the next layer with `0ni0n_0f_0bfu5c@ti0n`. It contains the encrypted text and Enigma configuration.
6. Configure the Enigma machine using the documented settings:

   ```text
   Rotors: Gamma VI VII VIII
   Reflector: C-Thin
   Ring settings: R I N G
   Display: AMTU
   Plugboard: AM BY CH DR EL FX GO IV JN KU PS QT WZ
   Ciphertext: JHSLPGLWYSQODQVLPAOTPCYKPUDTF
   ```

7. The decrypted phrase `ctfforensicsisnotrealforensics` is the final archive password.

### Flag

```text
CCT{Well_that_wasn’t_such_a_chore_now_was_it?}
```

> Note: The flag above preserves the typographic apostrophe found in the supplied reference. If the challenge rejects it, check whether the original flag uses the straight ASCII apostrophe (`'`).

**What I learned:** Forensic analysis benefits from examining metadata, file signatures, embedded archives, and contextual clues instead of assuming one tool will solve the entire challenge.

## Task 4 — CCT2019 - Crypto1

### Objective

Solve three sequential cryptography puzzles. Each completed stage reveals its flag and unlocks the next stage.

### Crypto1a — Keyboard layout substitution

The first stage uses a Dvorak/QWERTY keyboard-layout substitution. The decrypted clue yields the archive password:

```text
dvorakdvorakdvorak
```

**Crypto1a flag:**

```text
CCT{Actu411y_a_w@rmup}
```

### Crypto1b — Rail Fence cipher

The second stage uses a Rail Fence transposition with **five rails**, beginning from the bottom according to the reference. The resulting clue gives the password:

```text
teerrrriiffiicccccc
```

**Crypto1b flag:**

```text
CCT{th@t_w4s_th4_ea5y_bu770n!}
```

### Crypto1c — Binary run-length encoding

The final stage represents a binary message as lengths of alternating runs. The important hint is to **start with 0**, not 1. Each input digit specifies the number of times to repeat the current bit; then switch bits for the next digit. Finally, split the expanded binary string into groups of eight and decode ASCII.

Here is a reusable decoder (paste the digits from your own `crypto1c.txt`):

```python
encoded = input('Enter the run-length digits: ').strip()
if not encoded.isdigit():
    raise ValueError('Expected a string of decimal digits')

bits = ''.join(('0' if i % 2 == 0 else '1') * int(n)
               for i, n in enumerate(encoded))
if len(bits) % 8:
    raise ValueError(f'Expanded bit length {len(bits)} is not divisible by 8')

message = ''.join(chr(int(bits[i:i + 8], 2))
                  for i in range(0, len(bits), 8))
print(message)
```

**Crypto1c flag:**

```text
CCT{I_see_dead_ciphers_all_the_time}
```

**What I learned:** Cryptography puzzles require recognizing the transformation before attempting brute force. The run-length variant is simple once the starting bit and alternating-run rule are understood.

## Flag summary

| Task | Flag |
| --- | --- |
| pcap1 | `CCT{h3's_a_pc@p_w1z@rd_th3re_h4s_g0t_to_6e_a_7w1st}` |
| re3 | `31C02DCFDE2FCF727016E2A7054B6DA5` |
| for1 | `CCT{Well_that_wasn’t_such_a_chore_now_was_it?}` |
| crypto1a | `CCT{Actu411y_a_w@rmup}` |
| crypto1b | `CCT{th@t_w4s_th4_ea5y_bu770n!}` |
| crypto1c | `CCT{I_see_dead_ciphers_all_the_time}` |

## Blue Team takeaways

- **Packet reconstruction:** Validate capture integrity and recovered file sizes before interpreting traffic artifacts.
- **Protocol visibility:** USB, ICMP, TCP, and application-level sessions may each reveal different evidence in an investigation.
- **Artifact triage:** Metadata and appended archives can reveal hidden content that ordinary file viewers miss.
- **Static analysis:** Understanding executable logic helps analysts identify suspicious program behavior without blindly executing binaries.
- **Encoding vs. encryption:** Recognizing simple transformations can speed up malware configuration extraction and incident response.
- **Evidence discipline:** Preserve original files, document commands, and distinguish observed outputs from third-party write-up claims.

## References

- [TryHackMe](https://tryhackme.com/) — CCT2019 challenge.
- Supplied community reference: `CCT2019-U.S.-Navy-Cyber-Competition-Assessment-main/README.md`.
- [Wireshark](https://www.wireshark.org/), [Ghidra](https://ghidra-sre.org/), [CyberChef](https://gchq.github.io/CyberChef/).

---

*Personal cybersecurity learning portfolio — authorized lab research and technical documentation.*
