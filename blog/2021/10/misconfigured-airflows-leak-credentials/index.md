# Misconfigured Airflows Leak Thousands of Credentials from Popular Services

> Misconfigured Apache Airflow instances exposed credentials for services such as Slack, PayPal and AWS through variables, connections, logs and config files.

- Source: https://research.intezer.com/blog/2021/10/misconfigured-airflows-leak-credentials/
- Published: 2021-10-04
- Authors: Nicole Fishbein
- Tags: cloud_security, vulnerability
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


*This research refers to misconfigured Apache Airflow managed by individuals or
organizations ('users'). As a result of the misconfiguration, the credentials
of users are exposed, including their own credentials to the different
platforms, applications and services mentioned in this article. This article
doesn't refer to exposed credentials of the entities behind the development of
the platforms, applications and services themselves.*

## Apache Airflow is the #1 starred open-source workflows application on GitHub

Workflow management platforms are an indispensable tool for automating business
and IT tasks. These platforms make it easier to create, schedule and monitor
workflows. They are typically hosted on the cloud to provide increased
accessibility and scalability. On the flip side, misconfigured instances that
allow internet-wide access make these platforms ideal candidates for
exploitation by attackers.

While researching a misconfiguration in the popular workflow platform, Apache
Airflow, we discovered a number of unprotected instances. These unsecured
instances expose sensitive information of companies across the media, finance,
manufacturing, information technology (IT), biotech, e-commerce, health,
energy, cybersecurity, and transportation industries. In the vulnerable
Airflows, we see exposed credentials for popular platforms and services such as
Slack, PayPal, AWS and more.

## Key Findings

- A number of misconfigured Airflow instances have exposed the credentials of
  popular services including cloud hosting providers, payment processing, and
  social media platforms.
- Exposing secrets such as user credentials can cause data leakage or provide
  attackers with the ability to spread further in the system.
- Customer data exposed as a result of a data leak can lead to violation of
  data protection laws and the possibility of legal action.
- There is also the possibility that malicious code execution and malware can
  be launched on the exposed production environments and even on Apache Airflow
  itself.
- We have notified the identified entities to fix their misconfigured Airflow
  instances as part of the responsible disclosure policy.

All Apache Airflow users are urged to update to the latest version immediately
and make sure their deployments are only accessible to authorized users. In
addition, adopt [secure coding practices](#secure-coding-practices).

## What is Apache Airflow?

Apache Airflow is an open-source workflow management platform. The platform
simplifies the process of creating and managing complex workflows. Airflow
provides plug-and-play integrations with many technologies. Its
[GitHub](https://github.com/apache/airflow/) repository has 22.8K stars, making
it the world's most
[popular](https://www.datarevenue.com/en-blog/airflow-vs-luigi-vs-argo-vs-mlflow-vs-kubeflow)
open-source workflow project.

Airflow uses standard Python to create and schedule workflows, providing users
with a dynamic and convenient way to work with the platform. There are several
concepts and features in Airflow that make it flexible and popular among users:

- Directed Acyclic Graph (DAG) – the primary concept in Airflow that represents
  a collection of tasks with defined dependencies and relationships.
- Task – the basic unit of execution in Airflow, with each node in the DAG
  represented by a task.
- Variables – a way to store content and settings in a key value storage,
  including passwords and API keys that are stored as masked strings. Airflow
  also supports variable encryption using
  [Fernet](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/fernet.html#security-fernet).
- Connections – A feature that stores parameters (username, password, host)
  needed to connect to external systems.
- Logs – Airflow supports logging mechanisms as well as the ability to emit
  metrics.

Version 1 of Airflow was released in
[2015](https://github.com/apache/airflow/releases/tag/1.0.0). From the new web
server UI from version 1.10.0 many security changes were included, for
instance, removing the dangerous Ad Hoc query. In December 2020, version 2.0.0
was released and included many changes in functionality, performance, UI, and
security of the platform.

Version 2 presents a new REST API that requires authentication for all
operations, including logging into the platform. Additionally, the logs do not
not contain sensitive information and specification of the Fernet key is
required for variable encryption. Finally, the configuration is adapted. This
means that several fields are deprecated and other settings require explicit
values rather than using default values. Most of our research is focused on the
older, less secure versions of Apache Airflow, which also emphasizes the risks
associated with procrastinating on making software updates.

## Misconfiguration Risk

### Credential Exposure

One of the main risks of a misconfigured Airflow instance is the credentials
that are exposed with it. These credentials can give a threat actor access to
legitimate accounts and databases, with the ability to perform lateral
movement. If a large number of passwords are visible, a threat actor can also
use this data to detect patterns and common words to infer other passwords.
These can be leveraged in dictionary or brute-force-style attacks against other
platforms. Popular platforms that we found with exposed credentials as a result
of misconfigured Airflow instances include database passwords, API keys, and
cloud credentials. These include but are not limited to:

![](images/fig1.png)

Most of these credentials are exposed through insecure coding practices. We
document some of the ways in which the credentials have been exposed below.

#### Insecure Coding Practices

The most common way to leak credentials in Airflow is through insecure coding
practices. We discovered many instances with hardcoded passwords inside the
Python DAG code. An example of a hardcoded password for a production PostgreSQL
database is shown below.

![](images/fig2.png "Hardcoded PostgreSQL password in DAG code.")

#### Variables

The next most common way for credentials to be leaked is through the
'variables' feature in Airflow. The variables feature allows a user to define a
variable value that is able to be globally used across all DAG scripts. It is
very common to see hardcoded passwords in these variables. An example of a
Slack token used in a variable is shown below.

![](images/fig3.png "Hardcoded Slack token in variables.")

#### Connections

Connections are the correct way to store credentials in Airflow. When a
connection with a password or token is added to Airflow, the password is
[securely encrypted](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#securing-connections)
in a database using the Fernet key. In some cases, this feature is misused and
the credentials are stored in the `Extra` field of the connection in plaintext.
This means that they are not encrypted and can be viewed by anyone. An example
with an AWS key stored in the extra field is shown below.

![](images/fig4.png "AWS key located in misused connection.")

#### Logs

In versions of Airflow prior to 1.10.13, credentials added via the
[command line interface (CLI)](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#creating-a-connection-from-the-cli)
would be logged in plaintext in the logs section; now registered as
[CVE-2020-17511](https://www.cvedetails.com/cve/CVE-2020-17511/). Below shows a
log entry for a SQL Lakehouse platform.

![](images/fig5.png "Password located in logs.")

#### Configuration

The configuration file (airflow.cfg) is created when Airflow is first started.
It contains Airflow's configuration and it is able to be changed. This file can
contain sensitive information such as passwords and keys. If the setting in the
file `expose_config` is set to `True`, anyone can access the configuration from
the web server UI. Accessing from the UI can expose credentials. It is common
to see plaintext fernet keys in these files as shown below.

![](images/fig6.png "Key located in exposed configuration file.")

### Leakage of Sensitive Data

Many Airflow instances contain sensitive information. When these instances are
exposed to the internet the information becomes accessible to everyone, since
the authentication is disabled.

Leakage of sensitive data essentially means that attackers have access to
information on the organization that owns the exposed server. They can steal
and use the information in many ways. For example, attackers can use leaked
information to gather more information about the organization (reconnaissance
stage as defined in
[The Cyber Kill Chain®](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html))
and launch an attack that is tailored towards that organization. Leaked
information can also reveal details about the compromised organization's
clients. The consequences of data leakage can lead to serious reputational
damage for the company and the potential loss of existing and potential new
clients.

In versions prior to v1.10 of Airflow, there is a feature that lets users run
Ad Hoc database queries and get results from the database. While this feature
can be handy, it is also very dangerous because on top of there being no
authentication, anyone with access to the server can get information from the
database.

![](images/fig7.png "Screenshot of the Ad Hoc Query feature in Airflow UI.")

Many exposed Airflow instances that we found revealed information about the
services and platforms that companies are using in their software development
environments. For example, several instances included the private names of
Docker images or internal dependencies used in the workflow. Exposing
information about tools and packages used in the organization's infrastructure
can jeopardize the organization and also be leveraged by threat actors in
supply chain attacks. This can lead to attacks that leverage dependency or
image short names to deliver malicious code instead of the intended code. A
really good example of this type of attack is covered
[here](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610).

![](images/fig8.png "Private container registry.")

### Legal Action

Exposing customer information can also lead to violation of data protection
laws and the possibility of legal action. One such data protection law is
General Data Protection Regulation (GDPR), which applies to organizations
handling the data of European citizens or any entities located in the European
Union (EU). Violating this law by leaking sensitive customer information can
lead to sizable administrative fines.

Disruption of clients' operations through poor cybersecurity practices can also
result in legal action such as class action lawsuits. In the May 2021 Colonial
Pipeline hack, consumers that had their business disrupted
[filed class action lawsuits](https://www.washingtonpost.com/technology/2021/07/25/ransomware-class-action-lawsuit/)
against Colonial Pipeline.

### Malware

There is also the possibility that Airflow plugins or features can be abused to
run malicious code. An example of how an attacker can abuse a native
'Variables' feature in Airflow is if any code or images placed in the variables
form is used to build evaluated code strings.

Variables are able to be edited by any visiting user which means that malicious
code could be injected. One entity we observed was using variables to store
internal container image names to execute. These container image variables
could be edited and swapped out with an image containing and running
unauthorized or malicious code.

![](images/fig9.png "Image name variables vulnerable to exploitation.")

Another possible route for malicious code execution can come through unofficial
third-party plugins. In some instances, we noticed that the plugin
[`airflow-code-editor`](https://github.com/andreax79/airflow-code-editor) was
being used. This can be leveraged to edit DAGs to include malicious code that
can then be triggered from the UI. Of all the exposed instances, we noted only
a small percentage using these types of vulnerable plugins.

![](images/fig10.png "Vulnerable code editor plugin.")

## Mitigation

### Versioning

Airflow made great progress in terms of security features implemented in
version 2.0. The changes included the following:

- The dangerous Ad Hoc query was removed from the GUI.
- The logs do not leak information.
- Enforced login and authentication required for all operations in the REST
  API.
- Security tab was added to the dashboard. It provides information about users
  and the permissions they have for each menu in the dashboard.
- The configuration file is more strict and requires explicit specifications of
  configuration values rather than using default values.

In light of the major changes made in version 2, it is strongly recommended to
update the version of all Airflow instances to the latest version. Make sure
that only authorized users can connect.

### Secure Coding Practices

Secure coding practices should be adopted whenever possible. Passwords should
not be hardcoded and the long names of images and dependencies should be
utilized. You will not be protected when using poor coding practices even if
you believe the application is firewalled off to the internet.

The connections should be utilized in the proper way specified by Airflow for
maximum security. That process for managing connections is documented
[here](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html).

If there is sensitive information that can not be placed within the
connections, consider using environment variables instead.

## Detecting Attackers Exploiting Misconfigurations

Misconfigurations and broken access control are among the top reasons for
access to unauthorized functionality and data. If you don't believe us, check
out our research about misconfigurations in another popular workflow platform
called
[Argo Workflows](../../../2021/07/new-attacks-on-kubernetes-via-misconfigured-argo-workflows/).

In the event that an attacker has already exploited a misconfiguration or
vulnerability, you will need to detect and terminate the malicious or
unauthorized code in your production environment as soon as it is executed.
