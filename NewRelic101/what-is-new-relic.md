# New Relic

**New Relic** is an **observability platform** that helps developers, devops, and non-technical management monitor and optimize the performance of their applications and infrastructure. It can give you application level insights by tracking performance metrics for applications (e.g., response time, throughput, error rates) and supports multiple languages like Java, Python, Node.js, .NET, Ruby, PHP, and Go. This is useful for developers and SRE teams that want to quickly get to the root cause of an issue from the applications side. It also provides visibility into servers, containers, cloud providers (like AWS, Azure, GCP), and Kubernetes environments - which is what we will be focusing on. It also provides customizable dashboards for visualizing metrics and logs and helps trace requests as they travel through various services and systems, useful for microservices architectures. The logs themselves are correlated with traces and metrics for faster debugging since you can immediately check the log associated with a particular trace or metric. One final advantage of New Relic is its ability to alert any issue with your instrumented infrastructure to just about any alerting system you may have in place.

So basically this is a single point of information regarding your entire system. It is a supercharged version of Prometheus, where you don't really have to do any setup or maintenance. However, unlike Prometheus, New Relic is a paid service that charges per each GB of data that is sent to their system. Similar to Prometheus' PromQL, New Relic has NRQL which can be used to query every aspect of your system using a query language. In this section, we will go through implementing New Relic on your Kubernetes clusters, best practices, how to keep your costs low, important NRQL queries to use, as well as the Kubernetes events and other areas that are useful when debugging issues on Kubernetes.

### Pricing:

Note that New Relic uses a usage-based pricing model, which includes:

* Free tier with limited data retention and capabilities.
* Paid plans based on the number of users and amount of data ingested (measured in GBs).

## Installation

Setting up the New Relic integration for Kubernetes is very simple. The best way to do this is to go to the add an entity option in the New Relic dashboard and follow the guided install instructions. Make sure to select the Helm chart option. The CLI option is easier but it is not very customizable in the same way a Helm chart can be changed using the values.yaml.