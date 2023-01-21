# Terraform

Imagine you had to introduce a DevOps process to a project you were working on. You would likely provision a VM on which you would set up the rest of your DevOps infrastructure. If you were working in a small organization, you would likely use a cloud service provider to save costs, and if you were working in a large organization, you would have the choice between using a cloud provider or simply using VMs that are on-premises. Whatever the case may be, you would be able to reach your goal and finish setting up the infrastructure.

Six months later, you release your application and you need to get your existing infrastructure working for the next release. However, you still need to provide support on your old release, meaning that you would want to provision another VM and get the infrastructure ready for it. This same process will have to be repeated every time you release your application while supporting previous releases, meaning that you would be wasting time repeating the same set of steps manually.

This is where infrastructure as code would come into play. Go back to the very first time you created your infrastructure. Now, instead of creating it manually, imagine that you wrote declarative code that specified which resources should be created, where, and how it should happen. Then the next time you wanted to get all that infrastructure up and running, you would only have to rerun the declarative code and have all infrastructure set up. That is essentially what Terraform is.

Terraform allows you to specify declarative code in files that can be applied to create all your necessary resources in one go. You simply specify which state you would like your infrastructure to be in, and Terraform creates, deletes, or updates the infrastructure to match that state. This largely resembles how Kubernetes works, where you specify what your cluster should be like, and Kubernetes handles the rest.

While you can use a configuration file to automate resource creation on your on-prem VMs, the largest use case for Terraform is when it gets used to set up infrastructure on the cloud. In this case, we will be using GKE with terraform to automate setting up Kubernetes clusters on Google cloud.

## Terraform providers

Now comes another question: how does Terraform connect to cloud services to set up infrastructure on them? For this, they use something called Terraform providers. Terraform has over a hundred providers that allow the automated creation of over a thousand resource types. For example, the [Google provider](https://registry.terraform.io/providers/hashicorp/google/4.47.0) gives management capabilities to many of the cloud services provided by Google. To use it.

## How does it work?

First, the Terraform core takes the existing state of your infrastructure, as well as any configurations from the input file. An input file would look something like this:

```
provider "google" {}

resource google_compute_network "mynetwork" {
name = [google_compute_network]
    # RESOURCE properties go here
}
```

In this case, we are setting the provider to `google`, meaning that this Terraform file can now automate infrastructure across Google cloud services. After that, we declare a network resource that needs to be created when the file is run. This is the file that is fed into Terraform.

Terraform then considers what the desired state should look like and creates an execution plan. Finally, it executes the plan. Before the plan is executed, you get a detailed list of all the changes that would happen to your infrastructure so that you can either confirm or deny that the changes need to be applied. For example

```
Plan: 4 to add, 1 to change, 0 to destroy.
```

For the plan to be executed, it would have to use providers that allow access to various resources.

## Lab

Before we start, you need to have Terraform installed. Use the [guide here](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) for that. Alternatively, since we are going to be using Terraform with GKE, you could use the Google cloud shell. The shell comes with Terraform (as well as many other tools) pre-installed so there is no need to spend time setting things up. The shell also has an online IDE that you can use if you are not comfortable with CLI editors such as Vim.

First, let's plan out what we are going to build. We will create a simple cluster with 2 worker nodes. The worker nodes (which are essentially VMs) will be of type n1-standard1 while the master node will be managed by GKE. After that, we will configure cluster access and generate a kubeconfig. All this will be contained in a VPC so that it doesn't intrude on the other cloud services you may have running on your account, so a VPC will also have to be set up using Terraform.

The first step is to make a directory that will hold all the .tf files used to declare the infrastructure. If you are using the Google cloud shell, you can use the "Open editor" icon in the top right corner to get into the online IDE. Now we can get started. If you have created a GKE cluster before, you already know that it is mandatory to specify a region in which the cluster gets created. So as a first step, create a file called `terraform.tfvars` and copy these two lines into it:

```
project_id = "<your-project-id>"
region     = "us-central1"
```

To get the project id, go to the [cloud console](https://console.cloud.google.com/apis/dashboard) and select "Manage all projects" to list out all the project's names and IDs. Or you can use this command in the cloud shell:

```
gcloud config get-value project
```

.tfvars files hold variables for Terraform. While there are multiple ways to pass variables into Terraform, .tfvars files are the simplest. They are especially useful if you have large configuration files that are difficult to read and you have several properties that change often, where you can specify those variables in .tfvars files and change them instead. It is also useful if multiple .tf files use a specific variable in which case that variable can be defined in a .tfvars file. If the value needs updating, you now only need to change one file.

The next file you will be creating is the `versions.tf` file. This file holds the definitions of the required versions for the provider as well as Terraform. Place the contents of [this file](https://github.com/hashicorp/learn-terraform-provision-gke-cluster/blob/main/versions.tf) into the `versions.tf` you created.

Now let's create our first resource file: the VPC. Name the file `vpc.tf` and paste the contents of [this file](https://github.com/hashicorp/learn-terraform-provision-gke-cluster/blob/main/vpc.tf). Let's take a closer look at this file. You will notice that the variables you declared in the .tfvars file are referenced here at the very top. After the variable references, you have the provider set as "google" where the vars are used.

Once the basics have been defined, the actual resources get listed. The definition starts with the `resource` keyword followed by the name of the resource within quotes, which is then followed by the resource type. So in this case:

```
resource "google_compute_network" "vpc"
```

The resource name is "google_compute_network" and the resource is a "vpc". The resource definition follows where the specifics of the resource are defined within the braces. In the same way, a subnet is also defined.

Finally, we get to the actual GKE cluster definition. For this, create a file called `gke.tf` and paste the contents of [this file](https://github.com/hashicorp/learn-terraform-provision-gke-cluster/blob/main/gke.tf). If we look at the configuration file from the top we can see that it is largely similar to the vpc configuration file. To start with, the GKE credentials are defined. For the moment, leave them as they are. Next, the number of worker nodes is defined which is 2 in this case. Afterward, the primary resources are declared. First is the cluster itself with the cluster name, region, and other information defined.

Since the worker nodes of the cluster are going to be made up of compute instances, we need to specify the information about these resources such as the machine type and region. This is done in the node pool configuration that comes below the cluster configuration.

Now that all the configuration files are in place, it's time to start running Terraform. For starters, run:

```
terraform init
```

Remember that you specified the provider as "google". Now that you used the init command, the provider will get downloaded. Then the values you provided in the .tfvars file will be used to initialize the provider. You should be able to see a bunch of logs output in the CLI. Once Terraform has been initialized, it's time to apply the changes. If you have been using your GCP account to create clusters in the past, then you can move ahead to the next step. If this is the first time you are doing this with GCP, you need to go into the GCP console and enable the Kubernetes and Compute engine APIs.