---
title: Configuring AWS for Cloud Foundry
---

This topic describes how to configure Amazon Web Services (AWS) for Cloud Foundry.

## <a id="create-cf-manifest"></a>Step 1: Create a Deployment Manifest

The Cloud Foundry deployment manifest is a YAML file that defines the deployment components and properties. The [*cloudfoundry/cf-release*](https://github.com/cloudfoundry/cf-release) repo
contains template deployment manifests that you can edit with your deployment information. The `minimal-aws.yml` file contains the minimum information necessary to deploy Cloud Foundry to AWS.

Create a manifest for your deployment as follows:

1. Create a deployment directory to store your manifest.

    <pre class='terminal'>
    $ mkdir ~/CF-deployment
    </pre>

1. Clone the *cloudfoundry/cf-release* repo.

    <pre class='terminal'>
    $ git clone https://github.com/cloudfoundry/cf-release.git
    </pre>

1. Navigate to the *example_manifests* subdirectory to retrieve the `minimal-aws.yml` template. Copy and paste the template into a text editor and save the edited manifest to your deployment directory.

    In the template, you must replace the following properties:
    * `REPLACE_WITH_DIRECTOR_ID`
    * `REPLACE_WITH_PRIVATE_SUBNET_ID`
    * `REPLACE_WITH_PUBLIC_SUBNET_ID`
    * `REPLACE_WITH_ELASTIC_IP`
    * `REPLACE_WITH_PUBLIC_SECURITY_GROUP`
    * `REPLACE_WITH_SYSTEM_DOMAIN`
    * `REPLACE_WITH_SSL_CERT_AND_KEY`
    

1. Run `bosh status --uuid` to retrieve your BOSH Director ID. Update `REPLACE_WITH_DIRECTOR_ID` in the example manifest with this value.

We describe replacing these properties in [Step 2: Configure AWS for Your Cloud Foundry Deployment](#config-aws).

##<a id="config-aws"></a>Step 2: Configure AWS for Your Cloud Foundry Deployment

To configure your AWS account for Cloud Foundry:

* [Create a NAT VM](#create-nat-vm)
* [Update the MicroBOSH Security Group](#update-mibo-sec-group)
* [Create a Subnet for Cloud Foundry Deployment](#create-cf-subnet)
* [Configure your Cloud Foundry System Domain](#config-cf-dns)
* [Multi-AZ full deployment prep](#config-cf-for-multi-az-deploy)

<p class="note"><strong>Note</strong>: Ensure that "N. Virginia" is selected as the AWS Region.</p>

###<a id="create-nat-vm"></a>Create a NAT VM

1. On the EC2 Dashboard, click **Launch Instance**.
1. Click **Community AMIs**.
1. Search for and select "amzn-ami-vpc-nat-pv-2014.09.1.x86_64-ebs".
1. Select "m1.small".
1. Click **Next: Configure Instance Details** and complete as follows:
    * **Network**: Select your 'microbosh' VPC.
    * **Subnet**: Select your "Public subnet".
    * **Auto-assign Public IP**: "Enable"
1. Click **Next: Add Storage**.
1. Click **Next: Tag Instance**.
1. Enter "NAT" as the **Value** for the "Name" **Key**.
1. Click **Next: Configure Security Group**.
1. Click **Create a new security group** and complete as follows:
    * **Security group name**: "nat"
    * **Description**: "NAT Security Group"
    * **Type**: "All traffic""
    * **Protocol**: "All"
    * **Port Range**: "0 - 65535"
    * **Source**: "Custom IP / 10.0.16.0/24"
1. Click **Review and Launch**.
1. Click **Launch**.
1. Specify "Choose an existing key pair" and "bosh" from the dropdown menus.
1. Click **Launch Instances**.
1. Click **View Instances**.
1. Select the "NAT" instance in the **Instances** list.
1. Click **Actions**, then **Networking**, then **Change Source/Dest. Check**.
1. Click **Yes, Disable**.


###<a id="update-mibo-sec-group"></a> Update the MicroBOSH Security Group

1. On the VPC Dashboard, click **Security Groups**.
1. Select the "bosh" security group.
1. Click **Inbound Rules** at the bottom.
1. Click **Edit** and **Add another rule** as follows:
    * **Type**: "Custom TCP Rule"
    * **Protocol**: "TCP (6)"
    * **Port Range**: "4443"
    * **Source**: "0.0.0.0/0"
1. Click **Save**.

###<a id="create-cf-subnet"></a> Create a Subnet for Cloud Foundry Deployment 

1. Click **Subnets** from the VPC Dashboard.
1. Click **Create Subnet** and complete as follows:
    * **Name tag**: cf
    * **VPC**: microbosh
    * **Availability Zone**: Pick the same Availability Zone as the MicroBOSH Subnet.
    * **CIDR block**: 10.0.16.0/24
    * Click **Yes, Create**.
1. Replace the following in your manifest:
    * `REPLACE_WITH_AZ` with the Availability Zone you chose.
    * `REPLACE_WITH_PRIVATE_SUBNET_ID` with the Subnet ID for the cf Subnet.
    * `REPLACE_WITH_PUBLIC_SUBNET_ID` with the Subnet ID for the MicroBOSH Subnet.
1. Select the "cf Subnet" from the **Subnet** list.
1. Click the **Route table** tab in the bottom window to view the route tables.
1. Click the route table ID link in the **Route Table** field.
1. Click the **Routes** tab in the bottom window.
1. Click **Edit** and complete as follows:
    * **Destination**: 0.0.0.0/0
    * **Target**: Select the NAT instance from the list.
    * Click **Save**.
1. Click **Elastic IPs** from the VPC Dashboard.
1. Click **Allocate New Address** and click **Yes, Allocate**.
1. Update `REPLACE_WITH_ELASTIC_IP` in your manifest with the new IP address.
1. Click **Security Groups** from the VPC Dashboard.
1. Click **Create Security Group**.
    * **Name tag**: cf-public
    * **Group name**: cf-public
    * **Description**: cf Public Security Group
    * **VPC**: Select the bosh VPC.
    * Click **Create**.
1. In **Inbound Rules** tab in the bottom window, click **Edit** and add the following inbound rules:
<table border="1" class="nice">
	<tr>
		<th>Type</th>
		<th>Port Range</th>
		<th>Source</th>
		<th>Purpose</th>
	</tr>
	<tr><td>HTTP</td><td>TCP</td><td>80</td><td>0.0.0.0/0</td></tr>
	<tr><td>HTTPS</td><td>TCP</td><td>443</td><td>0.0.0.0/0</td></tr>
	<tr><td>TCP</td><td>TCP</td><td>4443</td><td>0.0.0.0/0</td></tr>
</table>

1. Update `REPLACE_WITH_PUBLIC_SECURITY_GROUP` in your manifest with the new security group.

###<a id="config-cf-dns"></a> Configure your Cloud Foundry System Domain

If you have a domain you plan to use for your Cloud Foundry System Domain, set up your DNS as follows:

1. Create a wildcard DNS entry for your root System Domain in the form `*.your-cf-domain.com` to point at the Elastic IP address you created in the [Create a Subnet for Cloud Foundry Deployment](#create-cf-subnet) section.
1. Click on **Route 53** from the Amazon Web Services Dashboard.
1. Click on **Hosted Zones**.
1. Select your zone.
1. Click **Go to Record Sets**.
1. Click **Create Record Set** and complete as follows:
    * **Name**: *
    * **Type**: A - IPv4 address
    * **Value**: Enter the Elastic IP created above.
    * Click **Create**.

If you do not have a domain, you can use 0.0.0.0.xip.io for your System Domain and replace the zeroes with your Elastic IP.

1. Update `REPLACE_WITH_SYSTEM_DOMAIN` with your system domain value.

1. Run the following series of commands to generate an SSL certificate for your system domain:

    <pre class="terminal">
    $ openssl genrsa -out cf.key 1024
    </pre>

    <p class="note"><strong>Note</strong>: The following command displays multiple prompts for certificate request information. For the <code>Common Name</code> prompt, enter <code>*.your-system-domain</code>.</p>

    <pre class="terminal">
    $ openssl req -new -key cf.key -out cf.csr
    </pre>

    <pre class="terminal">
    $ openssl x509 -req -in cf.csr -signkey cf.key -out cf.crt
    </pre>

    <pre class="terminal">
    $ cat cf.crt && cat cf.key
    </pre>

1. Update `REPLACE_WITH_SSL_CERT_AND_KEY` in your manifest with the value from the above command.


###<a id="config-cf-for-multi-az-deploy"></a> Configure AWS for multi AZ deploy, with RDS, Load Balancer

#### Create additonal Subnets
- cf1/cf2
- cf_elb1/cf_elb2
- rds_az1/rds_az2

## Create Internet gateway

## Create additional security group
- in the VPC
- web that gets connected to the internet gateway and route tables

## Set up routes tables

## create RDS DBs
- bosh
- ccdb
- uaadb

## create blobstores/s3 buckets

## create load balancer
### create self signed cert for load balancer


Back to [Deploying to AWS](aws_steps.html)

Next: [Deploying Cloud Foundry on AWS](deploy_aws_cf.html)