---
subcollection: solution-tutorials
copyright:
  years: 2019
lastupdated: "2019-10-18"
lasttested: "2019-06-17"
---

{:java: #java .ph data-hd-programlang='java'}
{:swift: #swift .ph data-hd-programlang='swift'}
{:ios: #ios data-hd-operatingsystem="ios"}
{:android: #android data-hd-operatingsystem="android"}
{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:screen: .screen}
{:tip: .tip}
{:pre: .pre}
{:important: .important}
{:note: .note}

# Securely access remote instances with a bastion host
{: #vpc-secure-management-bastion-server}

This tutorial is compatible with VPC for Generation 1 compute and VPC for Generation 2 compute. Throughout the tutorial, you will find notes highlighting differences where applicable.
{:note}

This tutorial walks you through the deployment of a bastion host to securely access remote instances within a virtual private cloud. Bastion host is an instance that is provisioned in a public subnet and can be accessed via SSH. Once set up, the bastion host acts as a **jump** server allowing secure connection to instances provisioned in a private subnet.

To reduce exposure of servers within the VPC you will create and use a bastion host. Administrative tasks on the individual servers are going to be performed using SSH, proxied through the bastion. Access to the servers and regular internet access from the servers, e.g., for software installation, will only be allowed with a special maintenance security group attached to those servers.
{:shortdesc}

## Objectives
{: #objectives}

- Learn how to set up a bastion host and security groups with rules
- Securely manage servers via the bastion host

## Services used
{: #services}

This tutorial uses the following runtimes and services:

- [{{site.data.keyword.vpc_full}}](https://{DomainName}/vpc/provision/vpc)
- [{{site.data.keyword.vsi_is_full}}](https://{DomainName}/vpc/provision/vs)

This tutorial may incur costs. Use the [Pricing Calculator](https://{DomainName}/estimator/review) to generate a cost estimate based on your projected usage.

## Architecture
{: #architecture}

![Architecture](images/solution47-vpc-secure-management-bastion-server/ArchitectureDiagram.png)

1. After setting up the required infrastructure (subnets, security groups with rules, VSIs) on the cloud, the admin (DevOps) connects (SSH) to the bastion host using the private SSH key.
2. The admin assigns a maintenance security group with proper outbound rules.
3. The admin connects (SSH) securely to the instance's private IP address via the bastion host to install or update any required software eg., a web server
4. The internet user makes an HTTP/HTTPS request to the web server.

## Before you begin
{: #prereqs}

- Check for user permissions. Be sure that your user account has sufficient permissions to create and manage VPC resources. See the list of required permissions for [VPC for Gen 1](/docs/vpc-on-classic?topic=vpc-on-classic-managing-user-permissions-for-vpc-resources) or for [VPC for Gen 2](https://{DomainName}/docs/vpc?topic=vpc-managing-user-permissions-for-vpc-resources).
- You need an SSH key to connect to the virtual servers. If you don't have an SSH key, see the instructions for creating a key for [VPC for Gen 1](/docs/vpc-on-classic?topic=vpc-on-classic-getting-started#prerequisites) or for [VPC for Gen 2](/docs/vpc?topic=vpc-ssh-keys). 
- The tutorial assumes that you are adding the bastion host in an existing virtual private cloud. **If you don't have a virtual private cloud in your account, create one before proceeding with the next steps.**

## Create a bastion host
{: #create-bastion-host}

In this section, you will create and configure a bastion host along with a security group in a separate subnet.

### Create a subnet
{: #create-bastion-subnet}

1. Click **Subnets** under **Network** on the left pane, then **New subnet**.
   - Enter **vpc-secure-bastion-subnet** as name, then select the VPC you created.
   - Select a location and zone.
   - Enter the IP range for the subnet in CIDR notation, i.e., **10.xxx.0.0/24**. Leave the **Address prefix** as it is and select the **Number of addresses** as 256.
   
   If you are using VPC for Gen 1, select **VPC default** for your subnet access control list (ACL). You can configure the inbound and outbound rules later.
   {:note}
1. Switch the **Public gateway** to **Attached**.
1. Click **Create subnet** to provision it.

### Create and configure bastion security group
{: #create-configure-security-group }

Let's create a security group and configure inbound rules to your bastion VSI.

1.  Navigate to **Security groups** and click **New security group**. Enter **vpc-secure-bastion-sg** as name and select your VPC.
2.  Now, create the following inbound rules by clicking **Add rule** in the inbound section. They allow SSH access and Ping (ICMP).
    **Inbound rule:**
    <table>
      <thead>
        <tr>
          <td><strong>Protocol</strong></td>
          <td><strong>Source type</strong></td>
          <td><strong>Source</strong></td>
          <td><strong>Value</strong></td>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>TCP</td>
          <td>Any</td>
          <td>0.0.0.0/0</td>
          <td>Ports 22-22</td>
        </tr>
        <tr>
          <td>ICMP</td>
          <td>Any</td>
          <td>0.0.0.0/0</td>
          <td>Type: <strong>8</strong>,Code: <strong>Leave empty</strong></td>
        </tr>
      </tbody>
    </table>

    To enhance security further, the inbound traffic could be restricted to the company network or a typical home network. You could run `curl ipecho.net/plain ; echo` to obtain your network's external IP address and use that instead.
    {:tip }
3. Click **Create security group** to create it.

### Create a bastion instance
{: #create-bastion-instance}

With the subnet and security group already in place, next, create the bastion virtual server instance.

1. Under **Subnets** on the left pane, select **vpc-secure-bastion-subnet**.
1. Click on **Attached resources** and provision a **New instance** called **vpc-secure-bastion-vsi** under your own VPC and resource group.
1. Select a **Location** and make sure to later use the same location again.
1. Select **Compute** (2 vCPUs and 4 GB RAM) as your profile.
1. To create a new **SSH key**, click **New key**
   - Enter **vpc-ssh-key** as key name.
   - Leave the **Region** as is.
   - Copy the contents of your existing local SSH key and paste it under **Public key**.
   - Click **Add SSH key**.
1. Select **Ubuntu Linux** as your image. You can pick any version of the image.
1. Under **Network interfaces**, click on the **Edit** icon next to the Security Groups
   - Make sure that **vpc-secure-bastion-subnet** is selected as the subnet.
   - Uncheck the default security group and mark **vpc-secure-bastion-sg**.
   - Click **Save**.
1. Click **Create virtual server instance**.
1. Once the instance is created, click on **vpc-secure-bastion-vsi** and **reserve** a floating IP.

### Test your bastion

Once your bastion's floating IP address is active, try connecting to it using **ssh**:

```sh
ssh -i ~/.ssh/<PRIVATE_KEY> root@<BASTION_FLOATING_IP_ADDRESS>
```

{:pre}

## Configure a security group with maintenance access rules
{: #maintenance-security-group}

With access to the bastion working, continue and create the security group for maintenance tasks like installing and updating the software.

1. Navigate to **Security groups** and provision a new security group called **vpc-secure-maintenance-sg** with the below outbound rules
   <table>
     <thead>
       <tr>
         <td><strong>Protocol</strong></td>
         <td><strong>Destination type</strong></td>
         <td><strong>Destination</strong></td>
         <td><strong>Value</strong></td>
       </tr>
     </thead>
     <tbody>
       <tr>
         <td>TCP</td>
         <td>Any</td>
         <td>0.0.0.0/0</td>
         <td>Ports 80-80</td>
       </tr>
       <tr>
         <td>TCP</td>
         <td>Any</td>
         <td>0.0.0.0/0</td>
         <td>Ports 443-443</td>
       </tr>
       <tr>
         <td>TCP</td>
         <td>Any</td>
         <td>0.0.0.0/0</td>
         <td>Ports 53-53</td>
       </tr>
       <tr>
         <td>UDP</td>
         <td>Any</td>
         <td>0.0.0.0/0</td>
         <td>Ports 53-53</td>
       </tr>
     </tbody>
   </table>

 DNS server requests are addressed on port 53. DNS uses TCP for Zone transfer and UDP for name queries either regular (primary) or reverse. HTTP requests are on port 80 and 443.
 {:tip }

2. Next, add this **inbound** rule which allows SSH access from the bastion host.

   <table>
    <thead>
      <tr>
         <td><strong>Protocol</strong></td>
         <td><strong>Source type</strong></td>
         <td><strong>Source</strong></td>
         <td><strong>Value</strong></td>
      </tr>
    </thead>
    <tbody>
      <tr>
         <td>TCP</td>
         <td>Security group</td>
         <td>vpc-secure-bastion-sg</td>
         <td>Ports 22-22</td>
      </tr>
    </tbody>
   </table>

3. Create the security group.
4. Navigate to **Security Groups**, then select **vpc-secure-bastion-sg**.
5. Finally, edit the security group and add the following **outbound** rule.

   <table>
    <thead>
      <td><strong>Protocol</strong></td>
      <td><strong>Destination type</strong></td>
      <td><strong>Destination</strong></td>
      <td><strong>Value</strong></td>
    </thead>
    <tbody>
      <tr>
         <td>TCP</td>
         <td>Security group</td>
         <td>vpc-secure-maintenance-sg</td>
         <td>Ports 22-22</td>
      </tr>
    </tbody>
   </table>

## Use the bastion host to access other instances in the VPC
{: #bastion-host-access-instances}

In this section, you will create a private subnet with virtual server instance and a security group. By default, any subnet created in a VPC is private.

If you already have virtual server instances in your VPC that you want to connect to, you can skip the next three sections and start [adding your virtual server instances to the maintenance security group](#add-vsi-to-maintenance).

### Create a subnet
{: #create-private-subnet}

To create a new subnet,

1. Click **Subnets** under **Network** on the left pane, then **New subnet**.
   - Enter **vpc-secure-private-subnet** as name, then select the VPC you created.
   - Select a location.
   - Enter the IP range for the subnet in CIDR notation, i.e., **10.xxx.1.0/24**. Leave the **Address prefix** as it is and select the **Number of addresses** as 256.
   
   If you are using VPC for Gen 1, select **VPC default** for your subnet access control list (ACL). You can configure the inbound and outbound rules later.
   {:note}

1. Switch the **Public gateway** to **Attached**.
1. Click **Create subnet** to provision it.

### Create a security group

To create a new security group:

1. Click **Security groups** under Network, then **New security group**.
2. Enter **vpc-secure-private-sg** as name and select the VPC you created earlier.
3. Click **Create security group**.

### Create a virtual server instance

To create a virtual server instance in the newly created subnet:
1. Click on the private subnet under **Subnets**.
1. Click **Attached resources**, then **New instance**.
1. Enter a unique name, **vpc-secure-private-vsi**, select the VPC your created and resource group as earlier.
1. Select the same **Location** already used by the bastion virtual server.
1. Select **Compute** (2 vCPUs and 4 GB RAM) as your profile. To check other available profiles, click **All profiles**
1. For **SSH keys** pick the SSH key you created earlier for the bastion.
1. Select **Ubuntu Linux** as your image. You can pick any version of the image.
1. Under **Network interfaces**, click on the **Edit** icon next to the Security Groups
   - Select **vpc-secure-private-subnet** as the subnet.
   - Uncheck the default security and group and activate **vpc-secure-private-sg**.
   - Click **Save**.
1. Click **Create virtual server instance**.

### Add virtual servers to the maintenance security group
{: #add-vsi-to-maintenance}

For administrative work on the servers, you have to associate the specific virtual servers with the maintenance security group. In the following, you will enable maintenance, log into the private server, update the software package information, then disassociate the security group again.

Let's enable the maintenance security group for the server.

1. Navigate to **Security groups** and select **vpc-secure-maintenance-sg** security group.
2. Click **Attached interfaces**, then **Edit interfaces**.
3. Expand the virtual server instances and activate the selection next to **primary** in the **Interfaces** column.
4. Click **Save** for the changes to be applied.

### Connect to the instance

To SSH into an instance using its **private IP**, you will use the bastion host as your **jump host**.

1. Obtain the private IP address of a virtual server instance under **Virtual server instances**.
2. Use the ssh command with `-J` to log into the server with the bastion **floating IP** address you used earlier and the server **Private IP** address shown under **Network interfaces**.

   ```sh
   ssh -J root@<BASTION_FLOATING_IP_ADDRESS> root@<PRIVATE_IP_ADDRESS>
   ```

   {:pre}

   `-J` flag is supported in OpenSSH version 7.3+. In older versions `-J` is not available. In this case the safest and most straightforward way is to use ssh's stdio forwarding (`-W`) mode to "bounce" the connection through a bastion host. e.g., `ssh -o ProxyCommand="ssh -W %h:%p root@<BASTION_FLOATING_IP_ADDRESS" root@<PRIVATE_IP_ADDRESS>`
   {:tip }

### Install software and perform maintenance tasks

Once connected, you can install software on the virtual server in the private subnet or perform maintenance tasks.

1. First, update the software package information:
   ```sh
   apt-get update
   ```
   {:pre}
2. Install the desired software, e.g., Nginx or MySQL or IBM Db2.

When done, disconnect from the server with `exit` command.

To allow HTTP/HTTPS requests from the internet user, assign a **floating IP** to the VSI in the private subnet and open required ports (80 - HTTP and 443 - HTTPS) via the inbound rules in the security group of private VSI.
{:tip}

### Disable the maintenance security group

Once you're done installing software or performing maintenance, you should remove the virtual servers from the maintenance security group to keep them isolated.

1. Navigate to **Security groups** and select **vpc-secure-maintenance-sg** security group.
2. Click **Attached interfaces**, then **Edit interfaces**.
3. Expand the virtual server instances and uncheck the selection next to **primary** in the **Interfaces** column.
4. Click **Save** for the changes to be applied.

## Remove resources
{: #removeresources}

1. Switch to **Virtual server instances**, **Stop** and **Delete** your instances.
2. Once the VSIs are gone, switch to **Subnets** and delete your subnets.
3. After the subnets have been deleted, switch to the **Virtual private clouds** tab and delete your VPC.

When using the console, you may need to refresh your browser to see updated status information after deleting a resource.
{:tip}

## Related content
{: #related}

- [Private and public subnets in a Virtual Private Cloud](/docs/tutorials?topic=solution-tutorials-vpc-public-app-private-backend#vpc-public-app-private-backend)
