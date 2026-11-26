# MSI Installer Files

> How attackers abuse MSI installers and custom actions, how the MSI database is structured, and a hands-on malicious MSI analysis with msitools and msidiff.

- Source: https://research.intezer.com/docs/how-to-analyze/msi/
- Published: 2024-07-17
- Authors: Nicole Fishbein
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Threat actors choose to use MSI installers to deliver and execute malicious
payloads because these files can embed harmful executables and scripts within
legitimate-looking packages, evading detection. They can
[abuse custom actions](../../../blog/2024/06/ssload-technical-malware-analysis/#:~:text=This%20installer%20initiates%20a%20delivery%20chain%20comprising%20several%20loaders%2C%20ultimately%20deploying%20the%20final%20payload.)
within MSI files to run malicious code during installation and configure the
installers to download additional malware from remote servers. By masquerading
as legitimate software, attackers trick users into executing these files. MSI
installers are processed by trusted Windows services, making it easier for
malware to bypass security products and remain undetected. This combination of
stealth and versatility makes MSI files an attractive vector for cyberattacks.

MSI
([Microsoft Installer](https://learn.microsoft.com/en-us/mem/configmgr/develop/apps/how-to-create-the-windows-installer-file-msi))
files are database files that contain detailed information about software
installation. They store data on the
[installation process](https://www.advancedinstaller.com/what-is-msi-file.html#:~:text=The%20MSI%20file%20extension%20is%20used%20to%20install%20software%20on%20Windows%20operating%20systems.%20It%20can%20be%20used%20to%20install%2C%20uninstall%2C%20configure%2C%20and%20update%20programs%20on%20the%20computer.),
including configuration settings, files to be installed, and setup
instructions, ensuring a structured and efficient software deployment. Widely
used by software vendors, they provide a standardized installation experience,
streamlining software deployment by ensuring that all necessary files,
settings, and configurations are correctly installed. Additionally, MSI files
offer capabilities for repairing and updating existing installations, making
them a versatile and efficient tool for software management. Designed to work
with the Windows Installer service, a core Windows component responsible for
application management, this standardization, and versatility also make MSI
files a common target for malicious exploitation.

MSI files offer many advantages and capabilities that threat actors actively
abuse, such as in
[SSLoad](../../../blog/2024/06/ssload-technical-malware-analysis/)
campaign. Therefore, it is important for analysts and researchers to understand
the structure of these files and how to analyze them. This blog aims to provide
detailed information on the structure and function of MSI files, offering a
comprehensive step-by-step guide to analyze these files for potential threats.
It highlights the traditional tools and techniques used in the analysis
process, ensuring analysts and researchers can effectively identify and
mitigate risks associated with malicious MSI installers.

## MSI Structure

### MSI vs. EXE

MSI files and EXE (executable) files differ significantly in their
functionality. MSI files are specifically designed for installing software on
Windows systems, integrating with the Windows Installer service for a
standardized installation process. This allows for advanced features like
component installation, rollbacks, and patching. EXE files are general
executables that can perform various functions, including software
installation, but lack the structured installation logic of MSI files.
Additionally, MSI files support silent/unattended installations and are ideal
for network deployment and centralized management while EXE files are more
versatile but may not offer these capabilities.

### MSI File Structure

As mentioned above, MSI installers work with the
[Windows Installer](https://learn.microsoft.com/en-us/windows/win32/msi/overview-of-windows-installer).
Windows Installer is a software component and API developed by Microsoft for
installing, maintaining, and removing software on Windows operating systems. It
uses `.msi` files for installations, offering features like rollback,
versioning, and automatic uninstallation sequences. It maintains a consistent
database of installed products for reliable operation and facilitates
installations by proxy for unprivileged users. Installer packages can be
created using tools like Visual Studio, InstallShield, and WiX Toolset. The
[current version](https://learn.microsoft.com/en-us/windows/win32/msi/released-versions-of-windows-installer?redirectedfrom=MSDN),
5.0, was released on July 22, 2009.

MSI files are structured as
[relational databases](https://www.ibm.com/topics/relational-databases) used by
the Windows Installer for software installation, maintenance, and removal. The
database consists of several tables, each serving a specific purpose in the
installation process. The tables are linked through primary and foreign key
values. An installation database contains all the information needed to install
an application; therefore, it needs to populate the tables with the required
information. An MSI file can have
[multiple tables](https://learn.microsoft.com/en-us/windows/win32/msi/about-the-installer-database),
but we will focus on the tables that are most useful in our analysis.

### Components and Features

The Windows Installer organizes software installation using the concepts of
[**features**](https://learn.microsoft.com/en-us/windows/win32/msi/windows-installer-features)
and
[**components**](https://learn.microsoft.com/en-us/windows/win32/msi/windows-installer-components).
A feature represents a part of the application's functionality that users can
choose to install independently. Components are smaller pieces of the
application that are always installed or removed together. When a feature is
selected, the installer determines which components are required.

A component is a collection of resources (such as files, registry keys, and
shortcuts) that are installed or uninstalled as a single unit. Each component
is tracked by a unique Component ID. Components must be organized so that no
resource is part of more than one component, ensuring proper operation of the
Windows Installer's reference-counting mechanism.

[Component Table](https://learn.microsoft.com/en-us/windows/win32/msi/component-table?source=recommendations):
Contains the components that are part of the installation.

A feature is a distinct part of an application's functionality that users can
choose to install independently, like a spell-checker. Features can have
hierarchical relationships, meaning if a child feature is installed, its parent
feature is also automatically installed.

[FeatureComponents Table](https://learn.microsoft.com/en-us/windows/win32/msi/featurecomponents-table?source=recommendations):
Establishes the relationship between features and components. For each feature,
this table specifies all the components that constitute that feature. This
relationship is crucial for ensuring that all necessary components are
installed or updated when a feature is selected during the installation
process. The features themselves are defined in a
[Feature Table](https://learn.microsoft.com/en-us/windows/win32/msi/feature-table?source=recommendations).

The installer needs to know which files need to be installed. This information
is stored in the `File` Table, which contains information about files to be
installed, including file names, paths, and attributes. There is also a
`Directory` Table, which specifies the directory structure for the
installation.

### Binary vs. File Table

[The Binary](https://learn.microsoft.com/en-us/windows/win32/msi/binary-table)
table and `File` table in MSI files serve distinct purposes. The `Binary` table
stores binary data directly within the MSI database, such as icons, bitmaps,
animations, or custom action scripts, which are embedded directly in the MSI
file and accessed using Windows Installer APIs during installation. It is
typically used for smaller resources needed during the installation process
itself. On the other hand, the `File` table contains information about files
that will be installed on the target system, describing actual application
files to be copied to the user's computer. These files are usually stored
separately (e.g., in a CAB file) or as compressed streams within the MSI and
represent the files that make up the application being installed. The `File`
table is used to determine which files to install, where to install them, and
their attributes.

### Installation Actions and Sequences

An
[**action**](https://learn.microsoft.com/en-us/windows/win32/msi/standard-actions)
encapsulates a function performed during installation or maintenance, and they
are set by the developers/creators of the installation. There are many built-in
standard actions. Examples include:

- [`CreateShortcuts`](https://learn.microsoft.com/en-us/windows/win32/msi/createshortcuts-action):
  Manages shortcut creation using the `Shortcut` table.
- [`InstallFiles`](https://learn.microsoft.com/en-us/windows/win32/msi/installfiles-action):
  Copies files from the source to the destination directory using the `File`
  table.
- [`WriteRegistryValues`](https://learn.microsoft.com/en-us/windows/win32/msi/writeregistryvalues-action):
  Sets up required registry information using the `Registry` table.

These actions ensure that all necessary components and configurations are
correctly applied during installation.

[Sequence tables](https://learn.microsoft.com/en-us/windows/win32/msi/using-a-sequence-table)
in MSI files control the order of installation actions and user interface
displays. There are three installation modes:

- Simple
- Administrative — An installation that was initiated by an administrator.
- Advertisement — The Windows Installer can advertise an application's
  availability to users or other applications without actually installing the
  application.

Each has specific
[sequence tables](https://learn.microsoft.com/en-us/windows/win32/msi/using-a-sequence-table)
(UI and Execute). The tables have fields for `Action`, `Condition`, and
`Sequence`, determining what actions occur, under what conditions, and in what
order. Sequence tables include details for standard and custom actions,
enhancing installation flexibility and control. During a simple installation,
the `INSTALL` top-level action is executed, which processes the
`InstallUISequence` table and the
[`InstallExecuteSequence`](https://learn.microsoft.com/en-us/windows/win32/msi/installexecutesequence-table)
table. The
[`InstallUISequence`](https://learn.microsoft.com/en-us/windows/win32/msi/installuisequence-table)
table manages the user interface sequence of the installation. In contrast, the
`InstallExecuteSequence` table controls the actual installation actions,
ensuring the installation steps are carried out in the correct order.

There are several types of actions:

- Standard Actions: Predefined actions provided by Windows Installer, such as
  `CostFinalize`, `InstallFiles`, `RegisterProduct`, and `RemoveFiles`. These
  handle common installation tasks.
- Custom Actions: User-defined actions that can perform custom installation
  tasks, such as executing a script or calling a DLL function.

MSI files define sequences for executing actions. There are three main types of
sequences:

- `AdminExecuteSequence`: Actions executed during an administrative
  installation for individual users or groups
- `InstallExecuteSequence`: Actions executed during a typical installation.
- `AdvertiseExecuteSequence`: Actions executed during advertisement
  (installation without installing files).

Actions and sequences allow for precise control over the installation process,
ensuring that steps are executed in the correct order and under the correct
conditions.

### Custom Actions

A
[Custom Action](https://learn.microsoft.com/en-us/windows/win32/msi/customaction-table)
in an MSI file refers to a command or script that runs during the installation
process to perform custom tasks not handled by standard MSI actions. These
actions can execute executable files, DLLs, VBScript, JavaScript, or predefined
commands, enabling threat actors to execute the malicious payload in the
background of what looks like a legitimate installation.

Custom actions in an MSI file are defined in the `CustomAction` table. Each
entry in this table specifies a custom action, its type, source, and target.
Here is a breakdown of the columns used to define custom actions:

1. **`Action`:** This is the name of the custom action. It acts as the primary
   key and references this action in sequence tables. If the name matches any
   built-in action, the custom action will not be executed.
2. **`Type`:** This field contains a bitwise combination of flags that specify
   the custom action type and its execution options. The type determines
   whether the action calls a DLL, an executable, a script, or sets properties.
3. **`Source`:** This field can contain a property name or a key to another
   table, such as the `Binary` table (for embedded binaries), `File` table (for
   installed files), or `Directory` table (for paths). The specific table
   referenced depends on the type of custom action.
4. **`Target`:** This field provides additional information required by the
   custom action, such as the entry point for a DLL, command-line arguments for
   an executable, or script code.
5. **`ExtendedType`:** This field specifies additional options for the custom
   action, such as handling patch uninstallation.

For example, a custom action that runs an executable might have its `Source`
field reference a key in the `File` table, and its `Target` field contains
command-line arguments.

### Common File Extensions and Their Roles

- **`.msi`:** The primary extension for Microsoft Installer package files.
  Contains the installation database and the necessary instructions for
  installing the software.
- **`.mst`:** Windows Installer Transform files. Used to modify or customize
  MSI files without altering the original package.
- **`.msm`:** Merge Module files. Used to deliver shared components and ensure
  that the same versions of shared files are installed across multiple
  applications.
- **`.msp`:** Windows Installer Patch files. Used to apply updates or fixes to
  installed applications.
- **`.cab`:** Cabinet files. Used to compress and package multiple files into a
  single file for easier distribution.

## MSI-Based Installers

People use third-party tools to create MSI files because they offer a
standardized software installation format, simplifying creation, maintenance,
and removal across multiple systems, especially in enterprise environments.
These tools also integrate seamlessly with enterprise deployment systems,
facilitating centralized control and automated deployment.

There are several products that support the MSI format. We will describe the
most common software:

### Advanced Installer

[Advanced Installer](https://www.advancedinstaller.com/) uses MSI (Microsoft
Installer) as one of its primary installer formats. Advanced Installer's use of
MSI includes powerful tools for creating and editing MSI packages and an
intuitive graphical user interface for viewing and modifying MSI elements like
file structure, registry entries, etc.

Advanced Installer offers many useful features, even in its free edition -
which makes it easy for threat actors to abuse it. The software supports
creating MSI packages even in its free edition and can repackage EXE installers
as MSI files. Additionally, it supports modern Windows technologies and can
create installers for various formats beyond MSI, including MSIX, App-V, and
more. Advanced Installer also provides features for silent installations of
both MSI and EXE files.

### InstallShield

[InstallShield](https://www.revenera.com/install/products/installshield) is a
software tool used to create installers for Windows applications. It helps
developers produce installation packages, including MSI (Windows Installer)
packages. InstallShield can create MSI files that are installed on end-user
systems using the Windows Installer service. While InstallShield generates the
installation packages, Windows Installer performs the actual installation on
the user's system. InstallShield offers various project types, with the Basic
MSI Project being a popular option for creating Windows Installer packages.
Although other tools like WiX (Windows Installer XML) can also generate MSI
files, InstallShield remains a widely used commercial solution. Beyond creating
MSI files, InstallShield provides features such as creating MSIX packages,
integrating with Visual Studio, and supporting advanced installer capabilities.

## Tools for Manually Analyzing an MSI File

Several tools can be used to analyze MSI files:

- [msitools](https://wiki.gnome.org/msitools): This Linux package includes
  utilities like `msiinfo` and `msidump` that can be used to examine MSI files.
  It allows you to enumerate tables and streams, dump table contents, and
  extract data from MSI files.
- [msidump](https://github.com/mgeeky/msidump): A tool developed by Mgeeky that
  analyzes malicious MSI packages. It can extract files, streams, and binary
  data, and incorporates YARA scanning. It's particularly useful for quick
  triage and detailed examination of potentially malicious MSIs.
- [lessmsi](https://lessmsi.activescott.com/): This is a utility with both a
  graphical user interface and a command-line interface. It can be used to view
  and extract the contents of MSI files. It also features a Windows Explorer
  integration for easy extraction and an MSI table viewer.
- [MSI Viewer](https://apps.microsoft.com/detail/9mztr9qs01gn?hl=en-us&gl=NL):
  Available from the Microsoft Store, this app allows you to view the contents
  of MSI installer files and merge modules. It can also extract files without
  needing to run the installer.
- [Orca](https://learn.microsoft.com/en-us/windows/win32/msi/orca-exe): A
  graphical user interface (GUI) tool developed by Microsoft for analyzing and
  editing MSI files. Orca allows users to open MSI files and view their
  internal structure, including tables, properties, and other components of the
  installation package.

## MSI in the Wild

MSI files can be very useful from the perspective of threat actors. First, MSI
files are a common tool for installing a wide variety of software, making them
a perfect target for masquerading malicious payloads and abusing their
popularity. Next, MSI files offer multiple handy features for installing
software. Still, at the same time, they can be used to secretly execute
malicious payloads without the user knowing or noticing it.

Common examples of threats that use MSI files in their campaigns are:

- [DarkGate](https://www.trendmicro.com/en_us/research/24/c/cve-2024-21412--darkgate-operators-exploit-microsoft-windows-sma.html):
  Uses MSI files that masquerade as a legitimate installation, such as Notion
  or Apple iTunes. It contains DLL sideloading that delivers the DarkGate
  payload.
- [Maze Ransomware](https://nucleon-security.com/security-insights/how-maze-abuse-legitimate-installer-to-trick-your-defense/):
  Abuses the Custom Action to execute a malicious DLL.
- [IcedID](https://www.trendmicro.com/en_us/research/22/l/icedid-botnet-distributors-abuse-google-ppc-to-distribute-malware.html):
  Abuses MSI files in several ways. It is often distributed via malicious
  Office documents or ZIP files containing MSI files, tricking users into
  installing the malware. Distributors create MSI files masquerading as
  installers for legitimate software like AnyDesk or Adobe products and promote
  them through Google Ads. Once executed, the MSI files drop several files and
  use `rundll32.exe` to run the malicious loader routine. IcedID uses
  `msiexec.exe` to mask its activities, helping it blend in with legitimate
  processes. These MSI files deliver the initial IcedID loader, which then
  fetches the main payload, often a backdoor. The use of MSI files helps IcedID
  evade some security measures and establish persistence through methods like
  scheduled tasks.

Malware developers frequently abuse MSI installers to execute malicious
payloads by leveraging the flexibility and capabilities of Windows Installer
technology. They also often use custom actions in MSI files to execute
malicious code. Custom actions can run executables, scripts, or DLLs during
installation, allowing threat actors to execute and inject malicious code into
running processes.

By design, MSI installers can embed various types of files, including
executables and DLLs. These embedded files can contain malicious payloads that
are extracted and executed as part of the installation process. In addition,
these DLLs can be used to alter the execution flow by
[DLL Hijacking](https://attack.mitre.org/techniques/T1574/001/) or
[DLL Sideloading](https://attack.mitre.org/techniques/T1574/002/).

- **Binary Table**: Storing malware in the `Binary` table and executing it as
  part of the installation routine is a common tactic. The malware is disguised
  as a necessary component of the software being installed.

MSI installers have numerous features and capabilities that can be useful for
threat actors. For instance, MSI installers often run with elevated privileges,
which can be abused to perform actions that require administrative rights.
Attackers can also use the silent installation options to deploy malware
without user interaction or awareness.

## Hands-On Analysis of a Malicious MSI File

Let's take a look at a malicious MSI file that was part of a campaign that
delivered SSLoad malware. SSLoad is a malware written in Rust that collects
information about compromised victims and sends it to the C2. For more details
about the campaign,
[check our write-up](../../../blog/2024/06/ssload-technical-malware-analysis/).

We will examine the installer file using **msitools**. Our goal is to identify
and extract files that are being executed by the installer and are not part of
a legitimate installation.

The msitools package can be installed for
[Linux](https://wiki.gnome.org/msitools) and
[Mac](https://formulae.brew.sh/formula/msitools).

The sha256 of the MSI file:
[`90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750`](https://analyze.intezer.com/files/90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750)

First, let's examine the metadata of the file:

```
msiinfo suminfo 90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750.sample
Title: Installation Database
Subject: MS SharePoint
Author: Akamai Technology
Keywords: Installer, MSI, Database
Comments: This installer database contains the logic and data required to install MS SharePoint.
Template: *;1033*
Last author:
Revision number (UUID): {B61C58BB-6B98-4501-8BF7-0D9C8D568301}
Last printed: Fri Dec 11 06:47:44 2009
Created: Fri Dec 11 06:47:44 2009
Last saved: Fri Sep 18 10:06:51 2020
Version: 200 (c8)
Source: 10 (a)
Application: MS SharePoint
Security: 0 (0)
```

Based on the output, we can understand that the malicious file tries to
masquerade as an installer of MS SharePoint. The creation and update date can
help in triage and hunting, however, note that they can be spoofed or altered.

Next, examine the list of tables in this MSI file:

```
msiinfo tables 90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750.sample
_SummaryInformation
_ForceCodepage
Patch
Condition
AdvtExecuteSequence
PatchPackage
FeatureComponents
_Validation
AdminExecuteSequence
ActionText
AdminUISequence
TextStyle
Binary
File
Component
BootstrapperUISequence
Media
ComboBox
Control
Dialog
Property
Error
Feature
Directory
EventMapping
CheckBox
ControlCondition
ControlEvent
CreateFolder
CustomAction
InstallExecuteSequence
InstallUISequence
LaunchCondition
ListBox
ListView
RadioButton
Registry
UIText
Upgrade
```

While it might be overwhelming to see all of these tables, we should start our
analysis from places more prone to abuse, such as the `CustomAction` table.
While many actions are defined in this table, `LaunchFile` seems to execute a
specific DLL file.

```
msiinfo export 90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750.sample CustomAction
Action Type Source Target ExtendedType
s72 i2 S72 S0 I4
CustomAction Action
AI_DETECT_MODERNWIN 1 aicustact.dll DetectModernWindows
AI_Init_PatchWelcomeDlg 1 aicustact.dll DoEvents
AI_Init_WelcomeDlg 1 aicustact.dll DoEvents
AI_SET_ADMIN 51 AI_ADMIN 1
AI_InstallModeCheck 1 aicustact.dll UpdateInstallMode
AI_DOWNGRADE 19 4010
AI_DpiContentScale 1 aicustact.dll DpiContentScale
AI_EnableDebugLog 321 aicustact.dll EnableDebugLog
AI_PREPARE_UPGRADE 65 aicustact.dll PrepareUpgrade
AI_ResolveKnownFolders 1 aicustact.dll AI_ResolveKnownFolders
AI_RESTORE_LOCATION 65 aicustact.dll RestoreLocation
AI_STORE_LOCATION 51 ARPINSTALLLOCATION [APPDIR]
LaunchFile 1026 viewer.exe C:\Windows\System32\regsvr32.exe /S [LocalAppDataFolder]sharepointMenuEx.dll
SET_SHORTCUTDIR 307 SHORTCUTDIR [ProgramMenuFolder][ProductName]
SET_TARGETDIR_TO_APPDIR 51 TARGETDIR [APPDIR]
AI_CORRECT_INSTALL 51 AI_INSTALL {}
AI_SET_RESUME 51 AI_RESUME 1
AI_SET_INSTALL 51 AI_INSTALL 1
AI_SET_MAINT 51 AI_MAINT 1
AI_SET_PATCH 51 AI_PATCH 1
AI_DATA_SETTER 51 CustomActionData [AI_Init_PatchWelcomeDlg]
AI_DATA_SETTER_1 51 CustomActionData [AI_Init_WelcomeDlg]
```

Another way to view the files that are part of the installer is by examining
the `File` Table:

```
msiinfo extract 90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750.sample FileTable > FileTable.txt
```

To get the file (`MenuEx.dll`), we will use `msiextract`:

```
msiextract --directory extracted_files 90f1511223698f33a086337a6875db3b5d6fbcce06f3195cdd6a8efa90091750.sample
```

The suspicious DLL is in the `extracted_files` folder. Analyzing the DLL is
outside the scope of this blog, but you can read more about it in
[our blog](../../../blog/2024/06/ssload-technical-malware-analysis/).

In this example, we see how the threat actors used the MSI's custom action
feature to load malicious DLL. We also showed how to analyze MSI files and
identify this sort of manipulation.

### MSI Installer Analysis Using Intezer

So far, we have demonstrated how to investigate MSI files manually. However,
there is a more efficient method to extract malicious payloads from these and
other file types. Using [Intezer](https://intezer.com/forensic-ai-soc/), we
leverage static and dynamic execution combined with robust code and string
reuse analysis. This powerful approach enables analysts, SOC teams, and
researchers to quickly identify and classify malicious files from a high volume
of alerts.

Below, we can see the extracted DLL from the MSI installer. The DLL is being
detected as PhantomLoader.

![](images/fig1.png "Execution of the MSI file in Intezer Analyze.")

## Diffing MSI Files - MSIDIFF

Let's go back to manual analysis with **msitools**.

The **msidiff** is a command-line utility part of msitools. It compares two MSI
files or an MSI file against a directory structure. This utility is useful for
developers and IT professionals who need to identify differences between
different versions of MSI packages. It helps to ensure that the changes in the
installer packages are as intended, aiding in version control and
troubleshooting.

For investigators and researchers, it helps identify differences and changes
between versions of MSI packages.

The utility compares the tables of the input files. It provides a detailed
comparison report highlighting the differences between the two MSI files or
between the MSI file and the directory structure. The differences are
categorized and displayed in a readable format, typically showing:

- Table Changes: Differences in the database tables within the MSI files.
- File Changes: Differences in the files included in the MSI packages.
- Property Changes: Differences in the properties set within the MSI files.

Each difference is presented in a line-by-line comparison, indicating what has
been added, removed, or modified. The report helps in pinpointing exactly where
the changes have occurred, making it easier to manage and verify updates to the
installation packages.

**Example Output**

An example output might look like this:

```
Table: File
+ FileKey  Component_  FileName
- oldfile  comp1       oldfile.txt
+ newfile  comp2       newfile.txt

Table: Property
- ProductVersion  1.0.0
+ ProductVersion  2.0.0
```

In this example:

- In the `File` table, `oldfile.txt` has been replaced with `newfile.txt`.
- In the `Property` table, the `ProductVersion` has been updated from `1.0.0`
  to `2.0.0`.

Note that the file that is specified first will be treated as the old version,
and the second file will be the newer version.

### Real-World Example

According to HarfangLab's
[report](https://harfanglab.io/en/insidethelab/muddywater-rmm-campaign/), the
Iranian state-sponsored group MuddyWater has been using MSI files containing
Atera Agent, a legitimate remote monitoring and management (RMM) tool, in a
campaign targeting Israel and other countries. These MSI files were distributed
via spearphishing emails, with filenames crafted to appear relevant to the
intended victims, such as "Tourism Program" and "Leonardo Hotels-tourism
software." Hosted on various file-sharing platforms like Egnyte and Sync.com,
these MSI files were configured with Atera Agent linked to specific email
addresses from compromised accounts.

Our team identified another variant of the MSI installer that installs Atera
Agent in the wild.

Sha256:
[`66ea27e2e043adcfca5352089e2cbe7d4349f1f7e78dd4acefaf451b8c9585c4`](https://analyze.intezer.com/files/66ea27e2e043adcfca5352089e2cbe7d4349f1f7e78dd4acefaf451b8c9585c4)

We wanted to compare it to the installers from the report. We chose the file
with the sha256: `ffbe988fd797cbb9a1eedb705cf00ebc8277cdbd9a21b6efb40a8bc22c7a43f0`

```
msidiff ffbe988fd797cbb9a1eedb705cf00ebc8277cdbd9a21b6efb40a8bc22c7a43f0 66ea27e2e043adcfca5352089e2cbe7d4349f1f7e78dd4acefaf451b8c9585c4
diff -r -Nup old/CustomAction.idt new/CustomAction.idt
--- old/CustomAction.idt 2024-07-08 11:40:27
+++ new/CustomAction.idt 2024-07-08 11:40:27
@@ -1,6 +1,11 @@ CustomAction Action
Action Type Source Target ExtendedType
s72 i2 S72 S255 I4
CustomAction Action
+GenerateAgentId 2049 AlphaControlAgentInstallationBinary GenerateAgentId
+ReportInstallationStart 2049 AlphaControlAgentInstallationBinary ReportMsiStart
+ReportInstallationEnd 2049 AlphaControlAgentInstallationBinary ReportMsiEnd
+ReportInstallationFailure 2049 AlphaControlAgentInstallationBinary ReportMsiFailure
+ReportInstallationUserCancel 2049 AlphaControlAgentInstallationBinary ReportMsiUserCancel
SetCustomActionPropertyValues 51 PostUninstallCleanup SKIPCLEANUP=[SKIPCLEANUP]
PostUninstallCleanup 3137 AlphaControlAgentInstallationBinary PostUninstallCleanup
PormptInstallationDialog 2049 AlphaControlAgentInstallationBinary ShouldContinueInstallation
@@ -10,6 +15,6 @@ oldVersionUninstall 3170 INSTALLFOLDER unins000.exe /V
KillAteraTaskQuiet 3137 WixCA WixQuietExec
KillAteraService 3234 INSTALLFOLDER sc delete AteraAgent
oldVersionUninstall 3170 INSTALLFOLDER unins000.exe /VERYSILENT
-install 3090 AteraAgent.exe /i /IntegratorLogin="[INTEGRATORLOGIN]" /CompanyId="[COMPANYID]" /IntegratorLoginUI="[INTEGRATORLOGINUI]" /CompanyIdUI="[COMPANYIDUI]" /FolderId="[FOLDERID]" /AccountId="[ACCOUNTID]"
+install 3090 AteraAgent.exe /i /IntegratorLogin="[INTEGRATORLOGIN]" /CompanyId="[COMPANYID]" /IntegratorLoginUI="[INTEGRATORLOGINUI]" /CompanyIdUI="[COMPANYIDUI]" /FolderId="[FOLDERID]" /AccountId="[ACCOUNTID]" /AgentId="[AGENTID]"
uninstall 3154 AteraAgent.exe /u
DeleteTaskScheduler 3170 INSTALLFOLDER SCHTASKS.EXE /delete /tn "Monitoring Recovery" /f
diff -r -Nup old/File.idt new/File.idt
--- old/File.idt 2024-07-08 11:40:27
+++ new/File.idt 2024-07-08 11:40:27
@@ -1,10 +1,10 @@ File File
File Component_ FileName FileSize Version Language Attributes Sequence
s72 s72 l255 i4 S72 S20 I2 i4
File File
-AteraAgent.exe AteraAgent.exe culrpfxg.exe|AteraAgent.exe 144224 1.8.6.7 0 512 1
+AteraAgent.exe AteraAgent.exe culrpfxg.exe|AteraAgent.exe 145968 1.8.7.2 0 512 1
AteraAgent.exe.config AteraAgent.exe.config 7uho0yn3.con|AteraAgent.exe.config 1442 512 2
-ICSharpCode.SharpZipLib.dll ICSharpCode.SharpZipLib.dll fd-i8f6f.dll|ICSharpCode.SharpZipLib.dll 214880 1.3.3.11 0 512 4
-Newtonsoft.Json.dll Newtonsoft.Json.dll e8lrglzz.dll|Newtonsoft.Json.dll 709984 13.0.2.27524 0 512 5
-PubNub.dll PubNub.dll Pubnub.dll 602464 6.13.0.0 0 512 6
-System.ValueTuple.dll System.ValueTuple.dll 7t1dchnb.dll|System.ValueTuple.dll 73056 4.6.26515.6 0 512 7
-BouncyCastle.Crypto.dll BouncyCastle.Crypto.dll 8kw0ybbq.dll|BouncyCastle.Crypto.dll 3318624 1.9.0.1 0 512 3
+ICSharpCode.SharpZipLib.dll ICSharpCode.SharpZipLib.dll fd-i8f6f.dll|ICSharpCode.SharpZipLib.dll 215088 1.3.3.11 0 512 4
+Newtonsoft.Json.dll Newtonsoft.Json.dll e8lrglzz.dll|Newtonsoft.Json.dll 710192 13.0.2.27524 0 512 5
+PubNub.dll PubNub.dll Pubnub.dll 602672 6.13.0.0 0 512 6
+System.ValueTuple.dll System.ValueTuple.dll 7t1dchnb.dll|System.ValueTuple.dll 73264 4.6.26515.6 0 512 7
+BouncyCastle.Crypto.dll BouncyCastle.Crypto.dll 8kw0ybbq.dll|BouncyCastle.Crypto.dll 3318832 1.9.0.1 0 512 3
diff -r -Nup old/InstallExecuteSequence.idt new/InstallExecuteSequence.idt
--- old/InstallExecuteSequence.idt 2024-07-08 11:40:27
+++ new/InstallExecuteSequence.idt 2024-07-08 11:40:27
@@ -11,6 +11,11 @@ PublishProduct 6400
InstallFinalize 6600
PublishFeatures 6300
PublishProduct 6400
+GenerateAgentId (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") AND (NOT SKIPCLEANUP) 96
+ReportInstallationStart (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") AND (NOT SKIPCLEANUP) 97
+ReportInstallationEnd (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") AND (NOT SKIPCLEANUP) 6601
+ReportInstallationFailure (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") AND (NOT SKIPCLEANUP) -3
+ReportInstallationUserCancel (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") AND (NOT SKIPCLEANUP) -2
SetCustomActionPropertyValues 1602
PostUninstallCleanup (NOT UPGRADINGPRODUCTCODE) AND (REMOVE="ALL") 1604
PormptInstallationDialog (Installed = "") AND (REINSTALL = "") AND (UPGRADINGPRODUCTCODE = "") AND (REMOVE = "") 98
diff -r -Nup old/Property.idt new/Property.idt
--- old/Property.idt 2024-07-08 11:40:26
+++ new/Property.idt 2024-07-08 11:40:27
@@ -7,11 +7,11 @@ Manufacturer Atera networks
ALLUSERS 1
REINSTALLMODE dmus
Manufacturer Atera networks
-ProductCode {C5F5A288-85FF-4257-AF69-D5910E6268B5}
+ProductCode {E732A0D7-A2F2-4657-AC41-B19742648E45}
ProductLanguage 1033
ProductName AteraAgent
-ProductVersion 1.8.6.7
+ProductVersion 1.8.7.2
SecureCustomProperties NETFRAMEWORK35;PREVIOUSFOUND;WIX_UPGRADE_DETECTED
-INTEGRATORLOGIN mcsamaniego@mre.gov.py
+INTEGRATORLOGIN eduardomonica00214@hotmail.com
COMPANYID 1
-ACCOUNTID 001Q3000008IyacIAC
+ACCOUNTID 001Q300000AVzB7IAL
diff -r -Nup old/_SummaryInformation.idt new/_SummaryInformation.idt
--- old/_SummaryInformation.idt 2024-07-08 11:40:26
+++ new/_SummaryInformation.idt 2024-07-08 11:40:27
@@ -8,9 +8,9 @@ _SummaryInformation PropertyId
5 Installer
6 This installer database contains the logic and data required to install AteraAgent.
7 Intel;1033
-9 {D42E1648-2328-41CB-BC72-C0B32D77958E}
-12 2023/09/26 10:17:42
-13 2023/09/26 10:17:42
+9 {721AD955-79FD-4019-BBF5-9DCC4C1175BB}
+12 2024/02/28 04:52:02
+13 2024/02/28 04:52:02
14 200
15 6
18 Windows Installer XML Toolset (3.11.2.4516)
diff -r -Nup old/signature new/signature[...]
```

Based on the output the changes are in the following tables:

`CustomAction`, `File`, `InstallExecuteSequence`, `Property`, `ProductCode`,
`ProductVersion`, and `_SummaryInformation`.

### Using AI to Parse the Output of MSIDIFF

We can use generative AI to make the output more user-friendly and derive our
conclusions. We have used ChatGPT4-o with the following prompt:

***You are experienced malware analysts. Parse and summarize the output of
msidiff utility based on the provided input.***

Next, we gave it the output of the msidiff command and got the following
result:

```
The `msidiff` utility output shows the differences between two MSI package versions. Here's a summary of the changes:

### CustomAction.idt
- **Added Custom Actions:**
  - `GenerateAgentId` with target `GenerateAgentId`
  - `ReportInstallationStart` with target `ReportMsiStart`
  - `ReportInstallationEnd` with target `ReportMsiEnd`
  - `ReportInstallationFailure` with target `ReportMsiFailure`
  - `ReportInstallationUserCancel` with target `ReportMsiUserCancel`

- **Modified Custom Action:**
  - `install` action now includes an additional parameter `/AgentId="[AGENTID]"`

### File.idt
- **Modified File Sizes:**
  - `AteraAgent.exe` updated from `144224` bytes to `145968` bytes
  - `ICSharpCode.SharpZipLib.dll` updated from `214880` bytes to `215088` bytes
  - `Newtonsoft.Json.dll` updated from `709984` bytes to `710192` bytes
  - `PubNub.dll` updated from `602464` bytes to `602672` bytes
  - `System.ValueTuple.dll` updated from `73056` bytes to `73264` bytes
  - `BouncyCastle.Crypto.dll` updated from `3318624` bytes to `3318832` bytes

### InstallExecuteSequence.idt
- **Added Installation Steps:**
  - `GenerateAgentId` with condition and sequence number `96`
  - `ReportInstallationStart` with condition and sequence number `97`
  - `ReportInstallationEnd` with condition and sequence number `6601`
  - `ReportInstallationFailure` with condition and sequence number `-3`
  - `ReportInstallationUserCancel` with condition and sequence number `-2`

### Property.idt
- **Updated Product Information:**
  - `ProductCode` changed from `{C5F5A288-85FF-4257-AF69-D5910E6268B5}` to `{E732A0D7-A2F2-4657-AC41-B19742648E45}`
  - `ProductVersion` updated from `1.8.6.7` to `1.8.7.2`
  - `INTEGRATORLOGIN` updated from `mcsamaniego@mre.gov.py` to `eduardomonica00214@hotmail.com`
  - `ACCOUNTID` updated from `001Q3000008IyacIAC` to `001Q300000AVzB7IAL`

### _SummaryInformation.idt
- **Updated Summary Information:**
  - `ProductCode` changed from `{D42E1648-2328-41CB-BC72-C0B32D77958E}` to `{721AD955-79FD-4019-BBF5-9DCC4C1175BB}`
  - Creation and modification dates updated to `2024/02/28 04:52:02`

### General Summary
The changes indicate updates to the custom actions, file versions and sizes, installation sequences, and product properties. The new version of the MSI includes additional custom actions primarily focused on reporting installation statuses and generating agent IDs. There are also updates to the executable and library files, reflecting changes in file sizes and potentially code updates. Additionally, product metadata, such as version numbers and user-specific properties, have been updated.
```

We see some changes in the custom actions, the different email addresses that
were used, the updated `ProductVersion`, and the creation date.

## Conclusion

MSI files, widely used for software installation on Windows systems, offer a
standardized and efficient method for deploying software but are also
attractive targets for threat actors. These files can be exploited to deliver
and execute malicious payloads, leveraging their trusted status and complex
structure to evade detection.

Understanding the structure and functionality of MSI files, including their
database tables, custom actions, and installation sequences, is essential for
identifying potential threats. Tools such as `msitools`, `msidump`, `lessmsi`,
and MSI Viewer provide valuable capabilities for inspecting and analyzing MSI
files. By examining key tables like `CustomAction` and `File`, security
professionals can detect and mitigate risks associated with malicious MSI
installers.
