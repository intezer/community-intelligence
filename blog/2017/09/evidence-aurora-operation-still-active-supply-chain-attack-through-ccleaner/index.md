# Evidence Aurora Operation Still Active: Supply Chain Attack Through CCleaner

> A unique base64 function shared by the CCleaner 5.33 backdoor and earlier APT17 samples suggests the Operation Aurora group was still active in 2017.

- Source: https://research.intezer.com/blog/2017/09/evidence-aurora-operation-still-active-supply-chain-attack-through-ccleaner/
- Published: 2017-09-20
- Authors: Jay Rosenberg
- Tags: supply_chain_attack, apt, apt_china, backdoor, windows
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Recently, there have been a few attacks with a supply chain infection, such as
[Shadowpad](https://securelist.com/shadowpad-in-corporate-networks/81432/)
being implanted in many of Netsarang’s products, affecting millions of people.
You may have the most up to date cyber security software, but when the software
you are trusting to keep you protected gets infected there is a problem. A
backdoor, inserted into legitimate code by a third party with malicious intent,
leads to millions of people being hacked and their information stolen.

Avast’s CCleaner software had a backdoor encoded into it by someone who had
access to the supply chain. Through somewhere that had access to the source
code of CCleaner, the main executable in v5.33.6162 had been modified to
include a backdoor. The official statement from Avast can be found
[here](https://blog.avast.com/update-to-the-ccleaner-5.33.6162-security-incident)

## The Big Connection

Costin Raiu, director of Global Research and Analysis Team at Kaspersky Lab,
was the first to find a code connection between APT17 and the backdoor in the
infected CCleaner:

> The malware injected into #CCleaner has shared code with several tools used
> by one of the APT groups from the #Axiom APT 'umbrella'.
>
> — Costin Raiu (@craiu)
> [September 19, 2017](https://x.com/craiu/status/910059453948579840?ref_src=twsrc%5Etfw)

Using Intezer Analyze™, we were able to verify the shared code between the
backdoor implanted in CCleaner and earlier APT17 samples. The photo below is
the result of uploading the CCBkdr module to Intezer Analyze™, where the
results show there is an overlap in code. With our technology, we can compare
code to a huge database of malicious and trusted software — that’s how we can
prove that this code has never been seen before in any other software.

![](images/fig1.png)

A deeper analysis leads us to the functions shown below. The code in question
is a unique implementation of base64 only previously seen in APT17 and not in
any public repository, which makes a strong case about attribution to the same
threat actor.

![](images/fig2.jpg)

This code connection is huge news. APT17, also known as
[Operation Aurora](../../../2017/10/evidence-aurora-operation-still-active-part-2-more-ties-uncovered-between-ccleaner-hack-chinese-hackers/),
is one of the most sophisticated cyber attacks ever conducted and they
specialize in supply chain attacks. In this case, they probably were able to
hack CCleaner’s build server in order to plant this malware. Operation Aurora
started in 2009 and to see the same threat actor still active in 2017 could
possibly mean there are many other supply chain attacks by the same group that
we are not aware of. The previous attacks are attributed to a Chinese group
called PLA Unit 61398.

## Technical Analysis

The infected CCleaner file that begins the analysis is from
`6f7840c77f99049d788155c1351e1560b62b8ad18ad0e9adda8218b9f432f0a9`

A technical analysis was posted by Talos
[here](http://blog.talosintelligence.com/2017/09/avast-distributes-malware.html).

The flow-graph of the malicious CCleaner is as follows (taken from the Talos
report):

![](images/fig3.png)

Infected function:

![](images/fig4.png "Infected function")

Load and execute the payload code:

![](images/fig5.png "Load and execute the payload code")

After the embedded code is decrypted and executed, the next step is a PE
(portable executable) file loader. A PE file loader basically emulates the
process of what happens when you load an executable file on Windows. Data is
read from the PE header, from a module created by the malware author.

The PE loader first begins by resolving the addresses of imports commonly used
by loaders and calling them. GetProcAddress to get the addresses of external
necessary functions, LoadLibraryA to load necessary modules into memory and get
the address of the location of the module in memory, VirtualAlloc to create
memory for somewhere to copy the memory, and in some cases, when not
implemented, and memcpy to copy the buffer to the newly allocated memory
region.

![](images/fig6.png "The PE loader")

After the module is copied to memory, to load it properly, the proper loading
procedure is executed. The relocation table is read to adjust the module to the
base address of the allocated memory region, the import table is read, the
necessary libraries are loaded, and the import address table is filled with the
correct addresses of the imports. Next, the entire PE header is overwritten
with 0’s, a mechanism to destroy the PE header tricking security software into
not realizing this module is malicious, and after the malicious code begins
execution.

The main module does the following:

1. Tries an anti-debug technique using time and IcmpSendEcho to wait
2. Collect data about the computer (Operating system, computer name, DNS
   domain, running processes, etc)
3. Allocates memory for payload to retrieve from C&C server
4. Contacts C&C server at IP address `216.126.225.148`. If this IP address is
   unreachable, uses a domain generation algorithm and uses a different domain
   depending on the month and year
5. Executes code sent by C&C

By the time of the analysis, we were unable to get our hands on the code sent
by the C&Cs.

If you would like to analyze the malware yourself, you may refer to my tweet.

> #ccleaner malware DLL w/ IAT fix https://t.co/FprmtmkV64
> https://t.co/dgWiQVd31k @TalosSecurity @malwrhunterteam
> pic.twitter.com/TxsbveFoHJ
>
> — Jay Rosenberg (@jaytezer) September 18, 2017
