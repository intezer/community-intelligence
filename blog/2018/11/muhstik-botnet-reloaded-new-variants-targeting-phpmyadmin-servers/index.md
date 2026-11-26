# Muhstik Botnet Reloaded: New Variants Targeting phpMyAdmin Servers

> New Muhstik botnet variants add a scanner module targeting phpMyAdmin web servers, with code likely borrowed and modified from the open-source Pnscan tool.

- Source: https://research.intezer.com/blog/2018/11/muhstik-botnet-reloaded-new-variants-targeting-phpmyadmin-servers/
- Published: 2018-11-12
- Authors: Ignacio Sanmillan
- Tags: linux, botnet
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


*The Muhstik botnet was first exposed by
[Netlab360](https://blog.netlab.360.com/gpon-exploit-in-the-wild-i-muhstik-botnet-among-others-en/)
researchers in May 2018. This botnet targeted mainly GPON routers. At Intezer
we found that Muhstik is extending its spectrum of compromised devices by
targeting web servers hosting phpMyAdmin.*

[PhpMyAdmin](https://www.phpmyadmin.net/) is a well known open-source tool
written in PHP, intended to handle the administration of MySQL over the web.
This tool is fairly popular among web developers due to the convenience of
operating a database via a web browser. On the other hand, the use of these
type of tools leads the front-end of a server to be publicly exposed, and in
particular to phpMyAdmin, with a lack of protection against brute-forcing
attempts.

This blog will cover how the Muhstik malware has been using phpMyAdmin scanners
in its newer variants in order to extend the spectrum of its vulnerable devices
discovery and acquisition.

Another recent IoT malware campaign that targeted phpMyAdmin servers was
[Operation Prowli](https://www.guardicore.com/2018/06/operation-prowli-traffic-manipulation-cryptocurrency-mining/),
however, it is not very common to see these type of botnets targeting web
servers.

We will also disclose the relationship between Muhstik’s phpMyAdmin Scanner and
an open-source TCP port scanner tool called Pnscan, since we believe that the
threat actors behind Muhstik borrowed and modified code from this project.

## Hunting New Mushtik Variants

After the analysis of various known *Muhstik* samples, we observed that some
lower detected samples exposed what seem to be new features added to the
botnet. One of these examples is the following MIPS sample:

`d0d8f3c3e530a75768784c44772b2a71c5b3462d68a3217fee1f6767686cea4e`

The following is the known Muhstik’s implant download method from Netlab360
report:

![](images/fig1.png "Muhstik GPON exploit deployment")

When analyzing the strings we notice that there are some uncommon *Muhstik*
strings along with the known Muhstik payloads.

Known and unknown strings from the known *Muhstik* variant:

![](images/fig2.png "String reuse from Intezer Analyze™")

After analyzing the code, we concluded that the new functionality was a
phpMyAdmin scanner module. Furthermore, these new findings motivated us to
index some lower detected variants in our database in order to evaluate if the
discovery of further lower detected variants was possible.

Since we support ELF files in Intezer Analyze™ but do not currently support
MIPS binaries, we decided to rely on our 'String Reuse' feature in order to
find lower detected variants that shared large pieces of the phpMyAdmin Scanner
code base based on string reuse.

![](images/fig3.png)

## Muhstik.PMA.Scan Variant Analysis

This variant shares the same infrastructure as earlier Mushstik variants,
possessing scan, report, download, and control stages.

![](images/fig4.png "Muhstick’s phpMyAdmin Scanner Infrastructure")

### Scanner Stage

Sample: `d0d8f3c3e530a75768784c44772b2a71c5b3462d68a3217fee1f6767686cea4e`.

The scanner is a command line tool utility in which a file containing a list of
*CIDRs* to bruteforce is expected as the first argument.

In every analyzed sample we noticed a recurrent pattern of hardcoding the
report and download servers in the main function:

![](images/fig5.png "Hardcoded servers in phpMyAdmin Scanner")

The scanner then attempts to obtain the CIDR list it requires to commence the
scan. For this specific sample it obtains such lists from the download server,
but we are aware of samples that contain the hardcoded IP blocks in the binary
itself:

![](images/fig6.png "Retrieving CIDRs from download server")

The download server (`217.13.228.176`) hosts implants to be downloaded by the
compromised server and CIDRs to be used by the phpMyAdmin Scanner. In this
sample, the download server host these files at `/rescueshop_dev/x/`:

![](images/fig7.png "Download Server Open Directory")

We observe the names of the CIDRs correlate with the names that Muhstik’s
aioscan implant used based on a Netlab360 report.

![](images/fig8.png "Older variants of Muhstik using CIDRs")

The download server also contains a series of files which are instances of
*Muhstik* *phpmyAdmin* Scanner. These files were hosted in the root directory
of the download server under the name `pma<number>`.

In addition, once the scanner obtains the IP block to target, it will begin
brute forcing credentials in order to find vulnerable servers. The following
image demonstrates one of the several ways that the scanner crafts http
requests when attempting to brute force login credentials of a discovered
*phpMyAdmin* login page:

![](images/fig9.png "Muhstick’s phpMyAdmin HTTP headers")

Once the scanner has found a vulnerable server, it will then disclose it to the
report server.

![](images/fig10.png "Vulnerable phpMyAdmin URL crafting for report")

![](images/fig11.png "Function in charge to report Vulnerable phpMyAdmin URL")

### Report and Download Stages

After the scanner has found a vulnerable *phpMyAdmin* server, it will send the
full vulnerable url to the report server (`128.199.251.119`) in the following
format:

`http:://<reportserver>/pma.php?ip=<vulnerable url>`.

The report server IP can also be seen in the Netlab360 report used to deploy
Muhstik’s GPON exploit:

![](images/fig12.png "Muhstik’s GPON exploit deployment")

Throughout the investigation we were unable to find the exact operations of
pma.php hosted in the report server, although we assess with high confidence
that it was used to deploy a phpMyAdmin exploit since it shares the same
convention as previously observed Muhstik exploit deployments.

Furthermore, we were able to find a core dump in
[VirusTotal](https://www.virustotal.com/#/file/1dd04d7a09b0ae49cfd552fd7ae93c3f16ae78e629b7610d8dec96ec8325af48)
of this same variant that exposed some of pma.php’s behavior.

![](images/fig13.png "Muhstik’s phpMyAdmin Scanner Core-dump VirusTotal Analysis")

When analyzing the stack segment in the core file we found the following:

![](images/fig14.png "Core-Dump Stack Overview")

We observed that paths of the downloaded implant and *Muhstik phpMyAdmin*
Scanner instance were saved in suspicious environment variables. We also notice
based on the `$PWD` environment variable that this core file proceeded from a
`WordPress` server.

The binary was executed from `/tmp/` directory and its name was pma6, which
follows the same naming convention from the Muhstik phpMyAdmin Scanner
instances hosted in its download server.

Additionally, the environment variable `$SHLVL` has a value of 3. The value of
this environment variable increases by one each time a shell has started. In
other words, for the time that the core dump was generated, the binary was
running on 3 nested shells. These are solid indicators that this core file was
generated in a compromised system.

Summarizing, this core dump leaves hints on how the attack chain was deployed.
It seems that the implants may have been downloaded by an additional agent in
the exploitation stage, and that the scanner was one piece of a broader
post-exploitation tool set.

### Control Stage

If we submit the implants to Intezer Analyze™ we observe that they are packed
with a specific UPX variant.

![](images/fig15.png "Packed Muhstik sample")

In the related samples section we observe that the same UPX variant has been
seen in previous Kaiten variants reported by MalwareMustDie.

![](images/fig16.png) ![](images/fig17.png) ![](images/fig18.png)

After unpacking the sample we see that the unpacked *Muhstik* implant shares
genes with Tsunami based on code reuse. Tsunami and STDBot are known to be
derived from an IRC bot malware family called Kaiten. Therefore, this
connection shows that *Muhstik* implants are also based on Kaiten.

![](images/fig19.png)

This implant operates with different *IRC* servers, implementing *SSH*, *HTTP*
scanning functionalities, and multiple *DDoS* features which extends the usual
functionalities of regular Kaiten variants.

![](images/fig20.png)

Based on Netlab360’s report we can assume that this botnet has been active
since May 2018. However, when we analyze the username the botnet uses at some
point in one of its IRC channels, we notice the date as a suffix. We believe
this naming convention is used to track new variants among the botnet’s IRC
channels. This sample’s fixed IRC username suggests the build was released one
day after Netlab360 published its blog about *Muhstik*.

![](images/fig21.png "Origins of Muhstik phpMyAdmin Scanner")

## Origins of Muhstik phpMyAdmin Scanner

After researching this newer Muhstik variant, we noticed several other pieces
of malware not related to Muhstik, but using a similar scanner.

Based on code similarities, we found that the original scanner in which Muhstik
borrows code from is an open-source project called Pnscan. Pnscan was written
by Peter Eriksson, a computer systems manager from Linköping University,
Sweden.

![](images/fig22.png "Pnscan’s Github Repository")

It is important to understand that this Pnscan tool is not directly related to
the Linux.Pnscan.2 worm reported by
[DrWeb](https://news.drweb.com/show/?i=9548&lng=en&c=5) in August, 2015. Pnscan
is an open-source TCP port scanner tool which received its name as an
abbreviation of Parallel Network Scanner.

When we compare some of the functions found in the un-stripped Muhstik
phpMyAdmin Scanner against Pnscan, we observe undeniable evidence that they
share a common code base. We can see this based on function, string, and symbol
naming similarities.

![](images/fig23.png "Portion of Pnscan’s Main Function")
![](images/fig24.png "Portion of Muhstick’s phpMyAdmin Main’s Function")
![](images/fig25.png "Pnscan’s get_ip_range")
![](images/fig26.png "Muhstick’s phpMyAdmin Scanner get_ip_range Function")
![](images/fig27.png "Pnscan’s get_port_range Function")
![](images/fig28.png "Muhstick’s phpMyAdmin get_port_range")

## Conclusion

PhpMyAdmin scanners seem to be an increased resource leveraged by threat actors
to compromise systems in recent years. In particular, the Muhstik botnet has
adopted new techniques to compromise a broader spectrum of systems.

The phpMyAdmin scanner variant appears to have fallen under the radar in
comparison with previous variants targeting GPON routers and has not been seen
or reported to date.

In order to mitigate these types of attacks against phpMyAdmin, stronger
password conventions should be implemented, or some form of anti brute-forcing
schemes such as
[account lockout policy](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/account-lockout-policy),
or Google’s [reCaptcha](https://www.google.com/recaptcha/intro/v3.html), should
be applied.

Furthermore, we have observed in the past the repercussions of publishing
open-source projects that can be used for malicious purposes. This is another
example demonstrating how threat actors borrow and modify publicly released
code in order to conduct their campaigns.

## IOCs

Muhstik phpMyAdmin Scanner:

```
c356bf0fd5140f428c2f7c39b96095584c4b99fa6f077f0c1cf9d1ecd9d26e88
7109ba2b506fbcc2a99dfdee14bc1dbb0a8b5dbe14f530b12ffcfc3fa04f90b7
e6ee7da0d9e5d38b44be7f2c2c038708acc12819cb5d6635b8e31715de6026b7
61af6bb7be25465e7d469953763be5671f33c197d4b005e4a78227da11ae91e9
d0d8f3c3e530a75768784c44772b2a71c5b3462d68a3217fee1f6767686cea4e
e0da49d8def275d7e35b2ad4559330301debc9f10cc9bdde953748c18075994e
68002efcc5e13bf6a8c6738cdb324eb7d92bcc54080e3fa6beff2612e4f7252f
679fd3ff024750ef4753f6f52bc99c3d7bb9793e5d32742751099edbed558623
```

Muhstik WordPress Scanner

```
fbfc7b127ab9a03bb6b3550e6678e8224ab3d18d1e96ea473ec50eb271fcf05b
```

Muhstik Implants:

```
7296f5b14f5c55e1f359bb752e18323ba2819f4a43066d50cf4e0294c9d33766
9ae309db0fe53092e67bea17d37a6137bcca70e9c4c31491f15e493ebca3d1c7
fc367a6247e8dca792b54e49dc997e3bc48d28161d7d987043c12c9408933c61
094cd380b411951a2fe00c2d38208838463b83160199f4495062391ced106946
6d0a52bc9b3c6cdef438cbf22c9968e3c06eff6037ffc5ddc645f0f158513ef2
9519e0cd8ec702af86367c1245afea85fd98cc1160cf46e2874a60dfd0952ed8
82bf0be6debed7eb94d4d64e806e0c9f2458d58936443b12c8bd6ae805106e68
3756a7a180b4573efcb88bcba663c66d9474f19e7f646840469f2f60cad236d4
8426f4623aab0af7aaab2d5919dc86ffd9f167f2e423ca3838d7fe24591458f1
96867f503d65c564b146e8961dffae1f90962ba171dd0a5f856ed3f648cb7f4c
767cc42e1b6cc082bd41eecdb2743173d69ac5e8e02f5b63fa104e3c19d90345
```

Mushtik implant unpacked

```
b56f666944b3f6bc459546d34002e607c0dc51b1b8bc14999ff4a1e6665a1c9a
```

Mushtik GPON phpMyAdmin MIPS hybrid

```
Dd6c74d2942fe9e1cee82e39e7de6986589ae923530864d61b58e21318bf4f7b
```

Pnscan source code: https://github.com/ptrrkssn/pnscan
