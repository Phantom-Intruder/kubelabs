# New Relic

**New Relic** is an **observability platform** that helps developers, devops, and non-technical management monitor and optimize the performance of their applications and infrastructure. It can give you application level insights by tracking performance metrics for applications (e.g., response time, throughput, error rates) and supports multiple languages like Java, Python, Node.js, .NET, Ruby, PHP, and Go. This is useful for developers and SRE teams that want to quickly get to the root cause of an issue from the applications side. It also provides visibility into servers, containers, cloud providers (like AWS, Azure, GCP), and Kubernetes environments - which is what we will be focusing on. It also provides customizable dashboards for visualizing metrics and logs and helps trace requests as they travel through various services and systems, useful for microservices architectures. The logs themselves are correlated with traces and metrics for faster debugging since you can immediately check the log associated with a particular trace or metric. One final advantage of New Relic is its ability to alert any issue with your instrumented infrastructure to just about any alerting system you may have in place.

So basically this is a single point of information regarding your entire system. It is a supercharged version of Prometheus, where you don't really have to do any setup or maintenance. However, unlike Prometheus, New Relic is a paid service that charges per each GB of data that is sent to their system. Similar to Prometheus' PromQL, New Relic has NRQL which can be used to query every aspect of your system using a query language. In this section, we will go through implementing New Relic on your Kubernetes clusters, best practices, how to keep your costs low, important NRQL queries to use, as well as the Kubernetes events and other areas that are useful when debugging issues on Kubernetes.

### Pricing:

Note that New Relic uses a usage-based pricing model, which includes:

* Free tier with limited data retention and capabilities.
* Paid plans based on the number of users and amount of data ingested (measured in GBs).

## Installation

Setting up the New Relic integration for Kubernetes is very simple. The best way to do this is to go to the add an entity option in the New Relic dashboard and follow the guided install instructions. Make sure to select the Helm chart option. The CLI option is easier but it is not very customizable in the same way a Helm chart can be changed using the values.yaml. Once you enter your license key and fill in the necessary information, a Helm command will be given to you that you can directly use in the CLI to install New Relic onto your cluster. However, it is best to take the values.yaml that are also provided alongside the command and fine-tune options manually before you install the chart. This is because the guided install only provides very basic options when it comes to installation whereas the full chart has far more options.

For a full reference of the options that the chart has, check the [official chart GitHub page](https://github.com/newrelic/helm-charts/tree/master/charts/nri-bundle). We will discuss a few important options that are definitely worth looking into.

### Low data mode

By default, New Relic scrapes cluster metrics and other information every 15 seconds. By setting `lowDataMode: true` you can increase this scrape interval to 30 seconds which will reduce the amount of data being sent to New Relic without really affecting anything.

### Disable unwanted items

Depending on your business needs, its important to only enable the aspects of New Relic that you will actually be using. For example, if you don't need Prometheus OpenMetrics, you can disable it 

```yaml
nri-prometheus:
  enabled: false
```

newrelic-logging:
  # newrelic-logging.enabled -- Install the [`newrelic-logging` chart](https://github.com/newrelic/helm-charts/tree/master/charts/newrelic-logging)
  enabled: false

newrelic-pixie:
  # newrelic-pixie.enabled -- Install the [`newrelic-pixie`](https://github.com/newrelic/helm-charts/tree/master/charts/newrelic-pixie)
  enabled: false

pixie-chart:
  # pixie-chart.enabled -- Install the [`pixie-chart` chart](https://docs.pixielabs.ai/installing-pixie/install-schemes/helm/#3.-deploy)
  enabled: false

newrelic-infra-operator:
  # newrelic-infra-operator.enabled -- Install the [`newrelic-infra-operator` chart](https://github.com/newrelic/newrelic-infra-operator/tree/main/charts/newrelic-infra-operator) (Beta)
  enabled: false

newrelic-prometheus-agent:
  # newrelic-prometheus-agent.enabled -- Install the [`newrelic-prometheus-agent` chart](https://github.com/newrelic/newrelic-prometheus-configurator/tree/main/charts/newrelic-prometheus-agent)
  enabled: false

newrelic-eapm-agent:
  # newrelic-eapm-agent.enabled -- Install the [`nr-eapm-agent`](https://github.com/newrelic/helm-charts/tree/master/charts/nr-ebpf-agent)
  enabled: false

k8s-agents-operator:
  # k8s-agents-operator.enabled -- Install the [`k8s-agents-operator` chart](https://github.com/newrelic/k8s-agents-operator/tree/main/charts/k8s-agents-operator)
  enabled: false

newrelic-k8s-metrics-adapter:
  # newrelic-k8s-metrics-adapter.enabled -- Install the [`newrelic-k8s-metrics-adapter.` chart](https://github.com/newrelic/newrelic-k8s-metrics-adapter/tree/main/charts/newrelic-k8s-metrics-adapter) (Beta)
  enabled: false

### Using a proxy

If you are sending terabytes of data to New Relic (which is generally the case), and your cluster is inside a private VPC, then all that data will be going out of the NAT gateway. You can avoid this by using a proxy in a public subnet that will send the data via the internet gateway instead of the NAT gateway. Since data going through NAT gateway also goes through internet gateway anyways, the cost should go down. We will look at how we an achieve this with squid proxy in the next section.

```bash
helm repo update ; helm upgrade --install newrelic-bundle newrelic/nri-bundle -n newrelic --values values.yaml
```