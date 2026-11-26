# Executable and Linkable Format (ELF) files

> The Linux ELF executable format from the ground up: sections and segments, symbols, relocations and dynamic linking, and how malware authors abuse them.

- Source: https://research.intezer.com/docs/elf/
- Published: 2019-06-25
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


The Executable and Linkable Format is the generic file format for executables
on Linux systems. Understanding how an ELF file is put together is the
foundation for everything else in Linux malware analysis, from spotting a
packed binary to following a hooked library call.

This series builds that foundation from the ground up, covering the structures
the linker and loader rely on and the ways malware authors abuse them.

- [Sections and Segments](part1/) - The ELF header, the section header table,
  the program header table, and how sections map into the segments that get
  loaded into memory.
- [Symbols](part2/) - What symbolic information is, the `Elfxx_Sym` structure,
  and how the `.symtab` and `.dynsym` symbol tables relate to their string
  tables.
- [Relocations](part3/) - How symbolic references are connected to their
  definitions, the two relocation entry structures, and the x86 and x86_64
  relocation types.
- [Dynamic Linking](part4/) - Static versus dynamic linking, what the dynamic
  linker does at load time, lazy linking through the PLT and GOT, and the
  `LD_PRELOAD` feature malware uses to hook functions.

## Pages in this section

- [Sections and Segments](https://research.intezer.com/docs/elf/part1/): How an ELF file is laid out: the ELF header, the section header table, the program header table, and how sections map into segments loaded into memory.
- [Symbols](https://research.intezer.com/docs/elf/part2/): What symbolic information is in ELF files, the ELF symbol structure, and how the .symtab and .dynsym symbol tables relate to their string tables.
- [Relocations](https://research.intezer.com/docs/elf/part3/): How ELF relocations connect symbolic references to their definitions, the structure of relocation entries, and the x86 and x86_64 relocation types.
- [Dynamic Linking](https://research.intezer.com/docs/elf/part4/): How dynamic linking works in ELF files: static versus dynamic linking, what the dynamic linker does at load time, and lazy linking through the PLT and GOT.

