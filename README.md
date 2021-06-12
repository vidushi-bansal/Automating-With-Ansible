# Automating Deployments in AWS EC2 instances With Ansible  
In this module, we will explore aws EC2 and Ansible automation. Running on the cloud has many benefits, including the ability to rapidly scale up or down the number of systems in use for your application in any given moment of the time to react to the needs of your customers and workloads is pivotal and a game changer when using the virtualized world. This can additionally function as a stable disaster recovery solution fro pennies on the dollar. To do this, you need to be able to use the automation to rapidly deploy and configure your servers as instances on the cloud. Provisioning AWS EC2 instances with Ansible is a practical solution to this challenge. Ansible helps ensure fast, repeatable, compliant, and automatic deployment of your systems and can make it easier to apply updates and improvements quickly. Automation at its core is all about reduction and errors as well as increasing your throughput and efficiences. Being able to deploy your environments across multiple regions, or deploying the version upgrades, can be made easy and reduce the sources of human errors such as misreading instructions, mistyping commands, forgetting steps, or executing steps in the wromg order.  
### Planning and preparing for automation  
#### Overview of deployment process  
Creating the network:  
> Create an Amazon Virtual Private Cloud (VPC)  
> Create an internet gateway  
> Create a public subnet  
> Create a routing table  
> Create a security group  
[NOTE: Create an AWS user and its Access Key and Secret Key]  
### Deploying into Amazon EC2  
We will create a variable file for a specific key-value pairs. Then we'll create an Ansible playbook with the needed modules to construct a working virtual private cloud. In order to take advantage of the privileges that cloud provides, we will need to understand how to configure our Amazon Virtual Private Cloud. The VPC is a logically isolated network. This network spans an entire AWS region where our EC2 instances will be launched. A VPC mainly provides the ability to isolate our AWS resources from other accounts in the public cloud. It allows us to route network traffic to and from our instances and shape it as we see fit. Lastly, it gives us the opportunity to protect our instances from network intrusion.    
**CODE EXAMPLE**  
```  
-name: Start  
 hosts: localhost  
 remote_user: Vidushi  
 gather_facts: false  
  
 vars_files:  
   - vars/info.yml  
```  
This example is a typical start for our plays to manage EC2. It runs on a localhost because most of the EC2 cloud modules run on a managed host which talk to the EC2 API to make changes. Fact gathering is turned off tto speed up the play, but can be turned back on if the need be. The vars/info.yml file contains variables that set the credentials you need to access EC2.    
Now we will take a look at how to author playbooks and tasks specifically in order to configure a VPC within Amazon. Each of the core task will be performed with the matching Ansible module from the table below:  
![Module](https://github.com/vidushi-bansal/blob/main/Module.png)  
#### Configuring aws VPC  
Here is a snippet of tasks from a playbook that configures an AWS VPC.  
```  
tasks:  
  - name: create a VPC  
    ec2_vpc_net:  
       aws_access_key: "{{ aws_id }}"  
       aws_secret_key: "{{ aws_key }}"  
       region: "{{ aws_region }}"  
       name: test_vpc_net  
       cidr_block: 10.10.0.0/16  
       tags:  
         module: ec2_vpc_net  
       tenancy: default  
    register: ansibleVPC  
  - name: debugVPC  
    debug:  
      var: ansibleVPC  
```  
The variables **aws_id**, **aws_key**, and **aws_region** are being loaded from vars/info.yml. A **name** for the VPC and its network (in **cidr_block**) are required parameters. We may set one or more **tags** as key-value pairs. If **tenancy** is **default**, new instances in this VPC will run on a shared hardware by default. If we use **dedicated**, new instances will run on a single-tenant hardware by default. The results of the task are stored in the variable **ansibleVPC**. This includes the resource ID of the VPC you created (in **ansibleVPC['vpc']['id']**). To inspect **ansibleVPC**, **debug** module is used to display its contents.  
![VPC](https://github.com/vidushi-bansal/blob/main/VPC.png)  
#### Configuring aws Internet Gateway  
Next, we need to manage the VPC Internet Gateway. We will use the ec2_vpc_igw to attach an internet gateway to the newly created VPC. We will need the vpc_id parameter that was returned from the previous execution when we created the VPC. We can get the required vpc id from **ansibleVPC.vpc.id**. Let's look at the code snippet for internet gateway.  
```  
- name: create an internet gateway for ansibleVPC  
  ec2_vpc_igw:  
    aws_access_key: "{{ aws_id }}"  
    aws_secret_key: "{{ aws_key }}"  
    aws_region: "{{ aws_region }}"  
    state: present  
    vpc_id: "{{ ansibleVPC.vpc.id }}"  
    tags:  
      Name: ansibleVPC_IGW  
  register: ansibleVPC_igw  
  
- name: display ansibleVPC IGW details  
  debug:  
    var: ansibleVPC_igw     
```  
**vpc_id** is the VPC's ID, which is retrieved by reading the data in the variable registered when the VPC was created. **state** controls whether the IGW should be present or absent from the VPC. A tag of **Name:ansibleVPC_IGW** is set. We will need the IGW's ID later to create the route table, so we save the results of this task in ansibleVPC_igw. The debug task is not usually needed but will show you the contents of **ansibleVPC_igw**.  
![IGW](https://github.com/vidushi-bansal/blob/main/Internet_Gateway.png)   
#### Configuring subnets in aws VPC  
Now we are ready to create a subnet. We will manage our subnets in aws VPCs using **ec2_vpc_subnet** module. This will add a subnet to an existing virtual private cloud. We need to specify the vpc_id returned from the variable from our initial task. Let's look at the code snippet for creating subnets.  
```  
- name: create public subnet in "{{ aws_region }}"  
  ec2_vpc_subnet:  
    aws_access_key: "{{ aws_id }}"  
    aws_secret_key: "{{ aws_key }}"  
    aws_region: "{{ aws_region }}"  
    state: present  
    cidr: 10.10.0.0/16  
    vpc_id: "{{ ansibleVPC.vpc.id }}"  
    map_public: yes  
    tags:  
      Name: ansible_public_subnet  
  register: ansible_public_subnet  
- name: show public subnet details  
  debug:  
    var: ansible_public_subnet  
```  
Set the vpc_id parameter to the VPC's ID and **state** to present to specify that this subnet should exist. Next specify the CIDR block. To make it easier to find and manage, set the tag named **ansible_public_subnet**. Use **map_public** to assign instances a public IP address by default.   **public_subnet** contains results we may need later in the play.  
![SUBNET](https://github.com/vidushi-bansal/blob/main/Subnet.png)  
#### Configuring route tables for subnets  
Next we can manage our routing tables. In order for our VPC to route the traffic for the new subnet, it needs to understand how. We will do so by defining a route table entry. We will use the ec2_vpc_route_table module to create this routing table. It also is used to manage the routes in the table and associate them with an IG, or internet gateway. As we've been storing return results in variables throughout each of these tasks, we will need to recover the VPC ID and IGW ID from their respective variables. Let's look at the code snippet for creating route tables.  
```  
- name: create new route table for public subnet  
  ec2_vpc_route_table:  
    aws_access_key: "{{ aws_id }}"  
    aws_secret_key: "{{ aws_key }}"  
    region: "{{ aws_region }}"  
    state: present  
    vpc_id: "{{ ansibleVPC.vpc.id }}"  
    tags:  
      Name: rt_ansibleVPC_PublicSubnet  
    subnets:  
      - "{{ ansible_public_subnet.subnet.id }}"  
    routes:  
      - dest: 0.0.0.0/0  
        gateway_id: "{{ ansibleVPC_igw.gateway_id }}"  
  register: rt_ansibleVPC_PublicSubnet  
- name: display public route table  
  debug:  
    var: rt_ansibleVPC_PublicSubnet   
  
```  
  
**vpc_id** must be set to the ID of the VPC for which you are creating the route table. **subnets** is a list of subnet IDs to attach to the route table -- this example gets it from the **public_subnet** variable you registered earlier in the play. **routes** is a list of routes. Each route in the list is a directory:
 **dest** is the network being routed to, 0.0.0.0/0 is the default route. **gateway_id** is the ID of an IGW.
![Route_table](https://github.com/vidushi-bansal/blob/main/Route_table.png)  
#### Configuring aws security group  
Lastly, we will need to configure a security group. This will help in managing the firewall rules that pertain to our VPC. Basic parameters for defining a group using the ec2_group module are the name, that provides the name for the new group, region, that specifies the aws region for the group and rules, that defines the firewall inbound rules to enforce. Let's look at the code snippet for ec2-group module.   
```  
- name: Create a Security Group  
    ec2_group:  
        aws_access_key: "{{ aws_id }}"  
        aws_secret_key: "{{ aws_key }}"  
        region: "{{ aws_region }}"  
        name: "Ansible Test Security Group"  
        description: "Ansible Test Security Group"  
        vpc_id: "{{ ansibleVPC.vpc.id }}"  
        tags:  
          Name: Ansible Test Security Group  
        rules:   
          - proto: "tcp"  
            ports: "22"  
            cidr_ip: 0.0.0.0/0   
    register: ansible_vpc_sg  
  
- name: Set security group ID in variable  
  set_fact:  
    sg_id: "{{ ansible_vpc_sg.group_id }}"  
```  
In order to launch an instance in aws we need to assign it to a particular security group. We will give our security group a descriptive name. The security group must be in the same VPC as the resources we want in our project. A security group blocks all traffic by default. If we want to allow the traffic to a port we need to add a rule specifying it.  
![Security_Group](https://github.com/vidushi-bansal/blob/main/Security_Group.png)  
  
#### Configuring aws ec2 instance  
We will now look at the main module for managing our EC2 instances with Ansible. The ec2 module allows us to create and destroy aws instances. Here are the steps required to create an instance:  
1. Specify the AMI to use for this instance   
2. Declare the instance type you want to use  
3. Associate the SSH key with the instance  
4. Attach a security group  
5. Attach a subnet  
6. Assign a public IP address  
Once you create an instance, you can use other Ansible modules in subsequent tests to provision and configure it further, just like any other managed host. Before, we are ready to use EC2 to create instances, we need to know the ID of an AMI to use in order to deploy the instance. Many AMIs already exist in Amazon EC2. The IDs for these AMIs can vary from region to region, so we will use the **ec2_ami_info** module in order to find the AMI that's appropriate for us to use. Lets look at the code snippet for ec2_ami_info.  
```  
- name: Find the AMI  
  ec2_ami_info:  
    aws_access_key: "{{ aws_id }}"  
    aws_secret_key: "{{ aws_key }}"  
    region: "{{ aws_region }}"  
    filters:  
      architecture: x86_64  
      name: ubuntu*20.04*  
  register: amis  
- name: Show AMI's  
  debug:  
    var: amis  
- name: Get the latest one  
  set_fact:  
    latest_ami: "{{ amis.images | sort(attribute='creation_date') | last }}"       
```  
The **ec2_ami_info** searches for the AMI of Ubuntu. The **filter** dictionary filters the list of the AMIs returned by the module, based on the x86_64 architecture and using wildcards to match the name. All AMIs available for the region that match are returned, aw=nd we store the results in the **amis** variable. The **set_fact** task filters the list of images for the one with the most recent creation date and saves it in **latest_ami**.  
  
## Commands  
To run the syntax check:  
```  
ansible-playbook --syntax-check playbook.yml  
```  
After syntax is verified, run the play  
```  
ansible-playbook playbook.yml  
```  

