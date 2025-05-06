## Table of contents
1. [Introduction](#introduction)
2. [Goal](#goal)
3. [Setting up our working environment](#first)
4. [Issues with certificate validation](#issues)
5. [DNS synchronization to resolve certificate validation](#syncronization)
6. [Conclusion](#conclusion)

## Introduction <a name="introduction"></a>
Terraform is an open-source Infrastructure as Code (IaC) tool that allows you to define and provision infrastructure using code, hence enabling automation. Terraform is regularly used to manipulate hosted zones (HZ),  which using AWS's jargon are containers for NS records. Records contain information about how you want to route traffic for a specific domain. Terraform can easily manipulate HZs, create new hosted zones, remove an existing zone, or automatically update the information of an existing HZ. Manipulating HZs is necessary for example when dealing with SSL/TLS certificates required to set up services such as CloudFront, which supports HTTPS connections. With Terraform and AWS Certificate Manager (ACM), one can automatically validate certificates when setting up services. 

Unfortunately, when you set up a new hosted zone with Terraform, AWS assigns a random list of Name Servers to the zone. <mark>Name server matching between a domain and a hosted zone is critical, becoming a problem when automatically validating certificates</mark>. The certificate validation will not be approved unless the DNS records in the hosted zone match those in the registered domain.
This can be fixed by hand, using the AWS GUI, by copying the list of NS from the hosted zone (AWS Route53/Hosted Zone/Newhosted zone created) and pasting them to the domain registration (Route53/Registered Domains). However, this solution breaks the automatical workflow that characterizes Terraform, as while the domain is being validated, we need to access the AWS GUI and copy the NSs from the HZ to the Domain.

## Goal <a name="goal"></a>
<div class="alert alert-block alert-info">
Here I descrive an automatic solution that matches the NS between a hosted zone and a domain, hence achiving a smooth, automated Terraform workflow when deploying SSL/TLS certificates.
</div>

## Setting up our working environment <a name="first"></a>

Now I will set up my working environment, cloning the repo, setting up Terraform's backend and the variables accross the project.


First, I will clone the corresponding GITHub repo and update the cloud profile data.

### Cloning the repo
I will start by cloning the repo and updating the AWS profile info

 ```
git clone https://github.com/TorresAWS/ns-syncronization
cd global/providers/
vi cloud.tf     # make sure you update your AWS profile info according to your $HOME/.aws/credentials
```

### Starting Terraform's backend
Now I will start Terraform's backend. I will update the names in the backend to avoid conflict:

```
cd global/tf-state/
vi backendname.tf     # make sure you update the bucket and dynamodb names
bash start.sh    # at this point the backend is setup
```

### Terraform variables
Now that the backend is started, I will define all Terraform variables:

```
cd global/variables/
vi domain-var.tf     # make sure you update your domain name
bash start.sh    # at this point the backend is setup
```

## Issues with certificate validation <a name="issues"></a>

Now, I will demonstrate how the problem arises. I will create a hosted zone without syncing the DNS between the HZ and the domain. Make sure your registered domain is in the same account as the newly created hosted zone. Now, we will execute Terraform to create the new hosted zone. 

<h5 a><strong><code>cd vpcs/zone</code></strong></h5>

```
vi route53-domains_registered_domain.tf # make sure this file is commented
bash start  # at this point the new hosted zone is created
```

At this point, if you enter AWS GUI and go to Route53/hosted zones you should be able to see the newly created zone. After the zone is created I will attempt to validate a certificate using ACM, the AWS service in charge of validating certificates. Note that in the context of Amazon Route 53 is just a container of DNS records. In that container, you could create for example, and A (IPv4 address) or AAAA (IPv6 address) record, a  CNAME (canonical name, an alias), or MX (mail exchange). An much more. To validate the certificate I will do:
 
<h5 a><strong><code>cd vpcs/certs</code></strong></h5>

```
bash start.sh  # the validation will never be completed
```

You should see that the certificate is never validated, taking more than hours. 
![My image](../../img/sync-ns-img1.png)
{:.image-caption}
*Screenshot of the certificate deployment before DNS synchronization*

The reson for this problem is that AWS assigns a random set of DNS to newly created Hosted Zones, not necesarely corresponding to those listed under the domain configuration.


If you take a look at the DNS certificates listed on the HZ
![My image](../../img/sync-ns-img3.png)
{:.image-caption}
*DNS listed on the Hosted Zone*

and compare those with the DNS certificates listed on the domain, you will see they dont match.
![My image](../../img/sync-ns-img4.png)
{:.image-caption}
*DNS listed on the Domain*


## DNS syncronization to resolve certificate validation <a name="syncronization"></a>
Now we will destroy all infrastructure created and synchronize the Name Servers between a registered domain and a hosted:

```
cd vpcs/certs ; terraform destroy --auto-approve
cd vpcs/zone ; terraform destroy --auto-approve ;
cd vpcs/zone ; vi route53-domains_registered_domain.tf # make sure this file is UNcommented
bash start.sh
cd vpcs/certs ; bash start.sh  
```

![My image](../../img/sync-ns-img2.png)
{:.image-caption}
*Screenshot of the certificate deployment after DNS synchronization*

## Conclusion <a name="conclusion"></a>
<div class="alert alert-block alert-info">
Here I have shown how synchronizing the DNS between your hosted zone and domain can speed up certificate approval. With the help of Terraform, an automatic solution to this name server matching problem was developed, helping our workflow when deploying infrastructure such as Cloudfront distributions.
</div>

