# Knowledge Base

> Intezer’s Research Team guides to malware analysis and reverse engineering: triage, assembly, LNK, Office, PDF, MSI, ELF, .NET, memory and URL analysis.

- Source: https://research.intezer.com/docs/
- Published: 2025-10-02
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


A collection of practical guides to malware analysis and reverse engineering,
written by Intezer's Research Team. Each section starts with the fundamentals
of a format or technique and then works through real samples using open-source
and free tools.

- [Beginner's guide to malware analysis and reverse engineering](reverse-engineering/) -
  Start here. What malware analysis is, how to decide how deep an analysis
  needs to go, and the initial triage steps: identifying the format, checking
  whether the threat is known, spotting packing, and reading strings and the
  import table.
- [Malware Reverse Engineering for Beginners](malware_re_101/) - Assembly
  language, the CPU and the stack, disassemblers, and how malware is packed,
  obfuscated, and delivered.
- [How to Analyze](how-to-analyze/) - Format-by-format guides to the file types
  attackers use for delivery: LNK shortcuts, Microsoft Office documents, PDFs,
  phishing emails, and MSI installers.
- [Executable and Linkable Format (ELF) files](elf/) - The Linux executable
  format in depth: sections and segments, symbols, relocations, and dynamic
  linking.
- [ELF Malware Analysis 101](elf-malware/) - The Linux threat landscape,
  followed by hands-on static and dynamic analysis of ELF malware.
- [.NET Malware 101](dotnet/) - How managed code is compiled and executed, and
  how to read the metadata, tokens, and method bodies inside a .NET assembly.
- [Memory Analysis 101](memory/) - Finding fileless threats and injected code
  in volatile memory, and the forensic tools used to do it.
- [URL Analysis 101](url-analysis/) - How phishing URLs are manipulated to look
  trustworthy, and how to detect them at scale with machine learning.

## Pages in this section

- [Beginner's guide to malware analysis and reverse engineering](https://research.intezer.com/docs/reverse-engineering/): A beginner’s guide to malware analysis: setting the analysis goal, then initial triage of file format, known threats, packing, strings and the import table.
- [Malware Reverse Engineering for Beginners](https://research.intezer.com/docs/malware_re_101/): Reverse engineering basics for malware analysts with no RE experience: assembly, the CPU and stack, disassemblers, and how malware is packed and delivered.
- [How to Analyze](https://research.intezer.com/docs/how-to-analyze/): How to analyze the file types attackers use to deliver malware: LNK shortcuts, Office documents, PDFs, phishing emails and MSI installers, with free tools.
- [Executable and Linkable Format (ELF) files](https://research.intezer.com/docs/elf/): The Linux ELF executable format from the ground up: sections and segments, symbols, relocations and dynamic linking, and how malware authors abuse them.
- [ELF Malware Analysis 101](https://research.intezer.com/docs/elf-malware/): A training track for Linux malware: the ELF threat landscape, then static and dynamic analysis of real ELF samples with tools you can run on a Linux VM.
- [.NET Malware 101](https://research.intezer.com/docs/dotnet/): How .NET malware is compiled and run by the CLR, and how to navigate a .NET assembly’s metadata, tokens and method bodies with dnSpy, ILSpy and PEStudio.
- [Memory Analysis 101](https://research.intezer.com/docs/memory/): Memory forensics for fileless malware and injected code: what a memory image reveals, why manual analysis is hard, and the tools that make it practical.
- [URL Analysis 101](https://research.intezer.com/docs/url-analysis/): How phishing URLs are manipulated to look trustworthy, from cybersquatting to short URLs, and how to detect them at scale with machine learning.

