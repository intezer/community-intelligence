# Exposed Prefect Workflows Could Lead to Disruptive Attacks

> Hundreds of misconfigured Prefect workflow servers are exposed online, which could let attackers modify or delete workflows, tasks, agents and execution logs.

- Source: https://research.intezer.com/blog/2021/10/exposed-prefect-workflows-could-lead-to-disruptive-attacks/
- Published: 2021-10-26
- Authors: Nicole Fishbein
- Tags: cloud_security, vulnerability
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Introduction

Workflow management platforms are powerful tools for automating and managing
complex tasks. Integrating workflow platforms can help companies coordinate and
ease their business processes as well as increase the productivity of their
teams. When the instances are misconfigured, they can expose the systems to
potential attacks and data loss. This is why it's crucial to properly set up
these instances. Companies recently learned this the hard way when
misconfiguring popular open-source workflow platforms
[Argo](../../../2021/07/new-attacks-on-kubernetes-via-misconfigured-argo-workflows/)
and
[Airflow](../../../2021/10/misconfigured-airflows-leak-credentials/).
This post will focus on misconfigurations in the Prefect workflow platform. We
have identified hundreds of misconfigured Prefect instances which could be the
target of disruptive attacks. By using Prefect's web UI and API function,
attackers can potentially modify and delete workflows, tasks, and agents,
including the history logs of previous executions, belonging to companies with
exposed instances. Disruptive attacks can cause data loss and business delays
affecting teams that integrate workflows into their daily routines and the
clients that depend on these workflows functioning properly. We did not find
any evidence of Prefect users having their information leaked as of yet.
Prefect users should verify that their instances are not exposed to the
internet and enable multi-factor authentication.

## What is Prefect?

Prefect is a workflow management platform launched in 2018 and powered by the
open-source Prefect Core workflow engine. It's a popular platform marked by its
7.6K stars on [GitHub](https://github.com/prefecthq/prefect). There are two
ways to set up Prefect: 1) Using the
[Prefect Cloud](https://www.prefect.io/cloud/) engine or 2) Setting up a local
[Prefect server](https://github.com/prefecthq/server). The main difference
between the two options is Prefect Cloud has everything already configured and
ready to use while the latter option requires users to put more effort into
configuring the infrastructure. Prefect uses a full
[GraphQL API](https://docs.prefect.io/orchestration/ui/interactive-api.html)
for interacting with the platform and easily accessing all of the information
stored in the platform. The API can be accessed using the web UI or through a
Python client. When using Prefect Cloud, authentication is required to access
the platform. However, when using the Prefect server, authentication is not
required by default, meaning that anyone with access to the server can get
information from the Prefect platform. For companies that have instances
exposed to the internet and which don't incorporate authentication, essentially
this means that anyone can view the information stored by those companies on
Prefect. This includes flows, tasks, agents and projects. Unlike
[Argo](../../../2021/07/new-attacks-on-kubernetes-via-misconfigured-argo-workflows/)
and
[Airflow](../../../2021/10/misconfigured-airflows-leak-credentials/),
Prefect is a much more secure platform and we did not find any evidence of
companies that use Prefect having their information leaked. We examined the
functions exposed by the API and the actions that users can make in the web UI.
There are several functions in the web UI and GraphQL API that can be used by
attackers to delete elements in the workflow platform and launch disruptive
attacks on the companies that own the exposed instances.

## How is the Attack Executed?

Workflows can be deleted with a single button click in the web UI as shown
below.

![](images/fig1.png "Deleting a workflow in the web UI.")

Another way to delete workflows, as well as projects, tasks and agents, is by
using queries from the GraphQL API, which is available in the web UI or on port
4200 of the server that runs the Prefect platform. Using a simple query, an
attacker can retrieve the ID of the elements they plan to delete and then use
the suitable API functions to delete the elements. Those elements are
`delete_task`, `delete_flow`, `delete_project` or `delete_agent`.

![](images/fig2.png "Obtaining the ID of a task using GraphQL API.")

![](images/fig3.png "Deleting a task which has a specific ID.")

## The Risk of Exposed Prefect Instances

Prefect instances exposed to the internet don't enable authentication by
default, allowing attackers to delete elements in the platform and disrupt the
everyday work of the organization. This type of attack can affect an
organization in several ways:

- Data loss – losing information about workflows, tasks and data from previous
  deployments
- Disruption – workflows are a key utility leveraged by different teams to
  accomplish tasks. Damage in a company's Prefect instance affects all of the
  teams from that company that use the platform and the clients that depend on
  the service(s) provided by the victim company. For example, a company that
  uses workflows to automate its purchasing process could be blocked from
  making transactions until the issue is resolved. Another example is the
  disruption of client onboarding processes powered by automated workflows
- Delays in project deployment – teams and customers relying on workflows to
  accomplish tasks most likely would experience delays since it takes time to
  restore the platform or even set up a new one. For example, a business that
  uses automated workflows to send emails to its customers could experience
  delays. In addition, disruption of a service could tarnish a company's brand
  and reputation resulting in financial loss

## Stay Secure

Misconfigurations are the number one cause of security breaches in the cloud
today. Companies exposing their instances to the internet without enforcing
authentication is always a bad idea, especially if the platform provides a
powerful API that can also be leveraged by attackers. While Prefect is more
secure than workflow platforms like Argo and Airflow, exposed Prefect
deployments are still candidates for disruptive attacks in which attackers can
alter the workflow environment and greatly damage the company that uses the
platform. We advise on adding firewall rules to hide the instance from the
internet and ensure that authentication is configured. If the application does
not offer sufficient authentication mechanisms, consider leveraging tools
provisioned by the cloud provider to implement your own authentication such as
GCP's Identity-Aware Proxy or AWS's Session Manager.
