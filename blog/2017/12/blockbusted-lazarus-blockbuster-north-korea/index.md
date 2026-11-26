# BLOCKBUSTED: Lazarus, Blockbuster, and North Korea

> Code overlaps in Lazarus samples from 2014 to 2017, including shared C&C command handlers, link Blockbuster campaign malware to other North Korean attacks.

- Source: https://research.intezer.com/blog/2017/12/blockbusted-lazarus-blockbuster-north-korea/
- Published: 2017-12-12
- Authors: Jay Rosenberg
- Tags: apt, apt_dprk, dprk
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


As we have proven in previous research blog posts, malware authors often reuse
the same code. This evolution of code and code reuse is seen all throughout the
well known Blockbuster campaign and connections between other malware
attributed to the Lazarus group, a cyber threat organization attributed to
North Korea. You can read about excellent research on Lazarus and the
Blockbuster campaign by looking at reports by
[Novetta](https://www.operationblockbuster.com/wp-content/uploads/2016/02/Operation-Blockbuster-Report.pdf),
[Kaspersky](https://securelist.com/operation-blockbuster-revealed/73914/), and
[Palo Alto](https://researchcenter.paloaltonetworks.com/2017/11/unit42-operation-blockbuster-goes-mobile/).

The Lazarus group has been responsible for different campaigns and variations
of RATs, Trojans, backdoors, and malware in general. Some of the names given to
malware created by Lazarus are FALLCHILL, Destover, Hangman, Volgmer, and
Manuscrypt, among others. For someone who has not personally analyzed these
different malware families, a lot of the samples associated with each malware
could appear to be completely different, but if you examine them at the code
level, you can see that these attacks are most likely different components of
the same framework that has been pieced together or upgraded.

## Intezer Analyze™ Code Reuse

After uploading the different samples to Intezer Analyze™, the system shows
that there are code overlaps between a number of samples dating from 2014 to
2017. These are malware from different periods of time and binaries attributed
to different attacks.

![](images/fig1.png "https://analyze.intezer.com/#/analyses/7b059705-9d55-4fd9-a70b-9b4401aeecae")

![](images/fig2.png "https://analyze.intezer.com/#/analyses/d204da0f-a2a1-4f0e-9033-fe1e3b2105cc")

The analyzed samples were uncovered after uploading them to our Analyze system, generating YARA signatures from some of the unique code found in Lazarus group samples, and searching these signatures across a few malware databases. (Note: we found more samples than are being used for this report.)

## C&Cs & Command Handler

Throughout the years the Lazarus group has been active, not only have there been overlaps in the C&C infrastructure, the components of the C&C command handler have had pretty much the same functionality and code. Below is a comparison of two binaries from the Lazarus group, with shared code between the handler and functions included in the handler.

![](images/fig3.png "C&C Command Handler 1d195c40169cbdb0f50eca40ebda62321aa05a54137635c7ebb2960690eb1d82")

![](images/fig4.png "C&C Command Handler 2de5e99315a6cf42a46c8286ac4ea0bc842f6d78995833d2cab7de1cdad7dd8d")

The only visible differences are the control codes to execute certain functions; if you dive deeper, the functions themselves do not vary much. The first screenshot is of a binary with C&C commands 21687 to 21707 with a compilation date from February 5, 2014, and the second screenshot with C&C commands 32768 to 32835 is dated with the compilation date October 12, 2017. Not much has changed in three years. Interestingly enough, however, in reviewing more Blockbuster samples, the control code ranges appear to vary with the compilation date. Samples in the same range of compilation dates have the same control code sets. Also, we see the control codes remain the same regardless of whether the sample was compiled as an EXE or DLL.

![](images/fig5.png "A few of the collected samples, compilation date, C&C commands, and type being EXE or DLL")

## Infection Vectors & Targets

The Lazarus group has targeted a variety of victims — from individuals to corporations — and used multiple platforms like traditional PCs and more recently, Android devices. The group does not seem to engage in specific targeting as they look to be targeting everyone and anyone they can infect using different infection vectors over the years. Their C&Cs seem to consist of hijacked domains and servers.

Most recently, it looks like they have tried to spearfish users by sending them an infected document. The latest samples we have incorporate a malicious visual basic macro embedded in a document with a fake job position.

![](images/fig6.png "Malicious fake job position document")

![](images/fig7.png "Embedded macro")

During our research, we found a C&C located in China that is linked to a domain posted on the comment section of the threat intel news website, Israel Defense. These spam links are not just posted on Israel Defense, but all over the internet.

![](images/fig8.png "http://www.israeldefense.co.il/he/comment/112488 with link to lzruziniu[.]com")

It appears that during the time of this post, the C&C (114.215.107.218) was also hosting another binary from Lazarus. It also seems to be an automated spam comment, as there are multiple others on the same page. The links seem specific enough as if they were supposed to get someone browsing the web to click the link, which would lead to a download for infection. They may also have been using some type of web browser exploits. It is hard to tell at this point because the comments are from a couple years ago.

## Control Flow & Legacy Code

After reverse engineering one of the binaries from the Lazarus group, the rest become easily recognizable by a quick glance in IDA. From the entry point (or the ServiceMain export if the binary is a DLL), the binary begins by resolving all its imports, including those unused, by calling GetProcAddress and manually storing them in pointers.

![](images/fig9.png "2014 DLL vs 2017 EXE")

The string obfuscation code has changed throughout the binaries according to the time they were compiled as well. This is one of the updates the group did make to the code, but the most significant evidence here is that from the oldest binary to the newest, they use the GetProcAddress function with “GetProcAddress” as the parameter. This was obviously a coding error that has remained in all the binaries analyzed.

![](images/fig10.png)

The next part of the execution flow of the binary varies, but consists mainly of the same code with different options used. There may be features like registering persistence, checking for sandboxes or virtual machines, and more. All of these options seem to be interchangeably used, and a binary could be created as simply as filling out a few textboxes and checkboxes for all the different features, establishing the binary with a “generate RAT” button.

After the various components of optional code are executed, the malware creates a thread to bind and ‘listen’ to a specified port, waiting for a command from the C&C as discussed in the previous section of this post.

## Conclusion

Evidently, the Lazarus group is still active. Because the compilation dates seem to be an accurate measure of a timeline, that puts the latest sample at two months ago from the date of this report. Consistent code reuse throughout the samples shows that well-funded nation states are reusing code time and time again. As stated in the beginning, if you examine them at the code level, you can see that these attacks are most likely different components of the same framework that has been pieced together or upgraded.

## IoCs

### Files

```
a4a2e47161bbf5f6c1d5b1b3fba26a19dbfcdcf4eb575b56bde05c674089ae95
ee3ecf100fc2042cfadeb0509ae4f49647daa1afcee2bd3098912247e155a1e7
16eaa0298c66e0de40cc42568879fab9513f4d533c0fa1f5366346b4d70df50d
dbae68e4cab678f2678da7c48d579868e35100f3596bf3fa792ee000c952c0ed
9f177a6fb4ea5af876ef8a0bf954e37544917d9aaba04680a29303f24ca5c72c
80b5cc9feb10fac41ee2958ab0f751bf807126e34dcb5435d2869ef1cf7abc41
8edb59694ff239c90f33cdbb17bc67ebbd08ec77df5000c6616d7844040796b4
db1f4abd2a8d3b17c14e8f31cb1da3a273cf51ed194ad0e4520346ed95708dad
7429a6b6e8518a1ec1d1c37a8786359885f2fd4abde560adaef331ca9deaeefd
4e8c10a7fa51a3ab089b284e86a7daaca779ed82ba1750607fc3bfa91681f9b1
e79bbb45421320be05211a94ed507430cc9f6cf80d607d61a317af255733fcf2
4481e31d42499d084317b79a3a6250e50302aa00603dacdd2dff3a8e3b911404
9607ab45451bb6c52eeec8cf0669d1dd37c917baecef2d63564c7cec110932c0
a606716355035d4a1ea0b15f3bee30aad41a2c32df28c2d468eafd18361d60d6
0b4959764331ddbe7da71e6f8515d0be8f494b4107c2718bda9e0a85d8247488
9fa326adbd71e58aeb7ea404b6b8d6be5742c547c525286b2e95444ab099ef9e
8ba791b9611d5d6dfd40e08e43ad851675faea24c2f5bc4f541e475871999ad3
2de5e99315a6cf42a46c8286ac4ea0bc842f6d78995833d2cab7de1cdad7dd8d
4a84452752cf8e493ae820871096044edd9f6453366842927148e7d8e218dc87
76e0eec565f4e50b57d74fb1a09ea9e356da0709cba3d5dab9b3702ba19d662c
f09fb9a79bab6a927297e536594027016da7d7ab13e124c76a05889ec107ad02
6dcd635875625426298a1d7b4ab346ee318b3afa2e6440677ab935e473bf782d
959eb014a2d8ca8158d1f6a198205d331e0ebfb3ced5fd9004b8dcab4547987f
3d481d166f27b48f103db39fda3845dfc962addd44e92e3187e0ddd1ee01c741
1d195c40169cbdb0f50eca40ebda62321aa05a54137635c7ebb2960690eb1d82
de13155f4a4cb1af04539883545151905ad4496ea6ef34cb22e50831f7caa38a
```

### Network

```
41.131.29.59
176.35.250.93
64.86.34.24
58.6.21.11
114.215.107.218
118.140.97.6
190.216.219.247
201.26.209.137
14.161.14.196
220.132.191.110
92.42.54.184
182.180.143.39
193.251.27.90
```
