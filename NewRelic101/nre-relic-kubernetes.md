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
  AND containerName = '<container>' 
  AND clusterName = '<cluster>'
SINCE 1 day ago TIMESERIES
```

This will give you a graph that shows the memory used vs the memory limit given to the pod. If the memory used reaches the memory limit, you can expect OOM issues (not always however). You can then use the pods shown by the graph in the event explorer to get specific about what happened. You can also use the APM page to see if there was a memory leak from the application or if garbage collection is not happening prpoerly, etc...

You can also look at Kubernetes Jobs from here. Since jobs start and stop (shutdown), you might have no insights as to what happened to a job pod even if you have the logs. With events, you can get an idea as to the happening of a single job. However, if you wanted to get statistics about your job, you will need NRQL. For example, let's say you wanted to get the longest running job in the month of May. You would use this query:

```
SELECT max(completedAt - createdAt) AS 'Longest Duration (seconds)'
FROM K8sJobSample
WHERE completedAt IS NOT NULL
  AND clusterName = 'cluster'
  AND namespaceName = 'namespace'
  AND jobName LIKE 'your-job-%'
FACET jobName
SINCE '2025-05-01' UNTIL '2025-06-01'
LIMIT 10
```

This would give a table with the top 10 longest running jobs in May. You can then change the month limit to figure out the trend of your longest running jobs so that you can adjust them either from the application side or the infrastructure side. This isn't remotely close to what NRQL can offer you. To get a full list of metrics available to you, use:

```
SELECT keyset() FROM K8sJobSample SINCE 1 day AGO
```

This will output a JSON list of all the metrics and options that Cron jobs push to New Relic. This same approach can be used for other items such as `K8sContainerSample`, `K8sPodSample`, `K8sNodeSample`, etc...

Addtionally, you can use the New Relic AI to ask questions in plain english and have them transalted to NRQL queries, which will be automatically run to give you a direct answer. Note that if you were to use other generative AI such as chatgpt, you would get mostly correct NRQL queries, but the keysets that they use might be incorrect on occasion. For example, it might give you a query which uses `endTime` instead of `completedAt`, and New Relic doesn't do a great job of letting you know when you are trying to use keys that don't exist. Instead it gives you a blank response and you are left wondering if you don't have data. So if you notice some issue like that, you can use the `SELECT keyset()` to get the keysets first, then feed the result into gen AI so their answers can be more accurate.

## Dashboards

### Step 1: Create a New Dashboard

1.  In the New Relic UI, navigate to the **Dashboards** section from the main menu on the left.
2.  Click the **Create a dashboard** button in the top right corner.
3.  Give your dashboard a descriptive name, like "Production Cluster Health" or "E-commerce App Monitoring".
4.  Confirm the account you want the dashboard associated with and set permissions if needed. Click **Create**.

You will now have a blank canvas to add your charts (called "widgets" in New Relic).

### Step 2: Add Your First Widget (Chart)

You can add widgets using either the user-friendly Chart Builder or by writing a custom NRQL query. NRQL is more powerful and flexible but we will briefly look at using the chart builder.

#### Using the Chart Builder

This method is great for exploring data without knowing NRQL.

1.  On your new dashboard, click the **+ Add widget** button.
2.  Select **Build a chart**.
3.  Under "Data type", choose **Metrics**.
4.  In the "Find a metric..." search box, type in a Kubernetes metric. For example, search for `k8s.pod.cpuCoresUtilization`.
5.  New Relic will automatically generate a basic chart. You can now refine it using the UI:
      * **`View by`**: Choose an aggregator function like `average`, `max`, or `sum`.
      * **`Group by`**: This is very useful. You could group the CPU utilization by `podName` or `namespaceName` to see which pods or namespaces are using the most CPU.
      * **`Filter by`**: Narrow down the data. For example, filter to a specific `clusterName` or `deploymentName`.
6.  Once you are happy with the chart, click **Save** in the bottom right.

Next, let's take a look at using NRQL.

#### Using NRQL

This is the recommended method for creating precise, customized charts.

1.  On your dashboard, click **+ Add widget**.
2.  Select **Add a chart from a query**.
3.  In the query editor, type your NRQL query. The basic structure is `SELECT function(attribute) FROM DataType WHERE condition SINCE time`.
4.  Choose a visualization type (e.g., time series, pie chart, table, billboard).
5.  Click **Run** to preview your chart.
6.  Give the chart a title and click **Save**.

### Step 3: Add Essential Kubernetes Charts with NRQL

Here are some common and highly useful NRQL queries you can use to build a comprehensive Kubernetes dashboard. Just copy and paste them into the NRQL query editor.

-----

#### **Cluster-Wide CPU & Memory Usage**

These "billboard" charts give you a quick, at-a-glance view of your cluster's overall resource consumption.

  * **Total CPU Usage (%)**
    ```nrql
    SELECT average(k8s.cluster.cpuCoresUtilization) AS 'Cluster CPU %' FROM K8sClusterSample
    ```
  * **Total Memory Usage (%)**
    ```nrql
    SELECT average(k8s.cluster.memoryUtilization) AS 'Cluster Memory %' FROM K8sClusterSample
    ```

-----

#### **Node Status**

This chart helps you see the health and readiness of all the nodes (the servers running your pods) in the cluster.

  * **Node Count by Condition** (visualize as a Pie Chart)
    ```nrql
    SELECT count(entityName) FROM K8sNodeSample FACET condition
    ```

-----

#### **Pod Monitoring**

These charts are critical for understanding the health of your applications.

  * **Top 10 Pods by CPU Usage** (visualize as a Table or Bar Chart)
    ```nrql
    SELECT average(cpuCoresUtilization) FROM K8sContainerSample FACET podName SINCE 30 minutes ago LIMIT 10
    ```
  * **Top 10 Pods by Memory Usage** (visualize as a Table or Bar Chart)
    ```nrql
    SELECT average(memoryWorkingSetBytes) / 1024 / 1024 AS 'Memory (MB)' FROM K8sContainerSample FACET podName SINCE 30 minutes ago LIMIT 10
    ```
  * **Pod Restarts by Namespace** (visualize as a Time Series Line Chart)
    ```nrql
    SELECT sum(restartCount) FROM K8sPodSample TIMESERIES FACET namespaceName
    ```

This is also a great place to use the previous NRQL query:

```
SELECT max(memoryUsedBytes), max(memoryLimitBytes)
FROM K8sContainerSample 
WHERE podName = '<pod>' 
  AND containerName = '<container>' 
  AND clusterName = '<cluster>'
SINCE 1 day ago TIMESERIES
```

Without just showing the memory usage, it will also show the usage vs limit. This is also a great place to check the replica count over time:

```
SELECT max(podsAvailable) FROM K8sDeploymentSample WHERE deploymentName IN (FROM Transaction SELECT latest(deploymentName) WHERE appName IN ('<apm-agent>') SINCE 1 hour ago) AND clusterName = '<cluster>' TIMESERIES since 1 hour ago
```

-----

#### **Deployment & Workload Status**

This helps you ensure your deployments are running as expected.

  * **Deployments Not at Desired Pod Count** (visualize as a Table)
    ```nrql
    SELECT deploymentName, podsAvailable, podsDesired FROM K8sDeploymentSample WHERE podsAvailable != podsDesired
    ```


Using these queries you should be able to get a pretty good idea of what goes on inside your Kubernetes cluster.

## Conclusion

This brings us to the end of the section on Kubernetes with New Relic. A few resources that will help you greatly with the New Relic integration are:
