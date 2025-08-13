## Using New Relic with Kubernetes

Now that you have your entire cluster instrumented on Kubernetes, let's look at how you can use New Relic to its best. First, head over to the New Relic dashboard, then select your Kubernetes cluster. This will immediately give you an overview of your entire cluster, broken down to a namespace level. You can see all the pods running in each namespace, any issues they have, etc... You can use the filters at the top to drill down to specific pods and if you scroll down the page, you will see all recent events that happened in your cluster. If your cluster is small, this page alone should help you with your entire monitoring stack. However, if your cluster is larger (as it usually is), you will need to get more specialized insights.

## Kubernetes events

To start, go to the Kubernetes Events section. This is argulably the most important page in New Relic with regards to Kubernetes. Since all events are logged here, any issue you face in your cluster will also be logged here. If your pods/nodes/containers go out of memory, disk, or anything else, this page will log it. By disabling the "Warnings only" check box, you can get events for everything. You will know exactly when each container in each pod started, how long it ran, what problems it faced, and so on down to a granular level. This is going to be a major help when figuring out what went wrong with something.

The next most powerful debugging tool is using NRQL queries. Let's look at a few common ones:

## NRQL queries

To get further events and information from NRQL, you can use this query:

```
SELECT event.involvedObject.name, event.message FROM InfrastructureEvent WHERE `event.involvedObject.name` = '<pod-name>' AND (clusterName = 'prod' OR `k8s.cluster.name` = 'prod') SINCE 24 HOURS AGO 
```

Note that you can change the object to be a node name or container name and this query will still work. For example:

```
SELECT event.involvedObject.name, event.message FROM InfrastructureEvent WHERE `event.involvedObject.name` = 'ip-192-168-162-138.ec2.internal' AND (clusterName = 'prod' OR `k8s.cluster.name` = 'prod') SINCE 24 HOURS AGO 
```

would work as well. Another thing you may experience with running Kubernetes pods is that there might be memory issues that arise from time to time. To observe this, you can use the below query:


```
SELECT max(memoryUsedBytes), max(memoryLimitBytes)
FROM K8sContainerSample 
WHERE podName = '<pod>' 
  AND containerName = 'inc-prod-menu-service' 
  AND clusterName = 'inc-core-prod-primary'
SINCE 1 day ago TIMESERIES
```

This will give you a graph that shows the memory used vs the memory limit given to the pod. If the memory used reaches the memory limit, you can expect OOM issues (not always however). You can then use the pods shown by the graph in the event explorer to get specific about what happened. You can also use the APM page to see if there was a memory leak from the application or if garbage collection is not happening prpoerly, etc...