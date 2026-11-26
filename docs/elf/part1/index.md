# Sections and Segments

> How an ELF file is laid out: the ELF header, the section header table, the program header table, and how sections map into segments loaded into memory.

- Source: https://research.intezer.com/docs/elf/part1/
- Published: 2018-01-03
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


This marks the first of several blog posts that will focus on Executable and
Linkable Format (ELF) files. In this series, we'll introduce various topics,
ranging from the basics of ELF files built upon Linux
[malware technologies](https://intezer.com/blog/malware-analysis/analyzing-capabilities-in-PE-and-ELF-files/)
such as infection vectors, custom packer techniques and common malware
practices like dynamic API resolving techniques or ELF header crafting.

The overall goal of the series is to help both advanced and beginner Linux
users to acquire a sound knowledge of ELF files, along with improving their
understanding of the threat landscape in Linux systems.

Why focus on ELF? Our team is preparing for its flagship product, Intezer
Analyze, to be able to support these files. Because malware also strikes Linux
systems--and it can often happen through this file type--we thought it would be
a helpful place to start.

Here, we'll discuss the basics of ELF files in order to set a technical
foundation for readers with no previous experience on the subject. (Feel free
to skip ahead in the post if this isn't you!)

## Overview of The Executable and Linkable Format

The Executable and Linkable Format, also known as ELF, is the generic file
format for executables in Linux systems. Generally speaking, ELF files are
composed of three major components:

- **ELF Header**
- **Sections**
- **Segments**

Each of these elements play a different role in the linking/loading process of
ELF executables. We'll discuss each of these components and the relationship
between segments and sections. First, let's become familiar with the structure
of each constituent:

### ELF Header

The ELF header is denoted by an `Elfxx_Ehdr` structure. Mainly, this contains
general information about the binary. Definitions of these structure's fields
are the following:

![](images/fig1.png)

- `e_ident`: Array of 16 bytes containing identification flags about the file,
  which serve to decode and interpret the file's contents. Examples of these
  identification flags include:
  - `EI_MAG0-3`: ELF magic
  - `EI_CLASS`: File class.
  - `EI_DATA`: File's data encoding.
  - `EI_VERSION`: File's version.
  - `EI_OSABI`: OS/ABI identification.
  - `EI_ABIVERSION`: ABI version
  - `EI_PAD`: Start of padding bytes.
  - `EI_NIDENT`: Size of `ei_ident`.
- `e_type`: Type of executable.
- `e_machine`: File's architecture.
- `e_version`: Object file version.
- `e_entry`: Entry point of application.
- `e_phoff`: File offset of the Program Header Table.
- `e_shoff`: File offset of the Section Header Table.
- `e_flags`: Processor-specific flags associated with the file.
- `e_ehsize`: ELF header size.
- `e_phentsize`: Program Header entry size in Program Header Table.
- `e_phnum`: Number of Program Headers.
- `e_shentsize`: Section Header entry size in Section Header Table.
- `e_shnum`: Number of Section Headers.
- `e_shstrndx`: index in Section Header Table Denoting Section dedicated to
  Hold Section names.

In order to preview these fields for a given ELF binary, we can use any ELF
parser of choice. A common tool to quickly parse ELF files is the `readelf`
utility from GNU binutils.

In order to use `readelf` so that we can display the contents of the ELF header
for a given executable, we can use the following command:

```
readelf -h <executable>
```

### Sections

Sections comprise all information needed for linking a target object file in
order to build a working executable. (It's important to highlight that sections
are needed on linktime but they are not needed on runtime.) In every ELF
executable, there is a Section Header Table. This table is an array of
`Elfxx_Shdr` structures, having one `Elfxx_Shdr` entry per section. Definitions
of these structure's fields involve:

![](images/fig2.png)

- `sh_name`: index of section name in section header string table.
- `sh_type`: section type.
- `sh_flags`: section attributes.
- `sh_addr`: virtual address of section.
- `sh_offset`: section offset in disk.
- `sh_size`: section size.
- `sh_link`: section link index.
- `sh_Info`: Section extra information.
- `sh_addralign`: section alignment.
- `sh_entsize`: size of entries contained in section.

Some common sections are the following:

- `.text`: code.
- `.data`: initialised data.
- `.rodata`: initialised read-only data.
- `.bss`: uninitialized data.
- `.plt`: PLT (Procedure Linkage Table) (IAT equivalent).
- `.got`: GOT entries dedicated to dynamically linked global variables.
- `.got.plt`: GOT entries dedicated to dynamically linked functions.
- `.symtab`: global symbol table.
- `.dynamic`: Holds all needed information for dynamic linking.
- `.dynsym`: symbol tables dedicated to dynamically linked symbols.
- `.strtab`: string table of `.symtab` section.
- `.dynstr`: string table of `.dynsym` section.
- `.interp`: RTLD embedded string.
- `.rel.dyn`: global variable relocation table.
- `.rel.plt`: function relocation table.

In order to display sections using `readelf`, we can use the following command:

```
readelf -S <executable>
```

### Segments

Segments, which are commonly known as Program Headers, break down the structure
of an [ELF binary](../../elf-malware/part2/) into suitable chunks to prepare
the executable to be loaded into memory. In contrast with Section Headers,
Program Headers are not needed on linktime.

On the other hand, similarly to Section Headers, every ELF binary contains a
Program Header Table which comprises of a single `Elfxx_Phdr` structure per
existing segment. Definitions of these structure's fields are the following:

![](images/fig3.png)

- `p_type`: Segment type.
- `p_flags`: Segment attributes.
- `p_offset`: File offset of segment.
- `p_vaddr`: Virtual address of segment.
- `p_paddr`: Physical address of segment.
- `p_filesz`: Size of segment on disk.
- `p_memsz`: Size of segment in memory.
- `P_align`: segment alignment in memory.

There are a wide range of segment types. Some of common types are the following

- `PT_NULL`: unassigned segment (usually first entry of Program Header Table).
- `PT_LOAD`: Loadable segment.
- `PT_INTERP`: Segment holding `.interp` section.
- `PT_TLS`: Thread Local Storage segment (Common in statically linked
  binaries).
- `PT_DYNAMIC`: Holding `.dynamic` section.

Something important to highlight about segments is that only `PT_LOAD` segments
get loaded into memory. Therefore, every other segment is mapped within the
memory range of one of the `PT_LOAD` segments.

The following screenshot is a generic Segment layout for a dynamically linked
executable:

![](images/fig4.png)

In order to use `readelf` to display segment information we can execute the
following command:

```
readelf -l <executable>
```

### Sections and Segments

In contrast from other File formats, ELF files are composed of sections and
segments. As previously mentioned, sections gather all needed information to
link a given object file and build an executable, while Program Headers split
the executable into segments with different attributes, which will eventually
be loaded into memory.

In order to understand the relationship between Sections and Segments, we can
picture segments as a tool to make the linux loader's life easier, as they
group sections by attributes into single segments in order to make the loading
process of the executable more efficient, instead of loading each individual
section into memory. The following diagram attempts to illustrate this concept:

![](images/fig5.png)

Another important aspect of segments is that their offsets and virtual
addresses must be congruent modulo the page size and their `p_align` field must
be a multiple of the system page size.

The reason for this alignment is to prevent the mapping of two different
segments within a single memory page. This is due to the fact that different
segments usually have different access attributes, and these cannot be enforced
if two segments are mapped within the same memory page. Therefore, the default
segment alignment for `PT_LOAD` segments is usually a system page size.

The value of this alignment will vary in different architectures.
