# Technical Analysis of a Novel IMEEX Framework

> IMEEX is a custom Windows DLL framework seen targeting Djibouti and Afghanistan, with remote control features and signs of infrastructure reuse with ShadowPad.

- Source: https://research.intezer.com/blog/2024/10/technical-analysis-of-a-novel-imeex-framework/
- Published: 2024-10-10
- Authors: Nicole Fishbein
- Tags: framework, apt, china, apt_china
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


The IMEEX framework is a newly discovered, custom-built malware designed to
target Windows systems. Delivered as a 64-bit DLL, it offers attackers
extensive control over compromised machines. This framework is notable for its
robust capabilities, featuring a wide array of functionalities, including
execution of additional modules, file manipulation, process management,
registry modification, and remote command execution. It also performs system
reconnaissance by gathering critical system information, such as hostname,
operating system version, volume serial number, and system language, and relays
this data to its command-and-control (C2) server.

Our analysis indicates that the IMEEX framework has primarily targeted
[Djibouti](#imeex-targeting-djibouti), with a less capable variant appearing
targeting [Afghanistan](#imeex-targeting-afghanistan). At this time, we are
unaware of the wider distribution of the campaign but assume it's not limited
to these two countries. While the infection vector remains unclear, the
behavior of the malware suggests it is part of a carefully orchestrated attack
campaign aimed at maintaining persistence and avoiding detection through
advanced techniques such as masquerading as legitimate processes, mutex
creation, and encrypted communications with the C2 server.

Interestingly, there are signs of infrastructure reuse with ShadowPad, a
modular malware platform privately sold and employed by multiple Chinese threat
actor groups. This overlap might indicate a strategic rotation from ShadowPad
to the IMEEX framework, although it does not conclusively link the campaign to
any specific group. Instead, it suggests a possible evolution in tactics, where
the attackers leverage IMEEX as an alternative toolset.

This blog aims to provide a detailed technical analysis of the IMEEX
framework's capabilities. We will explore its core functionalities, ranging
from system reconnaissance and data collection to dynamic module loading and
execution of additional modules.

## Key Findings about IMEEX

- Geographic Distribution: The IMEEX framework samples were submitted to
  VirusTotal from Djibouti and Afghanistan, indicating potential targeting or
  deployment in those regions. This geographic pattern suggests a focused
  campaign, with some variations between samples in terms of functionality.
- Masquerading as Legitimate Processes: IMEEX framework expects to be loaded by
  svchost.exe or rundll32.exe, ensuring it blends with legitimate Windows
  processes to avoid detection.
- Configuration Decryption: IMEEX framework extracts its configuration from the
  last 108 bytes of its executable. The decrypted data includes C2 domains and
  a port number, usually 443.
- Command and Control (C2) Communication: The malware communicates with its C2
  server via custom structured packets, including checksums and identifiers for
  data integrity. It uses port 443, blending in with legitimate HTTPS traffic,
  over TLS.
- Advanced File, Process, and Registry Operations: IMEEX framework supports a
  wide range of operations, including file manipulation, process control,
  remote code execution, and extensive Windows Registry modifications.
- Modular Execution: The framework employs a modular approach, loading and
  executing components on-demand. The malware uses the Windows Registry to keep
  track of its modules, ensuring each is executed only once and monitoring
  their state throughout execution.
- Persistence and Evasion: By modifying registry keys and leveraging scheduled
  deletions for locked files, the malware maintains persistence while evading
  detection.

## Technical Overview

The malware checks the context in which its DLL is loaded by verifying if it is
running under legitimate system processes like svchost.exe or rundll32.exe.
Based on the process name, it either proceeds to establish network connections,
or terminates to avoid detection. This behavior suggests an attempt to and
blend in with trusted Windows processes, enhancing stealth and persistence.

The malware creates a mutex named either Global4B59AFCC or Global3796878E. This
mutex ensures that only one instance of the malware runs on the infected system
at any given time, preventing conflicts and redundant resource usage.

### Sample Analysis

We analyzed four samples, and most of their functions remained consistent
across versions. The samples analyzed include:

- `3e25798da0232d9039e570fb34d4bdccf7f082fa38b486a097d954f5f3debab3`
- `7a3113d3212605a33924ad16ab360b7d48cc94de0de1c1cf9dc44695d4a01648`
- `7d02ad54e4e56f34e59414f9b02397901fc61bb1158a31ab2586fe62564aeb93`
- `94b8a01ad4b53d202984afb6781d7f88cb5cd329349791516e985ea88e08ad66`

Four files were submitted to VirusTotal from Djibouti. During the
investigation, we found a similar framework variant from Afghanistan with fewer
capabilities, which we will also cover.

The files from Djibouti are nearly identical, with only minor differences in
mutex names and C2 domains. The consistent codebase and minimal changes suggest
they are variants deployed within the same campaign. The primary differences
lie in the names of the mutexes and the process names the framework uses for
masquerading.

### Configuration Extraction and Decryption

The framework extracts its configuration data from the last 108 bytes of its
own executable file. This data is encrypted using a custom XOR-based algorithm.
The decryption process involves:

- Read the encrypted data from the executable's end.
- Perform a reverse iteration through the data, XOR-ing each value with the
  value that precedes it
- Validate the decrypted data with the magic value `0xFEFCFDFB`.
- The first 2 bytes following the magic are the value of the port – 443.
- The rest of the bytes are XOR-ed with the value `0xE6` giving the decrypted
  domains,separated by a semicolon.

Appendix B includes Python configuration extractors for all variants observed.

We extracted the following C2 domains and addresses from the samples we
analyzed:

- `yurtumawat.wwwhost[.]us`
- `erkinhorshiden.onedumb[.]com`
- `45.141.139[.]146`

### Network Initialization and Operation

IMEEX initializes network communication by establishing a socket connection to
decrypted C2 server addresses. It attempts to connect over TCP port 443, likely
evading network monitoring by blending with legitimate encrypted web traffic.
Notably, the framework uses dynamic DNS providers for its command-and-control
servers.

The framework employs checksums and specific identifiers in its communication
protocol to ensure data integrity and accurate command execution. The use of
structured headers and data packets allows for a modular approach to command
handling and response generation.

Key Features:

- Checksums: Calculated to verify that data has not been tampered with during
  transmission. Each packet includes a checksum calculated over the header.
- IDs and Sub-IDs: Numeric identifiers specify the exact command or response
  type, ensuring proper interpretation of exchanged information between the
  malware and the C2 server. The response packets also use a unique ID to
  identify the operation that is sending the response.
- Error Handling: The malware sends error responses when operations fail,
  allowing the C2 server to handle exceptions or retry commands.

#### C2 Packet Data Structure

Communication between the malware and the C2 server uses specific packet
formats for both requests and responses. Each packet contains a header and a
data section, adhering to predefined structures to facilitate various
operations.

Structure of packets received from the C2:

| Packet | Offset | (Bytes)    | Size (Bytes)                                                    | Field Description                           |
| ------ | ------ | ---------- | --------------------------------------------------------------- | ------------------------------------------- |
| Header | 0-3    | 4          | Unspecified Operation-specific data.                            |                                             |
|        | 4-7    | 4          | Class ID                                                        | Specifies the main operation to perform.    |
|        | 8-11   | 4          | Command ID                                                      | Additional identifier for sub-commands.     |
|        | 12-15  | 4          | Checksum                                                        | Validates the integrity of the header data. |
| Data   | 16+    | Parameters | Contains parameters or data required for executing the command. |                                             |

Structure of packets that are sent to the C2:

| Packet | Offset (Bytes) | Size (Bytes) | Field              | Description                                 |
| ------ | -------------- | ------------ | ------------------ | ------------------------------------------- |
| Header | 0-3            | 4            | Packet size        | Specifies the size of the packet.           |
|        | 4-7            | 4            | ID                 | Identifies the operation or response.       |
|        | 8-11           | 4            | Checksum           | Validates the integrity of the header data. |
| Data   | 12+            |              | Command parameters | Contains the actual data or payload.        |

The ID field encodes a combination of the Class ID, Command ID, and an optional
flag in big-endian format, serving as a unique identifier to correlate specific
responses from the framework with the operations that generated them.

### Reconnaissance

Before initiating its main thread, which manages communication with the C2
server and executes various operations, the framework performs initial
reconnaissance on the infected endpoint. It collects system information and
sends this data to the C2 server. The information gathered includes:

- **System ANSI Code Page Identifier**: Obtained using the
  [GetACP](https://learn.microsoft.com/en-us/windows/win32/api/winnls/nf-winnls-getacp)
  function.
- **File System and Volume Information**: Retrieves details about the file
  system and volume of the current working directory.
- **Hostname**: The network name assigned to the host machine.
- **Operating System Version**: The version number and build information of the
  host's operating system.
- **IP Address**: The current IP address assigned to the host.

This collected information is sent to the C2 server with an identifier value of
`0x88`.

### Supported Commands by the Framework

The IMEEX framework supports a wide array of commands, enabling comprehensive
control over infected systems. These commands are categorized into several
classes, such as file management, process manipulation, remote command
execution, and registry operations. The RAT's functionality includes reading,
writing, deleting files and directories, enumerating and terminating processes,
and remotely executing programs and commands on the compromised machine. It
uses a structured communication protocol with custom packet headers to manage
command exchanges with its C2 server.

In addition to basic system control, the framework supports Windows Registry
operations, allowing it to write, update, and delete registry values—critical
for maintaining persistence and managing module execution. These capabilities
make it highly versatile for remote administration and execution of malicious
tasks.

| Command Class                        | Class ID | Command ID | Command                              | Extra Parameters              |
| ------------------------------------ | -------- | ---------- | ------------------------------------ | ----------------------------- |
| File Operation                       | 3        | 1          | Get Drive Info                       | –                             |
|                                      |          | 2          | File/Directory Discovery             | File path                     |
|                                      |          | 3          | Write File                           | File path                     |
|                                      |          | 4          | Read File                            | File path                     |
|                                      |          | 5          | No Operation/Reserved                |                               |
|                                      |          | 6          | Delete folder and content            | File path                     |
| Process Operation                    | 5        | 1          | Get Processes Info                   | –                             |
|                                      |          | 2          | Terminate Process                    | PID                           |
| Remote Process Execution             | 11       | –          | Create a Process and run a command   | Command to execute            |
| Command Execution Using Windows Pipe | 16       | 1          | Set a pipe connected to cmd          | –                             |
|                                      |          | 2          | Terminate Pipe                       | –                             |
|                                      |          | 3          | Write to pipe                        | Content                       |
| Modules Operation                    | 20       | 1          | Write File                           | Registry key and then content |
|                                      |          | 2          | Delete file from Registry entry path | Registry sub-key              |
|                                      |          | 3          | Create Registry Values               | key/val                       |
|                                      |          | 4          | ?                                    | ?                             |
|                                      |          | 5          | Get Data from the Registry           | ?                             |
|                                      |          | –          | Checksum                             | ?                             |
| Driver Operation                     | –        | 1          | Check if new drivers were added      | –                             |

#### File Operations

##### Get Driver Information

The framework iterates over all available drivers (A-Z) and determines their
type using the
[GetDriveTypeW](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getdrivetypew)
API (e.g., fixed, removable, CD-ROM). For CD-ROM drivers, it retrieves the
display name. For fixed drivers, it calls
[GetVolumeInformationW](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getvolumeinformationw),
which returns:

- **Volume Name**: The label of the specified volume.
- **Volume Serial Number**: A unique number assigned by the OS during disk
  formatting.
- **Maximum Component Length**: Maximum file name component length.
- **File System Flags**: Features supported by the file system.
- **File System Name**: The name of the file system (e.g., NTFS, FAT32).

All this information is sent to the C2 server in a header with the ID 0x31. As
mentioned above, the ID that is sent back consists of the class ID (3) and the
command ID (1).

##### Directory Discovery

The framework receives a path from the C2 and performs a directory scan to
enumerate the files and folders within it. It then gathers detailed information
for each file, including size, last modified timestamp, attributes, and
filename. This collected data is subsequently transmitted back to the C2 in
separate packets, with each packet bearing an ID of 0x324. The presence of a 4
in this ID signifies that the operation has not yet completed sending the data.
In its final stage, when the last packet with ID 0x32 is sent, it indicates the
successful completion of the data transmission.

##### Write File

The framework receives a file path where the file will be created, and after
the file is created, it sends an acknowledgment (ACK) packet to the C2. The
packet ID in this case is 0x33, and it returns the allocated file size.

Next, the framework waits to receive the file content from the C2. Each
packet's content is verified using a checksum, with the expected checksum value
stored in the 8th byte of the packet. For each chunk written, the framework
sends an ACK, indicating the status of the
[WriteFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile)
API call.

##### Read File

The framework receives a path to a file to be read. It will open the file, read
its contents, and send it back to the C2 in chunks of 0x8000 bytes. The packet
IDs associated with this operation are 0x34.

##### Delete File/Folder

The framework receives a path to a folder, which it will recursively delete. It
first sets the folder's file attributes to FILE_ATTRIBUTE_NORMAL, then iterates
over all files in the provided path, deleting each one. The operation sends
back an ACK with the success code and the ID 0x36.

#### Process Operations

##### Get Processes Info

This command enumerates all processes and collects information such as the
process ID (PID), parent process ID, executable name, and whether it is a
system process. The information is sent to the C2 in a packet with the ID
0x514. Once all the information is sent, a final ACK packet is sent with the ID
0x55.

##### Terminate Process

The command receives a PID to terminate. The framework elevates its privileges
by setting the SeDebugPrivilege, which allows it to debug other processes and
access their memory. It then terminates the specified process using the
[TerminateProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess)
API call. The operation sends back an ACK with the success code and the ID
0x52.

##### Remote Code Execution

This operation receives from the C2 a path to an executable on the victim
machine. It creates a new hidden process using the specified executable. An ACK
packet with the status code of this operation and the ID 11 is sent.

#### Command Execution

##### Set a pipe connected to the cmd

This operation creates a
[pipe](https://learn.microsoft.com/en-us/windows/win32/ipc/pipes) connected to
cmd.exe with a hidden window. The input is redirected to cmd.exe, and the
output is sent to the C2. Essentially, this connects the socket to the pipe and
gives the attackers the ability to execute commands. The header for this packet
has an ID of 0x101: the 0x10 represents the class ID 16 and 1 is the command
ID.

##### Terminate pipe

When this command is received, the framework terminates the pipe process using
[TerminateProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess)
and sends an ACK.

##### Write to pipe

This command sends input to the pipe connected to cmd.exe. The input data is
first processed by retrieving the system's OEM code page identifier using
GetOEMCP and converting it using
[WideCharToMultiByte](https://learn.microsoft.com/en-us/windows/win32/api/stringapiset/nf-stringapiset-widechartomultibyte).
The framework then writes the command line into the pipe, waits for 100
milliseconds, and captures the output. If any output is detected, it sends the
content to the C2 in packets with an ID of 0x104. Once all the output is
captured, a final packet with ID 0x1044 is sent.

#### Modules Operation

The framework has extended capabilities for remote code execution of modules,
and it uses information that is stored in the Windows Registry to track and
execute them.

##### Write a file and update the Windows Registry

This function receives data from the C2, verifies its integrity, writes it to a
temporary file, and adds the file path to the system registry. The framework
uses a Windows Registry path that is intended to store the location of the
temporary executable.

If the registry path already contains data, the file creation process is
skipped, assuming the executable is already present. Otherwise, a new file is
created, the name is set by the
[GetTempFileNameW](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-gettempfilenamew)
API function, and the extension is set to.dat. Next, the framework receives
from the C2 the content of the file and writes into the file.

After successfully receiving and writing all the data, the function sets the
file's attributes to System (`FILE_ATTRIBUTE_SYSTEM`) and copies timestamps
from kernel32.dll to the file, helping it blend in with legitimate system
files. Then, the path to the file is saved in the following Windows Registry
path: `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<num>N`.

![](images/fig1.png)

If any errors occur (such as checksum failures, write errors, or network
issues), the function deletes the temporary file and sends an error response to
the C2 server.

##### Delete File Specified in Registry

This function deletes a file based on the registry key it receives from the C2.
The operation accesses the registry at the path:
`HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<provided_key>`, and the subkey
"N" that stores paths to files on the machine. It attempts to delete the file
and, if unsuccessful, moves it to a temporary location and schedules it for
deletion upon the next system restart using
[MoveFileExW](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-movefileexw).

##### Write Values to Registry Keys

This operation writes specific values to the Windows Registry under the keys
"A" and "P" within the path `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<ID>`.
The function receives the data in a structure named RegistryWriteData. The
value of <ID> is received from the attacker.

| Byte Offset | Field Name     | Size (Bytes) | Description                                 |
| ----------- | -------------- | ------------ | ------------------------------------------- |
| 0           | RegistryPathID | 4            | Identifier for the registry path (a number) |
| 4           | Reserved       | 4            | Reserved for future use or alignment        |
| 8           | ValueAData     | 4            | Data for registry key "A"                   |
| 12          | ValuePSize     | 4            | Size of ValuePData                          |
| 16          | ValuePData     | ValuePSize   | Data for registry key "P"                   |

The function begins by determining whether it should write to the registry key
"A". It checks if ValueAData is equal to -1; if so, it interprets this as a
signal to skip writing to key "A". Otherwise, it attempts to write the provided
data to registry key "A".

Similarly, for the registry key "P", the function examines ValuePSize. If this
value is zero, it determines that writing to key "P" is unnecessary. Otherwise,
it attempts to write the data pointed to by ValuePData to the registry key "P".

The function constructs the full registry path as
`HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<ID>`, where it writes the data to
keys "A" and "P" after encoding the input string using a reversible XOR-based
encoding algorithm.

The function allocates memory and encodes the data for the registry keys using
a simple, reversible encoding process. Each character of the input string is
modified using an XOR operation, starting with an initial seed value of 0xE4.
The modified characters are then split into high and low 4-bit nibbles and
mapped to ASCII characters starting from 'A'. The encoded data is stored in an
output array, ensuring it does not exceed twice the length of the original
input.

Essentially, it the operation creates to WIndows Registry paths:

- `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<ID>\A`
- `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\P`
- `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<ID>\N` (Was created by the
  write function described above)

The framework uses these subkeys to track executed modules, with <ID> serving
as the module identifier. It's unclear whether these values follow a specific
scheme or are chosen arbitrarily.

- Subkey "A" stores execution type information.
- Subkey "P" holds configuration or additional parameters.
- Subkey "N" contains the module's file path.

##### Execute a Module

This operation handles dynamic retrieval and loading of modules, specifically
DLL files, from the Windows Registry based on instructions from the C2.

Below is a table that represents the structure of the packet that is being sent
to the Module Load operation:

| Byte Offset | Size     | Field Name         | Description                                                                                 |
| ----------- | -------- | ------------------ | ------------------------------------------------------------------------------------------- |
| 0           | 12 bytes | N/A                | The header of the packet as described in the previous section                               |
| 12          | 4 bytes  | moduleID           | Represents the identifier in the WIndows Registry for the module to be executed.            |
| 16          | 4 bytes  | executionType      | Specifies the type of execution (2, 3, or other) to determine how the module should be run. |
| 20          | 4 bytes  | numberOfParameters | Number of parameters passed to the module function.                                         |
| 24          | 4 bytes  | params[]           | Array of parameters passed to the module function as part of the execution call.            |

The function first checks if the module, identified by `moduleID`, is already
loaded in memory by traversing a linked list of loaded modules. If found, it
returns a pointer to the module's information, avoiding redundant loading.

If the module is not loaded, the function retrieves the module's name from the
Windows Registry (subkey "N"), allocates memory, and attempts to load the
module using `LoadLibraryW`. It then locates the "Execute" function with
`GetProcAddress`. Once found, the module is added to the linked list for future
use. If the function fails at any point, it cleans up by freeing memory and
unloading the module, returning an error code.

Once loaded, the function checks the `executionType` received from the C2, with
three possible options:

- **Load configuration from the Registry and execute (executionType = 2)**: The
  function checks if the module is in memory. If not, it loads it again and
  retrieves additional data from
  `HKEY_LOCAL_MACHINE\Software\Microsoft\IMEEX\<moduleID>\P`, invoking the "Execute"
  function. If successful, the module's execution state is updated to prevent
  re-execution.
- **Load and execute with predefined values (executionType = 3)**: Similar to
  the previous case, it finds the module in memory, executes it with hardcoded
  arguments, and updates its state post-execution.
- **Execute with parameters from the C2 (executionType is not 2 or 3)**: The
  function directly passes the C2 parameters to the module. If successful, the
  result is sent back to the C2; otherwise, an error is reported.

This module execution process allows the framework to dynamically load and
execute modules from the Windows Registry, ensuring that each module is
executed only once while tracking their execution states. Results from
successful executions are sent back to the C2, with error handling and cleanup
in case of failure.

The framework also has a dedicated process that iterates over the values stored
in the subkey "A" (stores the execution type of the module), and if the value
is not 0 – it would run the module if it's not already running.

##### Retrieve IMEEX framework Windows Registry Values

This operation retrieves numeric subkeys from the malware's registry location:
`Software\Microsoft\IMEEX`. It gathers the subkeys and their values,
transmitting the data over a network socket. If data is cached in memory, it is
included in the packet. The stored data, encrypted with XOR, is decoded before
transmission and sent to the C2 in chunks.

If no subkeys are found or an error occurs, an empty packet or error code is
returned. The function processes all subkeys until transmission is complete,
ensuring proper packet structure and network integrity. A final checksum is
computed and sent with the remaining data.

#### Driver Operation

This operation is executed if the command received by the framework does not
match any of the specified operations. The framework calls
[GetLogicalDrives](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getlogicaldrives)
to retrieve a bitmask representing available disk drives and compares it to a
stored global variable containing previously retrieved information. If a
mismatch is found, the framework calls the Get Driver Information operation,
described under File Operations.

## Attribution Indicators and The Geopolitics of The Region

### Network Pivoting

Our investigation into the IMEEX framework was assisted by Tom Hegel from
SentinelOne, who provided valuable information about C2 domains. The analysis
reveals a potential connection between several domains and IP addresses,
suggesting shared infrastructure with ShadowPad – a privately sold modular
malware platform used by at least four clusters of Chinese espionage activity,
allowing threat actors to access advanced anti-detection and persistence
techniques without developing their own backdoors.

The domain `bbsnews.sytes.net` is resolved to the IP address 103.151.229.184
from Hong Kong, which is a generally unused address. Further investigation
shows that this IP has been linked to several other domains:
`cha.ylsyzxxwgov.cn`, `ftp-server.serveftp.com`, and
`it-technology.serveblog.net`, indicating a potential relationship between
these entities. Notably, prior to using 103.151.229.184,
`ftp-server.serveftp.com` was hosted on the similar IP address 103.151.229.52,
which hosted domains linked to ShadowPad activity in 2022, as reported by
[PT Security](https://web.archive.org/web/20220814120943/https://www.ptsecurity.com/upload/corporate/ww-en/pt-esc/winnti-2020-eng.pdf)
(such as point.linkpc.net). Another IP address, 38.60.200.224, also hosted
multiple shadowpad-linked domains, including connecter.publicvm.com, previously
identified as part of the Shadowpad malware
[by SentinelOne](https://www.sentinelone.com/labs/shadowpad-a-masterpiece-of-privately-sold-malware-in-chinese-espionage/)
in 2021, and which was also hosted on 103.151.229.52.

The investigation has uncovered a potential connection between several domains
and IP addresses, suggesting shared infrastructure with the use of ShadowPad by
multiple Chinese APT groups. However, this does not strongly indicate specific
attribution to a particular group, but rather suggests a possible rotation from
ShadowPad to another toolset.

### IMEEX targeting Djibouti

![](images/fig2.png)

Djibouti holds significant geopolitical importance due to its strategic
location at the crossroads of the Red Sea and the Gulf of Aden, controlling
access to one of the world's busiest maritime routes. This small East African
nation
[hosts military bases](https://moderndiplomacy.eu/2024/02/21/strategic-significance-of-djibouti-a-geopolitical-playground-for-global-powers/#:~:text=The%20militaries%20of%20France%2C%20Saudi%20Arabia%2C%20and%20Japan%20rub%20shoulders%20here%20with%20the%20US.%20What%E2%80%99s%20even%20more%20surprising%20is%20that%20so%20does%20China.%20India%20and%20Russia%20are%20keen%20to%20join%20in%20too.%20Despite%20its%20chaotic%20neighbors%2C%20Djibouti%20offers%20an%20excellent%20position%20for%20foreign%20militaries%20to%20project%20their%20power.)
from several global powers, including the U.S., France, and Japan, making it a
critical hub for military and trade operations. Notably, Djibouti is home to
[China's first overseas military base](https://thediplomat.com/2021/01/china-consolidates-its-commercial-foothold-in-djibouti/#:~:text=In%20recent%20years%2C%20China%2DDjibouti%20relations%20have%20developed%20and%20achieved%20fruitful%20results%20in%20various%20fields.%20In%202017%2C%20China%20established%20a%20naval%20base%20in%20Djibouti%2C%20representing%20the%20first%20time%20it%20has%20sought%20a%20permanent%20military%20presence%20beyond%20its%20borders.),
established in 2017, which has become a central part of China's Belt and Road
Initiative (BRI). This base helps China secure its economic and military
interests in the region, particularly along vital shipping lanes connecting the
Indian Ocean to Europe via the Suez Canal.
[China's involvement](https://www.cfr.org/blog/chinas-strategy-djibouti-mixing-commercial-and-military-interests)
in Djibouti extends beyond military interests, with heavy investments in
infrastructure like railways, ports, and a free trade zone. These efforts
support
[Djibouti's "Vision 2035" plan](https://djiboutiembassykuwait.net/en/p/index/14#:~:text=Djibouti%202035%27s%20core%20economic%20objective,than%20200%2C000%20jobs%20by%202035.),
aimed at becoming a commercial and logistics hub for Africa. However, these
investments have also deepened Djibouti's reliance on Chinese loans, raising
concerns about potential debt dependency.

Given Djibouti's strategic significance to China's global trade routes and
military expansion, it is plausible that China could leverage cyber operations,
including potential malware attacks, to safeguard its interests. The country's
proximity to important shipping lanes and the presence of multiple global
military powers, including the U.S. and France, make it a critical point for
espionage and intelligence gathering. Cyberattacks offer a low-risk,
high-reward method for China to monitor adversaries, gather intelligence, and
secure its foothold in Djibouti without escalating to open conflict. In this
competitive geopolitical environment, malware could be used to disrupt
operations that threaten China's regional strategic goals, aligning with
Beijing's broader approach of blending economic influence and military
presence.

### IMEEX targeting Afghanistan

![](images/fig3.png)

China's interests in Afghanistan are multifaceted, revolving around security,
economic influence, and geopolitical strategy. After the U.S. withdrawal and
the Taliban's return to power, China has sought to
[fill the power vacuum](https://www.rferl.org/a/afghanistan-taliban-china-goals/31420549.html)
by fostering stronger relations with Afghanistan. Key interests include
securing China's western border, ensuring stability in the region, and
safeguarding investments under the Belt and Road Initiative (BRI). The
expansion of the China-Pakistan Economic Corridor (CPEC) into Afghanistan is
crucial for enhancing regional connectivity and trade, linking China to Central
Asia and beyond.

[Security is a significant concern for China](https://global.chinadaily.com.cn/a/202305/07/WS6457244aa310b6054fad1807.html),
especially with regard to extremist groups that could threaten its investments
or spill over into its own territories, like Xinjiang. Afghanistan's
instability presents risks to China's projects and personnel in the region,
which may motivate the use of malware as a tool to monitor potential threats,
gather intelligence, and protect its strategic interests. Cyber operations
could also help China track the activities of other foreign powers or influence
local dynamics without direct intervention.

Given the volatile situation and China's deep economic and strategic interests,
the deployment of malware could serve as a low-cost, effective means of
advancing China's goals in Afghanistan.

### Analysis of Variants or Related Samples

sha256: `6fcd206752cd87c26909ed3751b94eb8ef14cd1567d3757cae7fa0b89d3f77c7`

The malware sample submitted to VirusTotal from Afghanistan features a
compilation timestamp of April 4, 2016. This date is more recent than other
known variants, which were compiled between 2014 and 2015. Despite the newer
timestamp, the Afghan variant has fewer capabilities compared to its
predecessors, indicating a reduction in functionality despite the newer
compilation date.

Upon execution, the malware creates a mutex with the value:

`Global42EC381F`

### Codebase and Functional Differences

While the core codebase of the Afghan variant remains largely consistent with
other versions, there are notable differences in its operational capabilities:

- **Execution of additional modules**: This variant does not support the
  execution of modules. Specifically, it lacks the functionality to store
  arguments and information required for the remote execution of modules within
  the Registry.
- **Process Management**: The malware cannot create or terminate processes.
  This absence of process manipulation capabilities reduces its ability to
  execute additional payloads or disrupt system processes, potentially limiting
  its impact compared to more feature-rich variants.

### Configuration Storage

The method of storing configuration data in this variant differs from other
versions:

- **Storage Location**: Unlike other variants, this version retrieves
  configuration values from the read-only (.rdata) section of the binary
  instead of loading configuration data from the end of the executable.
- **Data Obfuscation**: Each byte of the configuration data is XOR-encoded with
  the value 0xEC. This simple obfuscation technique serves to conceal the
  configuration information from straightforward analysis, requiring reverse
  engineers to decode the data to understand the malware's settings.

The C2 used by this variant is: `bbsnews.sytes[.]net`

A script to decode the configuration of this variant is provided in the
appendix below.

## Overall Assessment

Considering the reduced functionality in execution of modules and process
operations, along with the modified method of configuration storage, this
variant appears to be a more **compact iteration of the IMEEX Framework**. The
simplifications suggest a focus on specific operational objectives, potentially
prioritizing stealth and resource efficiency over comprehensive system control.
This makes the Afghan sample a potentially less versatile but more discreet
threat compared to its more feature-rich counterparts.

Summary of supported commands by the Afghan variant:

| Operation                            | Commands Id | Sub-Command Id  | Command                            | Extra Parameters    |
| ------------------------------------ | ----------- | --------------- | ---------------------------------- | ------------------- |
| File Operation                       | 3           | 1               | Get Driver Info                    | –                   |
|                                      |             | 2               | File/Directory                     | Discovery File path |
|                                      |             | 3               | Write File                         | File path           |
|                                      |             | 4               | Read File                          | File path           |
|                                      |             | 6 (No number 5) | Delete folder and content          | File path           |
| Remote Process Execution             | 11          | –               | Create a Process and run a command | Command to execute  |
| Command Execution Using Windows Pipe | 16          | 1               | Set a pipe connected to cmd        | –                   |
|                                      |             | 2               | Terminate Pipe                     | –                   |
|                                      |             | 3               | Write to pipe                      | Content             |
| Driver Operation                     | –           | 1               | Check if new drivers were added    | –                   |

A comparative analysis of the command processing functionalities in the
Djibouti and Afghan variants reveals similarities between the two versions.
Notably, the absence of the execution of modules is observed in the Afghan
variant, underscoring its functional deficiencies compared to the more
comprehensive Djibouti implementation.

The function that processes the command in the Djibouti variant.

![](images/fig4.png)

The function that processes the command in the Afghan variant.

![](images/fig5.png)

## Technical Appendix: Indicators of Compromise

### Appendix A: File Names and Process Indicators

Attempts to run as svchost.exe, imaadp.dll or rundll32.exe

#### Network Infrastructure

##### Domains:

- `yurtumawat.wwwhost[.]us`
- `erkinhorshiden.onedumb[.]com`
- `45.141.139[.]146`
- `bbsnews.sytes[.]net`

##### Port

- 443/TCP

#### Mutexes

- `Global\4B59AFCC`
- `Global\42EC381F`
- `Global\3796878E`

#### Registry Keys

- `Software\Microsoft\IMEEX\<number>`
- Subkeys: N, A, P

### Appendix B: String Decryption Script

Djibouti variant:

```python
import struct
import os
from typing import Union, Tuple
import sys

def decrypt_and_store_in_buffer(file_path: str) -> Union[Tuple[int, str], None]:
    """
    Decrypts data from an executable, validates the decrypted data, and extracts a port and string.
   
    Args:
        file_path (str): Path to the executable file containing the encrypted data.
   
    Returns:
        Union[Tuple[int, str], None]: Returns a tuple containing the port and decrypted string,
                                      or None if decryption fails.
    """
    try:
        # Open the file for reading
        with open(file_path, 'rb') as f:
            # Move the file pointer to 108 bytes before the end
            f.seek(-108, os.SEEK_END)

            # Read 108 bytes of data
            data = f.read(108)
           
            if len(data) != 108:
                return None  # Failed to read the correct amount of data

            # Decrypt the data
            data_buffer = list(data)
            for i in range(107, 0, -1):
                data_buffer[i] ^= data_buffer[i - 1]

            # XOR the first byte with 0xF0 as part of the decryption
            data_buffer[0] ^= 0xF0

            # Pack the first 4 bytes to check for validation
            decrypted_header = struct.unpack_from('<I', bytes(data_buffer))[0]
            if decrypted_header != 0xFEFCFDFB:
                return None  # Validation failed

            # Extract the port from the next 4 bytes
            port = struct.unpack_from('<I', bytes(data_buffer), 4)[0]

            # Decrypted string starts at offset 8
            decrypted_string = bytes(data_buffer[8:]).rstrip(b'x00').decode('latin1')

            # Decrypt the string using XOR with 0xE6
            decrypted_string = "".join(chr(ord(c) ^ 0xE6) for c in decrypted_string)

            return port, decrypted_string

    except Exception as e:
        print(f"Error: {e}")
        return None

def main():
    """
    Main function to handle command-line arguments and execute the decryption.
    """
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <path_to_executable>")
        sys.exit(1)

    file_path = sys.argv[1]

    # Check if file exists
    if not os.path.isfile(file_path):
        print(f"File '{file_path}' does not exist.")
        sys.exit(1)

    result = decrypt_and_store_in_buffer(file_path)
   
    if result:
        port, decrypted_string = result
        print(f"Port: {port}")
        print(f"Decrypted String: {decrypted_string}")
    else:
        print("Decryption failed or validation failed.")

if __name__ == "__main__":
    main()
```

Afghan Variant

```python
def decode_hex_string(hex_string):
    # Convert the hex string to a byte array
    byte_array = bytearray.fromhex(hex_string)
   
    # XOR each byte with 0xEC and decode to string
    decoded_chars = [chr(byte ^ 0xEC) for byte in byte_array]
   
    # Join the characters to form the decoded string
    decoded_string = "".join(decoded_chars)
   
    return decoded_string

# The encoded hex value
encoded_value = '8E8E9F82899B9FC29F9598899FC28289'

# Decode the value
decoded_value = decode_hex_string(encoded_value)

# Print the decoded string
print("Decoded Value:", decoded_value)
```

### Appendix C: YARA

```yara
import "pe"

rule IMEEX_Framework
{
    meta:
        desc = "Detect IMEEX Framework"
        author = "Intezer"
        last_modified = "11-09-2024"
        hash = "3e25798da0232d9039e570fb34d4bdccf7f082fa38b486a097d954f5f3debab3"

    strings:

        /*
        The code that loads the string //Global\4B59AFCC
        */
        $s1 = {C7????47006C00C7????6F006200C7????61006C00C7????5C00}

        /*
        The code that loads the string: "Software\Microsoft\IMEEX"
        */

        $s2 = {C7????68 53006F00 C7????6C 66007400 C7????70 77006100 C7????74 72006500 C7????78 5C004D00 C7????7C 69006300 C7458072 006F00C7 45847300 6F00C745 88660074 00C7458C 5C004900 C745904D 004500C7 45944500 5800}

        $a1 = "server finished" ascii wide
        $a2 = "extended master secret" ascii wide
        $a3 = "client finished" ascii wide
        $a4 = "SRVRCLNT\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\
        $a5 = "key expansion" ascii wide
        $a6 = "SeDebugPrivilege" ascii wide
        $a7 = "AES-256-ECB" ascii wide
        $a8 = "AES-256-CTR" ascii wide
        $a9 = "AES-256-CBC" ascii wide
        $a10 = {fb fd fc fe}

    condition:
        pe.is_dll() and pe.is_64bit() and (all of ($s*) or 80% of ($a*))
}
```
