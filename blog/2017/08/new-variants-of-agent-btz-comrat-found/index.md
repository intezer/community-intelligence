# New Variants of Agent.BTZ/ComRAT Found: The Threat That Hit The Pentagon In 2008 Still Evolving; Part 1/2

> New Agent.BTZ/ComRAT variants from the Turla group: about a dozen new samples and over 70 unknown IPs and domains abusing satellite internet providers.

- Source: https://research.intezer.com/blog/2017/08/new-variants-of-agent-btz-comrat-found/
- Published: 2017-08-07
- Authors: Omri Ben Bassat
- Tags: rat, apt, apt_russia, backdoor
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Agent.BTZ–also known as ComRAT–is one of the world’s oldest known
state-sponsored threats, mainly known for the
[2008 Pentagon breach](http://articles.latimes.com/2008/nov/28/nation/na-cyberattack28).
Technically speaking, Agent.BTZ is a sophisticated user-mode RAT developed and
operated by the Turla group in conjunction with Snake/Uroburos rootkit. In the
past few months, we conducted research on Agent.BTZ’s code-base and how it
evolved using Intezer Code Intelligence™ technology. **Based on our research
conclusions, we were able to hunt about a dozen new samples and more than
seventy previously unknown live IP & DNS addresses indicating the ongoing abuse
of
[satellite internet providers](https://securelist.com/satellite-turla-apt-command-and-control-in-the-sky/72081/)
operating in both Africa & the Middle East**.

This is a short memo regarding our findings from the past few months; get more
details in Part 2 of this blog,
[a whitepaper we wrote on Agent.BTZ/ComRAT](../../../2017/09/new-variants-of-agent-btz-comrat-found-part-2)
describing in more details how we found these new variants using our
technology, along with a thorough analysis of the new samples.

## Dropper

Although the code itself was written from scratch and has nothing to do with
WinRAR, the adversary tried to mimic WinRAR’s SFX installer. Resource data was
duplicated, including icons and layouts used by the original installer, as you
can see in the following screenshot:

![](images/fig1.png)

Once executed, the dropper installs activeds.dll – a proxy dll which is loaded
directly to explorer.exe once the machine reboots. The purpose of this proxy
dll is to load the malware’s main payload – stdole2.tlb. The dropper then also
deletes any previous installation of Agent.BTZ if it exists. This is done using
a hard-coded file path:

- `C:\Documents and Settings\<USER>\Application Data\Microsoft\Windows\Themes\termsvr32.dll`
- `C:\Documents and Settings\<USER>\Application Data\Microsoft\Windows\Themes\pcasrc.tlb`

\*\*Note: These file names were first used by Agent.BTZ in late 2014, as you
can see in this
[automatic Dr.WEB report](https://vms.drweb.ru/virus/?i=4268384)

Once finished, the dropper renames and self delete using the following command
line:

- `C:\WINDOWS\system32\rundll32.exe C:\DOCUME~1\<USER>~1\APPLIC~1\MICROS~1\Windows\stdole2.tlb,UnInstall C:\~$.tmp`

Samples found:

- `69690f609140db503463daf6a3699f1bf3e2a5a6049cefe7e6437f762040e548`
- `6798b3278ae926b0145ee342ee9840d0b2e6ba11ff995c2bc84d3c6eb3e55ff4`

## stdole2.tlb

As previously mentioned, this file is the main component installed by the
fake-sfx dropper and loaded by activeds.dll. We extracted the configuration
from each sample in order to obtain the c2 address and inner version
("`PVer`"), which is built into every Agent.BTZ sample. In the past,
Agent.BTZ’s developers have used an incremental value to indicate the inner
build version, the last known value is 3.26 as
[published by G-Data in late 2014](https://www.gdatasoftware.com/blog/2014/11/23937-the-uroburos-case-new-sophisticated-rat-identified),
It seems that the developers have reacted to G-Data’s publication and have
stopped using an incremental value. New variants are now using a different
numbering system of `0.8/9.<RANDOM_VALUE>`, making it more difficult for
researchers to track the exact version of the samples.

![](images/fig2.png "Example configuration extracted from one of the samples – PVer 0.9.1528434231.")

Even without the PVer numbering, we were able to determine using our technology
that these samples are from a newer version, which is based on the latest known
versions of Agent.BTZ – 3.25 / 3.26. These are the two top files you can see in
the following screenshot:

- [4e553bce90f0b39cd71ba633da5990259e185979c2859ec2e04dd8efcdafe356 (VirusTotal)](https://www.virustotal.com/en/file/4e553bce90f0b39cd71ba633da5990259e185979c2859ec2e04dd8efcdafe356/analysis/)
- [3a6c1aa367476ea1a6809814cf534e094035f88ac5fb759398b783f3929a0db2 (VirusTotal)](https://www.virustotal.com/en/file/3a6c1aa367476ea1a6809814cf534e094035f88ac5fb759398b783f3929a0db2/analysis/)

Both of these files were uploaded almost three years ago to VT(!)

![](images/fig3.png "A screenshot from the Intezer Analyze™ product displaying a list of files in our database that share pieces of code with one of the new samples.  These pieces of code are specific to the Turla malware family, and were not seen in any other malicious or legitimate software.")

Samples found:

- `6ad78f069c3619d0d18eef8281219679f538cfe0c1b6d40b244beb359762cf96`
- `49c5c798689d4a54e5b7099b647b0596fb96b996a437bb8241b5dd76e974c24e`
- `e88970fa4892150441c1616028982fe63c875f149cd490c3c910a1c091d3ad49`
- `89db8a69ff030600f26d5c875785d20f15d45331d007733be9a2422261d16cea`

## Indicators of Compromise

### Files

```
69690f609140db503463daf6a3699f1bf3e2a5a6049cefe7e6437f762040e548 	dropper
6798b3278ae926b0145ee342ee9840d0b2e6ba11ff995c2bc84d3c6eb3e55ff4 	dropper
73db4295c5b29958c5d93c20be9482c1efffc89fc4e5c8ba59ac9425a4657a88 	activeds.dll
50067ebcc2d2069b3613a20b81f9d61f2cd5be9c85533c4ea34edbefaeb8a15f 	activeds.dll
380b0353ba8cd33da8c5e5b95e3e032e83193019e73c71875b58ec1ed389bdac 	activeds.dll
9c163c3f2bd5c5181147c6f4cf2571160197de98f496d16b38c7dc46b5dc1426 	activeds.dll
628d316a983383ed716e3f827720915683a8876b54677878a7d2db376d117a24 	activeds.dll
f27e9bba6a2635731845b4334b807c0e4f57d3b790cecdc77d8fef50629f51a2 	activeds.dll
a093fa22d7bc4ee99049a29b66a13d4bf4d1899ed4c7a8423fbb8c54f4230f3c 	activeds.dll
6ad78f069c3619d0d18eef8281219679f538cfe0c1b6d40b244beb359762cf96 	stdole2.tlb
49c5c798689d4a54e5b7099b647b0596fb96b996a437bb8241b5dd76e974c24e 	stdole2.tlb
e88970fa4892150441c1616028982fe63c875f149cd490c3c910a1c091d3ad49 	stdole2.tlb
89db8a69ff030600f26d5c875785d20f15d45331d007733be9a2422261d16cea 	stdole2.tlb
0e0045d2c4bfff4345d460957a543e2e7f1638de745644f6bf58555c1d287286 	other
bdcc7e900f10986cdb6dc7762de35b4f07f2ee153a341bef843b866e999d73a3 	other
fac13f08afe2745fc441ada37120cebce0e0aa16d03a03e9cda3ec9384dd40f2 	backdoor
bae62f7f96c4cc300ec685f42eb451388cf50a13aa624b3f2a019d071fddaeb1 	other
```

### Network

```
81.199.34[.]150 	
elephant.zzux[.]com 	
angrybear.ignorelist[.]com 	
bigalert.mefound[.]com 	
bughouse.yourtrap[.]com 	
getfreetools.strangled[.]net 	
news100top.diskstation[.]org 	
pro100sport.mein-vigor[.]de 	
redneck.yourtrap[.]com 	
savage.2waky[.]com 	
tehnologtrade.4irc[.]com 	
81.199.160[.]11 	
forums.chatnook[.]com 	
goodengine.darktech[.]org 	
locker.strangled[.]net 	
simple-house.zzux[.]com 	
specialcar.mooo[.]com 	
sunseed.strangled[.]net 	
whitelibrary.4irc[.]com 	
bloodpearl.strangled[.]net 	
getlucky.ignorelist[.]com 	
proriot.zzux[.]com 	
fourapi.mooo[.]com 	
nopasaran.strangled[.]net 	
78.138.25[.]29 	
showme.twilightparadox[.]com 	
mouses.strangled[.]net 	
82.146.175[.]69 	
mouses.strangled[.]net 	
178.219.68[.]242 	
ftp.fueldust.compress[.]to 	
ftp.linear.wikaba[.]com 	
ftp.mysterysoft.epac[.]to 	
ftp.scroller.longmusic[.]com 	
ftp.spartano.mefound[.]com 	
fueldust.compress[.]to 	
linear.wikaba[.]com 	
mysterysoft.epac.to 	
safety.deaftone[.]com 	
salary.flnet[.]org 	
scroller.longmusic[.]com 	
spartano.mefound[.]com 	
88.83.25[.]122 	
robot.wikaba[.]com 	
41.223.91[.]217 	
smileman.compress[.]to 	
decent.ignorelist[.]com 	
dekka.biz[.]tm 	
disol.strangled[.]net 	
eraser.2waky[.]com 	
filelord.epac[.]to 	
justsoft.epac[.]to 	
smuggler.zzux[.]com 	
sport-journal.twilightparadox[.]com 	
sportinfo.yourtrap[.]com 	
stager.ignorelist[.]com 	
tankos.wikaba[.]com 	
grandfathers.mooo[.]com 	
homeric.mooo[.]com 	
jamming.mooo[.]com 	
pneumo.mooo[.]com 	
razory.mooo[.]com 	
anger.scieron[.]com 	
gantama.mefound[.]com 	
letgetbad.epac[.]to 	
rowstate.epac[.]to 	
memento.info[.]tm 	
196.43.240[.]177 	
bughouse.yourtrap[.]com 	
news100top.diskstation[.]org 	
169.255.102[.]240 	
harm17.zzux[.]com 	
mountain8.wikaba[.]com
```

## Related articles

- [https://www.gdatasoftware.com/blog/2014/11/23937-the-uroburos-case-new-sophisticated-rat-identified](https://www.gdatasoftware.com/blog/2014/11/23937-the-uroburos-case-new-sophisticated-rat-identified)
- [https://www.gdatasoftware.com/blog/2015/01/23927-evolution-of-sophisticated-spyware-from-agent-btz-to-comrat](https://www.gdatasoftware.com/blog/2015/01/23927-evolution-of-sophisticated-spyware-from-agent-btz-to-comrat)
- [http://blog.threatexpert.com/2008/11/agentbtz-threat-that-hit-pentagon.html](http://blog.threatexpert.com/2008/11/agentbtz-threat-that-hit-pentagon.html)
- [https://securelist.com/satellite-turla-apt-command-and-control-in-the-sky/72081/](https://securelist.com/satellite-turla-apt-command-and-control-in-the-sky/72081/)
- [https://www.symantec.com/content/en/us/enterprise/media/security_response/whitepapers/waterbug-attack-group.pdf](https://www.symantec.com/content/en/us/enterprise/media/security_response/whitepapers/waterbug-attack-group.pdf)
- [https://securelist.com/the-epic-turla-operation/65545/](https://securelist.com/the-epic-turla-operation/65545/)
- [http://artemonsecurity.com/snake_whitepaper.pdf](http://artemonsecurity.com/snake_whitepaper.pdf)
- [https://www.gdatasoftware.com/blog/2015/01/23926-analysis-of-project-cobra](https://www.gdatasoftware.com/blog/2015/01/23926-analysis-of-project-cobra)
