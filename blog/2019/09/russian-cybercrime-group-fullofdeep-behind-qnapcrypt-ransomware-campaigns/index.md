# Russian Cybercrime Group FullofDeep Behind QNAPCrypt Ransomware Campaigns

> A new QNAPCrypt sample shares code and encryption with the 15 campaigns we halted in July. Its ransom note and strings point to Russian group FullofDeep.

- Source: https://research.intezer.com/blog/2019/09/russian-cybercrime-group-fullofdeep-behind-qnapcrypt-ransomware-campaigns/
- Published: 2019-09-20
- Tags: linux, cryptomining
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

We previously reported on how we managed to
[temporarily shut down](../../../2019/07/seizing-15-active-ransomware-campaigns-targeting-linux-file-storage-servers/)
15 operative **QNAPCrypt ransomware campaigns** targeting Linux-based file
storage systems (NAS servers). We have now identified a new QNAPCrypt sample
which is being used by the same threat actor group. The authors behind this new
ransomware instance have revealed enough evidence for us to conclude the
establishment of **FullofDeep**, a Russian cybercrime group operating from the
Union State and the Ukraine. The group is mainly focused on ransomware
campaigns.

## Technical Analysis

Initially, we received a hit from an older YARA signature, which was created to
hunt for
[QNAPCrypt](../../../2021/03/when-viruses-mutate-did-suncrypt-ransomware-evolve-from-qnapcrypt/)
instances. Although the new ransomware variant does not share large portions of
code with the original QNAPCrypt campaigns, the naming convention used in the
new ransomware's functions has been observed in previous QNAPCrypt instances:

![](images/fig1.png)

In addition, the new variant utilizes geolocation information in order to
determine whether or not the malware will operate:

![](images/fig2.png)

The threat actors behind the new ransomware have filtered Belarusia(BY),
Russia(RU), and Ukraine(UA), in order to impede operation if the ransomware is
executed from one of these countries. This is similar to the QNAPCrypt implant
[Anomali previously reported on](https://www.anomali.com/blog/the-ech0raix-ransomware):

![](images/fig3.png)

The overall functionality of the ransomware remains similar to QNAPCrypt. The
main configuration of the ransomware is static and hard-coded within the
binary, much like the latest version of QNAPCrypt.

![](images/fig4.png)

As is similar to the original QNAPCrypt samples, the algorithm the attackers
chose to encrypt the filesystem with is AES CFB. One noticeable difference is
the main ransom note doesn't advertise a specific bitcoin wallet like it did in
the original QNAPCrypt samples. Instead, the attackers demand to be contacted
via a protonmail email account:

![](images/fig5.png)

The email address provided is "fullofdeep", and the ransom note annotates
'CONTACT US'. These strings draw us to the conclusion that this is a Russian
cybercrime group referred to as FullOfDeep, and that the group is deploying
different ransomware campaigns. Based on the samples' overlaps, including both
ransomware were written in Go, and in addition to similarities in the implant
function naming convention and the geolocation filtering applied, we can assess
with high confidence that this new ransomware sample is operated by the same
authors behind the original QNAPCrypt campaigns.

## Summary

In July, we identified and temporarily seized 15 active QNAPCrypt ransomware
campaigns targeting Linux-based file storage systems (NAS servers). This
discovery was significant in that it's rare to see ransomware being used to
target the Linux-operating system.

While we prevented the malware from infecting additional victims, we have since
hunted for and detected a new ransomware that shares many similarities with the
previous QNAPCrypt samples. We can assess with high confidence that FullofDeep,
a new Russian cybercrime group specializing in ransomware operations, is behind
both the original QNAPCrypt campaigns and the newly identified ransomware
instance.

In addition, both the original and new QNAPCrypt samples are indexed in Intezer
Analyze's code genome database. If you have a suspicious file that you suspect
to be QNAPCrypt, you can upload it to the free
[Intezer Analyze community edition](https://analyze.intezer.com/#/) in order to
view the verdict, code and string reuse, and more.

Please refer to the **mitigation recommendations** section in our blog post
titled,
[Why we Should be Paying More Attention to Linux Threats](https://intezer.com/blog-why-we-should-be-paying-more-attention-to-linux-threats/),
for best practices Linux users can adopt in order to mitigate the threats posed
by QNAPCrypt and other Linux-based threats.

## IOCs

```
50470f94e7d65b50bf00d7416a9634d9e4141c5109a78f5769e4204906ab5f0b
fullofdeep@protonmail.com
```
