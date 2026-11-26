# Silence of the Moles

> The loader of Silence, malware aimed at financial institutions, shares unique code with the Mole ransomware loader, a possible sign of the same threat actor.

- Source: https://research.intezer.com/blog/2017/11/silenceofthemoles/
- Published: 2017-11-01
- Authors: Jay Rosenberg
- Tags: backdoor
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Kaspersky Labs published a technical analysis of a new malware,
[Silence](https://securelist.com/the-silence/83009/) that is aimed at attacking
financial institutions. After uploading the loader of this malware to Intezer
Analyze™, we have found a possible connection through code reuse to the loader
of another campaign of malware,
[Mole](https://researchcenter.paloaltonetworks.com/2017/04/unit42-mole-ransomware-one-malicious-spam-campaign-quickly-increased-complexity-changed-tactics/)
previously discovered by Unit 42 of Palo Alto Networks.

This connection might be an indicator that these two attacks are originated
from the same threat actor, but currently it is too early to tell.

![](images/fig1.png)

(Intezer Analyze™ public report available
[here](https://analyze.intezer.com/#/analyses/f704b93d-0a31-4196-b8ab-2af3d50bf42c).)

Silence Loader:
`f24b160e9e9d02b8e31524b8a0b30e7cdc66dd085e24e4c58240e4c4b6ec0ac2` Mole Loader:
`50117ce3fe5dba572cf23584dc7541a7cfd4026d4316e69d29cdf536873fdf20`

If we look at the code of the two loaders used by both campaigns side by side,
we can see that the code is very similar and according to our system is unique
to these families of malware.

![](images/fig2.png "sub_4079A0 vs sub_4023A0")

Through the disassembly in the photo above, we can also see there is a string
initialized through an array, “RtpEncodePointer,” that is later used for a call
to GetProcAddress. This looks like a typo and the author of the code meant to
write “RtlEncodePointer” because “RtpEncodePointer” does not exist in
ntdll.dll. The evidence suggests that this code was being reused. There are no
references to “RtpEncodePointer” available publicly online besides automated
reports of a couple unclassified malware from
[Hybrid Analysis](https://www.hybrid-analysis.com/sample/1a8fc5c94009fd570a20397a05aaf53fc3e16c9d14d59a824ec16f663950219e?environmentId=100).

In addition to the links within the code, there are several other similarities
we have witnessed between the Mole and Silence malware, such as the attack
vectors (spear phishing, packaging of the malware) and motives — which can be
extra evidence for this connection.
