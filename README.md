# CCT2019 TryHackMe Challenge
I have finished the CCT2019 Challenge (Insane level) from TryHackMe which originated from the U.S. Navy Cyber Competition Team 2019 Assessment, and here is the explanation:
#
## Task 1: CCT2019 - pcap1
This task involves analyzing a PCAP file to uncover an embedded challenge leading to an IRC-based flag retrieval. Here's a high-level overview of the solution process:

**[Quick Steps Summary]**
1. Open the PCAP in Wireshark: Filter for USB traffic (usb.device_address == 7 && usb.capdata) to spot bulk transfer data in "Leftover Capture Data."
2. Extract Raw Data: Use tshark to pull hex from relevant USB packets:
   ```text
   tshark -r pcap2.pcapng -Y 'usb.capdata and usb.device_address==7' -T fields -e usb.capdata > raw.hex
   ```
   Convert hex to binary: xxd -r -p raw.hex output.bin.
3. Stego Extraction: Use stegoveritas output.bin (or binwalk) to reveal an embedded PCAP (pcap_chal.pcap).
4. Analyze Embedded PCAP: Filter ICMP packets (frame.len < 98 && icmp) for a conversation hinting at a movie reference (*The Net*) for the cryptcat key (`BER5348833`). Extract and decrypt TCP data on port 4444 using cryptcat to get an ELF binary.
5. Reverse the Binary: Analyze in Ghidra or similar; it connects to an IRC server (irc.cct, #flag channel). Set up a local IRC listener (with irssi or netcat) and run the binary to trigger the flag from the bot.

**[The Flag]**

`CCT{h3's_a_pc@p_w1z@rd_th3re_h4s_g0t_to_6e_a_7w1st}`

This translates to a cheeky leet-speak nod: "There's a PCAP wizard there, has got to be a twist!"
#
## Task 2: CCT2019 - re3
This task is a reverse engineering challenge involving a .NET executable (re3.exe) with a GUI featuring four sliders (0-1024 range). The goal is to set the sliders to specific values that satisfy mathematical conditions, unlocking a 32-character hexadecimal flag via decryption.

**[Quick Steps Summary]**
1. Decompile the Binary: Use a .NET decompiler like dnSpy, ILSpy, or JetBrains dotPeek to analyze re3.exe. Identify the button click event handler in the form class.
2. Understand Conditions: The sliders represent four values (value, value2, value3, value4) that must:
    - Sum to 711.
    - Multiply to 711,000,000.
    - Be in strictly descending order (value > value2 > value3 > value4).

      Failure shows `badBoy`; success calls `goodBoy()` to decrypt and display the flag.
3. Find Slider Values: Factorize 711,000,000 (prime factors: 2<sup>6</sup> x 3<sup>2</sup> x 5<sup>6</sup> x 79). Brute-force combinations of divisors to find quadruplets summing to 711. The solution is 316, 150, 125, 120.
4. Extract and Decrypt Flag: In `goodBoy()`, a byte array is XOR-decrypted using one slider value (150) as the key, combined with constant 177 (and possibly 51). The result is a concatenated string checked for valid 32-char hex. Use this Python snippet to compute:
```python
byteB = [20, 22, 100, 23, 21, 99, 100, 97, 99, 98, 21, 97, 100, 97, 16, 21, 16, 23, 22, 17, 98, 21, 102, 16, 23, 18, 19, 101, 17, 99, 102, 18]
c = 177
i = 150  # Second slider value
key = ''.join(chr(v ^ (c ^ i)) for v in byteB)
print(key)  # Outputs the hex flag
```
5. Verify: Set sliders to 316 (bar1), 150 (bar2), 125 (bar3), 120 (bar4). Run the exe; it displays the flag on success.

**[The Flag]**

`31C02DCFDE2FCF727016E2A7054B6DA5`

This hex string is the direct output; no further decoding needed, per the challenge hint.
#
## Task 3: CCT2019 - for1
This forensics task revolves around steganography layers in a JPEG image (for1_8f90d68390b565c308871a52c6572de8.jpg), leading to an Enigma-decrypted password for the final flag ZIP. It's a nod to *The Matrix* with escalating passwords and a cheeky meta-comment on CTF forensics.

**[Quick Steps Summary]**
1. Initial Analysis: Download the file (a 514x480 JPEG). Use exiftool to extract metadata, revealing Morse code in the Description: . — — ..- … — .- . — .- .-. — ..- . — . .-. .. — . …. — .. — .. (decodes to `JUSTAWARMUPRIGHT?`).
2. Stego Carving: Run stegoveritas (or binwalk) on the image to extract a trailing ZIP at offset 0x7212: 7212.zip (password-protected).
3. Unzip First Layer: Use password justawarmupright? to extract fakeflag.txt, which hints at *The Matrix* (`Peer into the Matrix... - Morpheus`) and provides next password: `Z10N0101`.
4. Deeper Stego: Use steghide on the carved for1.jpg with `Z10N0101` to extract archive.zip (password-protected).
5. Unzip Second Layer: Use password `0ni0n_0f_0bfu5c@ti0n` (from prior challenge or hint) to get:
    - cipher.txt: Ciphertext JHSL PGLW YSQO DQVL PFAO TPCY KPUD TF.
    - config.txt: Enigma settings ("C G. VI. VII. VIII. AMTU RING AM BY CH DR EL FX GO IV JN KU PS QT WZ").
    - flag.zip (password-protected).
6. Decrypt with Enigma: Use a Python script with py-enigma library (or online Enigma simulator) configured per config.txt:
```python
from enigma.machine import EnigmaMachine
import enigma.plugboard
enigma.plugboard.MAX_PAIRS=13
machine = EnigmaMachine.from_key_sheet(
    rotors='Gamma VI VII VIII',
    reflector='C-Thin',
    ring_settings='R I N G',
    plugboard_settings="AM BY CH DR EL FX GO IV JN KU PS QT WZ")
machine.set_display('AMTU')
ciphertext = 'JHSLPGLWYSQODQVLPAOTPCYKPUDTF'
plaintext = machine.process_text(ciphertext).lower()
print(plaintext)  # Outputs: ctfforensicsisnotrealforensics
```
  This yields the final password: `ctfforensicsisnotrealforensics`.

7. Final Unzip: Extract flag.zip with the password to get flag.txt containing the flag.

**[The Flag]**

`CCT{Well_that_wasn’t_such_a_chore_now_was_it?}`

A fun twist on the "easy" first forensics task; turns out it was a chore! 
#
## Task 4: CCT2019 - Crypto1
Task 4 encompasses this entire "crypto1" section, where each part unlocks the next via a password-protected ZIP file. The ultimate goal is to decrypt and decode the content to reveal the final flag.

**[Quick Overview of Steps]**
1. Crypto1a (Keyboard Layout Substitution Cipher):
     - Download `crypto1.zip` and extract `crypto1a.txt`.
     - Decrypt the ciphertext using a Dvorak-to-QWERTY keyboard layout shift (via dcode.fr tool).
     - Clean up punctuation/symbols to reveal the hint: The key is "dvorak" repeated three times (`dvorakdvorakdvorak`).
     - Use this to unzip `crypto1a.zip` and get the first flag: `CCT{Actu411y_a_w@rmup}`.

2. Crypto1b (Rail Fence Transposition Cipher):
     - From the previous unzip, access `crypto1b.txt`.
     - Decrypt using a rail fence cipher with 5 rails, starting from the bottom (e.g., via dcode.fr or a custom Python script).
     - The plaintext hints at a movie misspelling: "teerrrriiffiicccccc" (six 'c's, from Charlotte's Web goose scene).
     - Use this to unzip `crypto1b.zip` and get the second flag: `CCT{th@t_w4s_th4_ea5y_bu770n!}`.

3. Crypto1c (Run-Length Encoded Binary):
     - From the previous unzip, access `crypto1c.txt` (a string of digits 1-6).
     - Interpret as RLE for alternating 0s/1s (starting with 0): Even indices = 0s, odd = 1s, repeated by digit value.
     - Convert the resulting binary string to ASCII (via Python or CyberChef).
#
# Final Flag 
`CCT{I_see_dead_ciphers_all_the_time}`

**This flag completes Task 4**











