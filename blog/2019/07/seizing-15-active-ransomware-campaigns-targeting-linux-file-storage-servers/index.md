# How We Seized 15 Active Ransomware Campaigns Targeting Linux File Storage Servers

> We simulated over 1,091 infections to exhaust QNAPCrypt’s static Bitcoin wallet list, temporarily halting 15 ransomware campaigns targeting QNAP NAS servers.

- Source: https://research.intezer.com/blog/2019/07/seizing-15-active-ransomware-campaigns-targeting-linux-file-storage-servers/
- Published: 2019-07-10
- Authors: Ignacio Sanmillan
- Tags: linux, ransomware
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

It is rare to see ransomware being used to target the Linux operating system.
However, cyber criminals seem to adapt to this emerging environment and use a
variety of creative methods to gain profits from this landscape.

We at Intezer have detected and temporarily DoS’d the operation of a ransomware
targeting Linux-based file storage systems (NAS servers).

We have named the ransomware QNAPCrypt, as this is the name the authors have
appeared to label the malware. QNAP is a well-known vendor for selling NAS
servers, which the malware was intended to infect and encrypt the containing
files for ransom. NAS servers normally store large amounts of important data
and files, which make them a valuable target for attackers and especially a
viable target for ransomware campaigns.

This malware currently has very low detection rates in all major security
solutions.

The first two sections of this blog post will explain in brief how QNAPCrypt
operates and how we were able to take advantage of two design flaws in the
ransomware infrastructure in order to temporarily stop the campaign—preventing
the malware from infecting additional victims and forcing the authors behind
this malware to deploy new instances. Lastly, we will present a detailed
technical analysis of the malware and the investigation of the entire campaign.

For reference, here is the genetic analysis of the QNAPCrypt malware:

- [ARM variant](https://analyze.intezer.com/#/files/3d7ebe73319a3435293838296fbb86c2e920fd0ccc9169285cc2c4d7fa3f120d)
- [x86 variant](https://analyze.intezer.com/#/files/076a6fa4e051c061e19b9e3e37da9c63a9bc7c1a99111ac13b32eb2f70b7fa5c)

![](images/fig1.png "QNAPCrypt malware")

## How the Ransomware Works

The QNAPCrypt ransomware works similarly to other ransomware, including
encrypting all files and delivering a ransom note. However, there are several
important differences:

1. The ransom note was included solely as a text file, without any message on
   the screen—naturally, because it is a server and not an endpoint.

2. Every victim is provided with a different, unique Bitcoin wallet—this could
   help the attackers avoid being traced.

3. Once a victim is compromised, the malware requests a wallet address and a
   public RSA key from the command and control server (C&C) before file
   encryption.

## How We Seized the Campaign

![](images/fig2.jpg "QNAPCrypt ransomware")

In order to further research the malware and its operation, we wrote a script
to simulate infections on a wide scale to see how the wallet generation
mechanism worked in the attackers’ back end.

After simulating the infections of hundreds of virtual “victims”, we discovered
two major design flaws in the ransomware infrastructure which led us to seize
the operation:

1. The list of bitcoin wallets was created in advance and it was static.
   Therefore, it does not create a new wallet for each new victim in real time,
   but rather it pulls a wallet address from a fixed, predetermined list.

2. Once all of the wallets are allocated (or sent), the ransomware would not be
   able to continue its malicious operation in the victim’s machine.

After simulating the infection of more than 1,091 victims from 15 different
campaigns, we encountered that the attackers ran out of unique Bitcoin wallets
to supply to their victims. As a result, any future infection will be
unsuccessful and the authors behind this malware were forced to update their
implants in order to circumvent this design flaw in their infrastructure to
continue with their malicious operations.

After several days of continuously DoS'ing their infrastructure, we have
observed a newer variant in the wild that
[shares a significant amount of code](https://analyze.intezer.com/#/files/076a6fa4e051c061e19b9e3e37da9c63a9bc7c1a99111ac13b32eb2f70b7fa5c)
with previous QNAPCrypt instances and Linux.Rex. This time, the newer variant
uses an embedded static wallet and RSA public key in contrast to previous
instances.

## Technical Analysis

The initial implant we found came in the form of a statically linked Golang
binary built with the Go linker for ARM architecture. Throughout our research,
we were able to confirm that other variants exist for additional architectures
such as x86 / x64.

Go binaries may seem difficult to analyze when they come stripped, since trying
to make sense of stripped statically linked binaries is usually a more
difficult task than analyzing stripped dynamically linked binaries.

![](images/fig3.png)

We can observe that this binary is indeed a Go executable by looking at the
section names in its section header table.

If we know the location of these sections, in particular the .gopclntab
section, we will be able to reconstruct symbol names and offsets. This
methodology is illustrated in the following diagram:

![](images/fig4.png)

For further insights into populating function names in Go binaries we highly
recommend to view Tim Strazzere’s presentation and [scripts in GitHub](https://github.com/strazzere/golang_loader_assist) which
document this technique.

After retrieving Go function names, analyzing the binary becomes much less
complex since we can highlight the relevant functions of the application. Let’s
not forget that the binary is 4MB in size.

![](images/fig5.png)

After several cryptography algorithm initializations and parsing of arguments
for directory whitelisting and alike functionalities, the malware will send a
GET request to the CNC as a means to communicate that a new victim has been
compromised and that system locking is taking place:

![](images/fig6.png)

After sending this GET request, the malware will attempt to retrieve victim
keys configuration using a client for the SOCKS proxy protocol version 5.

![](images/fig7.png "SOCKS proxy protocol version 5")

This proxy will request to connect to an onion domain name. The following
represents the relevant packets for this connection:

![](images/fig8.png)

After successful connection through the proxy to the onion domain, an
additional GET request to the ransomware REST API is completed in order to
retrieve the RSA public key that will be used to encrypt the file system—a
unique Bitcoin wallet and the ransom note specific to the victim. All of these
artifacts seem to be retrieved based on a specific campaign ID.

![](images/fig9.png)

The response from the server is the following:

![](images/fig10.png)

After victim configuration has been retrieved, the malware will proceed to
remove itself and then it will parse the retrieved RSA public key.

![](images/fig11.png "RSA public key")

This RSA public key will be used to encrypt a random sequence of bytes that
would be used to encrypt the file system later on. This encrypted key will be
base64 encoded and it will be written at the end of the ransom note file called
README_FOR_DECRYPT.txt. We also noted that the ransomware distributes a
different Bitcoin wallet per each compromised system:

![](images/fig12.png)

After this file is created, the malware will proceed to execute the locking
mechanism by walking the file system encrypting files using AES CFB with the
derived encrypted key, avoiding to encrypt the ransom note just created:

![](images/fig13.png)

The malware will target files with the following extensions:

![](images/fig14.png)

After encryption, the malware will rename the affected files so that they will
be prefixed with `.encrypt`:

![](images/fig15.png)

In order for system decryption to take place the base64 encoded random sequence
encrypted with the RSA public key will be needed to be sent to the ransomware
operator via the onion domain site after paying the demanded ransom:

![](images/fig16.png)

After system locking has taken place, the ransomware will communicate that it
has finished with the victim once again to the CNC:

![](images/fig17.png)

## Looking Outside of the Binary

One of our intended goals that we wanted to achieve when analyzing QNAPCrypt
was to assess the scale of victims the ransomware was dealing with.

We were able to find a
[Reddit thread](https://www.reddit.com/r/linux/comments/bucepu/linux_ransomware/)
in which we contacted some of the affected victims:

![](images/fig18.png)

While talking to some of the victims related to the various campaigns of this
malware, we were able to identify the initial attack vector as SSH brute force
attacks and that they were targeting mainly NAS server providers, which
corresponds to how the attacker has chosen to label this malware:

![](images/fig19.png)

After making these findings we studied their infrastructure to determine if
there was anything we could do to interact with this threat actor’s operations.

While researching the ARM instance of the malware, we observed that there was a
request through their REST API in order to retrieve new victim configuration
keys as previously discussed. The following diagram is a high level overview of
the ransomware operation:

![](images/fig20.png)

The connection to the SOCKS5 proxy is completed without any authentication
enforced, and anyone would have the capability to connect to it.

Therefore, we decided to interact with the ransomware infrastructure in order
to retrieve configuration keys and potentially temporarily shut down the
operation of the ransomware to prevent infection of future victims that were
compromised by instances of the ransomware that followed the previous design
architecture:

![](images/fig21.png)

This idea simply abuses the fact that no authentication is enforced to connect
to the SOCKS5 proxy as previously mentioned. Since the authors behind this
ransomware were delivering one Bitcoin wallet per victim from a static pool of
already generated wallets, we could replicate the infection packets to retrieve
all of the wallets until they had no further wallets under their control.
Therefore, when a genuine infection would occur, the ransom client would not be
able to retrieve configuration artifacts.

We wrote the following script in order to implement the methodology described
above:

```python
import socket
import hexdump
import json
import sys

HOST = '192.99.206.61'  
PORT = 65000        

for i in range(15):
    BTC_WALLETS = list()
    while True:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((HOST, PORT))
        
        s.send(b'x05x01x00')
        data = s.recv(1024)
        hexdump.hexdump(data)
        
        s.send(b'x05x01x00x03x16' + b'sg3dwqfpnr4sl5hh.onionx00' + b'x50')
        data = s.recv(1024)
        hexdump.hexdump(data)
        
        s.send(b'GET /api/GetAvailKeysByCampId/%.2d HTTP/1.1x0dx0a' % i +
                b'Host: sg3dwqfpnr4sl5hh.onionx0dx0a' +
                b'User-Agent: http/2x0dx0a' +
                b'Accept-Encoding: gzipx0ax0dx0a')
        data = s.recv(1024)
        print '[+] Campaign id %.2d' % i
        hexdump.hexdump(data)

        try:
            data = json.loads(data[data.find('{'):])
            print data['BtcPublicKey']
            s.close()
            
            if data['BtcPublicKey'] not in BTC_WALLETS:
                BTC_WALLETS.append(data['BtcPublicKey'])
            else:
                sys.exit()

        except ValueError as e:
            print "[+] CAMPAIN HAS NO WALLETS LEFT"
            with open("wallets_%0d.txt" % i, 'w+') as fd:
                for wallet in BTC_WALLETS:
                    fd.write(wallet+'n')
            break
```

We were able to collect a total of 1,091 unique wallets meant to be delivered
to new victims distributed among 15 different campaigns.

![](images/fig22.png)

Furthermore, by depleting the attacker’s stored Bitcoin wallets we were able to
stop this malware from infecting new victims temporarily, since if there is a
failure to parse the RSA public key the client will just exit:

![](images/fig23.png)

The following screenshot shows the packets that the onion domain will retrieve
after the entire static Bitcoin wallet pool was depleted:

![](images/fig24.png)

The HTTP request returns a 200 but with a content length of 0, therefore
failing to retrieve configuration, and thus the ransomware client stops
execution. This implies that we were able to identify an easy method to prevent
further infections of this ransomware by constantly depleting its static
bitcoin wallet pool.

## Attribution and Attackers Reaction

After several days of continuously DoS'ing QNAPCrypt clients, we encountered
another QNAPCrypt sample—but this time targeting x86 systems.

![](images/fig25.png)

Based on Genetic Malware Analysis, we observed that this specific implant
reused a large portion of code with old instances of x86 Linux.Rex builds.
Linux.Rex is known for deploying
[exploits](http://news.softpedia.com/news/crooks-used-sql-injections-to-hack-drupal-sites-and-install-web-ransomware-504300.shtml)
against Drupal servers in 2016, in order to conduct ransomware and DDoS
operations.

The following represents some of the code similarities between Linux.Rex and
newer QNAPCrypt variants:

![](images/fig26.png)

Although both implants implement different functionality, it is noticeable that
both were written in a similar manner.

Furthermore, we can observe similarities with the ARM instance of QNAPCrypt but
with a major difference—the RSA public key, Bitcoin wallet and ransom note are
hardcoded in the binary:

![](images/fig27.png)

We can also see that the hardcoded onion domain is exactly the same as in the
ARM variant, and the site design to pay the ransom is also the same, although
the demanded ransom in Bitcoin seems to be lower than in previous variants:

![](images/fig28.png)

We interpret the discovery of these newer instances with hardcoded
configuration to be a response from the threat actors behind this campaign to
attempt to circumvent the DoS that their non connectionless instances were
suffering. This implied that they were forced to change their implants and to
centralize their bitcoin wallets, making the tracking of their income via their
ransomware campaigns more convenient.

## Conclusion

We have covered the operation of the QNAPCrypt ransomware, and how we were able
to find design flaws to prevent the malware from running in newer victims’
machines and forcing the attackers behind the malware to update their implants
in order to circumvent these flaws.

Additionally, Golang malware seems to be on the rise, since it appears to be a
very convenient language to create cross-platform malware.

Furthermore, we have discussed how Linux ransomware has slightly different
targets than Windows ransomware, in this case targeting NAS servers rather than
Linux endpoints.

Unfortunately detection rates of QNAPCrypt are low, and the ransomware could
create significant monetary losses and economic damage in comparison to other
types of Linux threats.

We have created a custom
[YARA signature](https://github.com/intezer/yara-rules/blob/master/QNAPCrypt.yar)
for detecting future variants of QNAPCrypt.

## Genetic Analysis

The QNAPCrypt malware variants are now indexed in Intezer's genetic database.
If you have a suspicious file that you suspect to be QNAPCrypt or other malware
from the Rex group, you can upload it to Intezer Analyze to detect code reuse
to this threat family and many others.

![](images/fig29.png "Genetic Analysis of the QNAPCrypt ARM variant")

## IOCs

```
sg3dwqfpnr4sl5hh[.]onion
192.99.206[.]61
3d7ebe73319a3435293838296fbb86c2e920fd0ccc9169285cc2c4d7fa3f120d
076a6fa4e051c061e19b9e3e37da9c63a9bc7c1a99111ac13b32eb2f70b7fa5c
```
