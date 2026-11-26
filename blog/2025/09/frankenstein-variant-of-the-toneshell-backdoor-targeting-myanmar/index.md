# Frankenstein Variant of the ToneShell Backdoor Targeting Myanmar

> A new variant of Mustang Panda’s ToneShell backdoor targeting Myanmar uses anti-analysis stalling, API hashing and updated GUID creation. Includes new IoCs.

- Source: https://research.intezer.com/blog/2025/09/frankenstein-variant-of-the-toneshell-backdoor-targeting-myanmar/
- Published: 2025-09-10
- Authors: Nicole Fishbein
- Tags: apt, china, threat_bulletin
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


ToneShell is a lightweight backdoor tied to the China-nexus group Mustang
Panda. Typically delivered via DLL sideloading inside compressed archives with
legitimate signed executables and often spread through cloud-hosted lures.
[Zscaler’s](https://www.zscaler.com/blogs/security-research/latest-mustang-panda-arsenal-toneshell-and-starproxy-p1)
2025 analysis described updates to its FakeTLS C2 (shifting from TLS 1.2- to
1.3-style headers), use of GUID-based host IDs, a rolling-XOR scheme, and a
minimal command set for file staging and interactive shell access. Notably,
some of this activity was observed in Myanmar, a region of strategic importance
to China. Targeting Myanmar is particularly interesting as it reflects China’s
broader geopolitical interests, spanning border security, infrastructure
projects, and political developments, and highlights how cyber operations are
leveraged to maintain influence in neighboring states.

This blog is a technical analysis of another variant of the backdoor. While
this variant does not introduce major new features, it is worth highlighting
the anti-analysis techniques it employs and the new indicators that can support
threat hunting and detection. In addition, the continued targeting of Myanmar
underscores China’s sustained use of cyber operations to maintain its interests
abroad.

## Technical Analysis

As previously documented, ToneShell is a dll that is sideloded by a legitimate
executable, usually bundled in the archive file. In this case it is a ZIP file,
named: TNLA နှင့် အခြားတော်လှန်ရေးအင်အားစုမျာ. Which translate to: TNLA and
other revolutionary forces.

sha256: `1272a0853651069ed4dc505007e8525f99e1454f9e033bcc2e58d60fdafa4f0`

The backdoor is a DLL named: SkinH.dll, compiled on 2025-07-14 22:38:08.

sha256:
[e7b29611c789a6225aebbc9fee3710a57b51537693cb2ec16e2177c22392b546](https://analyze.intezer.com/files/e7b29611c789a6225aebbc9fee3710a57b51537693cb2ec16e2177c22392b546)

![Showing amount shared code with other ToneShell samples](images/fig1.png "Code reuse with ToneShell")

The backdoor first checks the current module path for the string
`Google\DriveFS\Scratch`. If not found, it enumerates processes to identify
the parent process, retrieves its image path, and verifies whether it contains
**GoogleDrive**. If found, the backdoor will terminate its execution. This
check might be an attempt by the threat actor to prevent infecting themselves.
If this validation fails, the code enforces a single-instance policy using a
named mutex, `Global\SingleCorporation12AD8B`. It attempts to create the
mutex, storing the handle globally and setting a flag to indicate success or
failure. If creation fails or the mutex already exists, it releases any handles
and marks it as not created. Next, the backdoor checks its persistence
location. It compares its module path with the user profile directory
(CSIDL_APPDATA, 0x1A). If the binary is not under the user profile, it copies
itself and the following files: msvcr100.dll, msvcp100.dll, mfc100.dll to a new
folder it creates. The new directory name consists of a 6-character random
uppercase folder name (e.g., `C:\Users\<user>\AppData\<random-6-chars>`).

![](images/fig2.png "The function that copies the DLLs to a new folder in AppData")

Once installed, it attempts to spin up the Task Scheduler COM service, connect
to the local scheduler, open the root folder (`"\"`) and obtain an
IRegisteredTask for a task named dokanctl. If the task doesn’t exist, it will
create it using RegisterTaskDefinition. The task is set to execute every
minute, it sets the execution path to the folder in AppData created earlier:
`%APPDATA%\<random-6-chars>\svchosts.exe`, and sets it as the action Path. The
task is registered in the root folder with the name `dokanctl`.

![](images/fig3.png "Task creation")

If the backdoor is already under the user profile, it performs junk arithmetic
before entering its C2 loop, followed by a bounded timing/zero-check loop that
appears to serve as anti-analysis noise without affecting the main flow.

### API Function Hashing

The malware uses a custom API-resolving scheme based on a simple rolling hash,
applying it to module and function names instead of storing them in cleartext.
This technique, previously analyzed in previous blogs, is a common evasion
method seen in different malware types. With the
[HashDB project](https://github.com/OALabs/hashdb/blob/main/algorithms/tonepipeshell.py),
resolving these hashes back to the original imported functions is
straightforward, making analysis and detection easier.

![](images/fig4.png "Hash calculation")

![](images/fig5.png "Resolved Windows API functions")

### Anti-Analysis and Anti-Sandboxing

This ToolShell variant employs several stalling and anti-sandboxing tricks
designed to waste time, confuse automated analysis, and evade lightweight
sandboxes. These include:

- Repeated file churn: creating, writing, closing, and deleting a temporary
  file in a loop with Sleep(100) delays, which burns execution time and
  stresses filesystem emulation.
- Randomized sleep loops: several loops sleep for pseudo-random intervals
  ranging from ~800 ms to over a second per iteration, accumulating 20+ seconds
  of startup delay before meaningful behavior begins.
- Tick count checks: uses GetTickCount64() combined with jittered sleeps,
  waiting until at least 10 seconds of wall-clock time has elapsed. This
  ensures that emulators that don’t advance the system clock realistically can
  get stuck.
- Opaque string comparisons: slides across a large embedded wide-string buffer,
  repeatedly calling lstrcmpW() between overlapping substrings until a break
  condition. The comparison results are discarded; the loop simply consumes
  cycles and obfuscates control flow.
- Nonsense randomization: calculates values like 0xBABE or 0xCAFE via contrived
  branches using rand(). These values are not security checks, but serve as
  junk arithmetic to make the code harder to follow.
- Decoy API use: occasionally calls APIs such as OutputDebugStringA and sets
  LastError to unusual constants (0xBADF00D), further muddying behavior without
  altering core logic.

![](images/fig6.png "File creation loops")

Notably, the large embedded string buffers used in the comparison loop contain
text copied from
[OpenAI’s blog](https://openai.com/index/introducing-4o-image-generation/#:~:text=From%20the%20first%20cave%20paintings%20to%20modern%20infographics%2C%20humans%20have%20used%20visual%20imagery%20to%20communicate%2C%20persuade%2C%20and%20analyze%E2%80%94not%20just%20to%20decorate.)
on image generation and from [Pega AI’s](https://www.pega.com/ai-innovation)
website. Pega AI refers to the artificial intelligence features integrated into
the Pega platform, a software suite for workflow automation and customer
relationship management. The backdoor does not use this content functionally;
instead, it serves as filler to inflate the binary and supply meaningless
strings for comparison.

![](images/fig7.png "Text taken from PegaAI’s website")

![](images/fig8.png "Text taken from OpenAI’s website")

Together, these techniques do not detect debuggers directly; instead, they
stall execution, pollute control flow with junk work, and frustrate automated
sandboxing systems that expect short-lived, straightforward malware runs.

### GUID Creation

Similar to earlier versions, this variant of ToneShell generates a unique
identifier for each infected machine. It first attempts to check if one was
already created by reading 16 bytes from `C:\ProgramData\SystemRuntimeLag.inc`.
The malware creates a new seed if the file is missing, unreadable, or does not
provide exactly 16 bytes. The primary method is to generate a GUID via
CoCreateGuid and use it as the seed. If that fails, it falls back to producing
16 bytes using an internal linear congruential generator (LCG) seeded from the
current PRNG state. Once a new seed is generated, it is written back to the
designated file path to ensure persistence across executions.

This logic is similar to Zscelar’s blog about the second backdoor variant.

### Networking

The malware sets up a socket with a C2 at the address:

`146.70.29[.]229:443`

The malware’s network protocol wraps its messages in a TLS-like record header
to blend in with legitimate traffic. Each packet begins with the fixed bytes 17
03 03 (TLS 1.2 Application Data) followed by a two-byte length field, but only
the low byte is honored, effectively capping payloads at 255 bytes. The 5-byte
header is read into a temporary stack buffer and discarded; only the payload
bytes are stored in the context structure’s receive buffer starting at offset
0\.

Once the payload is received, the malware XOR-decodes it in place with a
256-byte rolling key. The first decoded byte is treated as a “type/status”
field, the second as an additional code, and the remainder (len-2 bytes) as the
message body. The decoded buffer pointer is then saved in the structure for
later use. This design produces a simple packet format: \[TLS-like
header\][XOR-obfuscated type | code | body…], with the header stripped before
the data is available to the rest of the malware.

Interestingly, while the GUID generation logic in this sample aligns with the
first version of ToneShell described by Zscaler, the communication protocol
remains identical to the second version. Combined with the anti-analysis
techniques and the use of text copied from AI platforms, this sample stands out
as a hybrid variant.

### Pivoting on The C2 Address

Pivoting on the C2 address, we were also able to find another variant:

sha256: `a58868b3d50b775de99278eeb14da8b7409b165aa45313c6d9fa35ac30d2cda2`

This variant was compiled on 2025-03-23 22:42:42, named node.dll. It was part
of an archive named: update.zip with the sha256:

`543024edc9f160cc1cedcffc3de52bfa656daa0ec9ed351331d97faaa67d0d99`

This variant reuses the same C2 commands and anti-analysis techniques described
above, and it stores the GUID under the same file name.

## Conclusion

In summary, this analysis of a new ToneShell backdoor variant highlights the
evolving tactics of the Mustang Panda group, particularly their sustained
targeting of Myanmar. While not introducing revolutionary features, this
variant employs anti-analysis techniques, including elaborate stalling
mechanisms and obfuscated code. The blend of old and new elements, such as the
consistent communication protocol alongside updated GUID generation and the use
of seemingly random, copied text for filler, underscores the adaptive nature of
this threat. The continuous refinement of these evasion methods, coupled with
the geopolitical significance of the targeted region, reinforces the need for
ongoing research and threat hunting to counter cyber operations.

## IOCs

ToneShell variants:

`a58868b3d50b775de99278eeb14da8b7409b165aa45313c6d9fa35ac30d2cda2`

`e7b29611c789a6225aebbc9fee3710a57b51537693cb2ec16e2177c22392b546`

Archive files:

`543024edc9f160cc1cedcffc3de52bfa656daa0ec9ed351331d97faaa67d0d99`

`1272a0853651069ed4dc505007e8525f99e1454f9e033bcc2e58d60fdafa4f0`

C2:

`146.70.29[.]229:443`
