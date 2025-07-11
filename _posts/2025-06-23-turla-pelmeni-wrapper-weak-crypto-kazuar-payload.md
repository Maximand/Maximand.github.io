---
layout: article
title: "Turla’s Pelmeni Wrapper: How Weak Crypto Exposed Kazuar’s Payload"
tags:
  - Observations on Threat Intelligence
mathjax: true
mathjax_autoNumber: true
author: Max van der Horst
show_author_profile: true
show_edit_on_github: false
excerpt: "Turla’s Pelmeni Wrapper reveals how flawed cryptographic choices in malware design can create tracking opportunities. This post dissects how a weak and predictable pseudorandom generator exposed Kazuar’s payload and what this means for threat intelligence."
header:
  theme: dark
article_header:
  type: cover
  image:
    src: /assets/cassowary.png
  og_image:
    src: /assets/cassowary.png
---

> **Summary:**  
> This post analyzes a cryptographic flaw in Pelmeni, a custom loader used by the Turla APT group to deploy Kazuar. While the malware uses environmental keying based on the victim’s host name, it relies on a weak pseudorandom number generator (Ranqd1) and reuses it to decrypt both export names and the embedded payload. By exploiting this weakness, we can recover the decryption seed and extract Kazuar without access to the victim environment. The post provides a reproducible method to identify, decrypt, and track such samples in the wild.

Last year, I came across a great [blog post](https://lab52.io/blog/pelmeni-wrapper-new-wrapper-of-kazuar-turla-backdoor/) by Lab52 analyzing a new wrapper for the Kazuar malware: *Pelmeni*. Their breakdown showed how the wrapper encrypts the Kazuar payload and ties decryption to the victim’s computer name. This ensures the malware only runs in the intended environment. That design raised some interesting questions: Was Kazuar used post-compromise for persistence? What does the full infection chain look like?

This blog does not aim to answer those broader questions. Instead, it focuses on a specific and previously undocumented weakness in Pelmeni’s cryptographic design. Lab52 noted that *the algorithm used to decrypt the payload and the one used to decrypt the export names is the same, which makes it vulnerable to brute-force attacks,* but they did not explain how exactly. Based on their findings, and on additional details from [Unit 42’s analysis](https://unit42.paloaltonetworks.com/pensive-ursa-uses-upgraded-kazuar-backdoor/), this post explains how that vulnerability works and why the cryptographic primitive used in Pelmeni can be exploited.

# What is Pelmeni?
Pelmeni (Russian: *пельмени*, English: dumpling) is the name given by Lab52 to a wrapper used to drop recent variants of the Kazuar (*казуар*, cassowary) backdoor, which has been linked to Turla (also known as Secret Blizzard, Snake, Uroburos, or Venomous Bear), a threat actor that is allegedly linked to the Russian Federal Security Service (FSB). Rather than deploying Kazuar directly, Pelmeni is used as an intermediary wrapper to embed Kazuar as an encrypted payload. Upon execution, Pelmeni decrypts the payload and transfers execution to the decrypted binary. Moreover, even though Pelmeni is the dropper, it is itself a Dynamic Link Library (DLL) that is executed through the use of DLL Sideloading vulnerabilities in legitimate software from organizations like Nvidia, Brother, Intel, and others.
 
What makes Pelmeni noteworthy is its use of environmental keying. The decryption key for Kazuar is derived from the victim's host name. This means that the payload will only decrypt correctly when run on the specific machine it was intended for. This kind of environment binding can serve both as an evasion technique and an operational safeguard.

This approach reflects Turla’s broader pattern: they often rely on tailored malware loaders, multi-stage payload delivery, and environmental checks to ensure stealth and resilience. Their operations typically prioritize long-term access and espionage, with custom tooling that is designed to persist in specific, high-value environments.

In addition to the encrypted payload, Pelmeni contains (randomly generated) export names that it decrypts during execution. These are typically function names that might be used to identify or interact with the payload. According to Lab52, the same algorithm is used to decrypt both the export names and the payload. This shared mechanism turns out to be the weak point in the entire design. 

# Pelmeni's cryptographic primitives
As mentioned before, an interesting aspect of the Pelmeni wrapper (sha256:55580aedc4429496ef13080545f2ec6014e4e0f0774918dfa7fd12f048f2bdd2 in this post, but generally applicable) is the fact that its export names are decrypted during execution. The Lab52 blogpost describes that the second entrypoint of the DLL is responsible for executing one of the exported DLL functions. The decryption key to decrypt the export name is generated by providing the (hashed) hostname as a seed for a pseudorandom number generator (PRNG) called Ranqd1, which is a [Linear Congruential Generator (LCG)](https://en.wikipedia.org/wiki/Linear_congruential_generator). This PRNG was recognized by the constants that are, among the rest of the operation, shown in Figure 1 and generates a keystream per four bytes (hence the modulo 32).  

![](../../../assets/entrypoint_2_pelmeni.png){: .centered-img }
<span class="centered-text">Figure 1: Second entrypoint of Pelmeni showing generation of decryption key</span>

The algorithm that was used to turn the computername into a reliable seed is called [Jenkins' one at a time](https://www.drdobbs.com/database/algorithm-alley/184410284), which is part of a non-cryptographic hash function family designed by Bob Jenkins. While known to contain some weak mixing behavior using the avalanche effect over 3-byte keys and commonly having collisions due to the small hash size (8 bytes), it is considered to have preimage resistance at the moment of writing. Unfortunately, this means that brute forcing for the victim's hostname for victimology purposes may be infeasible. An implementation of Jenkins' one at a time is shown in Listing 1.

```c
uint32_t jenkins_one_at_a_time_hash(const uint8_t* key, size_t length) {
  size_t i = 0;
  uint32_t hash = 0;
  while (i != length) {
    hash += key[i++];
    hash += hash << 10;
    hash ^= hash >> 6;
  }
  hash += hash << 3;
  hash ^= hash >> 11;
  hash += hash << 15;
  return hash;
}
```
<span class="centered-text">Listing 1: Jenkins' one at a time implementation in C</span>

In summary: the victim host name is hashed using Jenkins' one at a time, that hash is XOR'ed with a hardcoded constant (`0x38a55104` in this case) and fed into the Ranqd1 PRNG algorithm. The outcome is the key that is used to decrypt the export names during execution. The process is also shown in Figure 2.

At the same time, as also shown in the Lab52 blogpost, the exact same constants and algorithm return right before dropping Kazuar, when Pelmeni has loaded the thread that is supposed to decrypt the payload. This gives away that the decryption routine for the export names and Kazuar itself are the same. 

![](../../../assets/encryption_scheme_pelmeni.png){: .centered-img }
<span class="centered-text">Figure 2: Pelmeni encryption routine</span>

# Weakness in key derivation
As mentioned, the core cryptographic weakness in Pelmeni stems from the reuse of its decryption logic across both the Kazuar payload and the export function names. This shared mechanism enables recovering the seed that can be used to decrypt Kazuar. Specifically the use of the LCG, which is a relatively weak PRNG algorithm, allows for recovering a keystream that can decrypt both the export names and the payload. 

Looking at the PRNG in more detail, it is defined as follows:

```c
// Provided parameter is the seed
uint32_t rand_nsmb(uint32_t *state){
    uint64_t value = (uint64_t)(*state) * 1664525 + 1013904223;
    return *state = value + (value >> 32);
}
```
<span class="centered-text">Listing 2: ranqd1 implementation in C</span>

While the XOR operations and processing seem to provide some obfuscation at first, its entire security hinges on the secrecy of the seed. Due to the fact that we know that the routine is based on XOR operations and we have access to both the ciphertext and plaintext of the export function names, we can launch a known plaintext attack to obtain the original seed.

This can be done by taking the first four bytes of the ciphertext and plaintext and XOR'ing those to get the keystream used for the export function name. Then, to obtain the seed, we can reverse the LCG with the known constants. 

The process is shown in Figure 3. Where the original keystream is calculated by solving for $$keystream = (seed * a + c) \mod 2^{32}$$, the seed can be calculated in a similar fashion by solving for $$seed = (keystream - c) * a^{-1} \mod 2^{32}$$. The important detail here is the use of the first constant: $$a$$. To reverse the LCG, we have to calculate the multiplicative inverse of this value, which is 1664525, alongside the modulus, which is 32. This leaves us with the number 5, which allows us to start testing the decryption of export function names.

![](../../../assets/seed_recovery_pelmeni.png){: .centered-img }
<span class="centered-text">Figure 3: Recovering the original seed</span>

# Brute-forcing the seed in practice
To reproduce this decryption process without knowing the victim's host name, we can now calculate the original seed produced by Jenkins' one at a time that was XOR'ed with the earlier found constant. But first, we need to extract the right data from the binary itself. We know that, on multiple locations in the binary, Pelmeni decrypts export function names. This behavior can be identified in the binary in multiple places, but always with the pattern shown in Listing 3.

```asm
703c15c0  c744240406000000   mov     dword [esp+0x4], 0x6       \\ Pushes string size onto stack at [esp+0x4]
703c15c8  c7042498413e70     mov     dword [esp], 0x703e4198    \\ Pushes RVA of string to [esp]
```
<span class="centered-text">Listing 3: Instructions prior to decrypting export function name</span>

The string size varies, as the export names are randomly generated per sample. Therefore, if we want to find the original seed of any sample, we will have to write a regular expression for both the Relative Virtual Address (RVA) and actual string size (6 bytes in the case shown in Listing 3). We can do this as follows.

```python
import re, struct, pefile
binary_loc = "./55580aedc4429496ef13080545f2ec6014e4e0f0774918dfa7fd12f048f2bdd2"
with open(binary_loc, "rb") as file:
    bin = file.read()

regex = re.compile(
    rb"\x85\xC0\xC7\x44\x24\x04(?P<exportname_size>....)" 
    rb"\xC7\x04\x24(?P<exportname_rva>.{4})",
    re.DOTALL
)
export_name_encrypted = regex.search(bin)
export_name_size = struct.unpack("<I", export_name_encrypted.group("exportname_size"))[0]
export_name_rva = struct.unpack("<I", export_name_encrypted.group("exportname_rva"))[0]
pe = pefile.PE(binary_loc)
export_name_rva = export_name_rva - pe.OPTIONAL_HEADER.ImageBase
file_offset = pe.get_offset_from_rva(export_name_rva)

final_encrypted_string = bin[file_offset:file_offset+export_name_size]
```
<span class="centered-text">Listing 4: Retrieving encrypted export names</span>

Now that we have one of the encrypted export names, we want to extract all the export names from the binary. With the ciphertext and plaintext versions of the export name, we can construct our known plaintext attack to retrieve the seed. We can obtain seed candidates by reversing the LCG with the earlier described method and attempting to decrypt the ciphertext with the seed candidate. The ranqd1 constants seem to be consistent across samples, so for now we can keep these hardcoded. When the decryption routine with a seed candidate results in a known export name, we know we have the correct and original seed. 

```python
def reverse_lcg(candidate) -> int:
    a, c, modulus = 0x19660D, 0x3C6EF35F, 2**32
    a_mult_inv = pow(a, -1, modulo)                      # Compute multiplicative inverse
    return (a_mult_inv * (candidate - c)) % modulus # Reverse LCG formula

def decrypt(ciphertext, candidate) -> bytes:
    a, c, modulus = 0x19660D, 0x3C6EF35F, 2**32
    decrypted = bytearray()
    for i in range(len(ciphertext)):                # Reconstruct LCG routine
        if i & 3 == 0:
            candidate = (candidate * a + c) % modulus
        lcg_bytes = struct.pack(">I", candidate)[::-1]
        decrypted.append(ciphertext[i] ^ lcg_bytes[i % 4])
    return bytes(decrypted)

# Extract plaintext export names
pe_bin = pefile.PE(binary_loc)
pe_bin.parse_data_directories(directories=[pefile.DIRECTORY_ENTRY["IMAGE_DIRECTORY_ENTRY_EXPORT"]])
export_names = []
for symbol in pe.DIRECTORY_ENTRY_EXPORT.symbols:
    identifier = symbol.ordinal
    name = symbol.name
    export_names.append((identifier, name))

# Start operating on earlier found ciphertext in order to rebuild key candidate
for identifier, name in export_names:
    name += b"\x00"                                  # Decrypted string ends with \x00
    key_attempt = bytearray()
    ciphertext_bytes = final_encrypted_string[:4]
    plaintext_bytes = name[:4]
    for index in range(4):
        key_attempt.append(ciphertext_bytes[index] ^ plaintext_bytes[index])
    
    # Reverse LCG on key candidate
    key_attempt = struct.unpack("<I", key_attempt)[0]
    candidate = reverse_lcg(key_attempt)
    print(f"Attempting decryption with {hex(candidate)} against export name {name}")
    if decrypt(final_encrypted_string, candidate):   # Seed found
        seed = hex(candidate)
        print(f"[!] Seed {candidate} found to be correct.")
        break
```
<span class="centered-text">Listing 5: Executing a known plaintext attack on the seed</span>

The terminal output in Listing 6 shows the correct seed for our sample, which was identified as `0xa7655e3f` matching on the export name `Shgyo`. This is also in line with the length of the encrypted export name that we found using the regular expression, as it is 6 bytes long when a nullbyte is appended. With this information, it is time to start seeking out the encrypted Kazuar sample for decryption. 

```
$ python3 decrypt.py
Attempting decryption with 0x44594765 against export name b'Aoimzajs@0\x00'
Attempting decryption with 0x91fde603 against export name b'Gmlkwiqm@4\x00'
Attempting decryption with 0x4261dddc against export name b'Jagiwl@4\x00'
Attempting decryption with 0xc671d3a1 against export name b'Mfnxhg\x00'
Attempting decryption with 0x29e60da1 against export name b'Mtkvsmmq@4\x00'
Attempting decryption with 0x8dc2197a against export name b'Pduouohn@0\x00'
Attempting decryption with 0x4cb5a27a against export name b'Pqasrqht@4\x00'
Attempting decryption with 0x332671b5 against export name b'Qirlbb\x00'
Attempting decryption with 0xa7655e3f against export name b'Shgyo\x00'
[!] Seed 0xa7655e3f found to be correct.
```
<span class="centered-text">Listing 6: Terminal output of the script</span>

# Decrypting Kazuar
The Lab52 blog post already mentioned some hints on where to find the encrypted payload. We have earlier confirmed that the decryption algorithm for the export names and Kazuar itself is the exact same. Hence, we can search for occurrence of the ranqd1 constants throughout the binary. Moreover, the blog post describes the preparation of a thread that decrypts Kazuar and shows a screenshot of what this code looks like. After searching the binary for the ranqd1 constants, the content of Figure 4 showed up. This assembly (and the belonging pseudocode, looking like Figure 1) gives three hints: 
* The address of the buffer containing Kazuar is `0x703e44e0`
* The LCG routine starts on `0x703e0d45` and performs the XOR operation with the LCG byte on `0x703e0d85`
* The size of the Kazuar buffer is included on `0x703e0d95`, showing that the payload has an allocated size of `0x2411ff` (~2.3mb)

This allows us to write a second regular expression that will allow us to extract and decrypt the Kazuar payload.

![](../../../assets/locating_kazuar.png){: .centered-img }
<span class="centered-text">Figure 4: Locating the Kazuar payload</span>

To make the decryption routine work independently of the sample being analyzed, we need two key details from the binary: the location of the payload and its size. We've already identified these values, now we need to extract them. Examining the surrounding opcodes preceding the payload location, there is a consistent instruction sequence:

* `mov` (`0x8B`),
* `add` (`0x05`), and
* `movzx` (`0x0F`).

Similarly, the size value is loaded in a loop whose upper bound is defined by:

* `cmp [ebp-0xC]` (`0x817DF4`)
* followed by a conditional jump `jbe` (`0x76`),
* and later on, a `call` instruction (`0xE8`).

Between the `jbe` and `call` instructions, there is one variable byte that is skipped over. We do need the `call` instruction as this makes the sequence specific enough to use. This will lead to the following regular expression and extraction method.

```python
payload_location = b"\x8B\x45\xF4\x05(?P<payload>....)\x0F"
payload_size = b"\x81\x7D\xF4(?P<payload_size>....)\x76.\xE8"
match_location = re.search(payload_location, bin)
match_size = re.search(payload_size, bin)

kazuar_address = struct.unpack("<I", match_location.group("payload"))[0]
kazuar_size = struct.unpack("<I", match_size.group("payload_size"))[0]
kazuar_rva = kazuar_address - pe.OPTIONAL_HEADER.ImageBase
kazuar_file_offset = pe.get_offset_from_rva(kazuar_rva)
print(f"Extracted Kazuar address {kazuar_address} and payload size {kazuar_size}")

# Start decryption of Kazuar
decrypted = decrypt(bin[kazuar_file_offset:kazuar_file_offset+kazuar_size], seed)
if decrypted.startswith(b"\x4d\x5a\x90\x00"):   # If extract starts with MZ header, encryption succeeded
    with open(f"kazuar_extract_{seed}.exe", "wb") as f:
        file.write(decrypted)
        print(f"Wrote decrypted Kazuar payload for seed {seed}")
```
<span class="centered-text">Listing 7: Extracting and decrypting Kazuar</span>

As can be seen from Listing 7, we are ready to start extracting and decrypting Kazuar payloads. Table 1 shows the results of running this script against all samples of Pelmeni I could find. I will include the file hashes, seeds, resulting Kazuar file hash, and Kazuar file size in the table (which will immediately also be the IoC list for this post). At first sight, there are two different variants based on the large deviation in size.

<span class="centered-text">Table 1: Details about various Pelmeni and embedded Kazuar samples</span>

|---
| Pelmeni Hash (SHA256) | Seed | Kazuar Hash (SHA256) | Kazuar File Size
|-|-|-|-
| 00256c7fd9a36c6a4805c467b15b3a72dbac2e6dbd12abe7d768f20ce6c8f09f | `0xf1784656` | c41b19a5c5f1fee9f9d7a0a943bec10b8838ff1862b8776c45ee6802d19bfae8 | 2.4mb
| 15f5e4808549ff67a79f84e23659da912ebbc1dc7c7b100c12b72384a27e412a | `0xecc08599` | 802e3b92dedcf36df2a3e4032ead5190a0ea856b8bb555f8c5e0e7389076fce2 | 2.4mb
| 174b10038dfa52b214e79c950bfdb67fe7489c0c8c06a1af83f96f1e1c6a996c | `0x7f6eefab` | ae4b2c3eb82bcfdac52d15c521a0b397c067f66f77be0c8a004b1ff824a3715e | 1.7mb
| 18cb3a47679f9c1e5e57d3e8c0d70b521c30227c77705fe20c0eb2057a278ce4 | `0xa77e7107` | c1db26f15d5c3721e45dc8f5a37528e731f0ca4b32ada0b26980e786ced5d5ab | 2.4mb
| 2164d54c415b48e906ad972a14d45c82af7cab814c6cf11729a994249690ed97 | `0xe257eb0e` | 2352d77bbd985eea66bfd02bb10a86a171505f981f022bc4451aed1b3cc3705b | 1.7mb
| 55580aedc4429496ef13080545f2ec6014e4e0f0774918dfa7fd12f048f2bdd2 | `0xa7655e3f` | 0fd34087b765db5da47c24b5333ddb63ecae0024977b09d9dcd4d2812eeac481 | 1.7mb
| 606d19b0ebc8fb039d6c7daccf31b62983dce3a3c517ffbede6a8b620930861e | `0xfc98a4db` | 69a23efb7f0990c1d4e6a594c9992e318a0e557f2d4e79d04ad3db47d82f45da | 2.4mb
| 7d4c1883c4b61ca5c4059f13d5e2c74ed3de728b53d464036c6dbd7828f23c2c | `0xd9ab6d40` | 70848496a300b977cba9a7bdaf89f573dc3f2e049d0ef3a5717abade085d52f5 | 1.7mb
| 9b97e740b65bc609210f095cd9407c990a9f71f580f001ea07300228c5256d62 | `0x741e02cd` | 2a1269a376cbb05c6449ae5c0079c3a59d10bd9670bbb2500f1fc63dda567067 | 2.4mb
| ac10acd6391e18cbbf2c11198175959caa7adfa15144126f1ce9aa3c9b0f10e4 | `0xd03a4b24` | e3362c7509ad77c6f3ec5ffd026f9d6bd3e8e43559c05e10d3060ef75b1c7f5e | 1.7mb
| c1fdbf5a33a0288065d99082bde39a9401bc3ab346fdef276bde746e15a889cb | `0xfbb7f137` | 3539f53ac43ac6b8392d1663070772077fe910da69c2a914bc917fd41ec04f4c | 1.7mb
| cccd6327dd5beee19cc3744b40f954c84ab016564b896c257f6871043a21cf0a | `0x4ba34f4c` | c8439ce0c1fd34bf56ff022e2b2640901daa61f7b20a8593cab60f96461f1b18 | 2.4mb
| d577e82aec669ab2817c135413e16acd3b871568de04ac8eb65a6dbcb5c7048f | `0xcb6f7d6` | 4111bdb0bc7a39dd10f2a552f0b481f2636079501b703e2e7d28421d76a3fe9f | 2.4mb
| ebf10222bdd19bd8f14b7e94694c1534d4fe1d1047034aee7ffe9492cad4a92f | `0x7cc6d6a6` | 046fc6ee4bfc51d12ab7b7a643111076744a898fb21011a4ef4a19eefdb152b1 | 1.7mb
| f6b7b3072b3e343c246057c2518693d353c72ee62f9d9aaaa9431af3ce621568 | `0x61f8817c` | 41b2449673d811d78d3ccfa354ccb2f19d8ad03f846d3797caf5cbabedc0417c | 2.4mb
| d216d3a993459cd48ef46afc64c9c341e143051cbfb77727cc7ad71d1eaef84f | `0xa7655e3f` | e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 | 2.4mb
|---

All Kazuar samples extracted from these Pelmeni samples are valid .NET binaries and are consistent with [Unit42's analysis](https://unit42.paloaltonetworks.com/pensive-ursa-uses-upgraded-kazuar-backdoor/). The metadata of the resulting files also show the altered timestamp and the Kazuar binaries can be decompiled using tools like ILSpy.

```
$ file kazuar_extract_0xa7655e3f.exe
kazuar_extract_0xa7655e3f.exe: PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows
```
<span class="centered-text">Listing 8: Kazuar file information</span>

# How to proceed?
The Pelmeni wrapper demonstrates that even advanced actors like Turla can weaken their own operations through poor cryptographic design. Environmental keying, such as using the victim’s host name to derive the decryption key, is a clever operational safeguard. However, reusing a weak pseudorandom number generator across both export name and payload decryption made the entire scheme vulnerable.

The analysis of the decrypted Kazuar payloads, including command and control infrastructure, is already well documented in the Lab52 post and other sources. This blog focused instead on making the vulnerability in Pelmeni’s cryptographic routine actionable. The ability to recover the seed without needing access to the victim environment provides a practical tool for tracking and analyzing samples in the wild.

This case reinforces a broader lesson in malware analysis: even small implementation flaws in custom cryptographic code can create valuable openings for visibility and intelligence.

If you are a reverse engineer, threat analyst, or incident responder, I hope this post helped you in your work or provided a new perspective. If you come across a new variant or a similar loader with weak crypto, feel free to get in touch.