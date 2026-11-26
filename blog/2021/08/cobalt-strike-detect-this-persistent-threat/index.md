# How to Detect Cobalt Strike

> Cobalt Strike Beacon arrives via Office macros, supply chain attacks, living off the land and EXE files. How to detect each with static and dynamic analysis.

- Source: https://research.intezer.com/blog/2021/08/cobalt-strike-detect-this-persistent-threat/
- Published: 2021-08-18
- Authors: Ryan Robinson
- Tags: framework, windows
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Intro

Cobalt Strike is a penetration testing tool created by Raphael Mudge in 2012.
To this day, it remains extremely popular in red team activities and used for
malicious purposes by threat actors. Cobalt Strike is popular due to its range
of deployment options, ease of use, ability to avoid detection by security
products, and a number of capabilities it has. It is for these reasons that
threat actors like Cobalt Strike. Since Cobalt Strike is widely used by a range
of actors, its lack of exclusivity makes attribution harder. Companies still
struggle to detect Cobalt Strike also due to the various defensive techniques
it has.

This blog explains Cobalt Strike and practical steps to take if you believe
that you are being targeted or have been compromised by Cobalt Strike. We will
demonstrate some real world examples of Cobalt Strike delivery and steps to
detect each.

## What is Cobalt Strike?

Cobalt Strike is [marketed](https://www.cobaltstrike.com/) as "Software for
Adversary Simulations and Red Team Operations."

It is a popular platform that allows users to
[emulate](https://www.cobaltstrike.com/features) advanced threats, perform
reconnaissance, hide communications, escalate privileges, move laterally across
the network, and deploy additional payloads. The main payload of Cobalt Strike
is called "[Beacon](https://www.cobaltstrike.com/help-beacon)." The Beacon
payload is used to model advanced APT malware, and it can do the following:

- Receive commands (either passively or from an interactive console)
- Egress communications over HTTP, HTTPS, and DNS
- Launch PowerShell
- Execute binaries
- Modify and query the Windows registry
- Inject malicious code into legitimate processes
- Log keystrokes
- Take screenshots
- Set up proxies
- Escalate privileges
- Bypass UAC
- Dump password hashes
- Scan ports among other abilities

This tool is mainly used in red team operations for government agencies and
private enterprises, but it's also a popular tool leveraged by cybercrime and
[APT groups in cracked versions](https://www.proofpoint.com/us/blog/threat-insight/cobalt-strike-favorite-tool-apt-crimeware).
It is evident why Cobalt Strike is used by organizations and threat actors
alike because of the extensive suite of capabilities it possesses, and also due
to its ability to bypass defenses. It also comes with the feature to generate
[reporting](https://www.cobaltstrike.com/help-reporting) in which the attacking
team or threat actor can continuously study and improve their campaigns.

## Why is it difficult to detect Cobalt Strike?

Cobalt Strike is difficult to detect because of its several defense techniques.
Cobalt Strike payloads are usually shellcode encrypted with a rolling XOR key.
This makes static analysis difficult to conduct. This, combined with the
ability to configure many parts of the payload, makes hash-based detection
almost impossible. Cobalt Strike stagers are designed to be loaded and executed
only in-memory. This opens up a ton of possibilities for how this shellcode is
shipped, making signature-based detection on the delivery method a cat and
mouse game. Depending on how the code is delivered, the code can be injected
into other legitimate running processes, bypassing defenses that do not scan
legitimate processes or code in-memory.

## How has Cobalt Strike been deployed?

Cobalt Strike has many different ways for deployment. This flexibility has
helped attackers find many unconventional and creative ways to infect victims
with a payload. For an in-depth technical analysis of Cobalt Strike's
deployment options and how they differ, check out either Avast's
[blog](https://decoded.avast.io/threatintel/decoding-cobalt-strike-understanding-payloads/)
or Cisco Talos'
[white paper](https://talos-intelligence-site.s3.amazonaws.com/production/document_files/files/000/095/031/original/Talos_Cobalt_Strike.pdf).
Let's take a look at some real world examples of how Cobalt Strike is being
used in the wild. We will cover the following:

- Macro-Laden Microsoft Office files
- Supply Chain Attack
- Living off the Land (LotL)
- Executables (EXE) files

## Macro-Laden Microsoft Office Files

An
[example](https://analyze.intezer.com/analyses/2eaedc79-0197-4907-b4e1-f609853bc04e)
of a Cobalt Strike payload being delivered to victims via Microsoft Excel
spreadsheets demonstrates that this tool is used in mass phishing campaigns,
not just targeted APT attacks. The attack starts by sending potential victims a
Microsoft OneDrive link from which an Excel (.xls) file is downloaded.

![](images/fig1.png "OneDrive URLs sent to victims")

Using cloud storage links to deliver malicious files is a well-known strategy.
It leverages the good reputation of cloud provider domains such as Microsoft,
Amazon, and Google to bypass domain reputation-based security controls. This
link delivers an Excel file pretending to be an Apple Store invoice requesting
that the target "enable content to view receipt."

![](images/fig2.png "Spreadsheet lure masquerading as an Apple Store receipt.")

Upon enablement of macros, the spreadsheet will fetch and execute the payload
in-memory.

### How to Detect?

This can be difficult to detect, as there are multiple degrees of separation
before the Cobalt Strike payload is executed. Detection first requires dynamic
analysis in order to reach the Cobalt Strike stage. When this stage is reached,
the best ways to detect the running Cobalt Strike code are through static
signatures or genetic code analysis.

When it comes to static signatures, it can be difficult to isolate the exact
area in-memory that you should run the signatures over. One way this can be
achieved is running the file through debugging tools and manually dumping
memory to perform signature analysis. This can be extremely time consuming and
requires a high degree of technical knowledge. Another possible way is to use a
sandbox and download memory dumps from a finished analysis in order to run
static analysis tools. This requires slightly less technical knowledge, but it
still can be time consuming. We suggest taking the suspicious document and
uploading it to Intezer Analyze to find out if Cobalt Strike is hidden
in-memory.

![](images/fig3.png "Intezer Analyze result for Cobalt Strike payload.")

## Supply Chain Attack

One of the biggest cybersecurity stories of 2020 was the SolarWinds supply
chain attack that compromised high-profile entities around the world. This
attack was done by an APT group known as NOBELIUM (UNC2452) leveraging the
"Orion" business software to distribute malware to private and public
organizations. Among the deployed malware was a Cobalt Strike loader dubbed
TEARDROP by FireEye. The variant was named
[Raindrop](https://symantec-enterprise-blogs.security.com/blogs/threat-intelligence/solarwinds-raindrop-malware)
by Symantec. The TEARDROP dropper is a memory-only DLL that runs as a service
spawning a thread that pulls the Cobalt Strike payload from a fake JPG file.

The Raindrop variant is built from a modified version of 7-ZIP source code. It
uses a different custom packer than TEARDROP, also leveraging steganography to
locate the start of the encoded payload. Once the encoded payload has been
located, it extracts, decrypts, and decompresses the data to be executed as
shellcode.

### How to Detect?

It can often be difficult to detect if your organization has been the victim of
a supply chain attack. It can be especially hard to collect forensic evidence
for an attack when it could be mixed in with the code of legitimate and large
files. Due to the nature of
[supply chain attacks](https://intezer.com/blog/malware-analysis/securing-the-software-supply-chain/),
there are often a large number of machines in an organization infected at one
time. An action you can take is to run Intezer's
[live endpoint scanner](https://analyze.intezer.com/endpoint-analyses) across
all machines in the organization, to get immediate visibility over all running
code and quickly identify infected machines by detecting any traces of
malicious code found in-memory. An example of a machine with Raindrop loading
Cobalt Strike is shown in the endpoint scan below.

![](images/fig4.png "Intezer Analyze endpoint scan result for Raindrop loading and executing a fileless Cobalt Strike payload.")

## Living off the Land (LotL)

Living off the Land (LotL) is the attack process of using legitimate and signed
tools, usually provided within the operating system, to execute malware. This
is a powerful tactic as it can result in unauthorized code being executed
within the memory space of a trusted process, evading malware defenses by
flying under the radar. This type of tactic also makes incident response
difficult, since analysts can't just filter out known legitimate processes
during triage. All processes must be inspected in order to find that one needle
in the haystack.

One popular tool used for LotL operations is the Microsoft.NET framework
utility called
[MSBuild](https://docs.microsoft.com/en-us/visualstudio/msbuild/walkthrough-using-msbuild?view=vs-2019).
MSBuild is the build platform used for Microsoft and Visual Studio. Visual
Studio relies on MSBuild to build projects for testing and releases. Attackers
are able to pass MSBuild.exe, a project (.proj) file, to build and execute. The
payload, usually shellcode, is injected into another process. This attack is
effective for attackers as many sandboxing solutions are not able to handle
project files and struggle with fileless malware. This technique was observed
by Cisco Talos researchers in
[2020](https://blog.talosintelligence.com/2020/02/building-bypass-with-msbuild.html)
to deploy Cobalt Strike.

![](images/fig5.png "Project file code.")

As shown above, the project file has an encoded and compressed payload. This
payload is decrypted, decompressed, and then copied into memory. The shellcode
is then executed in a new thread.

### How to Detect?

An endpoint with a system injected with Cobalt Strike via MSBuild is shown
below. Note the process tree at the bottom indicating the "fileless code."

![](images/fig6.png "Intezer Analyze endpoint scan of a Cobalt Strike-infected system via LotL technique.")

## Executables (EXE) Files

There is an acronym in the United States Armed Forces called "KISS." KISS
stands for "Keep it simple, stupid!" Sometimes simple is better, and another
way for Cobalt Strike to be deployed is in a simple Windows EXE form. This
requires either social engineering tactics to get the target to execute the
malware or another program/script to execute the file. This process involves
creation of a thread that sets up a named pipe for privilege escalation. Once
the shellcode is written to the named pipe, it is decrypted and executed in a
separate thread.

### How to Detect?

An example of one of these payloads is shown in the
[analysis](https://analyze.intezer.com/files/e233fc08c1a41ed8cc7d32c8e851614aeb159f95470f4491262c818137591afc)
below. Notice how the Cobalt Strike code is only shown when it is executed and
found in-memory.

![](images/fig7.png "Cobalt Strike found via memory analysis.")

## How can Cobalt Strike be detected and remediated?

Due to the many ways Cobalt Strike is deployed, detection can be hard. The use
of shellcode, encoding, compression, obfuscated strings, process injection,
hashing algorithms, domain fronting, different communication channels, and
dynamically loaded libraries all give malware and network defenses a run for
their money.

### Static Analysis

Static analysis involves examining the file using various techniques without
actually having to execute the file itself. Static analysis can involve hashing
the file and finding intel on it, taking a look at the strings to see if there
are functionality or network indicators, or checking imports and running
signatures such as YARA for the file. Although useful, static analysis on its
own is probably not sufficient to detect Cobalt Strike.

Using hash-based identification of Cobalt Strike is insufficient, since each
payload will be encrypted with different keys and each configuration will
uniquely change the hash value. It is trivial to generate a new payload for
each new target.

Checking strings may be insufficient also. Strings for pipe names are
dynamically generated and incorporate random numbers, meaning they can change
every time the malware is executed. Encrypted payloads will also obfuscate
useful strings from static analysis.

[API Hashing](https://www.ired.team/offensive-security/defense-evasion/windows-api-hashing-in-malware)
algorithms employed by Cobalt Strike hide imports from static analysis
techniques. Signature-based detection is great for detecting malware, but due
to the versatility of Cobalt Strike's deployment using multiple stages and
encrypted/obfuscated payloads, an analyst may only be able to detect that a
file is going to load and execute a payload in-memory. Without dynamic
analysis, they won't be able to detect exactly what that payload will be.

### Dynamic Analysis

Dynamic analysis is the process of executing the suspect file in order to
analyze its behavior and how it affects the environment it runs in. Dynamic
analysis can open up new areas to explore as one can follow the malware through
each stage of its deployment and functionality. Dynamic analysis can get the
malware to unpack, decode, or download additional stages. These new stages are
then subject to further dynamic analysis as well as the previously mentioned
static analysis techniques.

Dynamic analysis does not have many limitations, although some malware includes
functionality to detect if it is being observed or running inside a sandboxed
environment. There is also the possibility that during dynamic analysis, areas
of malicious code may not be intentionally executed, and thus not detected in
the behavior. The best way to detect malicious code is via genetic code
analysis which is done automatically in Intezer Analyze.

### Combination of Several Techniques

The best way to detect Cobalt Strike code is through a combination of dynamic,
static, and genetic analysis. Let's take a suspicious looking document from an
unknown entity as an example. Before opening the document, we submit it to
Intezer Analyze and get the verdict, as shown below.

![](images/fig8.png "Intezer Analyze result showing in-memory Cobalt Strike code.")

The document drops and executes Cobalt Strike in the memory space of
`rundll32.exe`. Signatures are leveraged to show capabilities and file
characteristics. Under the "TTPs" tab the user can see the
techniques/capabilities employed by the malicious document.

![](images/fig9.png "TTPs section showing capabilities detected during execution.")

The document displays interesting techniques such as macros with
auto-execution, network activity with a unique user agent, office process
starting martian subprocess, and process injection. You can also dive deeper
into capabilities specific to the injected Cobalt Strike process.

![](images/fig10.png)

The "IoCs" tab in Intezer Analyze shows indicators that can help you pivot and
search in your environment during investigations to map out the scope of an
attack. IoCs provide you with file hashes and network indicators such as URLs,
and IP addresses being contacted through irregular ports.

![](images/fig11.png "IoCs tab showing file and network indicators.")

The "Behavior" tab shows a more in-depth analysis of the file's behavior, where
you can see the process tree, network activity, screenshots and file/registry
activity.

![](images/fig12.png "Behavior tab showing observed behavior during sandbox execution.")

## The Only Abused Pen Testing Tool?

Cobalt Strike is
[not the only](../../../2020/09/advanced-pasta-threat-mapping-malware-use-of-open-source-offensive-security-tools/)
penetration testing or legitimate tool that has been co-opted and abused by
threat actors. In the past, tools such as
[Pafish](https://github.com/a0rtega/pafish) (Paranoid Fish) have been
[used by](https://cdn2.hubspot.net/hubfs/1903456/Whitepapers/CopyKittens.pdf)
Iranian actors in their tooling for virtual machine (VM) detection. The
"Sysinternals" suite has been used extensively by threat actors. Most notably,
[PsExec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) has
been used in high-profile attacks such as the 2017
[NotPetya](https://www.theregister.com/2017/06/28/petya_notpetya_ransomware/)
global ransomware outbreak.

More recently, legitimate and penetration testing tools for the cloud have been
used by threat actors. The threat actor TeamTNT has
[used](../../../2020/09/attackers-abusing-legitimate-cloud-monitoring-tools-to-conduct-cyber-attacks/)
Weave Scope, a trusted tool which gives the user full access to their cloud
environment, and is integrated with Docker, Kubernetes, the Distributed Cloud
Operating System (DC/OS), and AWS Elastic Compute Cloud (EC2). The attacker
installs this tool in order to map the cloud environment of their victim and
execute system commands without needing to deploy malicious code on the server.
The same group has also been
[documented](https://www.cadosecurity.com/post/botnet-deploys-cloud-and-container-attack-techniques)
using the penetration testing tool
[Break Out The Box](https://github.com/brompwnie/botb) (BOTB) for cloud and
containerized environments.

## Get Started for Free

With [Intezer Analyze](https://analyze.intezer.com/), you can analyze any
suspicious files that you encounter, including non-executable files such as
Microsoft Office documents, scripts, archives, and more. Stay on top of
analyzing and classifying Cobalt Strike among other threats.
[Get started](https://analyze.intezer.com/create-account) for free and start
with 50 file uploads per month.
