# Technical Analysis: Pacha Group Deploying Undetected Cryptojacking Campaigns on Linux Servers

> Pacha Group, likely of Chinese origin, runs undetected Linux.GreedyAntd cryptojacking campaigns whose IPC-linked multi-stage design evades dynamic analysis.

- Source: https://research.intezer.com/blog/2019/02/technical-analysis-pacha-group/
- Published: 2019-02-28
- Authors: Ignacio Sanmillan
- Tags: linux, botnet
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

Cryptomining malware, also known as cryptojacking or cryptocurrency mining
malware, refers to software developed to take over a computer's resources and
use them for
[cryptocurrency mining](https://www.webopedia.com/TERM/C/cryptocurrency-mining.html)
without a user's explicit permission.

There are several
[reports](https://www.symantec.com/blogs/threat-intelligence/cryptojacking-modern-cash-cow)
documenting this newer malware breed and how it has become more popular in the
last few years.

[Antd](https://translate.google.com/translate?hl=en&sl=auto&tl=en&u=https%3A%2F%2Fwww.acey.me%2Fantd-miner%2F)
is a miner found in the wild on September 18, 2018. Recently we discovered that
the authors from Antd are actively delivering newer campaigns deploying a broad
number of components, most of them completely undetected and operating within
compromised third party Linux servers. Furthermore, we have observed that some
of the techniques implemented by this group are unconventional, and there is an
element of sophistication to them. We believe the authors behind this malware
are from Chinese origin. We have labeled the undetected Linux.Antd variants,
Linux.GreedyAntd and classified the threat actor as **Pacha Group**.

![](images/fig1.png "String Reuse from Antd and GreedyAntd")

## Technical Analysis

### Infrastructure Overview

Based on our findings Linux.GreedyAntd’s operations closely resemble previous
[cryptojacking campaigns](https://intezer.com/blog/cloud-security/competition-for-cryptocurrency-mining-foothold-on-the-cloud/)
deployed by Pacha Group in the past. A resumed overview of the current
infrastructure is as follows:

![](images/fig2.png)

The attack chain commences by intruding into a given vulnerable server. Based
on the services the compromised servers were publicly exposing, we can assume
the attackers opted to launch a brute-force attack against services like
WordPress or PhpMyAdmin, or used a known exploit for an outdated version of
alike services. The following is an overview of the open services and known
vulnerabilities found in one of the compromised systems:

![](images/fig3.png)

Once the attackers are able to break into a given compromised server, they will
run a series of stages in their attack chain.

### Main Dropper

Once a system is compromised the first implant that will be executed is a UPX
packed statically linked stripped ELF. ![](images/fig4.png)

This ELF binary is the main component in the intrusion stage and it is worth
dedicating a separate section to. We will refer to this binary as the ‘first
stage’ or ‘main dropper’ throughout the blog.

This binary is responsible for various tasks. One of the first actions it will
take is to assure that the current compromised server is not infected by other
cryptominers by using a technique similar to that of a ‘bot kill’ approach (as
known in the DDoS scene) killing any other cryptominer that is currently
running in the system. The following list contains the file names belonging to
known foreign cryptominers: ![](images/fig5.png)

We recognized that one of the file names in this list is the Korkerds miner
reported by
[TrendMicro](https://www.trendmicro.com/vinfo/us/security/news/cybercrime-and-digital-threats/cryptocurrency-mining-malware-targets-linux-systems-uses-rootkit-for-stealth)
as well as other known miners such as
[DDG](https://www.bleepingcomputer.com/news/security/mining-botnet-targeting-redis-and-orientdb-servers-made-almost-1-million/)
or [XMRig](https://github.com/xmrig/xmrig). This reinforces the assumption that
the list is indeed a blacklist of file names of known miners operating in the
wild.

Processes with a filepath residing in `/tmp/`, `/usr/tmp` or `/dev/shm` will be
killed. We concluded that the purpose of this aggressive behavior is intended
to discover further miner processes or malware that were not covered in the
initial miner process blacklist.

![](images/fig6.png)

We have also noticed that some implants were checking for potential JBOSS
compromised servers by attempting to access specific paths in order to detect
and restrict potential operational webshells or dropped binaries by removing
all available file permissions to them. There is a github project called
[JexBoss](https://github.com/joaomatosf/jexboss) regarding JBOSS serialization
vulnerabilities that uses these same paths, suggesting that authors behind
other cryptomining campaigns could be using it to spread their infrastructures,
answering why these paths are being searched for:

![](images/fig7.png)

As previously mentioned, evaluation of the current system in order to know if
it is already compromised has been accomplished. Furthermore, there may be a
chance that the current system is already compromised by the same group. In
order to figure out whether this is the case all process names are checked
again with the end goal of recognizing any familiar process names used by the
same group and if found, the process will terminate. This could potentially
work as a possible vaccine to be used by some miner-protection solutions
against this specific miner:

![](images/fig8.png)

After the first stage has completed an initial reconnaissance for the running
processes, it proceeds to create a random string to rename itself. It also
overwrites some known memory locations where the original process name resides
and overwrites them with a fake process name (in this case `[kworker/1:7]`). An
example of such memory locations is `argv[0]`:

![](images/fig9.png)

Furthermore it will fork itself and detach from its parent to become an
independent process running on a different session as a means to create a fresh
new process:

![](images/fig10.png)

Lastly, the current session username is checked and the control flow will
diverge accordingly:

![](images/fig11.png)

As demonstrated in the previous screenshot, the main difference is that if the
file was executed as root, persistence mechanisms would be enforced.

### Persistence Mechanisms

The applied persistence mechanisms consist mainly of a given dropped implant
saved as ‘mand’ followed by installing a Systemd service which will grant its
persistence in the system. In addition, the timestamp of the dropped implants
will be replaced as for the one of `/usr/bin/find` as a means to make the
dropped file unnoticed in the filesystem.

![](images/fig12.png)

A Systemd unit file will be decoded and dropped as ‘systemd-mandb.service’
masquerading the genuine mandb service. The following is the decoded Systemd
unit file:

![](images/fig13.png)

It is important to highlight that this persistence measure will make the
intrusion harder to spot for the untrained eye since it is not the average
cron-job that most Linux malware tend to use. Furthermore we spotted other
components of this campaign dropping and installing initrd scripts as well as
the following one:

![](images/fig14.png)

After persistence measures have been enforced, several components will be
downloaded to the current compromised system to remain with the attack chain:

![](images/fig15.png)

### Multi-Stage Architecture

The following diagram is a simplified version of the various components that
make up the malware’s main infrastructure: ![](images/fig16.png)

We can assume that the main reason for having such a broad infrastructure
involving a large number of components is to make it more resilient to server
shutdowns as well as to provide a factor of modularity. Furthermore, having
this amount of components interconnected with each other also implies to invest
a much greater effort in order to clean a given compromised system.

These components will run according to a small protocol involving the main
dropper and all remaining components executed via a shared named pipe. This
execution protocol is the following:

![](images/fig17.png)

The majority of the secondary stagers create (or just open if it already
exists) a named pipe on execution with write permissions in which they write
their pid to:

![](images/fig18.png)

On the other hand, the main dropper serializes each stager by reading and
logging the contents of the named pipe therefore retrieving each stager’s pid.
This way the main dropper acts as a manager for each active stager in the
system:

![](images/fig19.png)

The first stage’s main threat will continue execution attempting to update the
available stagers by downloading them in intervals on an infinite loop:

![](images/fig20.png)

In addition, a timed routine will be executed by triggering a `SIGALRM` signal
also in intervals, handling this signal via sigaction sycall, and therefore
pivoting control of execution to its correspondent signal handler
intermittently. This same technique has been spotted in various components of
this malware’s infrastructure:

![](images/fig21.png)

This installed signal handler is mainly used to drop further artifacts using
embedded one-liner python scripts such as the following:

![](images/fig22.png)

This script will drop further stages as well as further scripts. The following
is an example of such scripts:

![](images/fig23.png)

### GreedyAntD Miner Client

The deployed miner instance is a XMRig variant. We can confirm this via code
reuse:

![](images/fig24.png)

We can also confirm the miner shares code with other components from the same
infrastructure, also based on code reuse:

![](images/fig25.png)

It uses the [Stratum](https://en.bitcoin.it/wiki/Stratum_mining_protocol)
mining protocol and connects to a XMRig proxy in order to conduct the mining
operation. We assume the reason to use this specific protocol is to prevent to
deploy its clients with encoded configurations containing the target wallet
address they will be mining to, instead they connect to already configured
proxies:

![](images/fig26.png)

The following are the Proxies that interacts with:

![](images/fig27.png)

Furthermore, we can also confirm it is using Stratum by sniffing the Miner’s
stream to these proxies:

![](images/fig28.png)

We notice that the client and server are exchanging information encoded as
json-rpc strings, which is commonly used in stratum mining protocol.

Highlighted are the different cryptocurrency mining algorithms that the client
supports. These names can be seen in the main XMRig-proxy GitHub repository:

![](images/fig29.png)

The following screenshot is a process list view on htop of a compromised
system. Highlighted are some of the malicious processes related to the
campaign:

![](images/fig30.png "htop of a compromised system")

## Connections with Linux.HelloBot

Among the artifacts hosted in GreedyAntd’s servers, we managed to find a single
component not related to the same cryptojacking operation just previously
discussed and leveraged by Pacha Group. This file was hosted on a compromised
third party server and its main purpose was to drop a xmr-stak json
configuration. This json file was the following:

![](images/fig31.png)

When we analyzed this binary for code reuse connections we found it shared a
significant amount of code with Linux.HelloBot, a Chinese bot discovered by
Intezer along with MalwareMustDie in January 2019:

![](images/fig32.png)

After analyzing the code connections we came to the realization that both
samples were sharing the same instance of some static libc implementation:

![](images/fig33.png)

Library similarities tend to not be as relevant in some specific scenarios in
regards to finding connections between threat actors. However, in this case
these library similarities seem to be relevant enough to consider a potential
link between these two threat actors. Especially since from viewing all
different x86 libc versions in our database, it only matched with
Linux.HelloBot’s statically linked libc. In addition, this libc instance has
identical code in both samples which implies it was compiled with the same
compilation flags. This reinforces that the particular libc instance may be a
potential link to connect these binaries with a single author, also taking into
account that both of them have indicators that suggest they have Chinese
origin.

## Conclusion

This
[cryptominer use](../../../2021/01/a-rare-look-inside-a-cryptojacking-campaign-and-its-profit/)
case is another example of an undetected Linux malware operating in the wild.
After conducting more research we concluded that the approach of
interconnecting all second-stagers to a manager (this being the main dropper)
via IPCs is successful for anti-dynamic analysis. In order for any of the
secondary-stages to run successfully they will need to be present with the main
dropper in a given system. This implies that behavior and dynamic analysis will
fail if any of the second-stager components are analyzed independently without
tampering the original sample. This may explain why the majority of the
components in this malware’s infrastructure remain practically undetected:

![](images/fig34.png)

## IOCs

```
185.165.169.6
185.10.68.100
4d1a6151166048ea066c6b5918fbb02a43ed357e3e7eb75bc73970a97008337e
chn
206287e22445431ccab0a574f3002e28d1aaffa5153ba66f2a754d1f92b90a78
chn.unpacked
9119d47ee2b6e7bf94245699fec1432042e30255d9f64289f8e0aca56570eab3
ofd
096f8f387200fa70dddb2bfe5a77a50c88acb155d4f296f1fa6cb09109053246
ofd.unpacked
ee0ab03909ca433deb9161e831512c5bd6c64ffbc9332c3eea14b85b996ba882
rlr
1161fbe0a9ae1c5e0792d23682b602990af31c6847865220cf4f2f91981d426c
rlr.unpacked
3c3379284417070983da222f8ea347a4166c28a2ba3445e19f92b10b9b539573
_j.jpg
9ff1bf60e35912141c74728738c3af105d06ea8fa9c0cbd7a4b196ec1cdc9e22
j.jpg
dafac060867643d27a81e99e3753d155658e5f4a7f359317e0e8609fc7d14373
jm
069a87fbd966df854f55d82fc98f89ef394cee59b352fba5fb402887892a4161
jm.unpacked
84165c21fc144894c5fe674cfd06edafd4b95d52abf86afef4d61db91099bf8a
jpp
544e71e3a7ff1f1f0e902cef00156aa157790f7c3450870ca9272936443e05af
jpp.unpacked
a9656439d1ac3881c1ba9e0f2fd462b8a4469bf79035233517eae65ed6afafd0
jrd
e43d381c43749d7d267d207273ef3b634bfaeb0ff76f8e2cd6e0b27c6e3b07c8
jrd.unpacked
3c3379284417070983da222f8ea347a4166c28a2ba3445e19f92b10b9b539573
_j.sh
b6455e0335ad99028656b5b847fd5d530bc079b2524b2ecbf4f06d5e69473e7d
lmmml
39904faf4a620aa3a9e9ece3022f0bced20ef7684e0f352f99267e7c462d227c
lmmml.unpck
910fea37c73fc328522e04c77d1ad555c990f0376960770698bd3590c5b1b485
nvn
81b8860ecf21a73de8663188962fb1dae5a5c17e7b6f4ac41e0198d12497838f
nvn.unpacked
371f52a238d4be6eb8d7fd0130684f4286681f09adb61fbca3bdfacef8c747f7
sds
b72074b6c75b4fe5ea74e2db716f488a356d9d879c6d3aa5e9ed4bb786993761
sds.unpacked
42a423b6107f2186964ac9a1e7882a50f6b5cb9f96926dd2a69b1fc5eaba81d6
lmmml.1
e9a06f7183f7e06d8e414e16caae769a3859fcca20acae735f6744712f84b3e5
sss
4f9e77b4e0d80ea74ba861ab54b7360df7b823f24fd9cedb1fd44a29da70b11f
sss.unpacked
0940472a185099df2f814bcedbc1c913a7075168ab90d63249c6301849c1d93f
dld
6ff5e36d2999f8593cc4daa5e7c633abe0f28b0cba9da0339fc8d3cb7f6090a3
dld.unpacked
c70423e5d44cc31df70a69d65e56be1621956bccec2a3a68a69195ecccd4e881
_jj.jpg
452ed9cf53aed0afdc7900ca855f652a7c1585c2b03b27e6f3224fb6204da25a
_jj.sh
6f3add7ec36a710973d09b814082e105b848cd78f2769ba6bbc946f59f463457
ldl
045d7afaf53692607e7433a4fe8f19b2e3790414c649c8630086039d88935a02
ldl.unpacked
0940472a185099df2f814bcedbc1c913a7075168ab90d63249c6301849c1d93f
olo
6ff5e36d2999f8593cc4daa5e7c633abe0f28b0cba9da0339fc8d3cb7f6090a3
olo.unpacked
7ae8c6c65955fb9340b07afd380bbf3383b5030a92ba204cd61ca21c13a955e8
 _z.jpg
9e049a51741f22403a9c08d3d7625ad4761cbfda5a8a051f6e8195e0f6a8e9cd
z.jpg
7ae8c6c65955fb9340b07afd380bbf3383b5030a92ba204cd61ca21c13a955e8
 _z.sh
9e049a51741f22403a9c08d3d7625ad4761cbfda5a8a051f6e8195e0f6a8e9cd
z.sh
Cceddd7e9a7ddb4991776239cb0b941d061ac21db00b1021a8c45660f52e56b7
E2e07782dbfddeb95661c7360db5113c9b035cfb8e43e038106bd0f537553b36
40d8d89aa19ca4121ab583758692752964402923917da766f39a32cbc8bdd6dd
```
