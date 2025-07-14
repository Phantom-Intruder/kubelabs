# New Relic APM

Setting up the New Relic APM agent is fairly easy. All you need to do is create a newrelic.yml file that contains your NR license key and application name, then reference it in your application startup script. However, it can get fairly complicated if you have a large number of microservices that all need to be configured with APM agents. In this case, we need the newrelic.yml to be shared amongst these microservices with only the differing parts overridden in each application. We will discuss how to do a setup like this in this section.

For starters, we will need a shared filesystem that can be accessed by all the pods avaiable in your cluster. In this example, we will use AWS EFS. This will hold the newrelic.yml file that will be used by the APM agents in all your pods as a starting point. However, we will not be placing the newrelic jar in this shared file system as well since that might cause issues when starting the application. We will instead bundle it with the application jar in the image. Something like this:

```dockerfile
RUN yum install -y curl
RUN mkdir -p /newrelic
RUN curl -o /newrelic/newrelic.jar https://download.newrelic.com/newrelic/java-agent/newrelic-agent/current/newrelic.jar
```
 
We can also include the APM agent name here. This is the name that will show on the New Relic dashboard for your application. Since it is unlikely your image hosts multiple applications, this is a good place to put it:

```dockerfile
ENV NEW_RELIC_APP_NAME="<you-application-name>"
```

This will set the APM agent name to whatever you specfiy, and will override any name that is set in the newrelic.yml.