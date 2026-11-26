# New Attacks on Kubernetes via Misconfigured Argo Workflows

> Misconfigured Argo Workflows instances let attackers run unauthorized code on Kubernetes clusters. We found operators already using them to drop cryptominers.

- Source: https://research.intezer.com/blog/2021/07/new-attacks-on-kubernetes-via-misconfigured-argo-workflows/
- Published: 2021-07-20
- Authors: Ryan Robinson
- Tags: cloud_security, vulnerability
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


## Key Points

- Intezer has detected a new attack vector against Kubernetes (K8s) clusters
  via misconfigured Argo Workflows instances.
- Attackers are already taking advantage of this vector as we detected
  operators dropping cryptominers using this method in the wild.
- We have identified infected nodes and there is the potential for larger scale
  attacks due to hundreds of misconfigured deployments. We have detected
  exposed instances of Argo Workflows that belong to companies from different
  sectors including technology, finance and logistics.
- [Argo Workflows](https://github.com/argoproj/argo-workflows) is an
  open-source, container-native workflow engine designed to run on K8s
  clusters.
- Argo Workflows instances with misconfigured permissions allow threat actors
  to run unauthorized code on the victim's environment.

## Overview

Kubernetes (K8s) is revolutionizing the cloud computing environment, having
become the most popular platform for container orchestration. It is also one of
the most popular repositories on
[GitHub](https://github.com/kubernetes/kubernetes), with over 100,000 commits
and over 3,000 contributors. Each year there is a steady increase in
enterprises using Kubernetes and the number of clusters they deploy. According
to the Cloud Native Computing Foundation (CNCF)
[2020 survey](https://www.cncf.io/wp-content/uploads/2020/11/CNCF_Survey_Report_2020.pdf),
91% of respondents are using Kubernetes, compared to 78% in 2019 and 58% in
2018\. In the same survey, it was reported that among the top challenges of
using and deploying containers were complexity, security and lack of training.

With these challenges that enterprises face using containers and K8s clusters,
there has never been a greater opportunity for attackers to exploit weaknesses
in security. Applications such as "Argo Workflows" orchestrate parallel jobs in
Kubernetes by providing a friendly user interface and the ability to run CI/CD
pipelines without having to configure "complex software development products."
The application is [marketed](https://argoproj.github.io/argo-workflows/) to
"Easily run compute intensive jobs for machine learning or data processing in a
fraction of the time using Argo Workflows on Kubernetes." Even with products
like Argo helping to reduce the complexity of deployment, there is still always
the possibility of misconfiguration or exploitation.

Recently, while studying the impact of exposed Argo Workflows instances, we
discovered a number of unprotected instances, operated by companies in several
industries including technology, finance and logistics. Exposed instances can
contain sensitive information such as code, credentials and private container
image names. We also discovered that in many instances, permissions are
configured which allow any visiting user to deploy workflows. We also detected
that some misconfigured nodes are being targeted by threat actors. An example
of an attack found in the wild, launching a popular cryptocurrency miner
container, is described later in this blog.

## Argo Overview

The core resource in Argo is the "Workflow." The workflow is defined using a
YAML file containing a `spec` for the type of work to be performed. Most
commonly, each step in an Argo workflow is a container. An example "Hello
World" workflow is shown below:

![](images/fig1.png)

These workflows are executed from a template or submitted directly through the
Argo user interface, as shown below:

![](images/fig2.png)

## How Argo is Abused by Attackers

In instances when permissions are misconfigured, it is possible for an attacker
to access an open Argo dashboard and submit their own workflow. In one cluster,
we noticed that a popular cryptocurrency mining container,
`kannix/monero-miner`, was being deployed. An example of an in the wild attack,
still running on an exposed cluster for nine months, is shown below.

![](images/fig3.png)

![](images/fig4.png)

The `kannix/monero-miner` (now removed from Docker Hub) was a popular image
which used XMRig to mine for Monero cryptocurrency. Its ease of use allowed it
to be conveniently used by threat actors of any skill level to conduct
cryptojacking; since all that was required was to change the address of who the
mined cryptocurrency would be deposited to. This same container was mentioned
in a blog post by
[Azure Security Center](https://azure.microsoft.com/en-us/blog/detect-largescale-cryptocurrency-mining-attack-against-kubernetes-clusters/),
which stated that this container was used in a large-scale cryptocurrency
mining attack against Kubernetes clusters.

In Docker Hub there are still a number of options for Monero mining that
attackers can use. With a simple search it shows that there are at least 45
other containers with millions of downloads.

![](images/fig5.png)

Using another popular container, `giansalex/monero-miner`, we detected an XMRig
Miner running on the cluster via Argo Workflows.

![](images/fig6.png)

## Mitigation Advice

For a quick check that an instance is misconfigured, try accessing the Argo
Workflows dashboard from an unauthenticated incognito browser outside your
corporate environment. Another option is to query the API of your instance and
check the status code. Make a HTTP GET request to
`[your.instance:port]/api/v1/info`. A returned HTTP status code of
`401 Unauthorized` while being an unauthenticated user will indicate a
correctly configured instance, whereas a successful status code of
`200 Success` could indicate that an unauthorized user is able to access the
instance.

![](images/fig7.png)

It is critical to ensure that best practices for permissions are followed in
order to prevent unauthorized activity in your environments. Methodologies such
as the
[principle of least privilege](https://www.cyberark.com/what-is/least-privilege/)
(PoLP) should be followed and always refer to the
[application documentation](https://argoproj.github.io/argo-workflows/security/)
for best practices on security.

Even if your cluster is deployed on a
[managed cloud Kubernetes service](https://intezer.com/blog/container-security/do-you-really-need-kubernetes/)
such as Amazon Web Services (AWS), EKS or Azure Kubernetes Service (AKS), the
shared responsibility model still states that the cloud customer, not the cloud
provider, is responsible for taking care of all necessary security
configurations for the applications they deploy.

If you suspect that your Argo instance has been misconfigured and exposed to
the internet with excessive permissions, check for any suspicious activity in
the logs and in the workflow timeline. Make sure that there are no workflows
that have been running for an excessive amount of time. This might be an
indicator of a cryptominer running on your cluster. You can also check out our
article on
[best practices for securing your Kubernetes environment](https://intezer.com/blog/cloud-security/best-practices-for-securing-a-kubernetes-environment/).

## Runtime Protection for Kubernetes Security

It is impossible to completely remove all vulnerabilities or misconfigurations
in cloud applications deployed by any entity. New attack vectors,
vulnerabilities and misconfigurations are being detected and exploited all the
time. Therefore, it is important to have a safety net that can always ensure
that only trusted code is running in your environment. In the event that an
attacker has exploited a misconfiguration or vulnerability in your K8s cluster
or applications running on clusters, runtime protection solutions give
immediate visibility over all code running on your pods, nodes, and clusters,
and help you detect and respond to attacks in your environment.
