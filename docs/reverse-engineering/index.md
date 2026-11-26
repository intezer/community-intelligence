# Beginner's guide to malware analysis and reverse engineering

> A beginner’s guide to malware analysis: setting the analysis goal, then initial triage of file format, known threats, packing, strings and the import table.

- Source: https://research.intezer.com/docs/reverse-engineering/
- Published: 2025-10-02
- Authors: Nicole Fishbein
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Malware analysis and [reverse engineering](../malware_re_101/part1/) are
powerful but can also be challenging and time-consuming. Performing a thorough
analysis typically requires deep knowledge, specialized tools, and extensive
experience. However, not every security analyst has the expertise or the
resources to conduct an exhaustive investigation for every suspicious file they
encounter. Moreover, a comprehensive, in-depth reverse engineering effort isn't
always necessary or practical, for example, if another researcher has already
reported and documented the file.

This blog series on "Breaking down malware" introduces a flexible, practical
approach to malware analysis. Our goal is to guide you through determining the
level of analysis required based on the context and initial findings. We will
explore various techniques and tools that can help you efficiently assess a
suspicious file, quickly determining whether a deeper dive is warranted or if
initial triage provides sufficient insight.

We'll start by detailing the foundational steps for analyzing suspicious files,
from initial triage and basic information gathering to deciding the right
analytical approach. Later in the series, we'll dive deeper into advanced
reverse engineering techniques. By the end, you will understand the technical
aspects of reverse engineering and the strategic mindset necessary to choose
the most effective analysis path. Ultimately, this blog aims to empower
analysts at any skill level to perform effective, context-driven malware
investigations, demonstrating that valuable insights can often be gained
without requiring expert-level reverse engineering skills.

## What is malware analysis?

Malware (short for malicious software) analysis involves examining malicious
software to understand its behavior, capabilities, and effects. By gaining
insights into how malware functions, security teams can create effective
detection, mitigation, and prevention strategies. It resembles digital
forensics, where analysts serve as detectives, dissecting malware to uncover
its mechanisms and defense methods. Just as doctors research diseases to
develop cures, security researchers study malware to improve defense systems.

There are two primary approaches to malware analysis:

- Static Analysis involves examining malware without executing it, like reading
  a machine's instructions without turning it on. Analysts inspect the code (if
  accessible) and its properties to predict its behavior.
- Dynamic Analysis - Running malware in a controlled environment to observe its
  real-time behavior. This approach typically uses a sandbox or a virtual
  machine with a debugger and monitoring tools. The main difference is that the
  latter option is more hands-on, giving the researcher more control over
  malware execution (particularly when debugging). In contrast, a sandbox
  encapsulates everything and produces a final report.

## Why malware analysis matters?

A thorough understanding of malware allows security teams to assess its
potential or actual impact on an organization. Different types of malware pose
different risks. Ransomware encrypts files for extortion and presents a
different threat than spyware, which silently steals sensitive data. Malware
analysis helps security teams determine how a threat operates, what
vulnerabilities it exploits, and how to defend against it effectively.

One of the most significant benefits of malware analysis is its role in
improving cybersecurity strategies. By studying malware behavior, security
teams can identify weaknesses in their defenses and implement better protective
measures. This includes enhancing endpoint security, refining detection rules,
and strengthening network monitoring to catch malicious activity before it
causes harm. Organizations can use insights from malware analysis to develop
stronger incident response plans, ensuring a fast and effective reaction to
future attacks. Additionally, security teams can create and distribute
Indicators of Compromise (IOCs), such as malicious file hashes, domain names,
and registry modifications, which help detect and block similar threats before
they spread.

Malware analysis also plays a crucial role in
[law enforcement](../../blog/2020/10/a-storm-is-brewing-ipstorm-now-has-linux-malware/)
and threat intelligence. Cybercriminals often reuse code, infrastructure, or
attack patterns across different operations. Security researchers and
intelligence agencies can track cybercriminal groups, attribute attacks, and
uncover larger threat campaigns by analyzing malware samples. This information
is invaluable for law enforcement agencies working to dismantle cybercrime
operations and prosecute attackers. Furthermore, malware analysis contributes
to the broader cybersecurity community by sharing threat intelligence, allowing
organizations and government agencies to collaborate on mitigating emerging
threats.

Ultimately, malware analysis is a critical component of modern cybersecurity.
It enables organizations to stay ahead of evolving threats, strengthens defense
mechanisms, and supports efforts to track and stop threat actors. Without it,
organizations would be left vulnerable, relying on reactive measures instead of
proactive security strategies.

## The fundamentals of malware analysis: Define the goal of malware analysis

Reverse engineering and malware analysis can quickly become complex and
time-consuming tasks, especially given the sophisticated techniques malware
authors employ. Analysts often face constraints on time, resources, and their
specific areas of expertise. Thus, it's essential to approach malware analysis
strategically, focusing efforts based on clearly defined goals.

The objectives of malware analysis differ greatly depending on the analyst's
role and organizational priorities. For instance, a frontline analyst might
only need to confirm whether a file is malicious before passing it to
specialized teams for further analysis. In contrast, incident response
professionals may require detailed insights into malware behavior to understand
the full scope of an attack, formulate response strategies, and document
findings comprehensively. Analysts or researchers tasked with producing
detailed threat reports, whether for internal use or public dissemination,
typically require in-depth analysis. Such comprehensive reports necessitate a
thorough understanding of the malware's capabilities, objectives, and technical
intricacies.

Clearly identifying your goal at the outset helps you determine the scope of
analysis, select the appropriate tools and techniques, and efficiently allocate
resources. By defining the precise information needed, whether it's identifying
indicators of compromise, understanding infection methods, extracting
actionable intelligence, or simply classifying the file's threat level, you
ensure that your analysis remains targeted and effective.

Always ask yourself: "What is the minimum critical information required to
achieve my goal?" This mindset prevents unnecessary deep dives into irrelevant
details and maximizes the efficiency and impact of your malware analysis
efforts.

![](images/fig1.png)

## Initial triage

Before diving deep into malware analysis, the first step is to conduct an
initial triage. This helps determine the file's basic characteristics, identify
potential challenges like packing or obfuscation, and extract valuable
indicators for further analysis.

In this section and the following posts in this series, we will use an example
file to demonstrate the malware analysis and reverse engineering process. The
file can be downloaded from
[MalwareBazaar](https://bazaar.abuse.ch/sample/8d5be7da9525bdd7e925fe9e5fb988e5e7c0db4d37cb8a6fa6469f08b8ee0bc1/).
We will name the file: `mal_mal`.

MD5: `3f6094edd7cad01c0f12decf36dcce95`

### What architecture and format does it have?

The first step in triage is identifying the file's architecture and format.
When it comes to executables, malware can target different operating systems
and may come in various formats, such as:

- [Portable Executable](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)
  (PE)
- [Executable and Linkable Format](https://refspecs.linuxfoundation.org/elf/elf.pdf)
  (ELF)
- [Mach-O](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html)

Determining the format helps analysts choose the appropriate tools and
techniques for further investigation. Different tools, such as
[CFF-Explorer](https://ntcore.com/explorer-suite/),
[Detect-It-Easy](https://github.com/horsicq/Detect-It-Easy),
[XPEViewer](https://github.com/horsicq/XPEViewer),
[Radare2](https://github.com/radareorg/radare2), and the
[file](https://www.man7.org/linux/man-pages/man1/file.1.html) command (Unix),
can help us identify the file format.

Example: Identifying the file format and architecture. In this case, we have a
64-bit DLL for x84-64 processors.

```
file mal_mal
mal_mal: PE32+ executable (DLL) (console) x86-64, for MS Windows
```

### Is it a known threat?

Once the format is identified, checking whether the sample is already known can
save significant time. Analysts can compare the file's hash against threat
intelligence databases to see if it has been previously documented.

Common sources for checking known threats include:

- VirusTotal – A multi-engine scanning service
- MalwareBazaar – A public malware repository
- Threat Intelligence Feeds – Databases from security vendors

If the malware is known, existing reports can provide insights into its
functionality and risks. If it is not recognized, further analysis is required.

In our case, the file was not submitted to any of the publicly available
services.

### Packed vs. unpacked malware

### Packers and crypters

Threat actors often use tools to pack, encrypt, and obfuscate the main payload.
Their goal is to make malware analysis and detection much more complex and
time-consuming. The reason is that the main payload is wrapped in layers of
obfuscation, which makes it difficult to inspect. That said, there are ways to
extract the main payload; some cases are easier than others.

Packers are tools that compress the executable to reduce the size of the
binary. Common packers include UPX, ASPack, and MPRESS.

Crypters alter the code to make it difficult to read, often by renaming
functions, adding junk instructions, or using encryption to hide key
functionality. Common crypters include: Themida, Armadillo Crypter, and
CyberSeal.

For analysts, packed and obfuscated malware presents a challenge. Static
analysis becomes less effective since much of the original code is concealed,
making extracting valuable indicators such as strings, imports, or API calls
difficult. Debugging and dynamic analysis are often required to observe the
malware in action, extract the unpacked code, and uncover its true intent. In
cases where advanced packers or custom obfuscation methods protect malware,
analysts may need to manually trace execution in a debugger or dump the
unpacked code from memory to proceed with their investigation.

### Is It Packed and/or Obfuscated?

To determine if a file is packed, analysts can check for:

High entropy –
[Entropy](<https://en.wikipedia.org/wiki/Entropy_(information_theory)>) is a
measure of randomness in data. In the context of files, it indicates how
uniformly the bytes are distributed. A standard executable contains a mix of
code, readable strings, and structured data, resulting in medium entropy.
However, when a file is compressed or encrypted, the byte distribution becomes
more random, increasing its entropy level.

A high entropy value (typically close to 8 on a scale of 0 to 8) suggests that
the file has been packed or encrypted. This is because packing tools replace
readable code with compressed or encrypted data, which is later decompressed or
decrypted when executed. Analysts use entropy analysis to detect potential
packing, as a high entropy section, especially in areas where plain code should
be, often indicates that the file has been modified to conceal its true
functionality. If a file shows unusually high entropy, unpacking it is usually
necessary before further analysis can be conducted.

Unusual section names – Some packers create non-standard section names in
executables

Missing or altered import tables – In a standard executable file, the import
table lists external functions and libraries the program relies on to perform
tasks, such as file operations, network communication, or process manipulation.
This table is crucial because it provides insight into what the program does
and which system functions it interacts with. Packed malware often has a
missing or heavily altered import table, making static analysis more difficult.
This happens because packers remove or encrypt the import table, preventing
analysts from easily identifying the malware's functionality. Instead of
listing necessary functions upfront, packed malware dynamically resolves them
at runtime, often using indirect calls or API hashing techniques. A complete
absence of imports or an unusually small number indicates that the file is
packed and may require unpacking before further analysis.

Example: Checking if the file is packed using entropy analysis

{Insert command output showing entropy calculation or section analysis}

### Unpacking

If a sample is packed, it needs to be unpacked before further analysis. Some
packers, like UPX, have easy-to-use unpacking tools, while others require
manual unpacking techniques such as debugging and memory dumping.

**Example: Unpacking a UPX-packed sample**

Let's take our file and test for UPX packing -

One way is to run the UPX utility with the test (`-t`) flag:

```
upx -t mal_mal
```

![](images/fig2.png)

Another way is to check the section names. In standard UPX packing, the section
names will be named `UPX<number>`

So now that we know that the file is packed, we will use the UPX utility to
unpack it.

```
upx -d -o unpacked_mal mal_mal
```

The next stage in an investigation would be to analyze the unpacked file.

### Extracting strings and indicators – Finding clues in plain text

Plaintext strings can provide valuable clues. Strings can reveal:

- C2 (Command and Control) domains
- File paths and registry keys
- Error messages or debug information

Analyzing extracted strings helps guide further investigation and can provide
early indicators of the malware's intent.

**Example: Extracting strings from the malware sample**

We have several options: run the strings utility, use the built-in strings
extraction option in BinaryNinja, or use other tools that process strings.

No matter the method, we usually get a very long list of strings, some don't
make sense, some are from library code, but we are looking for user-defined
strings because these strings might give away some information about the
malware.

It takes experience and knowledge to identify interesting user-defined strings.
If you see a string and are not sure if it is part of a library or what it can
be related to, you can:

- Google it
- Use [grep.app](https://grep.app/) - a free platform that searches for a given
  string in all public git repositories. In our use case, it provides context
  to strings and helps identify strings common in code or strings that are part
  of malicious code.

We will use the `strings` command in Unix and highlight the following strings
that look suspicious (and interesting):

```
killme
changeshell
chcp 437 > NUL
Client Ready
%s\%lu.bat
powershell.exe -nologo
ea2ced841239466e92dc53d49b45975e
[+] Endpoint changed
[+] ShortTimer and FailCounter changed. New ShortTimer is
[+] Short Timer changed. New Short Timeout is 1 minute
orer.exe\r\nstart  "%s"\r\nif exist {C2796011-81BA-4148-8FCA-C664324elete "HKEY_CURRe\Classes\CLSID\@echo off\r\nreg dexplorer.exe\r\nti5113F}" /F\r\ntask"%s" goto d\r\ndelmeout 5\r\n:d\r\ndelkill /F /IM explENT_USER\Softwar
```

### Analyzing the Import Table

When analyzing a suspicious executable, the import table is one key area to
examine. This table provides valuable insights into the file's functionality by
listing the external libraries and functions the program relies on. Before
diving into the import table, it is essential to understand the structure of a
Portable Executable (PE) file and how different sections contribute to a
program's execution.

### Understanding the PE Header, Sections, and Segments

A Portable Executable (PE) file is the standard format for executables, DLLs,
and other binary files in Windows. It consists of multiple components,
including:

- PE Header – Contains metadata about the executable, such as its entry point,
  section layout, and the locations of important tables, including the import
  table.
- Sections – The PE file is divided into sections, each serving a different
  purpose. Common sections include:
  - `.text` – Contains the executable code.
  - `.data` – Holds initialized global variables.
  - `.rdata` – Stores read-only data, such as imported function names and
    strings.
  - `.rsrc` – Contains resources like icons, images, and dialogs.
  - `.idata` – Stores information about imported functions, forming the import
    table.

### What is the Import Table and why is it important?

The import table lists the external functions that an executable calls from
shared libraries (DLLs). Instead of implementing everything from scratch,
programs rely on system-provided functions for common tasks such as file
access, network communication, and memory management. The import table provides
the following information:

- DLL names – The names of the libraries the executable depends on, such as
  `kernel32.dll`, `user32.dll`, and `ws2_32.dll`.
- Imported functions – The program's specific API functions, such as
  `CreateFileA`, `VirtualAlloc`, `WinExec`, or `send`.
- Addresses of imported functions – The memory locations where these functions
  will be mapped when the program runs.

By analyzing the import table, we can quickly assess the purpose of a program.
If an executable imports functions related to file manipulation, registry
modifications, or process injection, it might indicate potential malicious
behavior. For example:

- The presence of `WriteFile`, `CreateFileA`, and `DeleteFileA` may suggest
  file system modifications.
- The use of `RegCreateKeyEx` and `RegSetValueEx` could indicate registry
  modifications for persistence.
- The inclusion of `VirtualAlloc`, `WriteProcessMemory`, and
  `CreateRemoteThread` might hint at code injection or process hollowing,
  techniques often used by malware.

### How does the import table help in initial triage?

During the initial triage of a file, examining the import table can help
analysts quickly determine the following:

- Whether the file is packed or obfuscated – If an executable has very few or
  no imports, it might be packed since packers remove the original import table
  and restore it dynamically at runtime. (As we saw in the previous part).
- The intended functionality of the program – We can infer what the executable
  is designed to do by looking at the imported functions.
- Potential malicious behavior – Certain functions or DLLs, such as
  `advapi32.dll` for registry access or `ws2_32.dll` for network connections,
  can indicate specific attack techniques used by malware.
- Connections to known threats – Comparing the import table against known
  malware samples can help identify similarities and classify the file.

Analysts can gain crucial insights into a file's behavior by carefully
analyzing the import table before executing it. This allows for a more informed
approach to further analysis, whether that involves static code inspection,
sandboxing, or unpacking a protected sample.

**Example: We will use the triage view in BinaryNinja to inspect the imports**

In our case, several functions stand out as possible red flags.

- Registry Manipulation – Possible Persistence or Configuration Extraction

  `RegQueryValueExA`, `RegOpenKeyExA`, `RegEnumKeyExA`

  These functions are commonly used to query, open, and enumerate registry
  keys. Malware often uses them to check system configurations, read stored
  credentials, or modify registry settings for persistence.
- Process and Thread Manipulation – Possible Code Injection or Process
  Hollowing
  - `CreateProcessA` - This function creates new processes, which can indicate
    malware attempting to spawn additional payloads or execute commands on the
    system.
  - `CreateThread` - The ability to create new threads can be exploited for
    code injection, where malicious code is executed in the context of another
    process.
- File System Operations – Potential Indicators of Data Theft or Destructive
  Behavior

  `CreateFileA`, `CreateFileW`, `WriteFile`, `ReadFile`, `DeleteFileA`

  These functions allow the malware to create, modify, or delete files.
  Ransomware often uses `WriteFile` to encrypt data, while data-stealing
  malware may use `ReadFile` to exfiltrate information.
- Network Communication – Potential C2 Activity

  `WinHttpOpen`, `WinHttpConnect`, `WinHttpOpenRequest`, `WinHttpSendRequest`,
  `WinHttpReceiveResponse`, `WinHttpReadData`, `WinHttpWriteData`

  These functions are part of the Windows HTTP Services (WinHTTP) library and
  are typically used for network communication. Malware can use these APIs to
  establish contact with a C2 server, download additional payloads, or
  exfiltrate data.
- Anti-Analysis and Evasion Techniques

  `IsDebuggerPresent` - This function checks whether the process is running
  inside a debugger. Malware often uses it to detect security researchers or
  sandboxes and modify its behavior to evade analysis.
- Inter-Process and Memory Manipulation – Potential Code Injection or Process
  Tampering

  `OpenFileMappingA`, `CreateFileMappingA`

  These functions create or open shared memory regions, which can be used for
  inter-process communication. Some malware uses these APIs for process
  injection or to exchange data between different processes stealthily.

While none of these imports alone confirms that the executable is malicious,
their presence, especially when combined, can indicate suspicious activity. If
a file heavily relies on registry modifications, process creation, file
manipulation, and network communication, it needs deeper analysis to determine
its true intent. The suspicious strings that we saw in the previous stage
further support this assumption. The following steps may include examining its
behavior dynamically and investigating additional artifacts.

This first part of our "Breaking Down Malware" series has laid the groundwork
for understanding malware analysis and reverse engineering. We've explored the
initial triage process, from identifying file architecture and format to
determining if a threat is known, and understanding the challenges posed by
packed and obfuscated malware. We also dug into the significance of extracting
strings and analyzing the import table, highlighting how these initial steps
provide crucial insights into a file's potential malicious behavior. By
meticulously gathering this fundamental information, analysts can build a
strong foundation for the deeper static analysis and reverse engineering
techniques we will explore in the upcoming second part of this guide.

In the next blog in this series, we will dive into the static analysis to
showcase how we continue the investigation based on the information we have
gathered.
