# "EternalMiner" Copycats exploiting SambaCry for cryptocurrency mining

> CopyMiner: copycats exploiting the SambaCry Samba vulnerability to install a cryptominer and the Tsunami backdoor, with daily updates from backup servers.

- Source: https://research.intezer.com/blog/2017/07/eternalminer-copycats/
- Published: 2017-07-26
- Authors: Omri Ben Bassat
- Tags: cryptomining, linux
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


About eight weeks ago, a critical RCE vulnerability present in every Samba
version since 2010 was reported and patched. This vulnerability is mostly known
as "SambaCry" after the famous WannaCry attack targeting Windows systems
vulnerable to "EternalBlue" SMB exploit. The vulnerability lies in a logical
bug, which enables an attacker with write-only access to a share to load a
malicious samba module and execute arbitrary code.

Immediately after writing & publishing the first public
[POC code](https://github.com/omri9741/cve-2017-7494), I wrote a yara signature
for a possible exploit payload and we began monitoring our data feeds for
threats exploiting this vulnerability. Over time, we have noticed countless
bind/reverse shells and droppers. Most of them are either Metasploit payload or
other publicly available implants.

On June 9th, Kaspersky Labs published an article about
"[EternalMiner](https://securelist.com/sambacry-is-coming/78674/)" — a
financially driven cryptocurrency mining malware turning victim machines into
"a workhouse on a large farm, mining crypto-currency for the attackers".

Less than a week later, we were able to observe copycat cybercriminals actively
exploiting the vulnerability with a similar yet improved setup for better
cryptomining & control over the victim machines. Naturally, we decided to name
this threat "**CopyMiner**"!

## Overview

As we mentioned above, the copycats used a similar yet improved setup.
Implementing a multistage flexible approach with daily updates from multiple
backup servers, a persistent backdoor for better control of the victim machines
and even a more efficient
[cryptominer](../../../2021/01/a-rare-look-inside-a-cryptojacking-campaign-and-its-profit/)
for better consumption of the victim machine’s resources! **The copycats took
EternalMiner to the next level as an operation suitable for multiple victim’s
machines**. In the following sections, we will drill down on each component of
the following graph, comparing CopyMiner’s setup vs EternalMiner:

![](images/fig1.png)

## Technical Analysis

### "CopyMiner" Dropper

On the 14th of June a small unique dropper
[sample](https://www.virustotal.com/en/file/444d0fae73e1221b81dc6c5d53cf087afa88248fc22ef36e756841ab04df24a8/analysis/1497997544/)
was uploaded to VirusTotal. The dropper starts executing from
"samba_init_module" exports then tries to fetch a payload from the server &
execute it as root user in the background. This is done using a hard-coded
obfuscated bash one-liner as you can see in the picture below.

![](images/fig2.png "sample hash: 444d0fae73e1221b81dc6c5d53cf087afa88248fc22ef36e756841ab04df24a8")

### Payload

The payload (`http://update1.sdgndsfajfsdf[.]info/u1`) is a short bash script
which relays on system tools for 3 main tasks

1. Setup daily updates from 3 different backup servers using crontab.

- `http://update.sdgndsfajfsdf.info/upd`
- `http://update.sdgsafdsf.pw/upd2`
- `http://update.omfg.pw/upd3`

2. Drop and execute Tsunami backdoor & CPUMiner in the background.
3. Prevent further exploitation by other players who might compete for
   resources(patch `smb.conf`).

We were also able to fetch the daily update script from a live server
(`http://update.sdgndsfajfsdf.info/upd`). It seems that this script is just a
stripped version of the payload script, lacking the daily updates setup
functionality:

![](images/fig3.png)

### CPUMiner

CPUMiner is an open source cryptomining command line utility supporting
multiple coins/algorithms. It seems that the attackers used
"[cpuminer-multi](https://github.com/tpruvot/cpuminer-multi)" – multithreaded
more efficient version of the original @poolers cpuminer used by EternalMiner.
This sample was "upgraded" it in a similar way as EternalMiner so it could run
standalone without command line parameters. Once started, the upgraded cpuminer
automatically login into the attackers private mining pool server
(`p.theywant[.]in:8080`), unlike the public mining pool server used by
EternalMiner (`xmr.crypto-pool[.]fr:3333`).

![](images/fig4.png)

### Tsunami Backdoor

Alongside CPUMiner the payload script would also fetch & execute "Tsunami"
(also known as Kaiten) which is an old Linux irc backdoor/ddos botnet best
known for been used to infect
[IOT devices](https://researchcenter.paloaltonetworks.com/2017/04/unit42-new-iotlinux-malware-targets-dvrs-forms-botnet/)
and
[OSX systems](https://www.welivesecurity.com/2011/10/25/linux-tsunami-hits-os-x/)
in the past. The source is
[publicly available](https://dl.packetstormsecurity.net/irc/kaiten.c) and could
be used by anyone. This sample
(`d8e93252f41e8b0d10cffa92923eeab94c6c42e8acc308e91340df102042c8c8`) is
configured with hardcoded c2 irc server (`asdgsd.uselesslongdomain[.]info`). We
were able to connect to the live irc server using credentials hard coded in the
malware. In the following picture you can see two victims currently logged in
(running as root user).

![](images/fig5.png "Tsunami Backdoor")

### Quickly adopting

We found that the domain used to host the payload file (`sdgndsfajfsdf[.]info`)
was registered on June 13th. Not only that but the first malicious
CPUMiner/Tsunami samples were uploaded to VT a day after(June 14th), meaning
that the attacks started within a short time of only 4-5 days after the
original EternalMiner publication!

## Indicators of Compromise

### Files

```
444d0fae73e1221b81dc6c5d53cf087afa88248fc22ef36e756841ab04df24a8 Dropper
2b96805abdbd1d9ca03d584e48fcfb30740d051dfc93248ff3e21b3a831c0e1a Payload
22a8dc0603005e3eee49706330c6a5b90214dcb0b3d2f89411fa10a6b3942e3b update file(daily)
d8e93252f41e8b0d10cffa92923eeab94c6c42e8acc308e91340df102042c8c8 Tsunami
162de4e95e5e5d35d80ca4cf752c80b2b32b52c9e5fef5551caa20b0d5ed83af Tsunami
26a717a7a14f10880a2869949814400b31d1f4c9cc45384be38289b012587468 Cpuminer
```

### Network

```
update1.sdgndsfajfsdf[.]info
update.sdgndsfajfsdf.info
update.sdgsafdsf.pw
update.omfg.pw
188.138.109.103
asdgsd.uselesslongdomain.info
p.theywant.in
```
