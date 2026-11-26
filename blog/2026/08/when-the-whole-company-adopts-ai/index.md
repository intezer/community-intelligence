# When the Whole Company Adopts AI: What It Does to Your SOC

> Intezer platform data shows AI-related alerts are a small but fast-growing share of SOC alerts, mostly false alarms that bury the genuine AI exposures.

- Source: https://research.intezer.com/blog/2026/08/when-the-whole-company-adopts-ai/
- Published: 2026-08-25
- Authors: Nicole Fishbein
- Tags: llm, ai
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


Over the past year, we watched a new class of alert appear in enterprise
security operations centers and grow faster than anything else in the stream:
alerts that were triggered by AI tools and agents. Not attacks against AI, but
the ordinary, everyday footprint of an organization *using* it, from developers
running coding agents and non-technical staff signing consumer AI tools into
corporate accounts.

We reviewed AI-related activity across numerous enterprise environments. Two
numbers frame everything that follows. AI-related alerts still account for only
0.43% of all SOC alerts. And that share is climbing every single month, up 685%
between February and June 2026. AI is a small slice of the alert stream today
and the fastest-growing slice at the same time.

What makes those alerts worth a security team's attention is not their volume
but their composition. We sort everything an AI agent triggers in a SOC into
three buckets: real attacks, risks, and noise, with the split being 94.1%
noise, 5.8% genuine risk, and 0.02% real attacks. Meaning that across the data
we investigated, real attacks that use AI agents are a drop in the ocean. The
cost of AI in the SOC, so far, is not breaches. It is a rising tide of alerts
that look alarming and almost never are, and a small, quiet set of genuine
exposures that those alarms tend to bury.

This post walks through each of the three categories with anonymized examples.
All customer names, hostnames, usernames, and identifiers have been removed;
indicators are defanged.

## The New Shape of the Alert Stream

AI adoption inside an enterprise is not one behavior it is two very different
ones arriving at the same time.

The first is technical. Developers install coding agents that spawn shells,
read credential stores, open network tunnels, download packages, and run
security tooling all as legitimate work, and all of it indistinguishable to a
detection engine from the
[early stages of an intrusion](https://cyberpress.org/agent-tunnels-mimic-c2/).
This is the loud half, and it dominates the data.

The second is when employees grant OAuth consent to third-party AI
applications,
[share information](<https://concentric.ai/is-chatgpt-secure-10-prompts-you-dont-want-your-employees-trying-with-chatgpt/#:~:text=an%20average%20of,and%20HR%20(6.9%25)>),
and paste documents into generative-AI tools. This is the quiet half. It rarely
trips an endpoint detection, but it is where data leaves the building.

Both halves land in the same place, the SOC, and both look, at first glance,
like something to worry about. Sorting the signal from the noise is the entire
job.

## By the Numbers

AI accounts for a small share of the volume but is fast-growing\*\*.\*\* Of the
roughly 16.9 million SOC alerts we reviewed, about 73,000 (0.43%) were
AI-related. Read on its own, that is reassuringly small.

![](images/fig1.png "Number of AI-related alerts per month as seen in our system.")

The rise is monotonic. Every full month is higher than the one before, and
growth accelerated sharply in May 2026. Over the window when reporting is
stable across regions (February to June), volume grew by 685%. The 0.43% figure
is best understood as today's floor, not a ceiling. A team that sizes its
AI-alert handling to current volume will be under-provisioned within a quarter.

The composition is as lopsided as the trend is steep. Nearly all of the
AI-generated alerts are noise.

For this research, we investigated the AI-related population and sorted each
alert by the underlying activity. A real attack is a confirmed compromise. A
security risk is not a compromise but a genuine exposure (for example, a coding
agent running with its permission safeguards disabled). Noise is legitimate
activity that tripped a detection written before AI agents existed. By that
measure, nearly all of the AI-related alerts are noise (94.1%), a small portion
are genuine security risks (5.8%), and real attacks are a sliver (0.02%).

![](images/fig2.png "The breakdown of the AI-related alerts based on the final classification of each alert.")

The second measurement is how those same alerts were handled in production
without a human in the loop. When an alert reaches an automated triage
platform, two separate decisions are made about it.

- The verdict states how dangerous the activity looks: it can be benign,
  suspicious, or malicious.\
  79.8% received a benign verdict.
- The response states what happens next: the alert can be suppressed (closed
  automatically, so no analyst ever sees it), flagged for follow-up, or
  escalated to a human.\
  81.7% were automatically suppressed.

Of the AI-related population, only 5.4% were ever escalated to a human analyst;
the remainder were flagged for follow-up.

A high-severity alert does not necessarily mean an actual threat. For example,
a single detection at a single customer accounted for 55% of all "critical"
verdict alerts flagging a Windows binary (`Expand.exe`) as a
lateral-tool-transfer. Upon inspection, it was found that a developer's coding
agent was setting up a shell environment, and the behavior was normal for this
type of work.

The lesson for any SOC is the same: severity labels on AI activity have to be
read with suspicion, not taken at face value.

## Category 1: Real Attacks

A real attack is an actual compromise or an attacker operation enabled by, or
riding on, AI adoption. This is the category every executive asks about first,
and it is the smallest, accounting for roughly 0.02% of AI-generated alerts.

When it comes to actual threats that were detected in this class of alerts,
none was a compromise caused by an organization's own AI agent. Every alert
titled "AI agent running mimikatz," "reverse shell from a coding tool," or
"credential theft" was resolved, on inspection, to a developer doing legitimate
work or to a detection misfiring. We return to those in the Noise section.

What *was* real is an attack that rides on AI rather than through it: a live
phishing campaign that weaponizes AI brand names as lures. Across multiple
customers, and as we expanded to new ones during the window we studied, we
observed malicious emails with AI-themed subject lines featuring the biggest
names in AI. The lure works precisely because AI adoption has made these brands
familiar and their notifications routine. Employees now expect email from these
products, which is exactly what the attacker is counting on.

Here are some examples of incidents where we spotted the execution of tools or
commands that usually indicate real attacks (or penetration testing), only in
these cases they were invoked by Claude, Codex, etc. So the investigator also
needs to question why the agents were running these tools and whether it was
part of a real attack that exploited the agent.

- Anthropic is used as bait in the business context. In that alert, the email
  subject is `RE: Anthropic Engagement approval & payment`, and the analysis
  says the sender references a supposed contract/invoice with Anthropic to make
  a large payment request appear legitimate. So Anthropic is not the sender or
  the threat source, it’s part of the pretext used to support the invoice fraud
  story.

- An email uses a fake Google/Gemini Ads invitation lure to appear legitimate
  and trustworthy. It presents itself as a business-related workspace
  invitation, encouraging the recipient to connect or join what looks like an
  official Gemini Ads environment, but the sender and reply-to infrastructure
  are not associated with Google and instead rely on the suspicious domain
  `gemini-advertisers[.]com`, indicating a brand impersonation attempt designed
  to drive the user to a malicious site.

- The email impersonates OpenAI (“OpenAI Partner Summit 2026”) but originates
  from noreply-zoomevents@zoom.us. Although the URLs use legitimate zoom.us
  infrastructure, the content and registration flow are being used to lend
  credibility to a fraudulent invitation.

![](images/fig3.png "Phishing email impersonating OpenAI")

![](images/fig4.png "Device code phishing")

- The AI IDE Cursor seems to have moved from normal coding activity into unsafe
  low-level system actions: while likely attempting to complete a debugging or
  troubleshooting task, the agent used a known credential-dumping technique
  (MiniDump via comsvcs.dll) that can expose secrets from process memory. The
  parent-child chain Cursor.exe → powershell.exe → rundll32.exe, the temp .ps1
  scripts, and the memory-dump commands show the IDE initiating an automated
  action sequence that may have been intended to help development, but did so
  in a way that created a serious credential-access risk on the endpoint.

The pattern across all three is worth stating plainly: the closer we looked,
the more the "attack" dissolved into context. That is the defining
characteristic of AI-era triage.

## Category 2: Unsafe Use

About 5.8% of the AI-related alerts are the ones we think deserve the most
attention. These alerts detect an unsafe use of AI tools, not necessarily a
compromise (yet). It is the moment when an agent, behaving exactly as
instructed and with no attacker involved, does something that materially
exposes the organization or the user.

The main risk is agents running with a permission-bypass flag, the option that
tells the agent to stop asking the user before it acts. Many users choose to
trust the agent not to destroy their machines or execute dangerous commands,
but as experience and, now, the data show us, in many cases, agents will
attempt, and mostly succeed in executing commands that expose the organization
and the user to great risks. It is worth noting that, especially when running
the agent with the permission-bypass flag, it is recommended to use additional
configurations, also known as harnesses, to programmatically prevent the agent
from attempting to execute risky commands.

![](images/fig5.png "The split of permission-bypass flags as seen in our system.")

On every sample we examined, the invocation was legitimate developer work. That
is exactly why it matters. This is the same precondition abused in a publicly
documented supply-chain attack, where an attacker's malicious code executed
freely because a coding agent had been launched with its permission prompts
disabled. The exposure is not intent; it is that the rail is off, across many
customers and at scale, waiting for the one time the code the agent is asked to
run is not benign. Notably, these same permission-bypassed invocations are also
the single largest source of false positives.

Other instances of unsafe use we surfaced:

- A reverse tunnel opened by an AI IDE\*\*:\*\* In one of the environments, an
  AI code editor spawned PowerShell, which launched `ngrok` and opened a named
  reverse tunnel to the public internet using the user's own auth token. While
  the intent is benign, it is a real risk and exposure.

- An agent dumping the entire macOS keychain to read one token: To retrieve its
  *own* and cloud services’ stored credentials. An agent ran
  `security dump-keychain > /tmp/,` which writes *every* stored secret to a
  temp file, briefly exposing all of them.

- Granting OAuth access to AI agents means that employees might share sensitive
  information with third-party service providers. But on top of that, it
  increases the risk of unauthorized data access via prompt injection or a
  compromised AI account. We observed multiple alerts for OAuth application
  consent granted to ChatGPT across tenants, "first sign-in to a new
  application: OpenAI" events, and, at one customer, a sizeable cluster of
  data-protection alerts for generative-AI uploads. Most are benign. But this
  is the surface where corporate data is sent to a third-party model, and it is
  almost invisible to endpoint tooling.

## Category 3: Noise

Noise is the largest category by an order of magnitude, 94.1% of the
AI-generated, and it is the one that directly determines whether a SOC drowns.
Noise here is not random. It is specific and diagnosable: detections written
before AI agents existed, now firing at high severity on routine agent work.
This is not a new tendency in the SOC, as Sophos previously
[reported](https://www.sophos.com/en-us/blog/2607_agents_vs_telemetry).

The clearest example is the AI vendors' own software. The genuine Anthropic
Claude Desktop installer, verified by its code signature, triggers major EDR
rules such as "Ransomware Operations detected" and "Encoded PowerShell Download
and Run" across several customers. The installer is legitimate. The detection
describes installer behavior in the ransomware vocabulary.

Underneath that sit the agent-behavior false positives, all confirmed on
inspection to be developers using tools as intended:

- The update of a coding agent and the developers' use of the agent triggered a
  “Ransomware Operations detected”. The binary that triggered the alert is a
  legitimate, signed software package. The behavior that looked
  “ransomware-like” came from normal Electron/Squirrel installer activity and
  developer tool usage.

```
Claude Setup.exe
Path: \Device\HarddiskVolume3\Users\{REDACTED}\Downloads\Claude Setup.exe
CMD: "C:\Users\{REDACTED}\Downloads\Claude Setup.exe"

Update.exe
Path: \Device\HarddiskVolume3\Users\{REDACTED}\AppData\Local\SquirrelTemp\Update.exe
CMD :--install .
Parent: Claude Setup.exe
Parent Path: \Device\HarddiskVolume3\Users\{REDACTED}\Downloads\Claude Setup.exe
Parent CMD: "C:\Users\{REDACTED}\Downloads\Claude Setup.exe"

squirrel.exe
Path: \Device\HarddiskVolume3\Users\{REDACTED}\AppData\Local\AnthropicClaude\app-1.1.1093\squirrel.exe
CMD: --updateSelf=C:\Users\{REDACTED}\AppData\Local\SquirrelTemp\Update.exe
Parent: Update.exe
Parent Path: \Device\HarddiskVolume3\Users\{REDACTED}\AppData\Local\SquirrelTemp\Update.exe
Parent CMD: --install .
```

- A process originating from node.exe executing the OpenAI Codex CLI agent with
  `--yolo` or `--dangerously-skip-permissions` tripping *ClickFix*,
  *DisableTools*, and *DLL-injection* detections.

```
Path: C:\Users\{REDACTED}\AppData\Roaming\...\bin\codex.exe
CMD: codex.exe --yolo
Parent: node.exe
Parent Path: \...\Program Files\nodejs\node.exe
Parent CMD: "node" "C:\Users\{REDACTED}\AppData\Roaming\...\codex\bin\codex.js" --yolo
```

- Standard developer automation triggered “PowerShell created possible reverse
  TCP shell” detections. The process tree is consistent with normal developer
  automation rather than a real reverse shell. The parent process shows a
  signed OpenAI Codex sandbox binary that spawned `powershell.exe`, which in
  turn launched `cmd.exe`, `python.exe`, and `conhost.exe`. The PowerShell
  command is in clear text and shows benign orchestration logic: it checks
  ports on localhost on `127.0.0.1`, selects an available port, starts a Python
  script from the project directory, and writes stdout and stderr to local log
  files in the `artifacts directory`.

The false-positive rates are the story. Across the noisiest AI activity
detections, the benign share ranges from 77% to 99%. Several detections are
wrong on the AI-generated more than four times out of five:

![](images/fig6.png "View of the detection that was triggered on benign AI-related behavior.")

The one exception proves the rule. The *ClickFix* detection is the single
cluster that leans genuinely severe, only 37% benign, and it does so precisely
because it collides with the permission-bypass risk from the previous section:
it fires on coding agents launched with `--yolo`. Even the "real-looking" noise
traces back to legitimate AI use.

## What Security Teams Should Do

From our analysis, the first step for every SOC is clear: tune the noisiest
legacy detections, the ones firing at high severity on routine agent work.
Next, define policies on what information can be shared with third-party AI
platforms (as with any third-party platform) and, based on those policies,
proactively hunt for permission-bypass flags, unauthorized tunnels, and risky
OAuth grants rather than waiting to be alerted.

The second step is harder, because it touches how triage itself works. AI tools
execute commands on the user's machine, with the user's credentials,
essentially acting on the user's behalf. Alerts are therefore triggered by
actions attributed to the user, and in many cases the user was not aware those
actions took place. Before AI, suspicious activity executed on a user's machine
without their knowledge usually indicated a high probability that an attacker
had taken over the machine. Now SOC teams face a new layer of complexity: first
determine whether the action in question was executed by an AI agent or tool.

To separate the user's context from the agent's, and to keep the agent away
from credentials and sensitive information it should not have, we suggest
running AI tools in an isolated environment with restricted access, such as a
Docker container or a virtual machine. Isolation limits what the agent can
reach, and it makes the agent's behavior easier to distinguish from the user's
own activity.

## What This Means for the SOC

Pulling the three categories together, the operational reality of enterprise AI
adoption looks like this:

- Real attacks (0.02%): none of the confirmed attacks we investigated were
  carried out by an organization's own agents. The genuine attack activity we
  found rides on AI adoption from the outside, phishing lures built on brand
  names employees now trust, not on the agents themselves.
- Security risks (5.8%): real, standing, and largely invisible to alerting.
  Agents run with their permission safeguards disabled, open tunnels to the
  public internet, over-expose stored secrets, and send corporate data to
  third-party models. None of this is an incident, and all of it is exposure.
- Noise (94.1%): the dominant cost. The single highest-value action available
  to most SOCs today is not a new detection. It is tuning the legacy ones so
  that a developer running a coding agent does not generate a maximum-severity
  alert.

The uncomfortable synthesis is that AI adoption has not, so far, brought a wave
of AI-enabled breaches. It has brought a wave of *alerts*, small as a share of
total volume today, that have grown 18-fold in six months and are
overwhelmingly false, alongside a smaller, quieter set of genuine exposures
that the alerts tend to bury. A SOC that treats every agent action as a
potential intrusion will exhaust itself on false positives and, in doing so,
will be less likely to notice the `ngrok` tunnel or the keychain dump that
actually matters.

The work ahead is therefore less about detecting AI attacks and more about
teaching detection engines what normal AI behavior looks like before the volume
that is doubling and tripling month over month makes that work unavoidable.
Understanding this distinction is what separates a SOC that scales with AI
adoption from one that is buried by it.
