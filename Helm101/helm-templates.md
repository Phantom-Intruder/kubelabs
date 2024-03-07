# Helm templates

In even a small-scale organization, you would have at least a couple of applications that work together inside a Kubernetes cluster. This means you would have a minimum of 5-6 microservices. As your organization grows, you could go on the have 10, then 20, even 50 microservices, at which point a problem arises: the deployment manifests. Handling just one or two is fairly simple, but when it comes to several dozen, updating and adding new manifests can be a real problem. If you have a separate git repository for each microservice, you will likely want to keep each deployment yaml within the repo. If this is a regular organization that follows best practices, you will be required to create pull requests and have them reviewed before you merge to master. This means if you want to do something as simple as change the image pull policy for several microservices, you will have to make the change in each repo, create a pull request, have it reviewed by someone else, and then merge the changes. This is a pretty large number of steps that a Helm template can reduce to just 1.

To start, we will need a sample application. We could use the same charts that we used in the previous section, but instead let's go with a new application altogether: nginx.

This will be our starting point:

```
# nginx-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```
# nginx-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

The above is a rather basic implementaion of an nginx server with 3 replicas, and allows connections on port 80. For starters, let's create a Helm chart from this nginx application.

For starters, let's create the Helm chart. Go into a folder you plan to run this from and type:

```
helm create nginx-chart
```

This will create a chart with the basic needed files. The directory structure should look like this:

```
nginx-chart/
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

By looking at the above structure, you should be able to see where the deployment and service yamls fit in. You will see that there are sample yamls created here. However, you will also notice that these yamls are go templates which have placeholders instead of hardcoded values. We will be converting our existing yamls into this format. But first, update the Chart.yaml file to include relevant metadata for nginx if you require so. Generally, the default Chart.yaml is fine. You can also optionally modify values.yaml. Things such as the number of replicas can be managed here.

Next, we get to the templating part. We will have to convert our existing deployment yaml into a Helm template file. This is what the yaml would look like after it is converted:

```
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx-deployment
  labels:
    app: nginx
spec:
  replicas: {{ .Values.nginx.replicaCount }}
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.nginx.image.repository }}:{{ .Values.nginx.image.tag }}"
        ports:
        - containerPort: {{ .Values.nginx.containerPort }}
```

The first thing to change is the naming convention: In the metadata.name field, {{ .Release.Name }}- has been added to prefix the deployment name. This ensures that each deployment has a unique name when installed via Helm, with .Release.Name representing the release name generated by Helm. The replica count has been replaced with {{ .Values.nginx.replicaCount }}. This allows the user to set the number of replicas in the values.yaml file of the Helm chart. When it comes to the image tag and repository, the hardcoded image name nginx:latest has been replaced with {{ .Values.nginx.image.repository }}:{{ .Values.nginx.image.tag }}. This allows the user to specify the image repository and tag in the values.yaml file. Finally, the container port's hardcoded port 80 has been replaced with {{ .Values.nginx.containerPort }}, allowing the user to specify the container port in the values.yaml file.

These changes make the Helm template more flexible and configurable, allowing you to customize the deployment according to their requirements using the values.yaml file. Now let's take a look at the service yaml and how it would look after it is converted:

```
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: {{ .Values.nginx.servicePort }}
      targetPort: {{ .Values.nginx.containerPort }}
  type: {{ .Values.nginx.serviceType }}

```

Similar to the deployment template, the service name has been replaced with {{ .Release.Name }}- to ensure uniqueness when installed via Helm. For the service port, the hardcoded service port 80 has been changed to {{ .Values.nginx.servicePort }}. This allows you to specify the service port in the values.yaml file. We also replaced the hardcoded target port 80 with {{ .Values.nginx.containerPort }}, allowing you to specify the target port in the values.yaml file. This should match the container port defined in the deployment template. For the service type we replaced the hardcoded service type ClusterIP with {{ .Values.nginx.serviceType }}, allowing users to specify the service type in the values.yaml file. This provides flexibility in choosing the appropriate service type based on the environment or requirements.

Now that we have defined both the deployment and the service in a template format, let's take a look at what the overriding values file would look like:

```
nginx:
  replicaCount: 3
  image:
    repository: nginx
    tag: latest
  containerPort: 80
  servicePort: 80
  serviceType: ClusterIP
```

In this values.yaml file, the replicaCount specifies the number of replicas for the nginx deployment image.repository and image.tag specify the Docker image repository and tag for the nginx container. The containerPort specifies the port on which the nginx container listens and servicePort specifies the port exposed by the nginx service. Finally, the serviceType specifies the type of Kubernetes service to create for nginx. You might want to change this to NodePort or LoadBalancer if you plan to provide external access (or use kubectl port forwarding).

With this structure, users can now install your Helm chart, and they'll be able to customize the number of replicas and the Nginx image tag through the values.yaml file. Let's go ahead and do the install using the below command:

```
helm install my-nginx-release ./my-nginx-chart --values values.yaml
```

Make sure you run the above command in the same directory as the values.yaml. This will create a release called "my-nginx-release" based on the chart overriding the values.yaml in your Kubernetes cluster. You should be able to run and test the Nginx server that comes up as a result. However, you will notice that we have gone out of our way to define templates and overriding files for something that a simple yaml file could have accomplished. There is more code now than before. So what is the advantage?

For starters, you get all the perks that come with using Helm charts. But now you also have a template you can use to generate additional helm releases. For example, if you want to run another Nginx server with different arguments (different number of replicas, a different image version, different port, etc...), you can use this template. This is especially true if you are working in an organization that has multiple services that require different Nginx setups. You could even consider a situation where your organization has 10+ microservices where the pods you spin up for each microservice are largely boilerplate. The only things that would likely change are the names of the microservice and the image that would spin up in the container. In a situation like this, you could easily create a values file that has a handful of lines that override the Helm template.

Let's try this. Create a new values-new.yaml and set the below values:

```
nginx:
  replicaCount: 2
  image:
    repository: nginx
    tag: alpine3.18-perl
  containerPort: 80
  servicePort: 8080
  serviceType: ClusterIP
```

The new yaml has a changed replica count, gets a different image tag, and serves on port 8080 instead of 80. In order to deploy this, you can use the same 

```
helm install my-nginx-release-alpine ./my-nginx-chart --values values-new.yaml
```

The release name and the yaml that gets picked up need to be changed here. In this same way, you could create different values.yaml with different overriding properties and end up with an infinite number of nginx servers, each with different values.

This brings us to the end of the section on the powerful use of Helm templates. Now, let's move on to Chart hooks.

[Next: Chart Hooks](chart-hooks.md)