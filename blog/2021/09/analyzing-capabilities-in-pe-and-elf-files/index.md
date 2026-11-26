# Teaching Capa New Tricks: Analyzing Capabilities in PE and ELF Files

> Intezer extended FireEye’s capa to analyze ELF files and added Linux rules. Pairing extracted capabilities with code reuse helps connect malware families.

- Source: https://research.intezer.com/blog/2021/09/analyzing-capabilities-in-pe-and-elf-files/
- Published: 2021-09-15
- Authors: Joakim Kennedy
- Tags: tool
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

When analyzing malware, one of the goals in addition to identifying what
malware it is, is to understand what it does when it runs on a machine. There
are multiple ways of achieving this by either manual analysis, which requires
reverse engineering skills, or automated analysis. With Intezer Analyze we
provide a tool that aids both advanced reverse engineers and entry-level
analysts with the task of analyzing any suspicious file. Earlier this year we
[announced](https://intezer.com/blog/get-more-context-with-ttps/) a new feature
in Intezer Analyze that extracts the capabilities of a file. From the
capabilities, you can map these to known Tactics, Techniques, and Procedures
(TTPs). Under the hood, the functionality is powered by an open-source project
from FireEye called [capa](https://github.com/fireeye/capa). Capa analyzes
Windows portable executable (PE) files and matches the findings against a rule
set to determine a file's capability. Intezer Analyze is not limited to PE
files. It can also detect code reuse between Executable and Linkable Format
(ELF) files. With this in mind, we wanted to provide similar information for
ELF files. To achieve this, we extended capa support to ELF files and also
created rules for Linux ELFs. Since capa is an open-source project, we
[submitted the changes](https://www.fireeye.com/blog/threat-research/2021/09/elfant-in-the-room-capa-v3.html)
to the capa project to allow not only users of Analyze to take advantage of
this new functionality but also users of capa. So let's explore how we can use
these new features in Intezer Analyze when investigating both PE and ELF files.

## Supercharge Code Reuse with Capabilities

One beautiful marriage is the use of capabilities extracted by capa and
malicious code reuse (aka code genes) detected by Intezer Analyze. At Intezer,
we have a vast gene database of both known malicious code and code from trusted
vendors. This allows for the detection of different malware versions but also
detects when code is shared between different malware families. This can be
used to attribute new malware to previously known threats. Another strength of
code genes is that it allows for the determination of which functionality is
shared between different malware via code reuse. In biology, genes serve as the
blueprint for proteins and the proteins affect what a cell will do. A parallel
can be drawn to computer code. What a file will do, is dependent on its
capability. The capabilities are "codified" in its code genes. What this means
is that by connecting capabilities to code genes, we can detect that a
capability is shared between different malware families, thus making a strong
connection between them. To see this in action, let's look at a practical
example. The following example uses one of the Olympic Destroyer samples
([`3e27b6b287f0b9f7e85bfe18901d961110ae969d58b44af15b1d75be749022c2`](https://analyze.intezer.com/files/3e27b6b287f0b9f7e85bfe18901d961110ae969d58b44af15b1d75be749022c2))
that was part of Cisco Talos'
[report](https://blog.talosintelligence.com/2018/02/olympic-destroyer.html).
Olympic Destroyer is a good example because of all the false flags planted by
the threat actor. If we analyze the sample, you can see that it drops a number
of other files. The analysis of one of these files is shown in Figure 1.

![](images/fig1.png "Figure 1: Analysis of one of the dropped files by the Olympic Destroyer sample (3e27b6b287f0b9f7e85bfe18901d961110ae969d58b44af15b1d75be749022c2).")

The gene summary shows that the file shares code with Olympic Destroyer and
TeleBots. Looking at the related samples to Olympic Destroyer, Figure 2, is the
"System Stealer" from the Cisco Talos report. According to the report, the
stealer attempts to obtain credentials from LSASS. The technique used is
similar to Mimikatz. From the gene summary, we can see that it does not share
code with Mimikatz.

![](images/fig2.png "Figure 2: Olympic Destroyer related sample found is the system stealer (f188abc33d351c2254d794b525c5a8b79ea78acd3050cd8d27d3ecfc568c2936) from the original Cisco Talos report.")

If we look at the related TeleBots samples, we can see (Figure 3) that they
share code with a sample labeled XData. XData was a ransomware that targeted
Ukraine a month before the
[NotPetya](../../../2017/10/notpetya-returns-bad-rabbit/) incident. According
to an ESET
[report](https://www.welivesecurity.com/2017/06/30/telebots-back-supply-chain-attacks-against-ukraine/),
the malware was launched just after M.E.Doc had been executed on the machine,
suggesting that the same supply chain vector as NotPetya was used. XData,
according to ESET, had an embedded DLL that performed credential harvesting
like Mimikatz.

![](images/fig3.png "Figure 3: TeleBots related samples show that the samples share code with an XData sample.")

The connection between the two malware families can be made stronger by looking
at the functionality of the code that was reused. This you can do in Intezer
Analyze by looking at the capabilities that share genes between the TeleBots
sample and the Olympic Destroyer sample. The result is shown in Figure 4, where
you can see that the shared code is for the allocation of read-write-execute
memory. This is used by malware, for example, as part of unpacking or
decrypting payloads or for injecting into another process. What this is telling
us is that XData and Olympic Destroyer share a unique code that is used as part
of the allocation of read-write-execute memory.

![](images/fig4.png "Figure 4: Capabilities with shared code genes between the Olympic Destroyer sample and XData.")

## Analyzing Capabilities of ELF Files

Capa is a great tool for extracting capabilities of files and until now it only
supported PE files. To provide the same functionality for ELF files in Intezer
Analyze, we expanded capa's functionality. Intezer Analyze has been supporting
ELFs for some time now but with the release of
[capa v3.0](https://www.fireeye.com/blog/threat-research/2021/09/elfant-in-the-room-capa-v3.html),
which includes our improvements and rules for Linux ELFs, similar capability
extraction can be performed by using capa. As with Windows PE capability
detection, Intezer Analyze also correlates detected capabilities to code genes
for ELFs. This means a similar approach can be applied as documented above when
it comes to analyzing code reuse and capabilities. Figure 5 shows the
capabilities detected for a RedXOR sample that we released a
[report](../../../2021/03/new-linux-backdoor-redxor-likely-operated-by-chinese-nation-state-actor/)
on earlier this year. From what we can see in the table under the TTPs tab, the
way the capabilities have been implemented is unique to RedXOR.

![](images/fig5.png "Figure 5: Capabilities of a RedXOR sample detected by capa.")

ELF malware are sometimes statically linked. This means that libraries used by
the malware are included in the binary. That way, the malware doesn't have to
depend on the infected machine to have the correct version of the library that
the malware expects, which allows the malware to run on multiple Linux
distributions. This can cause some detection of capabilities that are part of a
shared library and not the actual malware. This can be detected in the
capabilities table under TTPs in Intezer Analyze. Figure 6 shows some of the
capabilities detected for a malware called TSCookie. TSCookie is a backdoor
used by BlackTech and the Linux version was first
[reported](https://blogs.jpcert.or.jp/en/2020/03/elf-tscookie.html) on by
JPCert. The table shows the detection of "calculate modulo 256 via x86
assembly" and that the same genes are shared between BlackTech, Bifrost, and
libc. What this allows analysts to deduce is that the shared genes between
BlackTech and Bifrost exist because their malware has been statically linked
against the same libc version.

![](images/fig6.png "Figure 6: Some of the capabilities for a TSCookie sample.")

## Conclusion

Code gene analysis is a great way to find shared code between different malware
families. By correlating code genes to the capabilities of a file, you can add
another dimension to your malware analysis. In addition to Windows PE files,
Intezer Analyze now allows this to be performed on Linux ELF files. This was
possible by extending the functionality of an open-source project called capa.
We are grateful that FireEye open-sourced capa as it allowed us to extend its
functionality to support ELF files. As we are not only users of open-source, we
also enjoy providing improvements and bug fixes to open-source projects.
Additionally, we also have
[released some of our own](https://github.com/intezer). Check out FireEye's
blog post on capa v3.0 release
[here](https://www.fireeye.com/blog/threat-research/2021/09/elfant-in-the-room-capa-v3.html).
