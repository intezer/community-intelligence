# Analyzing the .NET Executable File Structure

> Inside the .NET executable format: CLR compilation and runtime, metadata streams and tables, tokens, the manifest and method bodies, using the Sunburst sample.

- Source: https://research.intezer.com/docs/dotnet/exe-structure/
- Published: 2024-03-13
- Authors: Nicole Fishbein
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Welcome to our deep dive into the world of .NET malware reverse engineering. As
a security researcher or analyst, you're likely aware that the
[.NET framework](https://dotnet.microsoft.com/en-us/learn/dotnet/what-is-dotnet),
famed for its ability to enable rapid and robust application development, is a
double-edged sword. The same features that make it attractive to legitimate
developers also make it a favorite among malware authors.

So why invest time and effort in unraveling .NET malware? Simply put, the cyber
threat environment is filled with malware built using .NET frameworks, and
countering these threats necessitates a deep understanding of their underlying
code structures and the ability to analyze their operational intricacies. Armed
with knowledge and reverse engineering skills, you'll be able to unpack the
mechanics of malware, uncover its attack vectors, and strengthen defenses
against these threats.

This blog aims to demystify the process of .NET reverse engineering, making it
more approachable and comprehensible. Our journey will equip you with the
essential knowledge to analyze .NET files and gain insights into their harmful
functionality and behaviors. We'll start with an in-depth exploration of the
.NET executable format to provide you with a solid understanding of the
structure of these files. This will be followed by an extensive review of the
tools and techniques available for reverse engineering .NET malware, ensuring
you are well-prepared to tackle these threats.

## The .NET Framework

The .NET software development framework and ecosystem developed by Microsoft
was first released in 2002. It's designed to provide a controlled programming
environment where software can be developed, installed, and executed on
Windows-based operating systems. Although with the introduction of
[.NET Core](https://github.com/dotnet/core), now succeeded by .NET 5 and
onwards, it has expanded to offer cross-platform support for Linux and macOS.
Unlike traditional programming languages, .NET is not a language but a
framework supporting multiple managed languages, including C#, VB.NET, and F#.

The .NET framework includes a large class library known as the Framework Class
Library (FCL), which affords developers a comprehensive range of ready-to-use,
tested, and optimized functionality ranging from data access to encryption to
XML parsing. This extensive support and the managed execution environment make
.NET distinctive from languages and frameworks that require more manual
intervention for such tasks. Combining these features creates a productive
environment favored for a range of applications, from web to desktop to mobile.

## .NET Threat Landscape

Malware developers might prefer using .NET framework over C/C++ due to its
user-friendly development process, rich feature set, and smooth integration
with Windows. However, tools like dnSpy make it simpler to reverse engineer
.NET-based malware, prompting these creators to employ obfuscation methods to
make analysis harder. Additionally, .NET's capacity to enable malware to change
its behavior or hide makes it harder to detect and reverse-engineer. While
C/C++ allows for finer control over system resources and could lead to more
discreet and efficient malware, it requires a more in-depth knowledge of the
system's inner workings. This makes .NET a more appealing choice for those
looking for a balance between development speed and ease.

The .NET threat landscape continually evolves, with attackers regularly
exploiting the adaptable and widely adopted .NET framework to craft and deploy
a diverse array of sophisticated threats. This framework underpins many cyber
threats, like the notorious ransomware
[**Locky**](https://unit42.paloaltonetworks.com/unit42-powerware-ransomware-spoofing-locky-malware-family/)
and
[**Killnet**](https://www.acronis.com/en-eu/cyber-protection-center/posts/killnet-ransomware-a-wiper-from-the-chaos-family/).
Credential stealers, including
[**RedLine Stealer**](https://malpedia.caad.fkie.fraunhofer.de/details/win.redline_stealer),
and banking trojans, such as
[**CryptoClippy**](../../../blog/2023/05/cryptoclippy-evolves-to-pilfer-more-financial-data/)
are also .NET-based threats. Additionally, destructive wipers written in .NET
are emerging, with examples like
[**DoubleZero**](https://cert.gov.ua/article/38088) and the more recently
discovered
[**Hatef Wiper**](../../../blog/2023/12/stealth-wiper-israeli-infrastructure/)
illustrating this trend.

Moreover, .NET has also been instrumental in creating remote access trojans
(RATs). Examples include
[**QuasarRAT**](https://malpedia.caad.fkie.fraunhofer.de/details/win.quasar_rat)
and
[**NanoCore**](https://blog.talosintelligence.com/nanocore-netwire-and-asyncrat-spreading/),
praised in underground circles for their rich feature sets and the ease with
which they can be modified and obfuscated. Additionally, .NET is a common tool
for creating malware loaders, which discreetly install and execute other types
of malware.

## The Process of .NET Compilation and Runtime

### Compilation - Managed Code

The execution of managed languages (C#, F#, or VB.NET) is controlled by the
runtime. When the suitable language compiler compiles the source code, the
output is an **Intermediate Language (IL)**, also known as MSIL (Microsoft
Intermediate Language), **Managed Code**, or **Common Intermediate Language
(CIL)**.

For example, when C# code is compiled in the .NET framework, the output of the
C# compiler (`csc.exe`) is a .NET assembly. This assembly can be an executable
file (EXE) for standalone programs or a dynamic-link library (DLL) for reusable
libraries.

This is unlike the compilation of unmanaged languages like C/C++, where the
source code is translated directly to machine code. The
[managed code](https://learn.microsoft.com/en-us/dotnet/standard/managed-code)
is then packaged into an assembly accompanied by a manifest that contains the
necessary metadata.

The beauty of managed code in this process is its portability and flexibility;
the same assembly can run on any platform supported by .NET without
recompilation. Additionally, managed code allows for cross-language inheritance
code access security. It provides the advantage of late-binding support,
whereby method calls can be resolved at runtime rather than compile time. This
level of abstraction provided by the managed code and assembly structure is a
cornerstone of the versatility and strength of the .NET framework, enabling
developers to create applications that are more
[secure, manageable, and adaptable to change](https://learn.microsoft.com/en-us/dotnet/framework/get-started/overview#:~:text=.NET%20Framework%20consists,security%20and%20robustness).

The illustration below demonstrates the compilation and execution process in
.NET. We demonstrate it using C# but it is relevant to all .NET languages.

![](images/fig1.png "Compilation and execution of .NET.")

[Source](https://cihansol.com/blog/index.php/2021/07/28/decompiling-net-code/)

### Runtime Execution - The Common Language Runtime (CLR)

The **Common Language Runtime (CLR)** is a vital component of the Microsoft
.NET framework that manages the execution of .NET programs. It is essentially
the execution engine that provides a variety of services necessary for running
.NET applications, regardless of the programming language they're written in.

When a .NET binary is executed, the CLR sets up the execution environment, but
doesn't immediately translate all managed code into native machine code. The
**Just-In-Time (JIT)** compiler converts managed code to native code as needed,
compiling methods when they're called. This ensures efficient execution on
specific hardware. Additionally, technologies like
[NGEN](https://learn.microsoft.com/en-us/dotnet/framework/tools/ngen-exe-native-image-generator)
and
[AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/?tabs=net7%2Cwindows)
compilation in .NET Core and .NET 5+ can pre-compile managed code into native
code before execution, enhancing performance further.

The functionality of the
[CLR](https://learn.microsoft.com/en-us/dotnet/standard/clr) goes beyond the
execution of applications; it provides critical services such as
[memory management,](https://learn.microsoft.com/en-us/dotnet/standard/automatic-memory-management#:~:text=Automatic%20memory%20management%20is%20one%20of%20the%20services%20that%20the%20Common%20Language%20Runtime%20provides%20during%20Managed%20Execution.)
exception handling, garbage collection, type safety checks, and security.
Memory management, orchestrated by the CLR, abstracts the need for developers
to manually allocate and free memory, significantly reducing memory leaks and
related bugs. The provided automatic garbage collection manages the lifecycle
of objects, reclaiming memory by deallocating objects no longer used by the
application.

Furthermore, the CLR enforces strict type safety and helps ensure that an
application does not attempt to perform unsafe or unverified operations. It
also plays an important role in the security architecture of .NET, providing
Code Access Security (CAS), which controls what resources a program has access
to based on a trust level assigned to the application. Overall, the CLR creates
a high-level environment that reduces many low-level programming tasks required
by traditional programming languages, enabling faster development cycles,
increased productivity, and more secure and reliable applications. This makes
the CLR an indispensable component of the .NET ecosystem.

### Unmanaged Functions

Unmanaged functions in the .NET framework refer to code that operates outside
the managed environment of the Common Language Runtime (CLR). These functions
are typically written in languages like C or C++ and are directly compiled into
machine-specific code, bypassing the CLR's management. This means that features
such as automatic garbage collection, type safety, and exception handling,
intrinsic to the managed environment of .NET, are not applied to these
unmanaged functions. They are predominantly used for interoperability purposes,
allowing .NET applications to utilize legacy code or external libraries not
written in .NET-compatible languages. This capability is essential when there
is a need to use existing non-.NET libraries or to call system-level APIs that
are only accessible via unmanaged code.

However, this comes with added complexity and responsibility, as the developer
must manually handle memory management and error handling, increasing the
potential for issues like memory leaks and security vulnerabilities. In the
context of malware analysis, understanding unmanaged functions is crucial, as
they can be used to execute code that bypasses some of the safeguards of the
managed environment, posing unique challenges in analysis and detection.

### Example - Unmanaged Functions

To create a simple .NET program that uses an unmanaged function, we can use
Platform Invocation Services (PInvoke) in C#. PInvoke allows managed code to
call unmanaged functions from dynamic link libraries (DLLs).

Here's an example where we'll call the `MessageBox` function from the
`user32.dll`, a standard Windows library:

```c#
using System;
using System.Runtime.InteropServices;

class Program
{
    // Importing the MessageBox function from user32.dll
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    public static extern int MessageBox(IntPtr hWnd, String text, String caption, uint type);

    static void Main(string[] args)
    {
        // Calling the MessageBox function - this is an unmanaged function call
        MessageBox(new IntPtr(0), "Hello, World!", "Message Box", 0);
    }
}
```

Let's explore the flow of the code above:

1. We import the `MessageBox` function from `user32.dll` using the `DllImport`
   attribute. This is necessary to tell the CLR that this function is external
   and not managed by .NET's runtime.
2. The `MessageBox` function is declared with the same signature as the
   unmanaged function in the `user32.dll`.
3. In the `Main` method, we call the `MessageBox` function. The parameters
   include a window handle (we pass `IntPtr(0)` as we don't have a window
   handle), a text message, a caption, and the type of message box (0
   represents a simple OK button box).
4. When you run this program, it will display a message box with the text
   "Hello, World!" and the caption "Message Box".

### .NET Assembly

The
[.NET assembly](https://learn.microsoft.com/en-us/dotnet/standard/assembly/) is
the fundamental building block of a .NET application, serving as a collection
of one or more code modules or resource files. The contents of the generated
assembly include:

- [Intermediate Language (IL) Code](https://learn.microsoft.com/en-us/dotnet/standard/managed-code#intermediate-language--execution):
  IL is a CPU-agnostic instruction set that enables the same assembly to be
  executed on different platforms supported by the .NET framework.
- [Metadata](https://learn.microsoft.com/en-us/dotnet/standard/metadata-and-self-describing-components):
  It describes the structural elements managed by the CLR, such as assemblies,
  types (classes, interfaces, enums, structs), methods, and more. This includes
  information required for debugging, garbage collection, security attributes,
  and details necessary for the runtime to manage the code.
- [Manifest](https://learn.microsoft.com/en-us/dotnet/standard/assembly/manifest):
  is a specific part of the metadata that describes the assembly itself. It
  includes the assembly's name, version, culture, and potentially a strong name
  to uniquely identify the assembly. While metadata describes the contents
  within an assembly, the manifest provides a high-level overview of the
  assembly as a whole, ensuring the assembly interacts with the correct
  versions of other assemblies it depends on.

## Diving Into the .NET Executables Format

In this section, we will dive into the .NET internals. To demonstrate the
concepts we cover, we will use an
[example](https://analyze.intezer.com/files/32519b85c0b422e4656de6e6c41878e95fd95026267daab4215ee59c107d6c77) -
the notorious Sunburst (SolarWinds) so you can follow along. Hash:
`32519b85c0b422e4656de6e6c41878e95fd95026267daab4215ee59c107d6c77`

We will use [dnSpy](https://github.com/dnSpy/dnSpy),
[ILSpy](https://github.com/icsharpcode/ILSpy), and
[PEStudio](https://www.winitor.com/download) in our demonstrations.

The runtime header in the context of .NET assemblies is an essential element
within the Portable Executable (PE) file format designated for use by the CLR.
It holds metadata and critical details for CLR to execute the .NET assembly
properly. This runtime header is the 15th data directory entry in the PE header
and is called the "**CLR Runtime Header**".

The Data Directory within a PE file acts as an index or table of contents,
listing vital sections and providing information about their locations and
sizes. This structure provides efficient access to different parts of the PE
file, such as import and export tables, resources, and, notably, the CLR
runtime header for .NET assemblies.

This entry delineates the runtime header's Relative Virtual Address (RVA) and
its size, thereby guiding the CLR to this header for managing the .NET
assembly's execution upon loading.

The screenshot below shows the 15th data directory entry identified as .NET.

![](images/fig2.png "Analysis of a .NET file in PeStudio.")

Following the section, we get into the **metadata header**, which plays a
crucial role in organizing these streams. It provides a directory of sorts,
listing each stream, its size, and its offset within the metadata section. When
the CLR or a tool like dnSpy or ILDasm needs to access a piece of metadata, it
consults the metadata header to find the appropriate stream. Then, it navigates
to the correct position within that stream to read the data. Next, let's some
key fields in the content of the .NET metadata header:

![](images/fig3.png)

- The signature is always the same: `BSJB` (`0x42534a42`).
- The GUID is a 128bits long unique identifier.
- IL-Only: This flag indicates that the assembly contains only Intermediate
  Language (IL) code and no native CPU-specific code. It is possible for a PE
  to contain both managed and unmanaged code.
- 32-Bit Required: This flag, when set, indicates that the assembly requires a
  32-bit runtime environment even if it's running on a 64-bit OS. It's often
  used for assemblies that rely on 32-bit native dependencies or specific
  behaviors of the 32-bit runtime.
- Strong Name Signed: Indicates whether the assembly has been signed with a
  strong name. Strong naming involves using a public/private key pair to sign
  an assembly, providing a unique identity and ensuring that the assembly
  hasn't been tampered with.
- **Streams** refer to structured data segments that contain specific types of
  metadata. The key streams within a .NET assembly's metadata include
  - `#~` (Tilde) Stream: The main metadata stream contains the **metadata
    tables**. These tables store information about the types, methods, fields,
    parameters, and other elements defined in the assembly. The tilde stream is
    structured according to the metadata table schema defined by the CLI
    specification.
  - `#Strings` Stream: This stream stores strings used by the metadata, such as
    type names, method names, and field names. References in the metadata
    tables (in the `#~` stream) point to offsets within this stream for the
    actual string values.
  - `#US` (User Strings) Stream: This stream holds literal string values used
    in the assembly, such as default values for string variables or string
    constants in code. The metadata references these strings, particularly in
    the instructions for loading string literals.
  - `#GUID` Stream: Contains GUIDs (Globally Unique Identifiers) used by the
    assembly. Each entry in this stream is a GUID used to uniquely identify
    certain aspects of the metadata, like the module version ID (MVID).
  - `#Blob` Stream: The "Binary Large Object" stream stores binary data for
    various purposes, such as default values for fields, method signatures,
    property signatures, and marshaling information. Items in the metadata
    tables reference this stream for detailed binary information.
  - `#Pdb` Stream: This optional stream contains debugging information that
    correlates the metadata and IL code to source code lines and files.

It might be tricky to access the metadata of an executable in some .NET
assembly viewers - because they are represented in different ways. The
screenshots below represent the difference between dnspy and ILspy.

![](images/fig4.png "The metadata tables in ILSpy.")

![](images/fig5.png "The metadata tables in dnspy.")

### Metadata

The metadata in .NET is a set of binary information describing the programmatic
elements and their characteristics. This includes information about the types
defined in the code (classes, interfaces, enums, etc.), member definitions
(methods, properties, fields, events), references to other types and members,
and the assembly itself.

Metadata is divided into several tables, collectively known as **Metadata
Tables**, within the PE file that store this descriptive information. Each
table adheres to a specific schema that outlines the structure and nature of
the data it contains. Here are some of the key types of information you can
find in the metadata tables:

**Definition Tables**: Contain information about the code defined within the
current assembly. This includes:

- *`TypeDef` Table*: Details of each class or interface defined in the source
  code, including its name, visibility, base type, and the methods or
  properties it contains. Key fields include:
  - `TypeName`: The name of the type.
  - `TypeNamespace`: The namespace the type belongs to.
  - `BaseType`: An index into the `TypeDef`, `TypeRef`, or `TypeSpec` table,
    indicating the base class of the type.
  - `Flags`: Describes type attributes (visibility, abstract/sealed status,
    etc.).
- *`MethodDef` Table*: Details of each method, including its name, signature
  (parameter and return types), and the IL code associated with it. Key fields
  include:
  - `Name`: The name of the method.
  - `Signature`: A blob index pointing to the method's signature, which
    includes its calling convention, return type, and parameters.
  - `RVA`: Relative Virtual Address, pointing to the method's implementation in
    the PE file.
- *`FieldDef` Table*: Details of each field (class variables), including its
  name and type. Key fields include:
  - `Name`: The name of the field.
  - `Signature`: A blob index that points to the field's type signature.
  - `Flags`: Specifies field attributes like visibility, static/instance, and
    init only (readonly).

**Reference Tables**: Contain code information external to the assembly but
referenced by it. This includes:

- *`TypeRef` Table*: Information about types defined in other assemblies that
  the current assembly references.
- *`MemberRef` Table*: Descriptions of members (methods, properties, etc.)
  defined in another module or assembly.

**Manifest Metadata Table**: Describes the assembly itself, including:

- *`Assembly` Table*: Information about the assembly, such as its name,
  version, culture, and strong name signature.
- *`AssemblyRef` Table*: Details of other assemblies that this assembly depends
  on, including their names, versions, and public keys if they are strongly
  named.
- *Other Metadata Tables*: Apart from the above, there are several other tables
  storing information like:

**`Module` Table**: Information about the current module, such as its name and
the GUID (Globally Unique Identifier) that identifies it uniquely.

**`CustomAttribute` Table**: Contains details of custom attributes applied to
various elements within the assembly.

**`Event` Table and `Property` Table**: Describe the events and properties
declared in the types.

**`Param` Table**: Information about the parameters of the methods.

`StandAloneSig` Table: Standalone signatures can be used to encapsulate a type
or method signature.

**`Constant` Table**: Stores the constants defined in the code.

These tables are essential for the operation of the CLR as they provide the
contextual information required to execute the assembly. They are read during
runtime to perform various tasks such as type instantiation, method invocation,
security verification, and more.

These metadata tables are encoded in a highly optimized binary format that is
efficiently processed by the runtime. Through reflection, the metadata can also
be accessed programmatically, allowing .NET applications to examine their own
structure or the structure of other assemblies at runtime. This introspective
capability is one of the powerful features of the .NET framework, enabling a
range of dynamic programming scenarios.

### Metadata Unique Identifier (ID)

Metadata tokens are unique identifiers the CLR uses to reference metadata
elements within an assembly's metadata tables. Each entry within these tables
is assigned a metadata token, which serves as a stable reference to that
particular item.

Metadata tokens are essential for the CLR to interact with compiled code
because they provide a means for the runtime to identify and access metadata
efficiently. Every type, member, signature, or other metadata descriptor within
the PE file has a corresponding token.

In dnSpy each method contains a comment right above its declaration, the
information includes the token, RID, RVA and a file offset - as can be seen in
the screenshot below.

![](images/fig6.png)

Let's examine the meaning of these fields:

1. Token: The high byte (big-endian) of the metadata token specifies the type
   of metadata for ease of identification by the runtime. It indicates which
   table in the metadata the token refers to (`TypeDef`, `TypeRef`,
   `MethodDef`, etc). Refer to Appendix A for more information about the values
   of the token type.
2. Row Index (RID): The remaining 24 bits of the metadata token are used to
   index the relevant metadata table. They indicate the row number where the
   actual metadata for this element can be found. Since each table could have
   millions of entries, 24 bits allow for an ample range of indices.
3. RVA: The RVA is the address of the method's body (its compiled IL code)
   relative to the base address at which the assembly is **loaded into
   memory**. For instance, `0x00023B28` means that the method's IL code starts
   at that memory offset from the base address of the loaded module. The CLR
   uses the RVA to locate and execute the method's code at runtime. In the
   context of a PE (Portable Executable) file, which is the format used for
   .NET assemblies on Windows, RVAs are used extensively to reference various
   parts of the file when it's loaded into memory.
4. File Offset: This value represents the position of the method's IL code
   within the actual .NET assembly file (the `.dll` or `.exe` file) **on
   disk**. `0x00021D28` indicates the offset in bytes from the beginning of the
   file to where the method's code starts. This is useful for direct binary
   analysis or manipulation of the assembly file, as it tells you exactly where
   in the file the method's code can be found.

The metadata tokens' structure enhances the CLR's performance when resolving
the references at runtime. For example, when the JIT compiler needs to compile
IL to native code, it uses metadata tokens to look up method signatures, type
information, and more. The metadata token system also supports the dynamic
features of the CLR, like reflection. It enables various runtime services such
as type safety, security checks, and cross-language interoperability to be
carried out effectively.

### Example of the Metadata Tables

Below is the *start* function of the SolarWinds malware.

![](images/fig7.png "The token and RID field in of the start function in dnspy.")

The higher bits of the **token** value (`0x6`) correspond to the metadata table
number 6, which is the Method (`MethodDef`) table. The lower part of the token
is `0x5fa` (1530), which is the entry number in the Method table.

![](images/fig8.png)

Examining the lower part of the metadata of the *start* method, the value
`0x00058D66` is the offset from the beginning of the executable, the value of
the offset (`0x1EC15`) is the offset in the `#String` stream, which will
contain the method name: *Start*. Let's see how it looks like in the Hex editor
in dnSpy:

![](images/fig9.png "The value of the offset in the String stream. DnSpy automatically detects the value of the string.")

To view the data in the Strings stream, we will do the following:

![](images/fig10.png)

In the new window, go to the offset (relative to the start of the String Stream
- RVA) and see the string we were looking for - *Start*.

![](images/fig11.png "The offset in the string Start in the Strings stream.")

### Manifest

The .NET manifest is a critical component of .NET assemblies, serving as the
metadata hub that describes how the elements within the assembly relate to each
other. It is embedded in every assembly, whether it is a static or dynamic one,
and contains essential data needed for the assembly's operation, including its
version requirements, security identity, scope definition, and resolution of
references to resources and classes.

The primary function of the .NET manifest is to provide a comprehensive
metadata description that facilitates the assembly's identification,
versioning, and dependency management. It ensures that the assembly is
self-describing, aiding in the resolution of type references and the mapping of
these references to the files containing their declarations and
implementations. This is especially crucial for maintaining version control and
ensuring compatibility between different assemblies and the components that
depend on them.

### Contents of the .NET Manifest

The manifest includes a variety of information critical to the assembly's
identity and operation:

- **Assembly Name**: A text string that specifies the assembly's name.
- **Version Number**: Includes major and minor version numbers, along with
  revision and build numbers, used by the common language runtime to enforce
  version policy.
- **Culture Information**: Specifies the culture or language the assembly
  supports, particularly important for satellite assemblies containing culture-
  or language-specific information.
- **List of Files in the Assembly**: Includes a hash of each file contained
  within the assembly and their names, ensuring the integrity and completeness
  of the assembly.
- **Type Reference Information**: Used by the runtime to map a type reference
  to the file containing its declaration and implementation, crucial for type
  safety and correctness.
- **Information on Referenced Assemblies:** Lists other assemblies that the
  current assembly statically references, including their names, metadata (like
  version, culture, operating system), and public keys if they are
  strong-named.

![](images/fig12.png "Manifest in dnSpy.")

### Method Body Structure

In .NET, the method body structure can be encoded in two formats known as the
"Tiny" format and the "Fat" format, each serving different purposes based on
the method's complexity and requirements. The choice between Tiny and Fat
headers is made by the .NET compiler based on the complexity of the method
being compiled.

**The Tiny header** is the simpler of the two formats. The tiny header is used
when a method meets specific criteria:

- The method is smaller than 64 bytes.
- Its stack depth won't exceed 8 slots (one slot for each item on the stack,
  regardless of the item's size).
- It contains no local variables or structured exception handlers
  ([SEHs](https://learn.microsoft.com/en-us/archive/msdn-magazine/2003/september/write-msil-code-on-the-fly-with-the-net-framework-profiling-api)).

The tiny header is more compact and optimized for small methods.

The Tiny header is a single byte long, with the lower 2 bits set to `0x2`
(binary `10`), indicating that it's a Tiny header, and the remaining 6 bits
indicate the size of the method body in bytes. This compact format allows for
efficient storage of small method bodies, reducing the metadata overhead for
simple methods.

| Field       | Size (Bits) | Description                                          |
| :---------- | :---------- | :--------------------------------------------------- |
| Header Flag | 2           | Always set to `10`(b) to indicate a Tiny header.     |
| Method Size | 6           | Specifies the size of the method body in bytes (max 63 bytes). |

**The Fat header** is used for more complex method bodies that exceed the
limitations of the Tiny header:

- The fat header is used for larger methods that don't meet the criteria for
  the tiny header.
- It provides additional information, such as the method's local variables and
  SEHs.
- When a method exceeds the size or complexity limits of the tiny header, it
  uses the fat header format.

The Fat header is larger and consists of multiple fields, including a flags
field to indicate additional characteristics of the method body (such as
exception handling clauses or local variable initializations).

| Field           | Size (Bits) | Description                                    |
| :-------------- | :---------- | :--------------------------------------------- |
| `Flags`         | 2           | Specifies attributes of the method body, including the presence of local variables, init locals, and more. The lowest bit (`0x3`) indicates a Fat header when set to 1. |
| `Size`          | 2           | Size of the header in 4-byte words. Includes the size of the entire header, not just the method body. |
| `MaxStack`      | 2           | Maximum number of items on the operand stack at any point during method execution. |
| `CodeSize`      | 4           | Size of the method body's IL code, in bytes.   |
| `LocalVarSigTok`| 4           | Metadata token for the signature of local variables, present only if the method has local variables. |
| More sections   | Variable    | Additional sections for things like exception handling clauses, present if specified in the `Flags`. |

For more details about the structure of these headers you can check
[this](https://www.ntcore.com/files/dotnetformat.htm#:~:text=the%20Method%20format.-,Methods,-Every%20method%20contained)
resource.

### Example of Method Body

Let's look at the function `DeleteDiscoveryProfileInternal` for example:

![](images/fig13.png)

Clicking on the offset (or the RVA) will lead us to the header of the method in
the HEX view window:

![](images/fig14.png)

And here we can see DnSpy highlights the header fields once we hover with the
mouse over the bytes of the header. The content of the function - the
instructions follow the header and represented in dnSpy as
`image_core_ilmethod_fat.instruction[]`. To better understand and view the
instructions' values (opcodes) we will open the malware in IDA:

![](images/fig15.png "The function in IDA.")

On the left we can see the value of each opcode, followed the first 2
instructions in dnSpy:

![](images/fig16.gif)

## Conclusions

In this initial exploration of .NET executable file structures, we've delved
into the complexities of the .NET framework, highlighting its dual use for both
legitimate development and malware creation. By dissecting the .NET compilation
process, runtime execution, and the intricate details of metadata and
assemblies, we've laid the groundwork for understanding how .NET applications
function and how they can be manipulated for malicious purposes. This is the
foundation to get you started gaining the skills and knowledge to analyze and
counteract .NET-based threats effectively, as you delve deeper into the art of
reverse engineering .NET malware.

### Appendix

**A table of token types**

| Token Type               | Value (Hex) | Description                          |
| :----------------------- | :---------- | :----------------------------------- |
| `Module`                 | `0x00`      | References a module definition.      |
| `TypeRef`                | `0x01`      | References a type in another module. |
| `TypeDef`                | `0x02`      | Defines a type within the module.    |
| `FieldDef`               | `0x04`      | Defines a field within a type.       |
| `MethodDef`              | `0x06`      | Defines a method within a type.      |
| `ParamDef`               | `0x08`      | Defines a parameter for a method.    |
| `InterfaceImpl`          | `0x09`      | Defines interface implementations for a type. |
| `MemberRef`              | `0x0A`      | References a field or method in another module. |
| `CustomAttribute`        | `0x0C`      | Defines a custom attribute.          |
| `Permission`             | `0x0E`      | Defines declarative security permissions. |
| `Signature`              | `0x11`      | Defines a standalone signature.      |
| `Event`                  | `0x14`      | Defines an event within a type.      |
| `Property`               | `0x17`      | Defines a property within a type.    |
| `ModuleRef`              | `0x1A`      | References an external module.       |
| `TypeSpec`               | `0x1B`      | Specifies a type using a signature.  |
| `Assembly`               | `0x20`      | Defines assembly metadata.           |
| `AssemblyRef`            | `0x23`      | References another assembly.         |
| `File`                   | `0x26`      | Defines an external file associated with the assembly. |
| `ExportedType`           | `0x27`      | Defines a type exported from another assembly. |
| `ManifestResource`       | `0x28`      | Defines an embedded resource.        |
| `GenericParam`           | `0x2A`      | Defines a generic parameter for a type or method. |
| `MethodSpec`             | `0x2B`      | Specifies a method instantiation for a generic method. |
| `GenericParamConstraint` | `0x2C`      | Specifies constraints on a generic parameter. |
