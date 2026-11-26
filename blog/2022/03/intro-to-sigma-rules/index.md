# SOC Level Up: Introduction to Sigma Rules

> An introduction to Sigma rules for SOC analysts: what they are, how the metadata, log source and detection sections work, and how to compile them for your SIEM.

- Source: https://research.intezer.com/blog/2022/03/intro-to-sigma-rules/
- Published: 2022-03-22
- Authors: Nicole Fishbein
- Tags: tool
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


*Sigma rules are catching on more and more for SOC teams, as a way to write one
rule that can be used across multiple environments. By learning how Sigma rules
work and how to create them, you can take your SOC skills to the next level.*

Detecting security breaches inside an infrastructure is heavily based on
analyzing and monitoring events using logs. There are different types of logs,
aggregation systems, strategies and technologies that help SOC analysts in
their day to day job. While it's excellent that there are a wide range of tools
SOC teams and organizations can implement in their security posture, it also
complicates the process of sharing information and knowledge within the
organization and the community – each SIEM has its own query syntax (or
language) and each log has it's own unique fields.

Often analysts create rules to detect active threats or attacks and
organizations would want to use them to alert upon a possible breach. For
example, we might have an excellent rule to detect the creation of a specific
malicious process and we want to share the rule with the community, partners,
or clients. Having said that, sharing detection rules for behavior is
complicated because each organization has its own way to digest logs,
infrastructure and tools, so it might be more challenging for them to
understand the rules and to integrate them into existing infrastructures.

The solution is using Sigma rules. These rules are written in a well-defined
format using a markup language. Sigma is used for generating queries for
specific SIEMs and configurations. Using Sigma for writing detection rules
makes it easier to share and integrate them in the organization, regardless of
specific tools and logs that are used.

In this blog we will introduce you to Sigma rules: What are Sigma rules? What
is the syntax of Sigma rules and what makes them a powerful tool that should be
used in your organization. We will present examples of how to write rules based
on analysis or real threats and explain the subtleties of writing these rules.
Once you get familiar with the format you will be able to understand the rules
and write them by yourself.

This blog is the first part of a blog series in which we are going to present
key technologies and concepts that can be useful for SOC analysts who wish to
improve their skills and knowledge to do a better job in their current position
or to switch positions. In this series, we will introduce topics such as threat
intelligence.

## What Are Sigma Rules?

Sigma is an [open-source](https://github.com/SigmaHQ/sigma) framework that
provides the ability to write rules to analyze logs – similar to YARA for files
or Snort for network analysis. Sigma rules are written using a predefined
syntax in YAML format, and then they are converted (using sigmac or
[online converter](https://uncoder.io/)) to a format that fits the target SIEM
or platform used in the organization. There are many supported targets such as:
Splunk, Elasticsearch, Microsoft Defender, and many more. Sigma can be used
with different log sources.

The agility of Sigma rules and the fact that one rule can be used in
environments that use different configurations, makes it easier for analysts to
write these rules and share them with colleagues and the community.

Sigma rules are a powerful tool that makes it easy to analyze different types
of logs and find specific action or threat. It can be used in two ways:

### Identify and alert on suspicious activity

Sigma rules can be integrated into SIEM platforms and detect different events
as they happen thus helping to detect and stop them before any further damage.
For example we can create †rules to detect: unauthorized actions, web/resource
access, file modification, process creation and much more.

### Threat Hunting

Sigma rules can be used to hunt for threats:

- Use the rules to detect when a certain attack or threat targets your
  organization.
- Check if your organization was breached by applying Sigma rules to old logs
  (assuming your organization aggregates logs for at least a few months). Often
  it
  [takes an organization several months](https://www.upguard.com/blog/cost-of-data-breach#:~:text=In%202021%2C%20the,the%20damage%20costs.)
  before they discover that an attacker is already in the system. By analyzing
  the logs for suspicious activity you increase the chances of discovering a
  security breach and starting an incident response process sooner.

## Sigma Rules Syntax

![](images/fig1.png "Source: Sigma repo")

The structure of Sigma rules is made of optional and mandatory parts as can be
seen in the picture above.

### Metadata

These fields provide information about the rule and add informative comments
and notes. While it's not mandatory, it is especially important to add this
information if you plan to share the rule with other people. There are a number
of fields that can be used here:

- Title of the rule
- Author (optional)
- Date (optional)
- Unique ID (optional)
- License (optional) – it is recommended to add this field if you plan to share
  the rule. By default the distribution is very strict, you can choose to
  change it using [SPDX](https://spdx.dev/ids/) standard.
- Tag (optional) – For example, associate the rule with a technique from the
  MITRE ATT&CK framework.
- Status (optional) – possible values can be experimental or test, to indicate
  that the rule might need further tuning and testing.

### Log Source

The log source section describes which logs should be searched and analyzed.
There are several optional fields that specify which type of logs are relevant
for the Sigma rule. While these fields are optional a rule should include at
least one of them as it provides vital information relevant to the detection
and will help the user of the rule integrate the rule in his infrastructure.

- Category: specify the logs from a group of products. For example: firewall,
  process_creation, file_event.
- Product: a specific software or service. For example: Windows, Apache, Zeek.
- Service: select subset from the product.
- Definition: a place for comments and additional information regarding the
  log.

### Detection

In this section, we define what we are looking for in the logs. This section
will contain one or more blocks usually called `selection` or `filter` but it
can have any other name. Each block will contain the relevant fields and
information that are needed for the detection of a specific event. When looking
for a match we can look for a specific string, event id, or a combination of
them, using the following fields and properties. Let's look at the example
below:

A rule to detect mshta execution where the process's parent image ends with
`svchost.exe` and the image of the process ends with `mshta.exe`:

```yaml
logsource:
 category: process_creation
 product: windows
detection:
 selection:
   ParentImage|endswith:
     - 'svchost.exe'
     - 'cmd.exe'
     - 'powershell.exe'
   Image|endswith:
     - 'mshta.exe'
 condition: selection
```

Based on what we learned from the previous section we know that this rule can
be used to scan logs of process creation on Windows OS.

Next, there is a block called `selection` which is a
[map](https://github.com/SigmaHQ/sigma/wiki/Specification#maps) (or directory)
that contains pairs of keys and values. In the example above there are two
keys: `ParentImage` and `Image`. The elements of the map are linked with
logical AND meaning we are looking for a match for both the image of the
process and its parent's image.

For the parent image the strings are in a
[list](https://github.com/SigmaHQ/sigma/wiki/Specification#lists) format – they
are linked with OR and we need at least 1 match in the end (because of
`endswith`) of the path :`svchost.exe`, `cmd.exe` or `powershell.exe`. In SIGMA
a string is:

- Case insensitive.
- Can contain
  [regular expressions](https://github.com/SigmaHQ/sigma/wiki/Specification#general)
  (regex) – in this case it will be case sensitive.
- Wildcards can be used (\* and ? ) in the detection. If needed these
  characters can be escaped using the backslash.

Note that the value uses a modifier: `endswith` – it specifies where the string
should be found; there are
[many other](https://github.com/SigmaHQ/sigma/wiki/Specification#value-modifiers)
modifiers that can be used. In the examples, the image of the process needs to
end with `mshta.exe`.

Lastly, the condition specifies which conditions have to be fulfilled in order
to identify an event and trigger an alert if needed. When a rule has several
sections they can be linked with different logical operations such as AND, OR,
NOT and [more](https://github.com/SigmaHQ/sigma/wiki/Specification). We can use
the condition to create more detailed rules, filter out certain matches and
tune the rule to avoid false positives. In our example the detection will
happen when a log would contain the specified image paths.

## Compiling Sigma Rules

After we write the rule we need to save the file and use sigmac (available in
the
[repo](https://github.com/SigmaHQ/sigma/tree/0889a1fc3b1a950fab00a5334f1fef3cd8ba0cb2/tools))
to compile the rule. The command will look like this:

```
./sigmac -t <target> -c <path to configuration file> <path to the rule>
```

To compile a rule you must specify the target SIEM, provide a configuration
file and the path to the rule. Each argument is very significant for a
successful compilation of the rule and the integration of the rule in SIEM
systems. We will explain each argument in the command.

### Target

Target is the SIEM or the system that will analyze the logs: it can be splunk,
stix, sysmon, etc. Each system has its own syntax so the output will differ.

We will continue using our example for detecting mshta from the previous
section, let's compile it for splunk:

```
./sigmac -t splunk -c config/generic/windows-services.yml 
sigma_rules/detect_mshta.yml
((ParentImage="*\svchost.exe" OR ParentImage="*cmd.exe" OR 
ParentImage="*powershell.exe") (Image="*\mshta.exe"))
```

And now for sqlite:

```
./sigmac -t sqlite -c config/generic/windows-services.yml 
sigma_rules/detect_mshta.yml
SELECT * FROM eventlog WHERE ((ParentImage LIKE '%\svchost.exe' ESCAPE '' 
OR ParentImage LIKE '%cmd.exe' ESCAPE '' OR ParentImage LIKE 
'%powershell.exe' ESCAPE '') AND (Image LIKE '%\mshta.exe' ESCAPE ''))
```

We wrote one rule that can be easily integrated into two entirely different
environments and that's the power of Sigma rules. You can run `sigmac --list` to
view the full list of supported targets.

### Configuration File

Each environment and organization can use different log sources or index the
logs differently. To make Sigma rules relevant and usable in environments
regardless of the logs they use, Sigma relies on configuration files. A
configuration file contains the mapping of the logs and the fields that are
being used in the environment to the fields used in the rules. We can think of
it as a translation between the rule and our environment.

Valid log source definition must contain at least one category, service or
product specification that exactly corresponds to the same fields in the rule's
logsource section.

Sigma Git repository contains many
[configuration files](https://github.com/SigmaHQ/sigma/tree/0889a1fc3b1a950fab00a5334f1fef3cd8ba0cb2/tools/config)
for the different log sources and SIEM systems. Note that in order to use the
configuration file in your environment you might need to modify it a bit and
adjust the mapping, but for a simple testing environment they should be good
enough.

For example this is the output of our example rule when using windows-services
logs for splunk:

```
./sigmac -t splunk -c config/generic/windows-services.yml 
sigma_rules/detect_mshta.yml
((ParentImage="*\svchost.exe" OR ParentImage="*cmd.exe" OR 
ParentImage="*powershell.exe") (Image="*\mshta.exe"))
```

[Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) is a
service that is part of sysinternals, it monitors and logs a wide range of
events and activities, the output can be viewed in the Windows event log
viewer. This is the output of the example rule for sysmon for splunk:

```
./sigmac -t splunk -c config/generic/sysmon.yml 
sigma_rules/detect_mshta.yml
(EventID="1" (ParentImage="*\svchost.exe" OR ParentImage="*cmd.exe" OR 
ParentImage="*powershell.exe") (Image="*\mshta.exe"))
```

Let's check the
[configuration](https://github.com/SigmaHQ/sigma/blob/0889a1fc3b1a950fab00a5334f1fef3cd8ba0cb2/tools/config/generic/sysmon.yml)
for Sysmon. Below is a snippet of the configuration:

```yaml
logsources:
    process_creation:
        category: process_creation
        product: windows
        conditions:
            EventID: 1
        rewrite:
            product: windows
            service: sysmon
    process_creation_linux:
        category: process_creation
        product: linux
        conditions:
            EventID: 1
        rewrite:
            product: linux
            service: sysmon
```

The configuration file specifies the fields that will be used in a Sigma rule
and how they will be defined or where they are located in logs that are
generated by sysmon. The way the fields are defined in the rule must fully
match the to the values set in the configuration file.

The configuration above tells sigmac that for rules that contain logsource with
category process_creation and product windows replace these fields with product
windows and service sysmon. Depending on the OS targeted by the rule, the
compiled output will define the product type. EventID: 1 is added with a
logical AND to the query outputted by sigmac, meaning only events with this
event ID are applicable to this rule. This way rules that didn't use sysmon as
a log source could be integrated into infrastructure that uses sysmon in their
logging and use this rule.

You might ask yourself why we didn't write the rule with Sysmon as a logsource
from the beginning – and the answer is that we could have done that! There
might be more than one way to write a Sigma rule. We can make the rule target
sysmon logs, so if the users of the rule use sysmon in their environment they
will need minimal effort on their part to integrate it.

Our mshta rule for sysmon:

```yaml
logsource:
product: windows
service: sysmon
detection:
selection:
  EventID: 1
  ParentImage|endswith:
    - 'svchost.exe'
    - 'cmd.exe'
    - 'powershell.exe'
  Image|endswith:
    - 'mshta.exe'
condition: selection
```

The original example of our mshta rule uses a more "general" approach: we
didn't specify which logsource to use and the user can convert the rule to fit
his environment by mapping the fields in his configuration file. In the first
example it is up to the user to tune the rule and adjust the rule or the
configuration file while in the last option, we specifically mention which log
source and fields the rules use. Both rules will achieve the same result.

## Conclusion

In this blog we presented Sigma rules – a well-defined and formatted structure
for writing detection rules, that can be used in all types of operating systems
and environments. Sigma allows you to share information but also consume it,
making it easier to integrate new detection rules and protect your environment.
