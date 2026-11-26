# Evidence Aurora Operation Still Active Part 2: More Ties Uncovered Between CCleaner Hack & Chinese Hackers

> Code shared by the CCleaner stage 2 payload and APT17 samples strengthens the link to the Axiom group, in an attack we assess was most likely state-sponsored.

- Source: https://research.intezer.com/blog/2017/10/evidence-aurora-operation-still-active-part-2-more-ties-uncovered-between-ccleaner-hack-chinese-hackers/
- Published: 2017-10-02
- Authors: Jay Rosenberg
- Tags: supply_chain_attack, apt, apt_china, backdoor, windows
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Since my
[last post](../../../2017/09/evidence-aurora-operation-still-active-supply-chain-attack-through-ccleaner/),
we have found new evidence in the next stage payloads of the CCleaner supply
chain attack that provide a stronger link between this attack and the Axiom
group.

First of all, our researchers would like to thank the entire team at Cisco
Talos for their excellent work on this attack (their post regarding stage 2 can
be found
[here](http://blog.talosintelligence.com/2017/09/ccleaner-c2-concern.html)) as
well as their cooperation by allowing us access to the stage 2 payload. Also,
we would like to give a special thanks to Kaspersky Labs for their
collaboration.

## The Next Connection

Starting from the stage 2 payload, I reverse engineered the module, extracting
other hidden shellcode and binaries within. After uploading the different
binaries to Intezer Analyze™, the final payload (that I have access to) had a
match with a binary relating to the Axiom group.

![](images/fig1.png)

At first glance, I believed it was going to be the same custom base64 function
as mentioned in my
[previous blog post](../../../2017/09/evidence-aurora-operation-still-active-supply-chain-attack-through-ccleaner/).
A deeper look in the shared code proved otherwise.

Binary in screenshot:

- `f0d1f88c59a005312faad902528d60acbf9cd5a7b36093db8ca811f763e1292a`

Related APT17 samples:

- `07f93e49c7015b68e2542fc591ad2b4a1bc01349f79d48db67c53938ad4b525d`
- `0375b4216334c85a4b29441a3d37e61d7797c2e1cb94b14cf6292449fb25c7b2`
- `20cd49fd0f244944a8f5ba1d7656af3026e67d170133c1b3546c8b2de38d4f27`
- `ee362a8161bd442073775363bf5fa1305abac2ce39b903d63df0d7121ba60550`

![](images/fig2.jpg)

Not only did the first payload have shared code between the Axiom group and
CCBkdr, but the second did as well. The above photo shows the same function
between two binaries. Let me put this into better context for you: out of all
the billions and billions of pieces of code (both trusted and malicious)
contained in the Intezer Code Genome Database, we found this code in only these
APTs. It is also worth noting that this isn’t a standard method one would use
to call an API. The attacker used the simple technique of employing an array to
hide a string from being in clear sight of those analyzing the binary (although
to those who are more experienced, it is obvious) and remain undetected from
antivirus signatures. The author probably copied and pasted the code, which is
what often happens to avoid duplicative efforts: rewriting the same code for
the same functionality twice.

Due to the uniqueness of the shared code, we strongly concluded that the code
was written by the same attacker.

## Technical Analysis

The stage two payload that was analyzed in this report (dc9b5e8aa6ec86db8af0a7aa897ca61db3e5f3d2e0942e319074db1aaccfdc83), after launching the infected version of CCleaner, was dropped to only a selective group of targets, as reported by Talos. Although there is an x64 version, the following analysis will only include the x86 version because they are nearly identical. I will not be going too far in depth as full comprehension of the technical analysis will require an understanding of reverse engineering.

Instead of using the typical API (VirtualAlloc) to allocate memory, the attackers allocated memory on the heap using LocalAlloc, and then copied a compressed payload to the allocated memory.

![](images/fig3.png)

It looks like the attackers used version 1.1.4 of zlib to decompress the payload into this allocated memory region.

![](images/fig4.png)

Depending on if you’re running x86 or x64 Windows, it will drop a different module. (32-bit `07fb252d2e853a9b1b32f30ede411f2efbb9f01e4a7782db5eacf3f55cf34902`, 64-bit `128aca58be325174f0220bd7ca6030e4e206b4378796e82da460055733bb6f4f`) Both modules are actually legitimate software with additional code and a modified execution flow.

![](images/fig5.png)

![](images/fig6.png)

The last modified time on the modules is changed to match that of the msvcrt.dll that is located in your system32 folder–a technique to stay under the radar by not being able to check last modified files.

![](images/fig7.png)

Some shellcode and another module are written to the registry.

![](images/fig8.png)

After the module is successfully dropped, a service is created under the name Spooler or SessionEnv, depending upon your environment, which then loads the newly dropped module.

![](images/fig9.png)
The new module being run by the service allocates memory, reads the registry where the other payload is located, and then copies it to memory
![](images/fig10.png)
![](images/fig11.png)

The next payload is executed, which decrypts another module and loads it. If we look at the memory of the next decrypted payload, we can see something that looks like a PE header without the MZ signature. From here, it is as simple as modifying the first two bytes to represent MZ and we have a valid PE file. (`f0d1f88c59a005312faad902528d60acbf9cd5a7b36093db8ca811f763e1292a`)

![](images/fig12.png)

The next module is a essentially another backdoor that connects to a few domains; before revealing the true IP, it will connect to for the next stage payload.

![](images/fig13.png)

It starts by ensuring it receives the correct response from https://www.microsoft.com and https://update.microsoft.com.

![](images/fig14.png)

The malware proceeds to decrypt two more URLs.

![](images/fig15.png)
![](images/fig16.png)

The malware authors used steganography to store the IP address in a ptoken field of the HTML.

Here you can see the GitHub page with the ptoken field.

![](images/fig17.png)

The value is then XOR decrypted by `0x31415926` which gives you `0x5A093B0D` or the IP address: `13.59.9.90`

![](images/fig18.png)

## Conclusion

The complexity and quality of this particular attack has led our team to conclude that it was most likely state-sponsored. Considering this new evidence, the malware can be attributed to the Axiom group due to both the nature of the attack itself and the specific code reuse throughout that our technology was able to uncover.

## IOCs

```
Stage 2 Payload: dc9b5e8aa6ec86db8af0a7aa897ca61db3e5f3d2e0942e319074db1aaccfdc83
x86 Trojanized Binary: 07fb252d2e853a9b1b32f30ede411f2efbb9f01e4a7782db5eacf3f55cf34902
x86 Registry Payload: f0d1f88c59a005312faad902528d60acbf9cd5a7b36093db8ca811f763e1292a
x64 Trojanized Binary: 128aca58be325174f0220bd7ca6030e4e206b4378796e82da460055733bb6f4f
x64 Registry Payload: 75eaa1889dbc93f11544cf3e40e3b9342b81b1678af5d83026496ee6a1b2ef79

Registry Keys:
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Wbem\Perf\�01
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Wbem\Perf\�02
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Wbem\Perf\�03
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Wbem\Perf\�04
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Wbem\Perf\HBP
```
