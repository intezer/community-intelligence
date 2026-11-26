# Memory Analysis 101

> Memory forensics for fileless malware and injected code: what a memory image reveals, why manual analysis is hard, and the tools that make it practical.

- Source: https://research.intezer.com/docs/memory/
- Published: 2024-04-23
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Some threats never touch the disk. Fileless malware, injected shellcode, and
in-memory implants leave nothing for a file scanner to find, which makes
volatile memory the only place the evidence exists. Memory forensics is how you
recover it.

This section covers what can be pulled out of a memory image, why doing it by
hand is difficult, and which tools make the job practical.

- [Understanding Memory Threats and Forensic Tools](tools/) - What memory
  forensics is and the data it recovers, how in-memory threats differ from
  on-disk ones, the technical and logistical challenges of manual analysis, and
  a worked example using WinPmem, the Volatility Framework, and YARA to find a
  Cobalt Strike beacon in a memory image.

## Pages in this section

- [Understanding Memory Threats and Forensic Tools](https://research.intezer.com/docs/memory/tools/): Memory forensics for in-memory threats: what it recovers, why manual analysis is hard, and finding a Cobalt Strike beacon with WinPmem, Volatility and YARA.

