# Lokibot-Trojan-Decompilation-and-controlled-Debug

How I Caught a Stealthy Dropper 

I had one file: `sample.bin`. Two hashes told the story before I even ran it:

Packed: 9da91175...da622a6
Unpacked AU3 script: 788d259b...e5b6099

It was AutoIt. Old trick, new obfuscation. A giant `$J` blob being XORed with key `LRMKTJVUQI` inside function `EWFUGMIBNE`. One hex string kept bugging me: `4053797374656D44697222`. Decoded it - `binascii.unhexlify` after failing with .NET `FromHexString` - and got `@SystemDir\dllhost.exe`. Classic COM hijack target.

The Hunt

I attached x32dbg. PID 504, Main Thread 3968. [Elevated].

Found it in Symbols - Export 1484, `kernel32!VirtualAlloc @ 75DFF9F0`. Set INT3. Hit it instantly.

The stack was screaming malware:

lpAddress = NULL
dwSize = 00034108 (213KB)
flProtect = 00000040 = PAGE_EXECUTE_READWRITE = RWX
Return = 00DB5310 -> http://sample.bin

No legitimate program needs 213KB of RWX. That's an unpacking stub. I watched the trampoline: `75DFF9F6 FF25 6C13E675 jmp -> KERNELBASE!76986E63` then...

**Boom. EAX = 03000000.**

That's the base. Right there in CPU view - `EIP 00D10CB9 sample.bin:$20CB9, EAX 03000000`. The malware just got its playground.

Proving It Without Touching It

Requirement was clear: prove it writes there without patching code. So I didn't.

Command: bphws 03000000, w, 1
Log: Hardware breakpoint already set!

Non-invasive. Pressed F9. Instant hit:

`WRITE to 03000000`

Caught the decryption loop red-handed, writing the real payload into RWX.

The Grab

While frozen at that breakpoint, I jumped to Process Hacker 2. Memory tab. There it was:

`03000000 - 03034108, Size 00034108, Protection: Execute_ReadWrite, Type: Private`

Saved it. `memdump_03000000_RWX.bin`. Hashed it. Ran strings - `MZ` header, `This program cannot be run in DOS mode.` - that's not AutoIt anymore. That's an unpacked PE ready to inject into dllhost.

Never executed it. Didn't need to.

What It Is

An AutoIt dropper. Allocates RWX, decrypts with `LRMKTJVUQI`, targets `dllhost.exe`. MITRE: T1055, T1027, T1059.006, T1105, T1497.

18 minutes from first hash to final dump. Time Wasted Debugging: `0:00:18:22` - but nothing was wasted.

*Lab: FLARE-VM2 isolated VM | For education only | Evidence in /evidence folder*
