# Conducting Digital Forensics Incident Response (DFIR) on an Infected GitLab Server

> Runtime alerts on a GitLab server exposed Metasploit shellcode and an XMRig miner in a container, likely entering via weak credentials or CVE-2021-22205.

- Source: https://research.intezer.com/blog/2021/11/dfir-infected-gitlab-server/
- Published: 2021-11-04
- Authors: Nicole Fishbein
- Tags: ir, linux
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## GitLab servers are under attack with a now-patched critical vulnerability

Earlier this week we investigated an incident that occurred on a user's GitLab
server. After the user installed a sensor on their server, an initial runtime
scan was performed. An alert was immediately triggered on the execution of a
malicious metasploit shellcode named `gitlab.elf`, which occurred a few days
prior to the installation of the sensor. Several hours later another alert was
triggered, this time upon the execution of an XMRig Miner. The malware was also
executed several days before the sensor was installed. We notified the user and
began our own investigation with the goal of understanding how the attacker got
into the system and what the scope of the incident was. Consistent with our
findings,
[news broke](https://thehackernews.com/2021/11/alert-hackers-exploiting-gitlab.html)
later that day that attackers were exploiting a GitLab unauthenticated remote
code execution (RCE) vulnerability. In this post we describe the investigation
process conducted by Intezer's research and engineering teams using the runtime
security sensor installed on the victim's host.

## The Incident

The individual was a new user who had just installed the a sensor on a CentOS
host. Immediately after the install, an initial scan was performed and the
sensor identified a running shellcode inside a container on the user's host.
The alert presented important information about the execution of the process
including the process tree, execution path, details about the user's host, and
the files containing malware. Using this information, we were able to get more
details about the malicious process:

- The process was executed inside a container based on a
  `gitlab/gitlab-ce:12.10.10-ce.0` Docker image
- The path and filename of the malicious process is
  `/var/opt/gitlab/gitlab-workhorse/gitlab.elf`
- EUID of the process is 998
- The process connected to `172.96.190[.]95:904`
- The alert also
  [links](https://analyze.intezer.com/files/cd54a34dbd7d345a7fd7fd8744feb5c956825317e9225edb002c3258683947f1)
  to Intezer, which identifies the shellcode as a metasploit.

During this time, the second alert appeared, shown below, notifying the user
that an XMRig Miner was executed inside the container. The name of the file
that contains the malware was `syslogd64`. Using the parent process ID (PPID)
information from the alert, we were able to determine that it was executed by a
script called `okk.sh`. The shellcode in `gitlab.elf` provides a reverse TCP
shell that delivers `okk.sh`.

We inspected the events that occurred on the victim's host and found a curl
command executed inside the container. Shown below are Events which can be
accessed from the sensor dashboard. A curl command is seen which uses the
whoami value as part of the target's hostname.

![](images/fig1.png "Curl command seen under Events.")

We were also able to find another script called `tmp.sh` under the Events tab.

- The scripts install an XMRig Miner and implement persistence by creating a
  systemd service while adding the malware to a `.profile` file in the user's
  home folder.
- Another wallet ID is used in this script:
  `44wrH5Y3VXn577ZJHAJwZG8pLMNQXURsGKnNfbwUNmv3fJdHcedednZSna9hbFU5ojBkF83beV5CJYnqh9jnZNNdTJ4oXor`

![](images/fig2.png "View of tmp.sh execution event from the sensor.")

![](images/fig3.png "Tmp.sh execution event.")

We examined `okk.sh` and got the following details:

- The script removes previously installed cryptominers.
- The script tries to remove the malware that is executed in this attack but it
  doesn't seem to work.
- The installation command, Install XMRig miner, is copied from the pool's
  setup documentation page, shown in the image below.
- The wallet ID used by the attacker is
  `45P62r6YsHWagiRZ6K5tGeKPy2tFFwfYvdfatsKHCC7wATRrsKqqSFySocLVtrKFzYj1RzYu2mSD9HXBofYusbfdAiSCkvM`

![](images/fig4.png "Installation command for the cryptominer")

The victim's host exposed the GitLab authentication page on port 8080. We can
assume that the breach happened due to weak credentials picked up by the
attacker, or by abusing the recently discovered RCE vulnerability
([CVE-2021-22205](https://nvd.nist.gov/vuln/detail/CVE-2021-22205)) in GitLab.

## Looking Inside the Wallets

We were able to view the status of the wallets in the pool's dashboard. Each
wallet was using hundreds of workers at the time of this post. The images below
show details of the wallets including their hash rates and activity of the
cryptominers.

![](images/fig5.png "Hash rates of the wallet.")

![](images/fig6.png "Hash rates of the second wallet.")

## Threat Detection and Response

Attackers are constantly searching for internet-facing instances and
applications that are vulnerable, misconfigured, or have weak credentials.
Users should always install the latest security updates, patch their systems,
avoid exposing applications to the internet when they can, and enforce
multi-factor authentication. While it's important to implement all the right
security policies and to keep systems up to date, zero-days are bound to
happen. Runtime protection solutions come in handy for detecting when the
attacker has already compromised the environment and launched malware. This
post was a joint effort between Intezer's research and engineering teams.
Special thanks to Adir Shemesh.

## IoCs

### Gitlab.elf

`cd54a34dbd7d345a7fd7fd8744feb5c956825317e9225edb002c3258683947f1`

### Syslogd64

`be6ac1ac095545af4a192c55b259f882933269fc7abf06f283b07487b6253ea7`

### Okk.sh

`7d9adc0b7caebe60ab3e5709e1e6de48b6455f352afff61acf53620f9999139f`

### Tmp.sh

`00fc15eadd33b1055cd3f30869363c6498d2668b8f1e244e82ba1a9abff58bf1`

### Wallet IDs

`44wrH5Y3VXn577ZJHAJwZG8pLMNQXURsGKnNfbwUNmv3fJdHcedednZSna9hbFU5ojBkF83beV5CJYnqh9jnZNNdTJ4oXor`

`45P62r6YsHWagiRZ6K5tGeKPy2tFFwfYvdfatsKHCC7wATRrsKqqSFySocLVtrKFzYj1RzYu2mSD9HXBofYusbfdAiSCkvM`
