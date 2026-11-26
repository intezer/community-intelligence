# Phishing Emails

> How to inspect phishing email files: sender addresses, headers, attachments and links, plus a real campaign that used a short URL and a packed payload.

- Source: https://research.intezer.com/docs/how-to-analyze/phishing/
- Published: 2023-11-03
- Authors: Nicole Fishbein
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Emails were created as a method to pass messages between users, and now they
are used by individuals and organizations all around the globe, by both big and
small companies across all industries. But emails also have a dark side -
phishing emails that are used by threat actors to gain access to victims'
systems.

Traditionally phishing emails are associated with credential harvesting
attacks, but that's not the only goal of these attacks. As defined by
[MITRE ATT&CK® framework](https://attack.mitre.org/techniques/T1566/),
adversaries also send phishing emails containing malicious links or attachments
to deploy malware (such as backdoors and ransomware) and further exploit the
system. For the last four years, the
[FBI's Internet Crime Report](https://www.ic3.gov/Media/PDF/AnnualReport/2022_IC3Report.pdf)
has found phishing was the top type of cyber crime and also one of most
prevalent ways ransomware attackers gained initial access.

These days, we know better than to open and respond to sketchy emails that make
wild promises. Consequently, threat actors adapt their techniques and messages
to increase the chances of successfully luring the victims. Phishing remains a
lucrative endeavor for cyber criminals, with successful phishing attacks
leading to a 76% increase of financial losses in 2022, according to email
security company
[Proofpoint's 2023 report](https://www.proofpoint.com/us/resources/threat-reports/state-of-phish).

*[**Know the basics about phishing emails? Jump down to learn about automating tasks in your phishing email investigation pipeline.**](#investigating-phishing-email-campaigns-with-real-life-examples)*

Since emails are widely used as a means of communication between the company
and the outside world, security teams have to deal with large amounts of files,
filtering and inspecting them to prevent phishing emails from reaching the end
user's mailbox. In addition, threat actors implement different techniques to
evade detection. In 2023, a wave of phishing emails started using
[QR codes to hide malicious links](https://intezer.com/blog/alert-triage/automated-qr-code-phishing-investigation/)
and evade email security tools.

In this blog, we will cover techniques used by threat actors to make phishing
emails *look* legitimate and deliver malware to the victim's endpoint. We will
show practical tools in which you can identify malicious emails and what are
the next steps of the investigation you need to take following the discovery of
malicious emails in your organization.

## How Are Email Files Used by Threat Actors?

Threat actors use emails to get initial access to the victim, from there they
can launch an attack and steal sensitive information, install backdoors, set
persistence and cause further damage. Attackers mainly use
[attachments](https://attack.mitre.org/techniques/T1566/001/) or
[links](https://attack.mitre.org/techniques/T1566/002/), to deploy and execute
their malicious intentions.

Emails can be sent with **malicious attachments** of any type: documents,
executables, images, video, scripts, etc. Usually, threat actors create
phishing emails with a message that lures the victim into opening the attached
files. Once the file is opened the first stage of the attack is executed.

Another way of delivering threats to the victim's endpoint is by using **links
to fraudulent websites** in the message, either displayed in plain sight or
hidden behind text, images, or QR codes. The end goal of the attackers is to
trick users into opening the links, so they can steal sensitive information or
deploy the first stage of the attack to the endpoint.

## How to Inspect Phishing Email Files like an Experienced Analyst

Let's start by looking at the emails like an experienced investigator, then we
can explore some free or advanced tools we might want to use.

Emails consist of a **header** and **body**, so inspecting those can provide
helpful information for an investigation and indicate whether the emails are
malicious. The message of the email can raise suspicion – for a trained eye,
the attachments and the sender domain can also be a trigger to investigate the
email. As SOC analysts and investigators, it is important to recognize the
signs and methods of malicious emails because the threat can be delivered in
several ways, and some of them are sneaky.

### Inspecting the Sender Address and Email Header

Assuming that you have blocked a known spam address and you monitor specific
keywords that indicate that an email is malicious, phishing emails still can
find their way into your inbox. To identify a malicious email you need to
closely inspect and understand who is the original sender of the email and
whether it's a legitimate entity or an attacker.

First, examine the email in the inbox, where there are several indicators that
can give away malicious emails in this stage:

1. You didn't expect to receive this email – in many cases phishing emails do
   not relate to previous connection or correspondence. If you haven't had any
   contact with the sender or their organization, flag the email as suspicious.
2. Misspelled domain or poorly written emails are a strong indication of
   suspicious emails and you must proceed with caution when handling these
   emails.
3. Take into consideration that big organizations and service providers will
   not send emails from free domains such as `gmail[.]com` or `outlook[.]com`.
   Usually these organizations have their own private domain in the format of
   `XXX@<domain_organisation>.com`

A common method used by attackers to lure victims into opening emails is by
using spoofed email addresses (to make the email look like it came from a known
and trusted entity) – there are sites that let you send emails from a given
domain or use a certain name that will appear in the sender field. But using
the information in the email's header makes it possible to detect this method.

So, if you decide to dig deeper into the email, you need to view the full email
body and header. Usually you can get it by opening the additional option menu
of the email service provider.

### Email File Header

The header contains important fields that can help in identifying malicious
emails:

- **`From`**: Who sent the message, however this field can be modified as
  mentioned above. The anomalies we mentioned about the sender apply here as
  well.
- **`To`**: Recipient's name and email address. This field can help in
  determining the real target of an email, in case we determine that the email
  is malicious the recipient can determine if the attack targets specific
  victims.
- **`Received`**: The most reliable field that provides information about the
  origin of the email. An email that is being sent from one user to another
  passes through one or more servers until it reaches its destination, similar
  to post offices. Each agent adds a layer of information that contains a
  timestamp and information about the server from which the email was received
  from, that's the reason that some emails contain several **`Received`**
  fields. Therefore, you should **inspect the header from the bottom to the
  top.**
- **`Return-Path`**: Address to return the mail.
- **DKIM signature**: Checks the DKIM DNS record of the sender to confirm the
  authenticity of the sender using public key cryptography.
- **SPF**: Defines which mail servers are authorized to send emails for a
  certain domain based on DNS record, the goal is to prevent attackers from
  sending emails using your domain from any server.
  - `Received-SPF` field - email sent from permitted servers will show up as
    "Pass" but it can still be a non-legitimate email if the account has been
    compromised. If the value of the field is either "Fail" or "Softfail", it
    indicates that the email may be spoofed or the domains didn't update their
    SPF records. Here is an example of an email that pretends to be sent by
    Haesung Tech, but the address is spoofed and doesn't pass the SPF check:

![](images/fig1.png "Sender Policy Framework (SPF) record of a spoofed address.")

Domain-based Message Authentication, Reporting and Conformance (**DMARC**) is
another email authentication protocol that uses the information provided by
either DKIM or SPF (or both). DMARC is implemented to protect the domain from
unauthorized use in phishing and spoofing attacks. And DMARC enforces
predefined policy to handle unwanted emails. DMARC, DKIM and SPF are used to
prevent phishing and unauthorized use of domains. When you inspect emails make
sure to validate the values in these fields.

Other fields in the header that you should know about:

- **`MIME-version`** - Multipurpose Internet Mail Extensions (MIME) extends the
  standard email format, it provides support for non-ASCII character sets,
  audio, video and applications. Messages that use MIME must contain the
  version field.
- **`Message-ID`** - unique identifier used in emails.
- **`X-header`** - custom email headers that can be added to the other standard
  headers. Email providers add specific headers for authentication, information
  about spam, etc. Microsoft Outlook uses `X-MS-Exchange` header and Google
  uses `X-Google-DKIM`. For example:
  - **`X-Originating-IP`** - the IP address of the sender
  - **`X-Spam-Status`** & **`X-Spam-Level`** - Spam record set by the mail
    service.

### Inspecting Attachments

As described above, emails support a wide variety of attachments, and these
files can contain malicious code (especially executables or scripts). Commonly
the email provider will alert upon opening emails that contain executables. But
that's not always the case for documents like [PDFs](../pdf/) and
[Microsoft Office files](../office/), which are often used and shared between
organizations and individuals. These non-binary files are also commonly used by
threat actors to deliver malicious payloads, and using different techniques,
execute the first stage of the attack.

Threat actors use
[double extensions](https://attack.mitre.org/techniques/T1036/007/) to
masquerade the true file type to trick users into opening files that might seem
safe (such as images or text) while it is actually an executable file. By
default, Windows hides file extensions, so `example.txt.exe` would appear as
`example.txt`

The second extension is the real extension of the file and defines which tool
will be used to open the file. Another technique used to hide the real file
extension is
[Right-to-Left Override (RLTO)](https://attack.mitre.org/techniques/T1036/002/)
– Windows supports languages that are written from right to left using a
Unicode character that causes the text that follows it to be displayed in
reverse. Threat actors use this method to blend the real file extension with
the file name.

![](images/fig2.png "Double extension in Windows when (by default) file name extension is hidden.")

**Tools for inspecting attachments:**

Before inspecting the attached files you will need to extract them (if the
email service provider didn't do that for you), there are several free tools
you can use:

- [OutlookAttachView](http://www.nirsoft.net/utils/outlook_attachment.html) -
  NirSoft software to collect all of the attachments in your Outlook mailbox,
  you can download certain files and investigate them.
- [Msg-extractor](https://github.com/TeamMsgExtractor/msg-extractor) -
  Open-source tool to extract attachments from Outlook.
- [Eml-extractor](https://github.com/diogo-alves/eml-extractor) - Open-source
  CLI tool to extract attachments from `.eml` messages.
- [UUDWIN](https://www.marks-lab.com/) - Software that extracts attachments
  from `.eml` and `.msg` emails.

*You can also [try Intezer for free](https://intezer.com/pricing/), which
automatically extracts URLs and files attached to phishing emails, analyzes
collected evidence, and uses generative AI to give you insights about email
body content and hidden HTML elements.*

### Most Commonly Used Attachment File Types in Malicious Emails

As mentioned, emails can have different types of attachments, so usually threat
actors use specific file types that can be relatively easily exploited to
deliver threats. Based on
[SonicWall's 2021 cyber threat report](https://www.sonicwall.com/medialibrary/en/white-paper/mid-year-2021-cyber-threat-report.pdf),
the following types of attachments are commonly used by both legitimate and
malicious emails, therefore emails that contain these types of attachments
should be carefully inspected.

- **Microsoft Office files** - Utilize the macros or embedded OLE objects to
  deliver threats such as
  [Emotet](https://analyze.intezer.com/families/db6abbcc-d292-4991-bc90-207c077f5e53)
  or
  [TrickBot](https://analyze.intezer.com/families/b70bfda8-0db1-4559-bc12-2525e7ac9e3b).
  To properly analyze these files you need to understand the format and the
  techniques threat actors implement to hide malware – check out our previous
  blog about [how to analyze malicious Office files](../office/).
- **PDFs** - By nature these files can have links,
  [JavaScript code](https://intezer.com/blog/alert-triage/ai-insights-revolutionizing-threat-analysis/),
  and any file type can be embedded into a PDF file. All of these features are
  used by threat actors to hide and deliver threats. On top of that, there are
  hundreds of PDF readers and many known vulnerabilities that can be exploited
  by specially crafted PDF files and allow threat actors to execute code on an
  endpoint. There are tools and methods to inspect PDF files and identify
  malicious files, which we go into detail about in this post on
  [how to analyze PDF files](../pdf/).
- **ZIP and RAR archive files** - These are compressed file formats, a
  malicious executable can be compressed and attached to an email using one of
  the archive types. A well-configured email gateway *should* be able to alert
  and possibly prevent the email from arriving at its final destination. But
  that's not always the case for password-protected archive files, which
  usually are not blocked, as its contents are not able to be inspected due to
  encryption. Another technique threat actors use to evade detection is sending
  one archive that contains a decoy file and another archive that contains the
  actual malware. This technique was used by the group behind
  [NanoCore RAT](https://www.bleepingcomputer.com/news/security/specially-crafted-zip-files-used-to-bypass-secure-email-gateways/).
  Another technique using ZIP files we described in our research report about
  [PufferPhishing](../../../blog/2023/05/how-hackers-use-binary-padding-to-outsmart-sandboxes/),
  which involves using binary padding to inflate the size of a malicious file
  and then compressing it for delivery.
- **ISO and IMG disk image** - These files are commonly used by attackers to
  evade detection from email-based Antivirus scanners. The use of ISO files
  allows the threat actor to
  [bypass the Mark-of-the-Web controls](https://attack.mitre.org/techniques/T1553/005/),
  resulting in execution of the malware without warning to the user. In
  addition, starting from Windows 8.1, double clicking the file will
  automatically mount the ISO file and install the software (malware in the
  case of malicious files). Threat actors use this format to evade detection
  while fairly easily installing malware.

Inspecting attachments can be time consuming and hard and in some cases you may
come to the conclusion that the file is trusted. To reduce the time you spend
on analysis of files, you can connect
[Intezer](https://intezer.com/forensic-ai-soc/) to have it automatically
collect files and use its proprietary code reuse database to extracts valuable
information: whether the files are malicious, their behavior, IOCs, or strings
with other samples in the database. This way you will not have to waste time on
trusted files, known malicious files, or understanding unique threats.

### Inspecting Links, Manually or Automatically

Most of the malicious emails use links and trick victims into visiting
[malicious URLs](../../../docs/url-analysis/part1/),
which will download malware or steal victim's sensitive information. Links can
be in plain sight and appear as legitimate links to known sites such Amazon or
Facebook, where in fact the domain is fake or a typosquatter. In other cases,
the links are hidden behind text or images such as buttons or coupons, making
it harder to notice the malicious URL.

You'll need to collect any URLs from the emails and inspect them using a URL
scanner, to know if the URL was seen in previous attacks. If it is a known
malicious address then the email is certainly malicious as well. But if the
results are not conclusive or you wish to dig deeper, it is worth to further
investigate the domain and see if it was linked to any known campaigns or
threat actors.

One of the tools you can use for inspecting URLs is
[Intezer's URL scanner](https://intezer.com/blog/product-updates/automatically-scan-urls-and-analyze-malware/).
**Intezer can connect to abuse inboxes or email security systems to
automatically collect and classify file attachments or URLs, allowing you to
accelerate response.** When you submit the extracted URL, then the scanner will
produce a report that includes a screenshot of the website, the verdict of the
URL (is it malicious or not), information about the domain and indicators such
as if the address is blacklisted, was it used in phishing campaigns, etc.

![](images/fig3.png "URL scan report in Intezer.")

## Investigating Phishing Email Campaigns (with Real Life Examples)

Our research team discovered a
[phishing campaign](../../../blog/2021/07/global-phishing-campaign-targets-energy-sector-and-its-suppliers/)
that used **spoofed or typosquatted emails** to make them look like part of a
normal business-to-business (B2B) correspondence, targeting large international
companies in the energy, oil & gas, and electronics industries. The attachments
were primarily an **IMG, ISO or CAB file**. We submitted the attachments to
Intezer for analysis and discovered that they belong to known information
stealer malware such as Agent Tesla, Loki, Snake Keylogger, and Formbook.

![](images/fig4.png "Genetic analysis of the ISO sample containing Snake Keylogger.")

If the investigation of the file concludes that the file is malicious, it means
that the email that delivered it is also malicious and should be further
investigated.

### Real Life Phishing Email Example: A Short URL with a Packed Payload

Intezer's research team discovered a
[phishing campaign](../../../blog/2021/07/targeted-phishing-attack-against-ukrainian-government-expands-to-georgia/)
that targeted government entities in Georgia. The attack flow starts with a
phishing email containing a malicious shortened URL, as can be seen in the
screenshot below. The URL redirects to a Command and control (C2) where a ZIP
file or malicious document is hosted. The ZIP file contains a malicious file,
and in some of the phishing emails also a harmless PDF file. The malicious
attachment varies between RTF, DOC, PDF, JS, LNK, or EXE. The main goal of the
attachment is to drop the packed payloads from the C2. The packed executable
loads an AutoIt payload into memory, once executed the malware steals files
with predefined extensions.

Here's one of the phishing emails, which was sent to the Ukrainian government.
Translation from Ukrainian – Subject: "Payments to ATO Veterans." Content: "It
must be filled in and sent back."

![](images/fig5.png "Phishing email targeting government entities in Georgia.")

This campaign demonstrates the challenge of inspecting emails, since the
variety of file types that are compressed into the ZIP file can make the
analysis more challenging and time consuming. You need the skills and the tools
to analyze each file type.

Another campaign that was recently discovered by our research team is a
[new IcedID campaign](../../../blog/2022/03/conversation-hijacking-campaign-delivering-icedid/)
that was delivered by a phishing email that uses **conversation hijacking** –
threat actors use previously stolen email messages and use them in future
attacks. This technique makes the emails look more legitimate and convincing.
This campaign takes the hijacking a step further; the threat actors are now
using the email address of the victim that they stole the original email from
to make the phishing email even more convincing.

## Automating Phishing Email Investigations

Inspecting email files can be hard because emails are frequently used, meaning
there are lots of files to cover. Emails can also contain URLs to malicious
sites or different types of attachments, all of which require different sorts
of skills and methods to thoroughly inspect all of the artifacts.

There is a solution for handling a lot of phishing emails, even with QR codes
hiding links or tricky attachments containing malicious scripts –
[automate your email pipeline](https://intezer.com/blog/alert-triage/automated-phishing-investigation/).
Building custom workflows can help, but
[SOAR playbooks have some drawbacks](https://intezer.com/blog/alert-triage/intezer-and-soar/).
But to make phishing investigations easier and less demanding for you security
team, it is worth setting up a smart solution with native automation for
suspicious emails.

Automatically submitting suspicious emails and extracting evidence with using a
platform such as Intezer means that instead of spending time investigating
files that are trusted or known threats, you can focus on the
[serious threats](https://intezer.com/blog/incident-response/automated-threat-escalations-focus-on-what-matters/)
identified in your suspicious emails. By piping emails and attachments into
Intezer for analysis for you get comprehensive reports about each email, laong
file, link, and other artifacts. Your team doesn't have to waste time on false
positives. If the file is known malicious or a unique threat, you can
automatically trigger the next step in your incident response process. The goal
is to focus your time and effort only on the truly serious threats.

Without smart automation for handling phishing threats, security teams are
stuck manually triaging an endless queue of suspicious emails.
