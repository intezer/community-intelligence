# Technical Analysis: Pacha Group Competing against Rocke Group for Cryptocurrency Mining Foothold on the Cloud

> New Linux.GreedyAntd variants show Pacha Group targeting cloud environments and using blacklists to remove Rocke Group miners and cloud security agents.

- Source: https://research.intezer.com/blog/2019/05/technical-analysis-cryptocurrency-mining-war-on-the-cloud/
- Published: 2019-05-09
- Authors: Ignacio Sanmillan
- Tags: linux, cryptominer, cloud_security, threat_actor
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Pacha Group is a crypto-mining threat actor we at Intezer discovered and
profiled in a [blog post](../../../2019/02/technical-analysis-pacha-group)
published on February 28, 2019. This threat actor targeted Linux servers dating
back to September 2018 and implemented advanced evasion and persistence
techniques.

We have continued to monitor this threat actor and new findings show that Pacha
Group is also targeting cloud-based environments and conducting great efforts
to disrupt other crypto-mining groups, namely
[Rocke Group](https://unit42.paloaltonetworks.com/malware-used-by-rocke-group-evolves-to-evade-detection-by-cloud-security-products/)
who is also known to target cloud environments.

We believe that these findings are relevant within the context of bringing
awareness about cloud-native threats and our research may imply that cloud
environments are increasingly becoming a common target for adversaries.

## Technical Analysis

In monitoring Pacha Group we have identified new, undetected Linux.GreedyAntd
variants that share code with previous variants.

![](images/fig1.png)

Despite sharing nearly 30% of code with previous variants, detection rates of
the new Pacha Group variants are low:

![](images/fig2.png)

The main malware infrastructure appears to be identical to previous Pacha Group
campaigns, although there is a distinguishable effort to detect and mitigate
Rocke Group’s implants. Rocke Group was first reported by
[Cisco Talos](https://blog.talosintelligence.com/2018/08/rocke-champion-of-monero-miners.html)
researchers and has deployed sophisticated crypto-mining campaigns in Linux
servers and cloud-based environments as reported by
[Palo Alto Unit 42](https://unit42.paloaltonetworks.com/malware-used-by-rocke-group-evolves-to-evade-detection-by-cloud-security-products/).
The following image is a blacklist of miners in which Linux.GreedyAntd searches
to eradicate. We have recognized several file names in this blacklist known to
be used for Rocke Group’s implants:

![](images/fig3.png)

Furthermore, there are other strings within this file path blacklist which are
used to search for and disable cloud protection solutions, such as Alibaba
Server Guard Agent. Strings of malware
[implants](https://www.reddit.com/r/linuxquestions/comments/bknurg/is_is_exploit_or_what/)
known to have abused the
[Atlassian vulnerability](https://confluence.atlassian.com/doc/confluence-security-advisory-2019-03-20-966660264.html?_ga=2.116436864.700072651.1557244235-1322709726.1557244235)
were also found. Rocke Group is known to hunt for similar security products and
to have
[abused](https://community.atlassian.com/t5/Confluence-questions/How-come-my-confluence-installation-was-hacked-by-Kerberods/qaq-p/1054605#M141274)
the same vulnerability.

![](images/fig4.png)

Another interesting update in Pacha Group’s infrastructure in comparison to
previous campaigns is that further implants would only be able to be downloaded
from Pacha Group’s servers if the HTTP GET request was completed with a
specific User-Agent. In the following screenshot we can see how files can not
be downloaded unless the correct User-Agent is used:

![](images/fig5.png)

In addition, Pacha Group’s component update seems to include a lightweight
user-mode rootkit known as Libprocesshider, which is an open source project
hosted on [GitHub](https://github.com/gianlucaborello/libprocesshider) and has
also been used by Rocke Group.

![](images/fig6.png)

The malware updates `/etc/ld.preload` to include the path of the dropped
library masquerading libconv.so, a unicode conversion library.

![](images/fig7.png)

This shared object will export customized versions of readdir and readdir64
functions that will attempt to hide a process name from /proc filesystem of one
of the main components of the malware’s infrastructure, in charge to download
further implants in intervals along with enforcing process, file path and IP
blacklisting:

![](images/fig8.png)

Along with process and file path blacklisting measures seen in previous
variants, we also observed that newer variants implement IP blacklisting using
an interesting technique.

Right after process and file path black listing has been accomplished, we find
the following code:

![](images/fig9.png)

Each of the IPs in the blacklist IP table is decoded and then added to the
system routing table with host scope via ioctl.

This is more conveniently shown by observing the following system call trace:

![](images/fig10.png)

When we check the routing table of a compromised system we see the following:

![](images/fig11.png)

Each of the decoded IPs have been added to the routing table with host scope.
This implies that when any of these IPs will be requested, each request will be
routed back to the host to be resolved instead of redirecting them to the
gateway, causing a failure in the routing process.

In the following screenshot we can see the effect of this methodology by using
the ping utility:

![](images/fig12.png)

After analyzing the IP blacklist we discovered that some of these IPs, even
though they may not necessarily be malicious, are known to have been used by
Rocke Group in the past. As an example, systemten.org is in this blacklist and
it is known that Rocke Group has [used this](https://imgur.com/a/H7YuWuj)
domain for their crypto-mining operations. The following are some domains that
correspond to their hardcoded IPs in Linux.GreedyAntd’s blacklist that have
Rocke Group correlations:

![](images/fig13.png)

## Conclusion

We have presented evidence that Pacha Group is targeting cloud-based
environments and being especially aggressive towards Rocke Group. We have based
this conclusion on the process blacklist used by Pacha Group and the newly
added IP blacklist which contains Rocke Group correlated artifacts.

We have also provided a
[YARA rule](https://github.com/intezer/yara-rules/blob/master/GreedyAntd.yar)
in order to detect Pacha Group’s Linux.GreedyAntd implants based on reused code
among the implants.

Cloud infrastructure is quickly becoming a common target for threat actors,
particularly on vulnerable Linux servers. Unfortunately the detection rates of
Linux-based malware remain low and the security community needs more awareness
in order to more effectively mitigate these threats.

## IOCs

```
195.154.187[.]169
165.227.140[.]184
f46a9d2c3c9bfcc409534e0856f4614d6b42e792134dcf0f40df7295a777c879
d2e373c1341a28e18158272208a15decfa397640b6092b56158e0f52e4ff73a4
c098d5aeef316c3564b0b40a8a102147dae9c606fa92a2e2f0ad5c94cfe30222
42612f41befc57619646da5e91e7758dcc83cbaafbe5fdfa19d9f43a71f2504f
ce10e7a0fb517309b1e1141b44d3f9f7759e0f8889c0392774a5869f41006a3f
d94a6537adcea2f8ef3ed5ed41a548bc2b26b3acdeca9aaf6da4c933e7f47174
f83d75ab09634a7b818ef87c6509cca2c6f26f5f65b8d3448ebc86b52be62253
e5f6fbeb3981c9dfa126dc0a71a0aa41b56a09a89228659a7ea5f32aff4b2058
```

GreedyAntd Embedded IP Blacklist The following are IPs that the Pacha Group
attempts to block to prevent operation of other crypto-mining implants (notice
not to block these IPs. See the IPs to block in the above IOCs section):

```
139.99.120[.]73
47.95.85[.]22
62.210.75[.]99
113.55.8[.]24
62.210.75[.]99
42.56.76[.]104
198.204.231[.]250
47.90.213[.]21
116.62.232[.]226
134.209.104[.]20
198.12.156[.]218
207.148.76[.]229
188.165.254[.]85
58.56.187[.]66
89.35.39[.]78
37.139.22[.]136
37.44.212[.]223
54.36.137[.]146
139.99.120[.]50
37.120.131[.]220
104.20.209[.]21
198.12.156[.]218
34.196.173[.]143
34.193.88[.]221
35.168.52[.]211
104.248.4[.]162
130.61.54[.]136
139.99.120[.]50
198.12.156[.]218
166.62.38[.]167
185.193.125[.]146
132.148.148[.]79
188.165.254[.]85
104.20.208[.]21
37.187.95[.]110
158.69.25[.]62
104.31.93[.]26
104.25.140[.]10
60.191.25[.]101
104.248.53[.]213
60.191.13[.]119
104.130.210[.]206
193.56.28[.]207
37.187.95[.]110
89.35.39[.]78
81.4.122[.]134
37.44.212[.]223
148.251.133[.]246
52.41.214[.]241
52.25.124[.]181
54.68.226[.]153
136.243.89[.]164
104.20.209[.]21
176.9.2[.]144
37.59.43[.]136
78.46.89[.]102
37.59.45[.]174
91.121.2[.]76
176.9.53[.]68
37.59.55[.]60
178.63.48[.]196
37.187.154[.]79
37.59.44[.]93
78.46.91[.]134
37.59.54[.]205
23.175.0[.]142
104.140.244[.]186
136.243.102[.]157
5.254.96[.]150
51.15.56[.]161
```
