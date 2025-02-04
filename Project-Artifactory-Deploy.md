#### Deploying and Packaging applications into Kubernetes with Helm

##### A Jfrog Artifactory  set up as a private registry for an organisation's Docker images and Helm charts

In [Jenkins deploy](https://github.com/Johnstx/Deploying-Jenkins-into-an-EKS-cluster-with-HELM.git), HELM was used as a useful tool to deploy a Devops tool to a kubernetes cluster.
Lets explore use of HELM to do same to more Devops tools
This project applies the same principle to deploy Artifactory into an EKS cluster.

This projects creates a realistic experience encountered when deploying applications using HELM such as fixing  deployment issues, tweaking helm values files to automate the application deployment etc.

<!-- *NB: Check [here](https://github.com/Johnstx/Deploying-Jenkins-into-an-EKS-cluster-with-HELM.git) for introduction to the use of helm* -->

**The Core focus** -

1. Artifactory
2. Ingress controllers
3. Cert-manager.

Other DevOps tools that many be deployed with HELM include
Prometheus
Grafana
Elasticsearch ELK using ECK

Artifactory is part of a suit of products from a company called Jfrog. Jfrog started out as an artifact repository where software binaries in different formats are stored. Today, Jfrog has transitioned from an artifact repository to a DevOps Platform that includes CI and CD capabilities. This has been achieved by offering more products in which Jfrog Artifactory is part of. Other offerings include

JFrog Pipelines -  a CI-CD product that works well with its Artifactory repository. Think of this product as an alternative to Jenkins.
JFrog Xray - a security product that can be built-into various steps within a JFrog pipeline. Its job is to scan for security vulnerabilities in the stored artifacts. It is able to scan all dependent code.

### The Objective

**In this project**, the objective is to use Jfrog Artifactory as a private registry for an organisation's Docker images and Helm charts. This requirement will satisfy part of the company's corporate security policies to never download artifacts directly from the public into production systems. We will eventually have a CI pipeline that initially pulls public docker images and helm charts from the internet, store in artifactory and scan the artifacts for security vulnerabilities before deploying into the corporate infrastructure.

**Deploy Jfrog Artifactory into Kubernetes**

First, lets set up the Kubernetes cluster.

1. Create the cluster

```
eksctl create cluster --name staxx-tooling-app --region us-west-1 --nodegroup-name worker --node-type t3.xlarge --nodes 2
```

Then update the kubeconfig file with the new cluster created

```
aws eks update-kubeconfig --name staxx-tooling-app --region us-west-1
```

Create a namespace thst will set your work environment for the group of DevOps apps to be deployed. Lets create a namespace called ```dev```

```
kubectl create ns dev
```

This project will require data creating and attaching volumes as well as data persistence in pods. To achieve this,EBS volume will be used for storage. However, Kubernetes by default does not have connectivity to the EBS, to enable this connectivity, we will install a driver, **EBS-CSI diver.**

The **EBS CSI Driver** (Amazon Elastic Block Store Container Storage Interface Driver) is a crucial component in Kubernetes environments for managing Amazon Elastic Block Store (EBS) volumes. The CSI (Container Storage Interface) is a standard API that allows Kubernetes and other container orchestrators to manage storage systems. With the EBS CSI driver, Kubernetes can provision, attach, and manage EBS volumes directly within a Kubernetes cluster.

**Some Key Features of EBS CSI Driver**

**Dynamic Volume Provisioning**:
Automatically creates EBS volumes when Kubernetes PersistentVolumeClaims (PVCs) are created, without manual intervention.

**Volume Attach/Detach:**
Attaches the volume to the correct EC2 node and makes it available for the pod to use.
Detaches volumes when no longer needed, ensuring efficient resource use.

**Volume Resizing:**
Allows you to increase the size of the EBS volume without disrupting your workload, which is useful when workloads require more space.

**Snapshots**:
Supports taking snapshots of volumes for backup or cloning purposes.
Can be used to create new volumes from snapshots, which is useful for disaster recovery or scaling.

**Volume Cloning:**
Supports creating clones of EBS volumes, enabling fast provisioning of new volumes for development or testing.

**Availability & High Availability:**
Designed to work with EC2 instances across different availability zones, providing fault tolerance by ensuring volumes are attached to the correct nodes in the cluster.

**Installing EBS CSI Driver**

1. List the items in the **kube-system** namespace.

```
kubectl get pods -n kube-system
```

![alt text](<images/3 pods inkubesystem.jpg>)

Observe the pods running on the *kube-dns* namespace. Afer the ebs-csi install, its pods will also run on this namespace.

To Install the **EBS CSI** check this [AWS walkthrough](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html).

2, To see the required platform version, run the following command.

```
aws eks describe-addon-versions --addon-name aws-ebs-csi-driver
```

![EBS add-on versions](<images/4 ebs add on version.jpg>)

Check if cluster has an  [OpenID Connect (OIDC )](https://openid.net/connect/) issuer associated with it.
To use AWS Identity and Access Management (IAM) roles for service accounts, an IAM OIDC provider must exist for your cluster’s OIDC issuer URL.

You can create an IAM OIDC provider for your cluster using eksctl or the AWS Management Console.

2. **Create OIDC provider (eksctl)**

i. Determine the OIDC issuer ID for your cluster.

Retrieve your cluster’s OIDC issuer ID and store it in a variable. Replace *staxx-tooling-app* with your own value.

```
cluster_name=staxx-tooling-app
```

```
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

```
echo $oidc_id
```

![alt text](<images/5 create an oidc provider for the cluster oi issuer.jpg>)

ii. Determine whether an IAM OIDC provider with your cluster’s issuer ID is already in your account.

```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

If output is returned, then you already have an IAM OIDC provider for your cluster and you can skip the next step. If no output is returned, then you must create an IAM OIDC provider for your cluster.

If theres no return, then follow through with below:

Create an IAM OIDC identity provider for your cluster with the following command.

```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --region us-west-1 --approve
```

![alt text](<images/5 create an oidc provider for the cluster oi issuer contd.jpg>)

**Assign IAM roles to Kubernetes service accounts**.

Now we configure a Kubernetes service account to assume an AWS Identity and Access Management (IAM) role. Any Pods that are configured to use the service account can then access any AWS service that the role has permissions to access.

**Create an IAM role and policy and link to the k8s cluster**
STEPS

1. Create an IAM role: while creating an IAM role. An assume policy has to be specified in the role as an arguement, therefore create the assume-role-policy first before creating the IAM role.
2. create an assume role policy, then proceed to number 1.
3. Create a role policy for the IAM role created above.
4. Create the EBS CSI driver add-on

**The Role Policy**
Create a file aws-ebs-csi-driver-trust-policy.json that includes the permissions for the AWS services

```
cat >aws-ebs-csi-driver-trust-policy.json <<EOF
{
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::619071319479:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/$oidc_id"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "oidc.eks.us-west-1.amazonaws.com/id/$oidc_id:aud": "sts.amazonaws.com",
              "oidc.eks.us-west-1.amazonaws.com/id/$oidc_id:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
            }
          }
        }
      ]
}
EOF
```

![alt text](<images/8 assume role policy create.jpg>)

**The IAM Policy**

```
aws iam create-role \
      --role-name AmazonEKS_EBS_CSI_DriverRole \
      --assume-role-policy-document file://"aws-ebs-csi-driver-trust-policy.json"

```

![alt text](<images/9 iam create role.jpg>)

**The Role Policy**

```
aws iam attach-role-policy \
      --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
      --role-name AmazonEKS_EBS_CSI_DriverRole
```

**The EBS CSI add-on**

Run the command below -

```
aws eks create-addon --cluster-name $cluster_name --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::619071319479:role/AmazonEKS_EBS_CSI_DriverRole --region us-west-1
```

![alt text](<images/10 create ebs add-on.jpg>)

Check the kube-system namespace and view the pods added.

```
kubectl get pods -n kube-system
```

![alt text](<images/10. check ns.jpg>)

#### Deploy Tools in Kubernetes

**Artifactory**
The best approach to easily get Artifactory into kubernetes is to use helm.

1. Search for an official helm chart for Artifactory on [Artifact Hub](https://artifacthub.io/)

2. Add the jfrog remote repository on your PC

```
helm repo add jfrog https://charts.jfrog.io
```

3. Create a namespace ```tools``` where all DevOps tools will be deployed.

```
kubectl create ns tools
```

4. Update the helm repo index

```
helm repo update
```

5. Install artifactory

```
helm upgrade --install artifactory jfrog/artifactory --version 107.38.10 -n tools
```

![alt text](<images/11. artifactory helm install.jpg>)

**NB**

* We have used upgrade --install flag here instead of helm install artifactory jfrog/artifactory This is a better practice, especially when developing CI pipelines for helm deployments. It ensures that helm does an upgrade if there is an existing installation. But if there isn't, it does the initial install. With this strategy, the command will never fail. It will be smart enough to determine if an upgrade or fresh installation is required.

* The helm chart version to install is very important to specify. So, the version at the time of writing may be different from what you will see from Artifact Hub. So, replace the version number to the desired. You can see all the versions by clicking on "see all" as shown in the image below.

Check the output from the installation for some next step directives.

#### Getting the Artifactory URL

Lets break down the first Next Step.

1. The artifactory helm chart comes bundled with the Artifactory software, a PostgreSQL database and an Nginx proxy which it uses to configure routes to the different capabilities of Artifactory. Getting the pods after some time, you should see something like the below.

![alt text](<images/13. artifactory pods.jpg>)

2. Each of the deployed application have their respective services. This is how you will be able to reach either of them.

![alt text](<images/14. svc jfrog.jpg>)

3. Notice that, the Nginx Proxy has been configured to use the service type of LoadBalancer. Therefore, to reach Artifactory, we will need to go through the Nginx proxy's service. Which happens to be a load balancer created in the cloud provider. Run the kubectl command to retrieve the Load Balancer URL.

```
kubectl get svc artifactory-artifactory-nginx -n tools
```

4. Copy the URL and paste in the browser

5. The default username is admin
The default password is password

**How the Nginx URL for Artifactory is configured in Kubernetes**

Helm uses the **values.yaml** file to set every single configuration that the chart has the capability to configure. THe best place to get started with an off the shelve chart from artifacthub.io is to get familiar with the **DEFAULT VALUES**

* To work directly with the **values.yaml** file, you can download the file locally by clicking on the download icon.

**Is the Load Balancer Service type the Ideal configuration option to use in the Real World?**

Setting the service type to **Load Balance**r is the easiest way to get started with exposing applications running in kubernetes externally. But provissioning load balancers for each application can become very expensive over time, and more difficult to manage. Especially when tens or even hundreds of applications are deployed.
The best approach is to use [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) instead. But to do that, we will have to deploy an [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

A huge benefit of using the ingress controller is that we will be able to use a single load balancer for different applications we deploy. Therefore, Artifactory and any other tools can reuse the same load balancer. Which reduces cloud cost, and overhead of managing multiple load balancers. more on that later.

For now, we will leave artifactory, move on to the next phase of configuration (Ingress, DNS(Route53) and Cert Manager), and then return to Artifactory to complete the setup so that it can serve as a private docker registry and repository for private helm charts

**Deploying Ingress Controller and managing Ingress Resources**
Lets highlight the Ingress resource first

**An Ingress resource** in Kubernetes is an API object that manages external access to services within a cluster, typically using HTTP and HTTPS. It provides routing rules to expose services based on hostnames, paths, or other configurations.

Below shows how ingress sends traffic to a service.

![alt text](images/20.png)
*image credit:* kubernetes.io
An ingress resource for Artifactory would like like below

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.itentity.xyz"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: staxx-artifactory-artifactory-nginx
            port:
              number: 80
```

* An Ingress needs ```apiVersion```, ```kind```, ```metadata``` and ```spec``` fields
* The name of an Ingress object must be a valid DNS subdomain name
* Ingress frequently uses annotations to configure some options depending on the Ingress controller.
* Different Ingress controllers support different annotations. Therefore it is important to be up to date with the ingress controller's specific documentation to know what annotations are supported.
* It is recommended to always specify the ingress class name with the spec ```ingressClassName: nginx```. This is how the Ingress controller is selected, especially when there are multiple configured ingress controllers in the cluster.
The domain ```itentity.xyz``` should be replaced with your own domain.

**Ingress controller**
If you deploy the yaml configuration specified for the ingress resource without an ingress controller, it will not work. In order for the Ingress resource to work, the cluster must have an ingress controller running.

Unlike other types of controllers which run as part of the kube-controller-manager. Such as the Node Controller, Replica Controller, Deployment Controller, Job Controller, or Cloud Controller. Ingress controllers are not started automatically with the cluster.

Branches
Commits
Tags
Repository graph
Compare revisions
Snippets

Build

Deploy

Operate

Monitor

Analyze
Anthony Akoji
Darey-PBL-Temp
Repository
darey-pbl-temp
Project-25
project25.md
project25.md
AKOJI ANTHONY's avatar
add project 25,26 and 27
AKOJI ANTHONY authored 1 year ago
5bb751f9
project25.md
43.13 KiB
Deploying and Packaging applications into Kubernetes with Helm
In the previous project, you started experiencing helm as a tool used to deploy an application into Kubernetes. You probably also tried installing more tools apart from Jenkins.

In this project, you will experience deploying more DevOps tools, get familiar with some of the real world issues faced during such deployments and how to fix them. You will learn how to tweak helm values files to automate the configuration of the applications you deploy. Finally, once you have most of the DevOps tools deployed, you will experience using them and relate with the DevOps cycle and how they fit into the entire ecosystem.

Our focus will be on the.

Artifactory
Ingress Controllers
Cert-Manager
Then you will attempt to explore these on your own.

Prometheus
Grafana
Elasticsearch ELK using ECK
For the tools that require paid license, don't worry, you will also learn how to get the license for free and have true experience exactly how they are used in the real world.

Lets start first with Artifactory. What is it exactly?

Artifactory is part of a suit of products from a company called Jfrog. Jfrog started out as an artifact repository where software binaries in different formats are stored. Today, Jfrog has transitioned from an artifact repository to a DevOps Platform that includes CI and CD capabilities. This has been achieved by offering more products in which Jfrog Artifactory is part of. Other offerings include

JFrog Pipelines - a CI-CD product that works well with its Artifactory repository. Think of this product as an alternative to Jenkins.
JFrog Xray - a security product that can be built-into various steps within a JFrog pipeline. Its job is to scan for security vulnerabilities in the stored artifacts. It is able to scan all dependent code.
In this project, the requirement is to use Jfrog Artifactory as a private registry for the organisation's Docker images and Helm charts. This requirement will satisfy part of the company's corporate security policies to never download artifacts directly from the public into production systems. We will eventually have a CI pipeline that initially pulls public docker images and helm charts from the internet, store in artifactory and scan the artifacts for security vulnerabilities before deploying into the corporate infrastructure. Any found vulnerabilities will immediately trigger an action to quarantine such artifacts.

Lets get into action and see how all of these work.

Deploy Jfrog Artifactory into Kubernetes
The best approach to easily get Artifactory into kubernetes is to use helm.

Search for an official helm chart for Artifactory on Artifact Hub
 2. Click on **See all results** 3. Use the filter checkbox on the left to limit the return data. As you can see in the image below, "Helm" is selected. In some cases, you might select "Official". Then click on the first option from the result.  4. Review the Artifactory page  5. Click on the install menu on the right to see the installation commands.
<img src="https://darey-io-nonprod-pbl-projects.s3.eu-west-2.amazonaws.com/project25/click-install.png" width="936px" height="550px">
Add the jfrog remote repository on your laptop/computer
helm repo add jfrog <https://charts.jfrog.io>
Create a namespace called tools where all the tools for DevOps will be deployed. (In previous project, you installed Jenkins in the default namespace. You should uninstall Jenkins there and install in the new namespace)
kubectl create ns tools
Update the helm repo index on your laptop/computer
helm repo update
Install artifactory
helm upgrade --install artifactory jfrog/artifactory --version 107.38.10 -n tools
Release "artifactory" does not exist. Installing it now.
NAME: artifactory
LAST DEPLOYED: Sat May 28 09:26:08 2022
NAMESPACE: tools
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Congratulations. You have just deployed JFrog Artifactory!

1. Get the Artifactory URL by running these commands:

   NOTE: It may take a few minutes for the LoadBalancer IP to be available.
         You can watch the status of the service by running 'kubectl get svc --namespace tools -w artifactory-artifactory-nginx'
   export SERVICE_IP=$(kubectl get svc --namespace tools artifactory-artifactory-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo http://$SERVICE_IP/

2. Open Artifactory in your browser
   Default credential for Artifactory:
   user: admin
   password: password
NOTE:

We have used upgrade --install flag here instead of helm install artifactory jfrog/artifactory This is a better practice, especially when developing CI pipelines for helm deployments. It ensures that helm does an upgrade if there is an existing installation. But if there isn't, it does the initial install. With this strategy, the command will never fail. It will be smart enough to determine if an upgrade or fresh installation is required.
The helm chart version to install is very important to specify. So, the version at the time of writing may be different from what you will see from Artifact Hub. So, replace the version number to the desired. You can see all the versions by clicking on "see all" as shown in the image below.  
The output from the installation already gives some Next step directives.

Getting the Artifactory URL
Lets break down the first Next Step.

The artifactory helm chart comes bundled with the Artifactory software, a PostgreSQL database and an Nginx proxy which it uses to configure routes to the different capabilities of Artifactory. Getting the pods after some time, you should see something like the below.

Each of the deployed application have their respective services. This is how you will be able to reach either of them.

Notice that, the Nginx Proxy has been configured to use the service type of LoadBalancer. Therefore, to reach Artifactory, we will need to go through the Nginx proxy's service. Which happens to be a load balancer created in the cloud provider. Run the kubectl command to retrieve the Load Balancer URL.

kubectl get svc artifactory-artifactory-nginx -n tools
Copy the URL and paste in the browser

The default username is admin

The default password is password

How the Nginx URL for Artifactory is configured in Kubernetes
Without clicking further on the Get Started page, lets dig a bit more into Kubernetes and Helm. How did Helm configure the URL in kubernetes?

Helm uses the values.yaml file to set every single configuration that the chart has the capability to configure. THe best place to get started with an off the shelve chart from artifacthub.io is to get familiar with the DEFAULT VALUES

click on the DEFAULT VALUES section on Artifact hub
Here you can search for key and value pairs
For example, when you type nginx in the search bar, it shows all the configured options for the nginx proxy.
selecting nginx.enabled from the list will take you directly to the configuration in the YAML file.
Search for nginx.service and select nginx.service.type
You will see the confired type of Kubernetes service for Nginx. As you can see, it is LoadBalancer by default
To work directly with the values.yaml file, you can download the file locally by clicking on the download icon.
Is the Load Balancer Service type the Ideal configuration option to use in the Real World?
Setting the service type to Load Balancer is the easiest way to get started with exposing applications running in kubernetes externally. But provissioning load balancers for each application can become very expensive over time, and more difficult to manage. Especially when tens or even hundreds of applications are deployed.

The best approach is to use Kubernetes Ingress instead. But to do that, we will have to deploy an Ingress Controller.

A huge benefit of using the ingress controller is that we will be able to use a single load balancer for different applications we deploy. Therefore, Artifactory and any other tools can reuse the same load balancer. Which reduces cloud cost, and overhead of managing multiple load balancers. more on that later.

For now, we will leave artifactory, move on to the next phase of configuration (Ingress, DNS(Route53) and Cert Manager), and then return to Artifactory to complete the setup so that it can serve as a private docker registry and repository for private helm charts.

Deploying Ingress Controller and managing Ingress Resources
Before we discuss what ingress controllers are, it will be important to start off understanding about the Ingress resource.

An ingress is an API object that manages external access to the services in a kubernetes cluster. It is capable to provide load balancing, SSL termination and name-based virtual hosting. In otherwords, Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

Here is a simple example where an Ingress sends all its traffic to one Service:

 *image credit:* kubernetes.io
An ingress resource for Artifactory would like like below

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:

* host: "tooling.artifactory.sandbox.svc.darey.io"
    http:
      paths:
  * path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
An Ingress needs apiVersion, kind, metadata and spec fields
The name of an Ingress object must be a valid DNS subdomain name
Ingress frequently uses annotations to configure some options depending on the Ingress controller.
Different Ingress controllers support different annotations. Therefore it is important to be up to date with the ingress controller's specific documentation to know what annotations are supported.
It is recommended to always specify the ingress class name with the spec ingressClassName: nginx. This is how the Ingress controller is selected, especially when there are multiple configured ingress controllers in the cluster.
The domain darey.io should be replaced with your own domain.
Ingress controller
If you deploy the yaml configuration specified for the ingress resource without an ingress controller, it will not work. In order for the Ingress resource to work, the cluster must have an ingress controller running.

Unlike other types of controllers which run as part of the kube-controller-manager. Such as the Node Controller, Replica Controller, Deployment Controller, Job Controller, or Cloud Controller. Ingress controllers are not started automatically with the cluster.

Kubernetes as a project supports and maintains AWS, GCE, and NGINX ingress controllers.

There are many other 3rd party Ingress controllers that provide similar functionalities with their own unique features, but the 3 mentioned earlier are currently supported and maintained by Kubernetes. Some of these other 3rd party Ingress controllers include but not limited to the following;

* AKS Application Gateway Ingress Controller (Microsoft Azure)
* Istio
* Traefik
* Ambassador
* HA Proxy Ingress
* Kong
* Gloo

[Click here](https://kubevious.io/blog/post/comparing-top-ingress-controllers-for-kubernetes#comparison-matrix) to see the comaparison between some ingress controllers. It is important to understand key features among them so that one can make informed decisions when making a choice for a business need.
It is possible to deploy more than one ingress controller in the same cluster, applying the use of ```ingress class```. By specifying the spec ```ingressClassName``` field on the ingress object, the appropriate ingress controller will be used tby the ingress resource.

**Deploy Nginx Ingress Controller**
We will use the Nginx ingress controller as a choice of ingress controller. It is the default choice in kubernetes projects. It is reliable and easy to use.

For [official guide](https://kubernetes.github.io/ingress-nginx/deploy/) to install nginx ingress controller.

Using the **Helm** approach, according to the official guide;

1. Install Nginx Ingress Controller in the ```ingress-nginx``` namespace

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

OR

```
kubectl create ns ingress-nginx 
```

```
helm repo add ingress-nginx  https://kubernetes.github.io/ingress-nginx
```

```
helm repo update
```

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --version 4.8.3 -n ingress-nginx
```

![alt text](<images/17. nginx controller.jpg>)

**Notice:**
This command is idempotent:

* if the ingress controller is not installed, it will install it,

* if the ingress controller is already installed, it will upgrade it.

2. A few pods should start in the ingress-nginx namespace:

```
kubectl get pods -n ingress-nginx
```

3. Check the load balancer in the  AWS console

![alt text](<images/17. load balancers.jpg>)

```
kubectl get svc -n ingress-nginx
```

![alt text](<images/17. nginx controller.jpg>)

The ``ingress-nginx-controller`` service that was created is of the type ``LoadBalancer``. That will be the load balancer to be used by all applications which require external access, and is using this ingress controller.

Copy the **EXTERNAL-IP** and paste on the browser.

![alt text](<images/19. load balance dns in url.jpg>)

**Deploy Artifactory Ingress**

Now, it is time to configure the ingress so that we can route traffic to the Artifactory internal service, through the ingress controller's load balancer.

![alt text](images/20.png)

Notice the `spec` section with the configuration that selects the ingress controller using the **ingressClassName**.

* To get the ingressClass of the ingress controller, run -

```
kubectl get ingressclass -n ingress-nginx
```

![alt text](<images/19.B get ingressclass for the artifactory ingress resource.jpg>)

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory
spec:
  ingressClassName: nginx
  rules:
  - host: "tooling.artifactory.itentity.xyz"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 80

```

Create the ingress resource in the tools namespace.

```
kubectl apply -f <filename.yaml> -n tools
```

* To ge the details on the created ingress resource, run -

```
kubectl get ingress -n tools
```

![alt text](<images/19.C get ingress in tools (artifactory_ingress).jpg>)

**NB**

* CLASS -  The nginx controller class name ```nginx```
* HOSTS - The hostname to be used in the browser  tooling.artifactory.itentity.xyz
* ADDRESS - The loadbalancer address that was created by the ingress controller

**Configure DNS**
As we noticed, the load balancer address which we pasted on the browser is too long, Ideally, a DNS record that is more human readbale will be created to direct request to the load balancer. Which is what we have conigured here - ```- host: "tooling.artifactory.itentity.xyz"```
Without a DNS configuration, the host address will be unable to reach the load balancer.

The ```itentity.xyz``` part of the domain is the configured HOSTED ZONE in AWS.

If the domain is  purchased directly from AWS, the hosted zone will be automatically configured. But if the domain is registered with a different provider such as **freenon** or **namechaep**, create the hosted zone and update the name servers.

**Create Route53 record**
Within the hosted zone is where all the necessary DNS records will be created. Since we are working on Artifactory, lets create the record to point to the ingress controller's loadbalancer. There are 2 options. You can either use the *CNAME* or *AWS Alias*

**CNAME** Method

1. Select the **HOSTED ZONE** you wish to use, and click on the create record button
2. Add the subdomain `tooling.artifactory`, and select the record type `CNAME`
3. Record created successfully

![alt text](<images/21. CNAME.jpg>)

*After the CNAME set-up, confirm if the domain has propagated sucessfully using **<www.dnschecker.org>***

![alt text](<images/22. dns checker.jpg>)

**AWS Alias Method**

1. In the create record section, type in the record name, and toggle the ```alias``` button to enable an alias. An alias is of the ```A```DNS record type which basically routes directly to the load balancer. In the ```choose endpoint``` bar, select ```Alias to Application and Classic Load Balancer```

2. Select the region and the load balancer required. You will not need to type in the load balancer, as it will already populate.

For detailed read on selecting between CNAME and Alias based records, read the [official documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

**Visiting the application from the browser**
So far, we now have an application running in Kubernetes that is also accessible externally. Use the full URL of the domain to access the artifactory application (tooling.artifactory.itentity.xyz).

![alt text](<images/24. domain url check.jpg>)

The application has been successfullly setup for access, but is not secure yet. So next, we setup the certificates.

**Exploring the Artifactory Web UI**

1. Access the default username and password - Run a helm command to output the same message after the initial install.

```
helm test artifactory -n tools
```

2. Insert the username and password to load the Get Started page
![alt text](<images/24. domain url check.jpg>)

3. Reset the admin password

![alt text](<images/25. jfrog password change.jpg>)

4. Activate the Artifactory License. You will need to purchase a license to use Artifactory enterprise features.

5. For learning purposes, you can apply for a free trial license. [Simply fill the form here](https://jfrog.com/start-free/) and a license key will be delivered to your email in few minutes.

![alt text](<images/26. jfrog key.jpg>)

6. Set the Base URL. Ensure to use ```https```

![alt text](<images/27. jfrog setup.jpg>)

7. Skip the Proxy setting and Skip creation of repositories for now. You will create them yourself later on.

![alt text](<images/28. jfrog setup skip.jpg>)

![alt text](<images/29. create repo skip.jpg>)

8. finish Setup.
![alt text](<images/31. welcome to jfrog.jpg>)

Next, its time to fix the TLS/SSL configuration so that we will have a trusted HTTPS URL

**Deploying Cert-Manager and managing TLS/SSL for Ingress**

Transport Layer Security (TLS), the successor of the now-deprecated Secure Sockets Layer (SSL), is a cryptographic protocol designed to provide communications security over a computer network.
The TLS protocol aims primarily to provide cryptography, including privacy (confidentiality), integrity, and authenticity through the use of certificates, between two or more communicating computer applications.
The certificates required to implement TLS must be issued by a trusted Certificate Authority (CA).
To see the list of trusted root Certification Authorities (CA) and their certificates used by Google Chrome, you need to use the Certificate Manager built inside Google Chrome

**Certificate Management in Kubernetes**
Ensuring that trusted certificates can be requested and issued from certificate authorities dynamically is a tedious process. Managing the certificates per application and keeping track of expiry is also a lot of overhead. Complex scripts or programs have to be created to achieve this. [Cert-Manager](https://cert-manager.io/) offers solution to this complexity.

Cert-Manager is an open-source tool that automates the management and issuance of TLS (Transport Layer Security) certificates within Kubernetes clusters. It is commonly used in Kubernetes environments to handle the lifecycle of certificates, including acquiring, renewing, and managing them for services and applications.

cert-manager adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates.

Similar to how Ingress Controllers are able to enable the creation of *Ingress* resource in the cluster, so also cert-manager enables the possibility to create certificate resource, and a few other resources that makes certificate management seamless.

It can issue certificates from a variety of supported sources, including Let's Encrypt, HashiCorp Vault, and Venafi as well as private PKI. The issued certificates get stored as kubernetes secret which holds both the private key and public certificate.

**Let's Encrypt** with cert-manager will be used in this project. The certificates issued by Let's Encrypt will work with most browsers because the root certificate that validates all it's certificates is called **“ISRG Root X1”** which is already trusted by most browsers and servers.
Cert-maanager will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

**Cert-Manager high Level Architecture**
Cert-manager works by having administrators create a resource in kubernetes called **certificate issuer** which will be configured to work with supported sources of certificates. This issuer can either be scoped **globally** in the cluster or only local to the namespace it is deployed to.
Whenever it is time to create a certificate for a specific host or website address, the process follows the pattern seen in the image below.

**Deploying Cert-manager**

1 . Create a namespace for certificate manager

```
 kubectl create ns cert-manager
```

2. **IMPORTANT:** you MUST install the cert-manager CRDs **before** installing the **cert-manager Helm chart**

```
$ kubectl apply \
    -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.11/deploy/manifests/00-crds.yaml
```

3. Add the Jetstack Helm repository

```
helm repo add jetstack https://charts.jetstack.io
```

4. Install the cert-manager helm chart

```
helm install --name my-release --namespace cert-manager jetstack/cert-manager
```

![alt text](<images/33. CERT MANAGER SETUP.jpg>)

**Certificate Issuer**

In order to begin issuing certificates, you will need to set up a ClusterIssuer or Issuer resource. We will use a **Cluster Issuer** so that it can be scoped globally

1. Create a file **cluster-issuer.yml** and deploy with kubectl.

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  namespace: "cert-manager"
  name: "letsencrypt-prod"
spec:
  acme:
    server: "https://acme-v02.api.letsencrypt.org/directory"
    email: "infradev@oldcowboyshop.com"
    privateKeySecretRef:
      name: "letsencrypt-prod"
    solvers:
    - selector:
        dnsZones:
          - "itentity.xyz"
      dns01:
        route53:
          region: "us-west-1"
          hostedZoneID: "Z09002501VV2SCD0BNE49"
```

![alt text](<images/35. cert-manager ppods.jpg>)

Lets break down the content to undertsand all the sections

* Section 1 - The standard kubernetes section that defines the **apiVersion, Kind,** and **metadata**. The Kind here is a ClusterIssuer which means it is scoped globally.

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
namespace: "cert-manager"
name: "letsencrypt-prod"
```

* Section 2 - In the spec section, an **ACME** - Automated Certificate Management Environment issuer type is specified here. When you create a new ACME Issuer, cert-manager will generate a private key which is used to identify you with the ACME server.
Certificates issued by public ACME servers are typically trusted by client's computers by default. This means that, for example, visiting a website that is backed by an ACME certificate issued for that URL, will be trusted by default by most client's web browsers. ACME certificates are typically free.

Let’s Encrypt uses the ACME protocol to verify that you control a given domain name and to issue you a certificate. You can either use the let's encrypt **Production** server address ```https://acme-v02.api.letsencrypt.org/directory``` which can be used for all production websites. Or it can be replaced with the staging URL ```https://acme-staging-v02.api.letsencrypt.org/directory``` for all **Non-Production** sites.

The ```privateKeySecretRef``` has configuration for the private key name you prefer to use to store the ACME account private key. This can be anything you specify, for example letsencrypt-prod

```
spec:
 acme:
    # The ACME server URL
    server: "https://acme-v02.api.letsencrypt.org/directory"
    email: "infradev@darey.io"
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
    name: "letsencrypt-prod"
```

* section 3 - This section is part of the **spec** that configures **solvers** which determines the domain address that the issued certificate will be registered with. **dns01** is one of the different challenges that cert-manager uses to verify domain ownership. [Read more on DNS01 Challenge here](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge). With the **DNS01** configuration, you will need to specify the Route53 DNS Hosted Zone ID and region. Since we are using EKS in AWS, the IAM permission of the worker nodes will be used to access Route53. Therefore if appropriate permissions is not set for EKS worker nodes, it is possible that certificate challenge with Route53 will fail, hence certificates will not get issued.

The other possible option is the [HTTP01](https://cert-manager.io/docs/configuration/acme/http01/#configuring-the-http01-ingress-solver) challenge, but we won't be using that. Click on the link to read more.

```
    solvers:
    - selector:
        dnsZones:
          - "itentity.xyz"
      dns01:
        route53:
          region: "us-west-1"
          hostedZoneID: "Z09002501VV2SCD0BNE49"
```

*To access the hosted zone ID*

```
aws route53 list-hosted-zones
```

![alt text](<images/34. hosted_zone list.jpg>)

With the ClusterIssuer properlu configured, it is now time to start getting certificates issued.

**Configuring Ingress for TLS**

To ensure that every created ingress also has TLS configured, we will need to update the ingress manifest with TLS specific configurations.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.class: nginx
  name: artifactory
spec:
  rules:
  - host: "tooling.artifactory.itentity.xyz"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: artifactory
            port:
              number: 8082
  tls:
  - hosts:
    - "tooling.artifactory.itentity.xyz"
    secretName: "tooling.artifactory.itentity.xyz"
```

Deploythe updated artifactory-ingress file

```
kubectl apply -f artifactory-nginx.yaml -n tools
```

After updating the ingress above, the artifactory will be inaccessible through the browser.

![alt text](<images/36. after ingress tls update.jpg>)

The most significat updates to the ingress definition is the annotations and tls sections.

**Differences between Annotations and Labels**

**Labels** are used in conjunction with selectors to identify groups of related resources. Because selectors are used to query labels, this operation needs to be efficient. To ensure efficient queries, labels are constrained by RFC 1123. RFC 1123, among other constraints, restricts labels to a maximum 63 character length. Thus, labels should be used when you want Kubernetes to group a set of related resources.

**Annotations** are used for “non-identifying information” i.e., metadata that Kubernetes does not care about. As such, annotation keys and values have no constraints. Thus, if you want to add information for other humans about a given resource, then annotations are a better choice.

The Annotation added to the Ingress resource adds metadata to specify the issuer responsible for requesting certificates. The issuer here will be the same one we have created earlier with the name **letsencrypt-prod.**

```
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

The other section is ```tls``` where the host name that will require ```https``` is specified. The secretName also holds the name of the secret that will be created which will store details of the certificate key-pair. i.e Private key and public certificate.

```
  tls:
  - hosts:
    - "tooling.artifactory.itentity.xyz"
    secretName: "tooling.artifactory.itentity.xyz"
```

Redeploying the newly updated ingress will go through the process as shown below.

 ![alt text](images/ingress_process.png)

 To see the resources created at each stage, run the commands below -

* kubectl get certificaaterequest
* kubectl get order
* kubectl get challenge
* kubectl get certificate

![alt text](<images/37. debug list cert files - resources.jpg>)

Observe closely and find that the result of the queries shows the resources are either on ```pending``` or ```not ready```. So some troubleshooting is Needed!!

**Actions taken when troubleshooting**

1. describe the challenge

```
kubectl describe challange tooling.artifactory.itentity.xyz-1-XXXXXXXX -n tools
```

![alt text](<images/37. debug-challenge.jpg>)

The reason is linked to permission issues: *User, which is the IAM role not authorized to perform route53:ChangeResourceRecordSets on the host name'*

2. Resolve the issue by creating an IAM policy to enable route53:ChangeResourceRecordSets and then add to the IAM role.

*To find existing policies on the IAM role, run* -

```
aws iam list-attached-role-policies --role-name eksctl-staxx-tooling-app-nodegroup-NodeInstanceRole-xxxxxxx
```

![alt text](<images/38. list iam role policies.jpg>)

OR Using the AWS console -

![alt text](<images/38. iam role policies.jpg>)

Observe that the policy related to **route53:ChangeResourceRecordSets** is not listed

3. Create the Policy

```
cat <<EOF > ChangeResourceRecordSets.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement",
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:GetChange"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/Z09002501VV2SCD0BNE49",
                "arn:aws:route53:::change/*"
            ]
        }
    ]
}
EOF
```

4. Apply the updated IAM policy to the IAM role. You can do this through the AWS Management Console or by using the AWS CLI

```
aws iam create-policy --policy-name ChangeResourceRecordSets --policy-document file://ChangeResourceRecordSets.json
```
![alt text](<images/39. iam policy.jpg>)


*Observe the output: The **policy ARN will be used to attach the policy to the IAM role***


5. Attach the policy to the role
```
aws iam attach-role-policy --role-name eksctl-dybran-eks-tooling-nodegrou-NodeInstanceRole-SQcjbXR7eFcD --policy-arn <insert the policy ARN here>
``` 
![alt text](<images/40. attach role policy.jpg>)

List the policies to  find the updated **ChangeResourceRecordSets** policy

![alt text](<images/41. list role policies again.jpg>)

Delete the pod in the cert-manager namespace so it can restart.

Run the requests below again 

* kubectl get certificaaterequest
* kubectl get order
* kubectl get challenge
* kubectl get certificate

like this 
![alt text](<images/42. check the certificaterequests.jpg>)

Notice that all requests are ```ready``` and ```approved```.


If all goes well, running ```kubectl get certificate```, you should see an output like below.

```
NAME                                           READY                            SECRET                          AGE
tooling.artifactory.itentity.xyz       True             tooling.artifactory.itentity.xyz       108s
```

Notice the secret name there in the above output.  Executing the command ```kubectl get secret tooling.artifactory.itentity.xyz -o yaml```, you will see the ```data``` with encoded version of both the private key tls.key and the public certificate ```tls.crt```. This is the actual certificate configuration that the ingress controller will use as part of Nginx configuration to terminate TLS/SSL on the ingress.

![alt text](<images/43. tls cert key for ingress.jpg>)

Check the hostname on the browser, notice the padlock sign has appeared and there are no warnings of untrusted certificates.

![alt text](<images/44. secure url.jpg>)

Finally, delete the LoadBalancer created for artifactory. If you run a get service kubectl command like below;


```
kubectl get svc -n tools
```
![alt text](<images/45. lb reconfigure.jpg>)

You will see that the load balancer is still there.
Update the helm values file for artifactory, and ensure that the ```artifactory-artifactory-nginx``` service uses ```ClusterIP```.

The output should look like this.

![alt text](<images/45i. lb reconfigure.jpg>)

Finally, update the ingress to use `artifactory-artifactory-nginx` as the backend service instead of using `artifactory`. Remember to update the port number as well.
If everything goes well, you will be prompted at login to set the BASE URL. It will pick up the new ```https``` address. Simply click next
You can skip the `proxy` part of the setup.

Skip the `proxy` part of the setup.

Skip repositories creation.
Then complete the setup.
