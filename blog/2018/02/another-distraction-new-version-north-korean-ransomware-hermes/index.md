# Yet Another Distraction? A New Version of North Korean Ransomware Hermes Has Emerged

> Hermes 2.1 ransomware shares code with the Hermes version used as a diversion in a Taiwan bank heist and thought to originate from North Korea’s Lazarus group.

- Source: https://research.intezer.com/blog/2018/02/another-distraction-new-version-north-korean-ransomware-hermes/
- Published: 2018-02-08
- Authors: Jay Rosenberg
- Tags: apt, apt_dprk, ransomware, windows
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Detecting Reused Ransomware

Whether we’re dealing with a criminal threat actor looking to steal money from
their victims using ransomware or malware sponsored by a nation state, we can
consistently see the reuse of code. In this specific case, we have observed a
variant of a well-known ransomware, via a new version of Hermes from what may
have originated from a nation state threat actor.

According to reports by researchers at
[McAfee](https://securingtomorrow.mcafee.com/mcafee-labs/taiwan-bank-heist-role-pseudo-ransomware/)
and
[BAE Systems](http://baesystemsai.blogspot.com/2017/10/taiwan-heist-lazarus-tools.html),
a ransomware named Hermes was used as a diversion in an attack involving a bank
heist in Taiwan. The ransomware is thought to have originated from the Lazarus
group, a threat actor known to be affiliated with North Korea.
([You can be read about them in this blog post about the Blockbuster campaign.](../../../2017/12/blockbusted-lazarus-blockbuster-north-korea/)).
Security researcher
[@demonslay335](https://twitter.com/demonslay335/status/960590594707017729)
tweeted about the existence of a new sample Hermes 2.1, so our team decided to
take a deeper look.

## Code Reuse Analysis of Hermes 2.1

When examining new binaries, the first step we take in our research is to take
the binary and upload it to our Intezer Analyze™ system in order to identify
code reuse.

![](images/fig1.jpg "Code Reuse Analysis of Hermes")

Here we can see some code reuse between the Hermes samples that were originally
discovered as well as the latest sample. Since the sample came out mostly
unique — indicating that much of the binary has changed — we were still able to
catch some key parts that clearly reuse code.

Intezer Analyze™ caught these fragments, and with a deeper look into IDA Pro,
we find an exact function-for-function match:

![](images/fig2.png)

In other places, although the code is not exactly the same, we can see very
similar code to the original Hermes and techniques known to be used by Lazarus.

## An Evolving Threat

The last time this ransomware appeared there was a bank heist affiliated with
it, and now it is possible that this new sample was used in an attack where the
infected target was unaware of the intended result. It may have been used to
cover up intellectual property theft, bank fraud, or something even more
nefarious. At this moment in time, there is not enough information to make
definitive conclusions about the specific intent of the Lazarus group; however,
with the reappearance of Hermes, we can be confident that this likely won’t be
the last time this code will be used in an attack.

## IOCs

```
bcb96251c3e747c0deabadfecc4e0ca4f56ca30f8985cae807ca2ff29099d818 New Sample
851032eb03bc8ee05c381f7614a0cbf13b9a13293dfe5e4d4b7cd230970105e3 Related Sample
```
