
:projectid: cloud-aws
:page-layout: guide-multipane
:page-duration: 45 minutes
:page-releasedate: 2019-05-29
:page-description: Explore how to deploy microservices to Amazon Elastic Container Service for Kubernetes (EKS) on Amazon Web Services (AWS).
:page-tags: ['Kubernetes', 'Docker', 'Cloud']
:page-permalink: /guides/{projectid}
:page-related-guides: ['kubernetes-intro', 'kubernetes-microprofile-config', 'kubernetes-microprofile-health', 'istio-intro']
:common-includes: https://raw.githubusercontent.com/OpenLiberty/guides-common/prod
:source-highlighter: prettify
:page-seo-title: Deploying Java microservices on Amazon Web Services (AWS) with Kubernetes
:page-seo-description: A getting started tutorial with examples on how to deploy Java microservices to Amazon Elastic Container Service for Kubernetes (EKS) using Amazon Elastic Container Registry (ECR) as a private container registry.
:guide-author: Open Liberty
= Deploying microservices to Amazon Web Services

[.hidden]
NOTE: This repository contains the guide documentation source. To view the guide in published form, view it on the https://openliberty.io/guides/{projectid}.html[Open Liberty website^].

Explore how to deploy microservices to Amazon Elastic Container Service for Kubernetes (EKS) on Amazon Web Services (AWS).

:kube: Kubernetes
:hashtag: #
:win: WINDOWS
:mac: MAC
:linux: LINUX
:system-api: http://[hostname]:31000/system/properties
:inventory-api: http://[hostname]:32000/inventory/systems


// =================================================================================================
// Introduction
// =================================================================================================

== What you'll learn

You will learn how to deploy two microservices in Open Liberty containers to a {kube} cluster on Amazon Elastic Container Service for Kubernetes (EKS). 

Kubernetes is an open-source container orchestrator that automates many tasks that are involved in deploying, managing, and scaling containerized applications. If you would like to learn more about Kubernetes, check out the https://openliberty.io/guides/kubernetes-intro.html[Deploying microservices to Kubernetes^] guide.

There are different cloud-based solutions for running your Kubernetes workloads. Cloud-based infrastructure enables you to focus on developing your microservices without worrying about low-level infrastructure details for deployment. Using a cloud helps you to easily scale and manage your microservices in a high-availability setup.

Amazon Web Services (AWS) offers a managed Kubernetes service called Amazon Elastic Container Service for {kube} (EKS). EKS simplifies the process of running Kubernetes on AWS without needing to install or maintain your Kubernetes control plane. It provides a hosted {kube} cluster where you can deploy your microservices. You will use EKS with Amazon Elastic Container Registry (ECR). Amazon ECR is a private registry that is used to store and distribute your container images. Note, because EKS is not free, there is a small cost that is associated with running this guide. See the official https://aws.amazon.com/eks/pricing/[Amazon EKS pricing^] documentation for more details.

The two microservices you will deploy are called `system` and `inventory`. The `system` microservice returns the JVM system properties of the running container. It also returns the pod’s name in the HTTP header, making replicas easy to distinguish from each other. The `inventory` microservice adds the properties from the `system` microservice to the inventory. This demonstrates how communication can be established between pods inside a cluster.


// =================================================================================================
// Prerequisites
// =================================================================================================

== Additional prerequisites

Before you begin, the following additional tools need to be installed:

* *Docker:* You need containerization software for building containers. Kubernetes supports various container types, but you will use Docker in this guide. For installation instructions, refer to the official https://docs.docker.com/install/[Docker^] documentation.

* *kubectl:* You need the Kubernetes command-line tool `kubectl` to interact with your Kubernetes cluster. See the official https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl[Install and Set Up kubectl^] documentation for information about downloading and setting up `kubectl` on your platform.

* *IAM Authenticator:* To allow IAM authentication for your Amazon EKS cluster, you must install the AWS IAM Authenticator for Kubernetes. Follow the https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html[Installing aws-iam-authenticator^] instructions to install the AWS IAM Authenticator on your platform. 

* *eksctl:* In this guide, you will use the `eksctl` Command Line Interface (CLI) tool for provisioning your EKS cluster. Navigate to the https://github.com/weaveworks/eksctl/releases[eksctl releases page^] and download the latest stable release. Extract the archive and add the directory with the extracted files to your path.

* *AWS CLI:* You will need to use the AWS Command Line Interface (CLI). For this guide, use AWS CLI Version 2, which is intended for use in production environment. All installers for AWS CLI version 2 include and use an embedded copy of Python, independent of any other Python version that you might have installed. Install the AWS CLI by following the instructions in the official https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html[Installing the AWS CLI^] documentation. 

To verify that the AWS CLI is installed correctly, run the following command:

[role=command]
```
aws --version
```



// =================================================================================================
// Getting Started
// =================================================================================================

[role=command]
include::{common-includes}/gitclone.adoc[]

// no "try what you'll build" section in this guide because it would be too long due to all setup the user will have to do.


// =================================================================================================
// Creating a Kubernetes cluster on EKS
// =================================================================================================

== Creating a Kubernetes cluster on EKS

Before you can deploy your microservices, you must create a {kube} cluster.

// =================================================================================================
// Configuring the AWS CLI
// =================================================================================================

=== Configuring the AWS CLI

Before you configure the AWS CLI, you need to create an AWS Identity and Access Management (IAM)user. Navigate to the https://console.aws.amazon.com/iam/home#/users[Identity and Access Management^] users dashboard and create a user through the UI. While creating a user, you must give the user `programmatic access` when selecting the AWS access type. You will also be prompted to add the user to a group. A group allows you to specify permissions for multiple users. If you do not have an existing group, you need to create a new one. Be sure to take note of the `AWS Access Key ID` and `AWS Secret Access Key`. After the AWS CLI is installed, it must be configured by running the AWS configure command. 

You will be prompted for several pieces of information, including an `AWS Access Key ID` and an `AWS Secret Access Key`. These keys are associated with the AWS Identity and Access Management (IAM) user that you created.

[role=command]
```
aws configure
```

Next, you will be prompted to enter a region. This region will be the region of the servers where your requests are sent. Select the region that is closest to you. For a full list of regions, see the https://docs.aws.amazon.com/general/latest/gr/rande.html#eks_region[AWS Regions and Endpoints^].

Finally, enter `json` when you are prompted to enter the output format. 

After you are done filling out this information, the settings are stored in the default profile. Anytime that you run an AWS CLI command without specifying a profile, the default profile is used. 

You can verify your current configuration values by running the following command: 
[role=command]
```
aws configure list
```

// =================================================================================================
// Provisioning a cluster
// =================================================================================================

=== Provisioning a cluster

The `eksctl` CLI tool greatly simplifies the process of creating clusters on EKS. To create your cluster, use the `eksctl create cluster` command:

[role=command]
```
eksctl create cluster --name=guide-cluster --nodes=1 --node-type=t2.small
```

Running this command creates a cluster that is called `guide-cluster` that uses a single `t2.small` Amazon Elastic Compute Cloud (EC2) instance as the worker node. The `t2.small` EC2 instance is not included in the AWS free tier. See the official https://aws.amazon.com/ec2/pricing/on-demand/[Amazon EC2 pricing^] documentation for more details. When the cluster is created, you see an output similar to the following:

[source, role="no_copy"]
```
[✔]  EKS cluster "guide-cluster" in "us-east-2" region is ready
```

After your cluster is ready, EKS connects `kubectl` to the cluster. Verify that you're connected to the cluster by checking the cluster's nodes:

[role=command]
```
kubectl get nodes
```

[source, role="no_copy"]
----
NAME                            STATUS    ROLES     AGE       VERSION
ip.us-east-2.compute.internal   Ready     <none>    7m        v1.11.5
----

// =================================================================================================
// Deploying microservices to Amazon Elastic Container Service for Kubernetes (EKS)
// =================================================================================================

== Deploying microservices to Amazon Elastic Container Service for Kubernetes (EKS)

In this section, you will learn how to deploy two microservices in Open Liberty containers to a {kube} cluster on EKS. You will build and containerize the `system` and `inventory` microservices, push them to a container registry, and then deploy them to your {kube} cluster. 

// =================================================================================================
// Building and containerizing the microservices
// =================================================================================================

=== Building and containerizing the microservices

The first step of deploying to {kube} is to build your microservices and containerize them.

The starting Java project, which you can find in the `start` directory, is a multi-module Maven project. It is made up of the `system` and `inventory` microservices. Each microservice resides in its own directory, `start/system` and `start/inventory`. Both of these directories contain a Dockerfile, which is necessary for building the Docker images. If you're unfamiliar with Dockerfiles, check out the https://openliberty.io/guides/containerize.html[Containerizing microservices^] guide.

To build these microservices, navigate to the `start` directory and run the following command:

[role=command]
```
mvn package
```

Next, run the `docker build` commands to build the container images for your application:
[role='command']
```
docker build -t system:1.0-SNAPSHOT system/.
docker build -t inventory:1.0-SNAPSHOT inventory/.
```

The `-t` flag in the `docker build` command allows the Docker image to be labeled (tagged) in the `name[:tag]` format. The tag for an image describes the specific image version. If the optional `[:tag]` tag is not specified, the `latest` tag is created by default.

During the build, you see various Docker messages that describe what images are being downloaded and built. When the build finishes, run the following command to list all local Docker images:

[role=command]
```
docker images
```

Verify that the `system:1.0-SNAPSHOT` and `inventory:1.0-SNAPSHOT` images are listed among them, for example:

[source, role="no_copy"]
----
REPOSITORY                        TAG
system                            1.0-SNAPSHOT
inventory                         1.0-SNAPSHOT
icr.io/appcafe/open-liberty       kernel-slim-java11-openj9-ubi       
----

If you don't see the `system:1.0-SNAPSHOT` and `inventory:1.0-SNAPSHOT` images, then check the Maven build log for any potential errors.

// =================================================================================================
// Pushing the images to a docker registry
// =================================================================================================

=== Pushing the images to a container registry

Pushing the images to a registry allows the cluster to create pods by using your container images. The registry that you are using is called Amazon Elastic Container Registry (ECR).

First, you must authenticate your Docker client to your ECR registry. Start by running the `get-login` command:
[role=command]
```
aws ecr get-login-password
```

The `get-login` command returns a `[password_string]`; take a note of this `[password_string]`. Next, running the following will return the `[aws_account_id]` needed to authenticate your Docker client. 

[role=command]
```
aws sts get-caller-identity --output text --query "Account"
```

The `[aws_account_id]` is a unique 12-digit ID that is assigned to every AWS account. You will notice this ID in the output from various commands because AWS uses it to differentiate your resources from other accounts. 

Replace the `[password_string]`, `[aws_account_id]` and the `[region]` your account is configured under in the following `docker login` command, that is used to authenticate your Docker client. 

[role=command]
```
docker login -u AWS -p [password_string] https://[aws_account_id].dkr.ecr.[region].amazonaws.com
```

Next, make a repository to store the `system` and `inventory` images:
[role=command]
```
aws ecr create-repository --repository-name awsguide/system
aws ecr create-repository --repository-name awsguide/inventory
```

You will see an output similar to the following:

[source, role="no_copy"]
```
{
    "repository": {
        "registryId": "[aws_account_id]",
        "repositoryName": "awsguide/system",
        "repositoryArn": "arn:aws:ecr:[region]:[aws_account_id]:repository/awsguide/system",
        "createdAt": 1553111916.0,
        "repositoryUri": "[aws_account_id].ecr.[region].amazonaws.com/awsguide/system"
    }
}
```

Take note of the repository URI for both the `system` and `inventory` repositories, as you need them when you tag and push your images.

// Tagging images
Next, you need to tag your container images with the relevant data about your registry:

[role=command]
```
docker tag system:1.0-SNAPSHOT [system-repository-uri]:1.0-SNAPSHOT
docker tag inventory:1.0-SNAPSHOT [inventory-repository-uri]:1.0-SNAPSHOT
```

// Pushing images
Finally, push your images to the registry:

[role=command]
```
docker push [system-repository-uri]:1.0-SNAPSHOT
docker push [inventory-repository-uri]:1.0-SNAPSHOT
```

When you tag and push your images, remember to substitute `[system-repository-uri]` and `[inventory-repository-uri]` with the appropriate URI for the system and inventory repositories. 

// =================================================================================================
// Deploying the microservices
// =================================================================================================

=== Deploying the microservices

Now that your container images are built, deploy them using a Kubernetes resource definition.

A Kubernetes resource definition is a yaml file that contains a description of all your deployments, services, or any other resources that you want to deploy. All resources can also be deleted from the cluster by using the same yaml file that you used to deploy them. The `kubernetes.yaml` resource definition file is provided for you. If you are interested in learning more about the Kubernetes resource definition, check out the https://openliberty.io/guides/kubernetes-intro.html[Deploying microservices to Kubernetes^] guide.

[role="code_command hotspot", subs="quotes"]
----
#Update the `kubernetes.yaml` file in the `start` directory.#
`kubernetes.yaml`
----

kubernetes.yaml
[source, yaml, linenums, role='code_column']
----
include::finish/kubernetes.yaml[]
----
[role="edit_command_text"]
The [hotspot=18 hotspot=39]`image` is the name and tag of the container image that you want to use for the container. Update the system [hotspot=18]`image` and the inventory [hotspot=39]`image` fields to point to your `system` and `inventory` repository URIs.


Run the following commands to deploy the resources as defined in kubernetes.yaml:
[role='command']
```
kubectl apply -f kubernetes.yaml
```

When the apps are deployed, run the following command to check the status of your pods:
[role='command']
```
kubectl get pods
```

If all the pods are healthy and running, you see an output similar to the following:
[source, role="no_copy"]
----
NAME                                    READY     STATUS    RESTARTS   AGE
system-deployment-6bd97d9bf6-4ccds      1/1       Running   0          15s
inventory-deployment-645767664f-nbtd9   1/1       Running   0          15s
----

=== Making requests to the microservices

Before you can make a request to `[hostname]:31000` or `[hostname]:32000`, you must modify the security group to allow incoming traffic through ports `31000` and `32000`. To get the `group-id` of the security group, use the `aws ec2 describe-security-groups` command:
[role=command]
```
aws ec2 describe-security-groups --filters Name=group-name,Values="*eksctl-guide-cluster-cluster-*" --query "SecurityGroups[*].IpPermissions[*].UserIdGroupPairs"
```

You will see an output similar to the following:

[source, role="no_copy"]
```
...
    {
        "Description": "Allow nodes to communicate with each other (all ports)",
        "GroupId": "sg-035c858e1ff9c52f1",
        "UserId": "208872073932"
    },
    {
        "Description": "Allow managed and unmanaged nodes to communicate with each other (all ports)",
        "GroupId": "sg-04a0c50049c10ae54",
        "UserId": "208872073932"
    }
...
```

Copy the value of the `GroupId` which description is "Allow managed and unmanaged nodes to communicate with each other (all ports)". In this example output, the value is `sg-04a0c50049c10ae54`.

Then, add the following rules to the security group to allow incoming traffic through ports `31000` and `32000`. Don't forget to substitute `[security-group-id]` for the `GroupId` in the output of the previous command.

[role=command]
```
aws ec2 authorize-security-group-ingress --protocol tcp --port 31000 --group-id [security-group-id] --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --protocol tcp --port 32000 --group-id [security-group-id] --cidr 0.0.0.0/0
```

After you finish adding the inbound rules to the security group, you might need to wait a few minutes before you try to access the `system` and `inventory` microservices.

Take note of the `EXTERNAL-IP` in the output of the following command. It is the hostname you will later substitute into `[hostname]`: 
[role=command]
```
kubectl get nodes -o wide
```

Then, `curl` or visit the following URLs to access your microservices, substituting the appropriate hostname:

* `{system-api}`
* `{inventory-api}/system-service`

The first URL returns system properties and the name of the pod in an HTTP header called `X-Pod-Name`. To view the header, you can use the `-I` option in the `curl` when you make a request to `{system-api}`. The second URL adds properties from `system-service` to the inventory.

// =================================================================================================
// Testing microservices that are running on AWS EKS
// =================================================================================================

== Testing microservices that are running on AWS EKS

pom.xml
[source, xml, linenums, role='code_column']
----
include::finish/inventory/pom.xml[]
----

A few tests are included for you to test the basic functionality of the microservices. If a test failure occurs, then you might have introduced a bug into the code. To run the tests, wait for all pods to be in the ready state before you proceed further. The default properties defined in the [hotspot]`pom.xml` file are:

[cols="15, 100", options="header"]
|===
| *Property*                                | *Description*
| [hotspot=cluster file=0]`cluster.ip`                         | IP or hostname for your cluster.
| [hotspot=system-service file=0]`system.kube.service`         | Name of the Kubernetes Service wrapping the `system` pods, `system-service` by default.
| [hotspot=system-node-port file=0]`system.node.port`          | The NodePort of the Kubernetes Service `system-service`, 31000 by default.
| [hotspot=inventory-node-port file=0]`inventory.node.port`    | The NodePort of the Kubernetes Service `inventory-service`, 32000 by default.
|===

Use the following command to run the integration tests against your cluster. Substitute `[hostname]` with the appropriate value:

[role=command]
```
mvn failsafe:integration-test -Dcluster.ip=[hostname]
```

If the tests pass, you see an output for each service similar to the following:

[source, role="no_copy"]
----

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running it.io.openliberty.guides.system.SystemEndpointIT
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.673 sec - in it.io.openliberty.guides.system.SystemEndpointIT

Results:

Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
----

[source, role="no_copy"]
----
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running it.io.openliberty.guides.inventory.InventoryEndpointIT
Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.222 sec - in it.io.openliberty.guides.inventory.InventoryEndpointIT

Results:

Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
----

== Deploying new version of system microservice

Optionally, you might want to make changes to your microservice and learn how to redeploy the updated version of your microservice. In this section, you will bump the version of the `system` microservice to `2.0-SNAPSHOT` and redeploy the new version of the microservice.

Use Maven to repackage your microservice:
[role=command]
```
mvn package
```

Next, build the new version of the container image as `2.0-SNAPSHOT`:
[role=command]
```
docker build -t system:2.0-SNAPSHOT system/.
```

Since you built a new image, it must be pushed to the `awsguide/system` repository of your container registry again.

// Tagging images
Tag your container image with the relevant data about your registry:

[role=command]
```
docker tag system:2.0-SNAPSHOT [system-repository-uri]:2.0-SNAPSHOT
```

// Pushing images
Push your image to the registry:

[role=command]
```
docker push [system-repository-uri]:2.0-SNAPSHOT
```

Update the `system-deployment` deployment to use the new container image that you just pushed to the registry:

[role=command]
```
kubectl set image deployment/system-deployment system-container=[system-repository-uri]:2.0-SNAPSHOT
```

Use the following command to find the name of the pod that is running the `system` microservice:

[role=command]
```
kubectl get pods
```

[source, role="no_copy"]
----
NAME                                   READY     STATUS    RESTARTS   AGE
inventory-deployment-6fd959cc4-rf2m2   1/1       Running   0          7m
system-deployment-677b9f5d9c-nqzcf     1/1       Running   0          7m
----

Observe that in this case, the `system` microservice is running in the pod called `system-deployment-677b9f5d9c-nqzcf`. Substitute the name of your pod into the following command to see more details about the pod:

[role=command]
```
kubectl get event --field-selector involvedObject.name=[pod-name]
```

View the events at the bottom of the command's output. Notice that the pod is using the new container image `system:2.0-SNAPSHOT`.

[source, role="no_copy"]
----
LAST SEEN   TYPE     REASON      OBJECT                                   MESSAGE
97s         Normal   Scheduled   pod/system-deployment-56b4b765d4-9jz5l   Successfully assigned default/system-deployment-56b4b765d4-9jz5l to ip-192-168-85-136.us-east-2.compute.internal
97s         Normal   Pulling     pod/system-deployment-56b4b765d4-9jz5l   Pulling image "208872073932.dkr.ecr.us-east-2.amazonaws.com/awsguide/system:2.0-SNAPSHOT"
95s         Normal   Pulled      pod/system-deployment-56b4b765d4-9jz5l   Successfully pulled image "208872073932.dkr.ecr.us-east-2.amazonaws.com/awsguide/system:2.0-SNAPSHOT" in 1.082459294s
95s         Normal   Created     pod/system-deployment-56b4b765d4-9jz5l   Created container system-container
95s         Normal   Started     pod/system-deployment-56b4b765d4-9jz5l   Started container system-container
----

// =================================================================================================
// Tear Down
// =================================================================================================

== Tearing down the environment

It is important to clean up your resources when you are finished with the guide so that you do not incur additional charges for ongoing service.

When you no longer need your deployed microservices, you can delete all {kube} resources by running the `kubectl delete` command:
[role='command']
```
kubectl delete -f kubernetes.yaml
```

Delete the ECR repositories used to store the `system` and `inventory` images:
[role=command]
```
aws ecr delete-repository --repository-name awsguide/system --force
aws ecr delete-repository --repository-name awsguide/inventory --force
```

Remove your EKS cluster:
[role=command]
```
eksctl delete cluster --name guide-cluster
```


// =================================================================================================
// finish
// =================================================================================================

== Great work! You're done!

You just deployed two microservices running in Open Liberty to AWS EKS. You also learned how to use the `kubectl` command to deploy your microservices on a {kube} cluster.

// Multipane
include::{common-includes}/attribution.adoc[subs="attributes"]

// DO NO CREATE ANYMORE SECTIONS AT THIS POINT
// Related guides will be added in automatically here if you included them in ":page-related-guides"
