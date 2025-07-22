# New Relic APM

Setting up the New Relic APM agent is fairly easy. All you need to do is create a newrelic.yml file that contains your NR license key and application name, then reference it in your application startup script. However, it can get fairly complicated if you have a large number of microservices that all need to be configured with APM agents. In this case, we need the newrelic.yml file to be shared among these microservices, with only the differing parts overridden in each application. We will discuss how to set up a system like this in this section.

For starters, we will need a shared filesystem that can be accessed by all the pods available in your cluster. In this example, we will use AWS EFS. This will hold the newrelic.yml file that will be used by the APM agents in all your pods as a starting point. However, we will not be placing the New Relic jar in this shared file system as well, since that might cause issues when starting the application. We will instead bundle it with the application jar in the image. Something like this:

```dockerfile
RUN yum install -y curl
RUN mkdir -p /newrelic
RUN curl -o /newrelic/newrelic.jar https://download.newrelic.com/newrelic/java-agent/newrelic-agent/current/newrelic.jar
```
 
We can also include the APM agent name here. This is the name that will show on the New Relic dashboard for your application. Since it is unlikely your image hosts multiple applications, this is a good place to put it:

```dockerfile
ENV NEW_RELIC_APP_NAME="<you-application-name>"
```

This will set the APM agent name to whatever you specify, and will override any name that is set in the newrelic.yml. It will also package your jar and the New Relic jar together. You will not be setting other things like your license key or other common attributes here, since all that will be hosted in the shared newrelic.yml file.

There are two ways your application can start. You either have the execution instructions in your Dockerfile, or you use `command` at your deployment file level to run your application. We will use the latter example in this case, but it is the same method if you have your execution command baked into your Dockerfile itself. Let's take an example startup command:

```bash
exec java -javaagent:/newrelicjar/newrelic.jar \
 -Dnewrelic.config.license_key=${NEWRELIC_KEY} \
         -Dnewrelic.config.file=/newrelic/newrelic.yml \
 -jar your-app.jar
```

First, the jar is specified. Since we did `RUN curl -o /newrelic/newrelic.jar`, the jar is in the newrelic folder and needs to be referenced as such. We also have the `license_key` added as a secret in Kubernetes, then referenced here. However, you can also choose to have the license key specified in the newrelic.yml that is referenced by all the applications. The `newrelic.yml` itself is also referenced here, but you may have noticed that we never created the newrelic.yml anywhere. So let's do that now.

Since the APM agent yaml is large, we won't have the whole thing here. Instead, view it in the [NewRelic docs](https://docs.newrelic.com/docs/apm/agents/java-agent/configuration/java-agent-config-file-template/). The important fields are `license_key`, `log_level`, `proxy_*` (if you are using one), and anything else that you either want to enable or disable. Things like distributed tracing are very useful, but if you have a different workflow for tracing transactions, you don't need it considering it is rather expensive.