# Radare Plugin is Here for Intezer Community

> The r2analyze community plugin brings Intezer code genomics into Radare2, so reverse engineers can attribute a sample to a malware family or threat actor.

- Source: https://research.intezer.com/blog/2022/02/radare-plugin-automate-reverse-engineering/
- Published: 2022-02-08
- Authors: Joakim Kennedy
- Tags: tool
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


When you reverse engineer code as part of an incident response team, you want
to quickly get information about what kind of threat you're dealing with.

A while back we released Intezer Analyze plugins both for
[IDA Pro](https://intezer.com/blog/malware-analysis/ida-pro-plugin-now-available-to-the-community/)
and
[Ghidra](https://intezer.com/blog/malware-analysis/community-ghidra-plugin-is-here/)
to help you zero in on a file's malicious and unique code. Now it is Radare's
turn. [Radare2](https://rada.re/n/radare2.html) (r2) is an open-source tool
chain for reverse engineering and forensics. With the release of the community
plugin `r2analyze`, r2 users can now supercharge their reversing session with
code genomics from Intezer to attribute the malware family or threat actor.

## The Radare Plugin for Reverse Engineering

How to get started:

1. Make sure you have an Intezer Analyze community account, or a paid team
   account. (If not, register [here](https://analyze.intezer.com/).)
2. Submit the file to Intezer Analyze.
3. Install the plugin via pip: `pip install r2analyze`.
4. Add your API key as an environment variable named `INTEZER_API_KEY`.
5. Open the file in r2 and perform an initial analysis (`aaa`).
6. Run the plugin as a r2pipe command (`#!pipe r2analyze`).

Here is an example using a ScarCruft sample
([`7c82689142a415b0a34553478e445988980f48705735939d6d33c17e4e8dac94`](https://analyze.intezer.com/files/7c82689142a415b0a34553478e445988980f48705735939d6d33c17e4e8dac94)).
The
[result](https://analyze.intezer.com/files/7c82689142a415b0a34553478e445988980f48705735939d6d33c17e4e8dac94)
from Intezer Analyze is shown below.

![](images/fig1.png "Intezer Analyze result for a ScarCruft sample.")

If you open the sample and run the plugin, you can see below that four items in
the flag space called `gene` have been created.

![](images/fig2.png "Executing r2analyze as a r2pipe plugin.")

If selecting only that flag space and listing all the flags, you can see that
four functions have been identified as unique to ScarCruft.

![](images/fig3.png "Listing detected functions.")

If Radare2 is your preferred framework for reverse-engineering and analyzing
binaries, now you can use this Intezer Analyze plugin to save time and get
additional insights for your incident response team.

Intezer automates the malware analysis process to quickly identify and classify
malware families. Analyze malware and unknown files for free at
[analyze.intezer.com](https://analyze.intezer.com/)

## Additional Resources

- [IDA Pro plugin](https://intezer.com/blog/malware-analysis/ida-pro-plugin-now-available-to-the-community/)
- [Ghidra plugin](https://intezer.com/blog/malware-analysis/community-ghidra-plugin-is-here/)
