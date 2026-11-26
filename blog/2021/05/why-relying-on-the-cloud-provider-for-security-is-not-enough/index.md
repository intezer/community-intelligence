# Why Relying on the Cloud Provider for Security is Not Enough

> Cloud Snooper, flaws in security tools, misconfigurations and unauthorized workload access show where the built-in security of AWS, Azure and GCP falls short.

- Source: https://research.intezer.com/blog/2021/05/why-relying-on-the-cloud-provider-for-security-is-not-enough/
- Published: 2021-05-20
- Authors: Nicole Fishbein
- Tags: cloud_security
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


[73% of organizations](https://blog.paloaltonetworks.com/2020/06/cloud-native-security-genome/)
using the cloud are not sure which parts of security fall
under their responsibility. Ultimately, the customer is responsible for
security in the cloud, meaning protecting the workloads (applications and code)
hosted on top of the virtual resources created in the cloud provider's
platform. Whereas the cloud provider is responsible for the security of the
cloud, meaning the physical infrastructure (e.g., data centers, network and
server equipment) and for operating that infrastructure (e.g., physical
security, power redundancy, connectivity between facilities). The big three
cloud providers, Amazon Web Services (AWS), Microsoft Azure and Google Cloud
Platform (GCP), offer various built-in tools and services designed to prevent a
wide range of attacks and reduce the attack surface. Yet, breaches still
happen. Let's examine some cases where the security tools provided by the cloud
provider were not enough to fully secure the customer's environment.

## Security in the Cloud

Before we get started, it's important to note that most cloud monitoring tools
are based on one or more of the following categories:

- **Network security** – Firewalls, networking segmentation and protection
  against DDoS attacks.
- **Vulnerability management** – Scanning and patching vulnerabilities in the
  cloud system.
- **Cloud Workload Protection Platforms (CWPP)** – Focus on securing the
  workloads themselves with runtime visibility and protection.
- **Cloud Security Posture Management (CSPM)** – Find misconfigurations and
  security risks based on predefined security policies in the cloud
  infrastructure. In some cases, remediation can be automated to terminate an
  attack.
- **SIEM capability** – A combination of security event management (SEM) and
  security information management (SIM). The goal is to collect logs from
  different sources and take the appropriate action when an incident happens.

The security tools provided by the cloud providers are not bulletproof though.
Certain attack vectors, such as unknown vulnerabilities and supply chain
backdoors, are not preventable.

## Cloud Snooper Attack Bypasses Firewall

The
[Cloud Snooper](https://news.sophos.com/en-us/2020/02/25/cloud-snooper-attack-bypasses-firewall-security-measures/)
attack bypassed the on-premise
[AWS Network Firewall](https://aws.amazon.com/network-firewall/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc)
using a rootkit. The rootkit hid the communication with the Command and control
(C2) server by disguising it as "innocent" packets and then dropped a RAT. The
attackers had control over the victims and a safe channel to communicate with
the C2. Despite having a well-configured firewall the attackers still managed
to infiltrate the system, meaning the security tools of AWS couldn't stop this
attack. Having runtime visibility is an important last line of defense, since
it detects the execution of any unauthorized code or commands which are
critical for an attacker to carry out a successful attack. AWS doesn't support
this capability, therefore the cloud customer should leverage third party tools
for workload-centric security and runtime protection.

## Security Tools Have Vulnerabilities Too

Cloud services
[have their own vulnerabilities](../../../2021/05/cve-2021-27075-microsoft-azure-vulnerability-allows-privilege-escalation-and-leak-of-data)
which can be exploited as an initial attack vector. The problem is it's nearly
impossible to eliminate all software vulnerabilities, especially for large
deployments spread across multiple clouds and when using third parties.
Consider that not every software has a fix and not every vulnerability is
known. Not to mention, fixing vulnerabilities takes time and third party
vulnerable software also exists. Relying only on the cloud provider's tools is
not enough because once a vulnerability in the tool is exploited by an
attacker, the only way to detect and stop the attack is by detecting
unauthorized code or commands in runtime. This capability is not natively
supported by AWS and GCP. Azure offers
[Microsoft defender](https://intezer.com/blog/incident-response/autonomous-soc-for-microsoft-defender/)
however it has limited support for Linux which runs 90% of the cloud.

## Misconfigurations are the Leading Cause of Attacks

A whopping 99% of
[cloud security failures](https://www.cio.com/article/3529426/posture-management-cloud-security-tools-rise-in-wake-of-breaches.html)
are expected to be the fault of the customer through 2025. Despite the variety
of tools and policies, setting up and maintaining a secured environment relies
on skilled and experienced professionals. Yet, people can make mistakes that
can compromise information or expose the infrastructure to attacks. For
example:
[FedEx exposed information by leaving an exposed storage server](https://www.infosecurity-magazine.com/news/fedex-s3-bucket-exposes-private/).
The
[Pentagon](https://money.cnn.com/2017/11/17/technology/centcom-data-exposed/)
exposed [100 GB of data](https://www.bbc.com/news/technology-42166004) from a
joint intelligence program with the U.S. Army and National Security Agency. The
data had been available for years in a publicly exposed S3 bucket. The attack
using a
[misconfigured Docker port](../../../2020/07/watch-your-containers-doki-infecting-docker-servers-in-the-cloud/)
drops the Doki malware which installs a backdoor on the victim's machine and
executes a cryptomining malware. 73% of companies leave SSH ports
[publicly exposed](https://www.eweek.com/security/misconfiguration-of-aws-services-by-users-exposes-cloud-security-risks/).
Sometimes the user is not aware of the misconfiguration only until a breach
occurs and the damage is done. In this case, a runtime protection solution can
stop the attack before it causes more damage.

## Unauthorized Access to Cloud Workloads

Credentials and permissions have an important role in securing the
infrastructure and cloud providers offer dedicated tools like Identity and
Access Management (IAM). However, when attackers or malicious insiders gain or
steal credentials to cloud assets they
[can cause extensive damage](https://www.darkreading.com/attacks-breaches/compromised-credentials-show-that-abuse-happens-in-multiple-phases/d/d-id/1340179)
to the entire cloud environment. Attackers used
[Tesla's Kubernetes dashboard](https://arstechnica.com/information-technology/2018/02/tesla-cloud-resources-are-hacked-to-run-cryptocurrency-mining-malware/)
that was exposed to the public internet and not password protected. One of the
pods contained access credentials for Tesla's AWS environment which included an
Amazon S3 bucket that stored sensitive information and telemetry. The attackers
ran
[cryptomining malware](../../../2021/01/a-rare-look-inside-a-cryptojacking-campaign-and-its-profit/)
on the server and took measures to hide the activity of the miner. The
attackers used atypical ports for communicating with the mining pools and they
also did not use well-known mining pool hosts. When an attacker bypasses all
security measures, having clear visibility into what code is running on your
systems enables you to identify and stop the attack before the attacker gains
remote access to other assets. Having runtime security measures in the
environment will help to identify and mitigate attacks on cloud workloads. But
not all cloud vendors have these tools, specifically for Linux.

## You Need Runtime Protection

Currently, AWS and GCP do not offer a runtime Cloud Workload Protection
Platform (CWPP) focused on what is running "inside the machine." Azure has a
CWPP but it's mostly focused on Windows threats, which means you still need a
third party tool to secure your Linux workloads. Runtime visibility and
protection for your cloud-hosted applications is a critical control that is not
supported by the cloud providers. At Intezer, we recommend protecting the
runtime environment as a key security control. Most research shows that nearly
all successful attacks are based on running unauthorized code or commands,
regardless of how the attacker got in. Risk reduction programs like
vulnerability scanning and misconfiguration management are very important but
they take a lot of time. If you take an assume breach mentality, an attacker
will eventually get through, making the last line of defense a priority.
Protecting the runtime environment buys you time to do larger programs to
reduce risk and have a more holistic cloud security. Runtime protection is also
needed to detect and respond to attacks. If an attacker exploits an unknown
vulnerability or a backdoor in the supply chain you must be able to detect it.
