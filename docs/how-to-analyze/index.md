# How to Analyze

> How to analyze the file types attackers use to deliver malware: LNK shortcuts, Office documents, PDFs, phishing emails and MSI installers, with free tools.

- Source: https://research.intezer.com/docs/how-to-analyze/
- Published: 2024-07-17
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Attackers rarely deliver a plain executable. They hide payloads inside the
everyday file types that pass through every inbox and endpoint: shortcuts,
documents, installers, and email attachments. Each format has its own
structure, its own abuse techniques, and its own set of tools for pulling it
apart.

This section walks through the file types most commonly used to deliver
malware, explaining the format first and then showing how to identify and
analyze malicious samples with open-source and free tools.

- [LNK Files](lnk/) - The Shell Link Binary format, the indicators that give
  away a malicious shortcut, and how to inspect one with `LnkParse3`.
- [Microsoft Office Files](office/) - OLE, OOXML, and RTF internals, plus
  macros, DDE abuse, template injection, and the known vulnerabilities
  attackers still exploit.
- [PDF Files](pdf/) - The four parts of the PDF structure, how JavaScript and
  embedded streams are used to deliver malware, and two worked examples using
  `peepdf` and `pdf-parser`.
- [Phishing Emails](phishing/) - Reading email headers, spotting spoofed
  senders, and inspecting the attachments and links that phishing campaigns
  rely on.
- [MSI Installer Files](msi/) - The MSI database format, the tables worth
  checking first, and a hands-on analysis of a malicious installer using
  `msitools` and `msidiff`.

## Pages in this section

- [LNK Files](https://research.intezer.com/docs/how-to-analyze/lnk/): How threat actors abuse Windows LNK shortcut files, how the Shell Link Binary format works, and how to analyze and detect malicious LNK files with LnkParse3.
- [Microsoft Office Files](https://research.intezer.com/docs/how-to-analyze/office/): How malicious Office files deliver malware through OLE, OOXML and RTF: macros, DDE abuse, template injection and known vulnerabilities, and how to analyze each.
- [PDF Files](https://research.intezer.com/docs/how-to-analyze/pdf/): How the PDF file format is abused to deliver malware, and how to investigate suspicious PDFs with free tools such as peepdf, with two malicious examples.
- [Phishing Emails](https://research.intezer.com/docs/how-to-analyze/phishing/): How to inspect phishing email files: sender addresses, headers, attachments and links, plus a real campaign that used a short URL and a packed payload.
- [MSI Installer Files](https://research.intezer.com/docs/how-to-analyze/msi/): How attackers abuse MSI installers and custom actions, how the MSI database is structured, and a hands-on malicious MSI analysis with msitools and msidiff.

