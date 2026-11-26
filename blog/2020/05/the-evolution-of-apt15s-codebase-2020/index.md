# The Evolution of APT15's Codebase 2020

> Ketrum, a new APT15 backdoor family merging features of Ketrican and Okrum, shows the group reworking the same codebase rather than changing tactics.

- Source: https://research.intezer.com/blog/2020/05/the-evolution-of-apt15s-codebase-2020/
- Published: 2020-05-21
- Authors: Paul Litvak
- Tags: apt, apt_china
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


The Ke3chang group, also known as APT15, is an alleged Chinese
government-backed cluster of teams known to target various high-profile
entities spanning multiple continents. Examples include attacks on
[European ministries](https://www.fireeye.com/content/dam/fireeye-www/global/en/current-threats/pdfs/wp-operation-ke3chang.pdf),
[Indian embassies](https://unit42.paloaltonetworks.com/operation-ke3chang-resurfaces-with-new-tidepool-malware/),
and
[British military contractors](https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2018/march/apt15-is-alive-and-strong-an-analysis-of-royalcli-and-royaldns/).
The group's activities have been traced back to 2010 and it is known to boast a
large number of custom tools, most often tailored to their specific target.

In mid May, we identified three recently uploaded samples from VirusTotal that
share code with older APT15 implants. We named this new family of samples,
"Ketrum", due to the merger of features in the documented backdoor families
"Ketrican" and "Okrum".

We believe the operation was conducted very recently. Below we present a
technical analysis of these samples and explain the evolution of APT15's
codebase over the last year.

## Overview

The three samples we discovered seem to be a mix of the Ketrican and Okrum
backdoors documented by researchers at
[ESET](https://www.welivesecurity.com/wp-content/uploads/2019/07/ESET_Okrum_and_Ketrican.pdf)
in 2019. Features have been merged from these two malware families to create a
different RAT class for the group. We've decided to call this umbrella of
malware "Ketrum."

The new samples we found continue the Ke3chang group's strategy of using a
basic backdoor to gain control over the victim's device, so that an operator
can then connect to it and run commands manually to conduct further operations.

Before beginning our technical analysis, we were able to connect these binaries
to Ke3chang using [Intezer Analyze](https://analyze.intezer.com/):

![](images/fig1.png)

*[Genetic Analysis](https://analyze.intezer.com/#/files/271384a078f2a2f58e14d7703febae8a28c6e2d7ddb00a3c8d3eead4ea87a0c0)*

All three samples contacted the same C2 server and appear to have been used in
two different time periods, judging by the PE timestamps and VirusTotal upload
date:

| SHA256s | Name | VirusTotal Date | PE Timestamp | C2 | Family |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `a142625512e5372a1728595be19dbee23eea50524b4827cb64ed5aaeaaa0270b` | `RavAudio64.exe` | 2019-12-03 | 7 Jan 2010 | `menu.thehuguardian[.]com` | Ketrum1 |
| `271384a078f2a2f58e14d7703febae8a28c6e2d7ddb00a3c8d3eead4ea87a0c0` | - | 2020-05-16 | 13 May 2020 | `www.thehuguardian[.]com` | Ketrum2 |
| `aacaf0d4729dd6fda2e452be763d209f92d107ecf24d8a341947c545de9b7311` | - | 2020-05-17 | 13 May 2020 | `www.thehuguardian[.]com` | Ketrum2 |

The C2 was registered towards the end of 2019, which makes us believe the first
PE timestamp was tampered with, and the latter two timestamps are at least
close to the real compilation date.

It's also important to note the C2 was registered in China and ceased operating
in mid May.

## The Best of Two Worlds

We documented several interesting differences between the backdoors:

|  | Ketrican | Okrum | Ketrum1 | Ketrum2 |
| :--- | :--- | :--- | :--- | :--- |
| Identify installed proxy servers and use them for HTTP requests | ❌ | ✅ | ✅ | ✅ |
| Special folder retrieval using registry key `[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders]` | ✅ | ❌ | ✅ | ✅ |
| The response from the server is an HTTP page with backdoor commands and arguments included in the HTML fields | ✅ | ❌ | ❌ | ✅ |
| Backdoor commands are determined by a hashing value received from C2 | ❌ | ✅ | ❌ | ❌ |
| Communication with the C&C server is hidden in the Cookie and Set-Cookie headers of HTTP requests | ❌ | ✅ | ✅ | ❌ |
| Impersonate a logged in user's security context | ❌ | ✅ | ✅ | ❌ |
| Create a copy of `cmd.exe` in their working directory and use it to interpret backdoor commands | ✅ | ❌ | ✅ | ❌ |
| Usual Ke3chang backdoor functionalities - download, upload, execute files/shell commands and configure sleep time | ✅ | ✅ | ✅ | ✅ |
| Screenshot-grabbing functionality | ❌ | ❌ | ✅ | ❌ |

## Ketrum 1

The Ketrum 1 sample was uploaded to VirusTotal in December 2019. This version
registers itself as a "WMI Provider Host" service *if* it is able to obtain
`SeDebugPrivilege`; otherwise it creates an entry in the startup directory.

This sample incorporates many features from Okrum as can be seen in the table
above, however, it abandons more advanced Okrum features such as offering a
reflective injection via an export and the use of hashes to receive command
IDs.

In the past, APT15 has used the
[IWebBrowser2](https://bohops.com/2018/03/17/abusing-exported-functions-and-exposed-dcom-interfaces-for-pass-thru-command-execution-and-lateral-movement/)
COM interface to manage its network communications. This time, the Ketrum
developer abandoned this technique and used simple HTTP APIs:

![](images/fig2.png)

Interestingly, this sample also incorporates a screenshot-grabbing command.

## Ketrum 2

Ketrum 2 seems to have been built for minimalism. As can be seen in Table 2,
many functionalities have been dropped.

Unlike the Ketrican variant, Ketrum implants no longer try to weaken the
system's security configurations. In previous implants, Powershell was used for
this end. Interestingly, a string still remains in Ketrum 2, which refers to
this deleted feature—perhaps an unintentional left-over from copy-pasting:

![](images/fig3.png)

Several other interesting unused file names are included in the binary such as
`%sadult.sft` and `%sMessage`.

The malware first collects basic system information to track the infected
endpoint and then sends it to the C2 server together with a hash of the system
info:

![](images/fig4.png)

![](images/fig5.png)

All incoming and outcoming payloads are fed through an RC4 encryption and
base64 encoding. The RC4 encryption uses an unusual key:

![](images/fig6.png)

These are actually constants used in the MD5 and SHA1 algorithms. The Ketrum
developer most likely intended to confuse researchers reversing this function.

A command is then extracted from the HTML in the response:

![](images/fig7.png)

This backdoor only supports a limited number of commands, which is typical of
Okrum and Ketrican backdoors. Unlike Ketrum 1, Ketrum 2 does not support
screenshot grabbing. This is the list of possible backdoor commands:

| Command ID | Description |
| :--- | :--- |
| 1 | Adjust sleep time |
| 2 | Execute a shell command |
| 3 | Upload a file |
| 4 | Download a file |
| 5 | Execute a file |
| 7 (there is no 6) | Execute a shell command with adjusted sleep time |
| 8 | Adjust execute shell sleep time |
| 9 | Download "Notice" file to working directory - * it is unclear how this is used |

## Code Reuse

Both Ketrum samples resemble a similar layout to previous Ke3chang tools, apart
from low level implementation and use of system APIs. Even in the two Ketrum
samples, there are differences between the low-level APIs used to achieve the
same functionality. For example, the file upload feature is implemented using
different APIs throughout the families; mostly using a constant value of
`0x20000` when reading files:

![](images/fig8.png)

![](images/fig9.png)

*Ketrum 1 — Ketrum 2*

![](images/fig10.png)

![](images/fig11.png)

*Ketrican 2018 — Okrum*

As reported by FireEye and ESET, it's likely the Ke3chang cluster of malware is
developed by multiple teams and the developers of Ketrican/Okrum belong to a
different team than the developers of Ketrum, albeit related. This could
explain the high-level and flow similarities but also the low-level
differences.

## Conclusion

Ke3chang's numerous tools such as Okrum, Ketrican, TidePool, Mirage, Ketrum,
and others all serve the same purpose, give or take a few techniques or
functionalities tailored for specific targets. We can regard these tools under
the same umbrella of BS2005 malware, distributed as different versions per
operation. However, the distinction created by naming them differently is
useful for tracking the group's operations and different development cycles.

The Ke3chang's group tools have not deviated much from the same tools reported
in FireEye's first Ke3chang report. The group continues to morph its code and
switch basic functionalities in their various backdoors. This strategy has been
working for the group for years and there is no indication yet that it will
deviate from this modus operandi.

The information security field has seen many improvements since the group's
inception, however, surprisingly, this is not reflected in the group's
persistence to use the same old TTPs in their tools.

## IOCs

```
271384a078f2a2f58e14d7703febae8a28c6e2d7ddb00a3c8d3eead4ea87a0c0
aacaf0d4729dd6fda2e452be763d209f92d107ecf24d8a341947c545de9b7311
a142625512e5372a1728595be19dbee23eea50524b4827cb64ed5aaeaaa0270b
thehuguardian[.]com
45.56.84[.]25
```
