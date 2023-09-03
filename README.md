# Project 11: Infrastructure as Code with Ansible

Greetings linkedin family, ever wondered about writing code and launching an infrastructure without having to open the different services on the console and create and wait for the creation to finish before moving on to the next service? Well Ansible brings us good news. With Ansible Playbooks, I was able to build an Infrastructure from a Control machine(EC2) instance. The code is stored in my repo on Github (edited on my local machine), then the Control machine, with an Ansible installation clones the code, from it, builds the infrastructure. I actually used a variable file, which was imported into the playbook, the vars file serves the purpose of making infrastructure code reusable (say launch the same infrastructure in another region, by just modifying the region's specs say the ami etc). One key lesson here never never neveeeeer to be forgotten: use IAM Roles for the control machine rather than hard coding your Secret keys into the playbook, for it can be mistakenly pushed to Github, by the way if you push your secret keys to Github, you will be detained in DevOps top security detention center for 3 months(i hope you know am joking 🤣 . As a teacher, i used to associate an important concept to a joke, the students will pause for a while and laugh, the concept will stick since they'll remember the joke)

## Architecture
![Project Architecture](https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible/blob/main/images/project11_architecture.jpg)

## Steps

1; Create a project on VSCode, create files:
bastion-instance.yml:

```yml
  ---
- name: Setup Vprofile Bastion Host
  hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: Import VPC Setup variable
      include_vars: vars/bastion_setup
    
    - name: Import VPC Setup variable
      include_vars: vars/output_vars

    - name: Create vprofileec2 key
      ec2_key: 
        name: vprofile-key
        region: "{{region}}"
      register: key_out

    - name: Crete Sec group for bastion hosts
      ec2_group:
        name: Bastion-host-sg
        description: Allow port 22 from everywhere and all port within Bastion-host-sg
        vpc_id: "{{vpc_id}}"
        region: "{{region}}"
        rules: 
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: "{{MYIP}}"
      register: BastionSG_out
    
    - name: Creating Bastion Host
      ec2:
        key_name: vprofile-key
        region: "{{region}}"
        instance_type: t2.micro
        image: "{{bastion_ami}}"
        vpc_subnet_id: "{{ pubsub1id }}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "Bastion-host"
          Project: vprofile
          owner: Devops Team
        exact_count: 1
        count_tag:
          Name: "Bastion-host"
          Project: Vprofile
          Owner: Devops Team
        group_id: "{{BastionSG_out.group_id}}"
      register: bastionHost_out
    
    - debug:
        var: key_out

```

vpc-setup.yml:

```yml
- hosts: localhost
  connection: local 
  gather_facts: False 
  tasks:
    - name: Import VPC variables
      include_vars: vars/vpc_setup
    
    - name: Create vprofile VPC
      ec2_vpc_net: 
        name: "{{vpc_name}}"
        cidr_block: "{{vpcCidr}}"
        region: "{{region}}"
        dns_support: yes
        dns_hostnames: yes
        tenancy: default
        state: "{{state}}"
      register: vpcout

    - name: Create Public subnet 1 in zone 1
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PubSub1Cidr}}"
        state: "{{state}}"
        az: "{{zone1}}"
        map_public: yes
        resource_tags:
          Name: vprofile-pubsub1
      register: pubsub1_out

    - name: Create Public subnet 2 in zone 2
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PubSub2Cidr}}"
        state: "{{state}}"
        az: "{{zone2}}"
        map_public: yes
        resource_tags:
          Name: vprofile-pubsub2
      register: pubsub2_out

    - name: Create Public subnet 3 in zone 3
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PubSub3Cidr}}"
        state: "{{state}}"
        az: "{{zone3}}"
        map_public: yes
        resource_tags:
          Name: vprofile-pubsub3
      register: pubsub3_out

    - name: Create Private subnet 1 in zone 1
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PrivSub1Cidr}}"
        state: "{{state}}"
        az: "{{zone1}}"
        map_public: no
        resource_tags:
          Name: vprofile-privsub1
      register: privsub1_out
    
    - name: Create Private subnet 2 in zone 2
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PrivSub2Cidr}}"
        state: "{{state}}"
        az: "{{zone2}}"
        map_public: no
        resource_tags:
          Name: vprofile-privsub2
      register: privsub2_out

    - name: Create Private subnet 3 in zone 3
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        cidr: "{{PrivSub3Cidr}}"
        state: "{{state}}"
        az: "{{zone3}}"
        map_public: no
        resource_tags:
          Name: vprofile-privsub3
      register: privsub3_out

    - name: Internet Gateway Setup
      ec2_vpc_igw:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        state: "{{state}}"
        resource_tags:
          Name: vprofile_IGW
      register: igw_out

    - name: Route tables
      ec2_vpc_route_table:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        tags :
          Name: vprofile-pubRT
        subnets: 
          - "{{pubsub1_out.subnet.id}}"
          - "{{pubsub2_out.subnet.id}}"
          - "{{pubsub3_out.subnet.id}}"
        routes: 
          - dest: 0.0.0.0/0
            gateway_id: "{{igw_out.gateway_id}}"
      register: pubRT_out

    - name: Create new nat gateway and allocate new EIP if a nat gateway does not yet exist in the account
      ec2_vpc_nat_gateway:
        state: "{{state}}"
        subnet_id: "{{pubsub1_out.subnet.id}}"
        wait: yes
        region: "{{region}}"
        if_exist_do_not_create: true
      register: NATGW_out

    - name: Setup private subnet Route tables
      ec2_vpc_route_table:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        tags :
          Name: vprofile-privRT
        subnets: 
          - "{{privsub1_out.subnet.id}}"
          - "{{privsub2_out.subnet.id}}"
          - "{{privsub3_out.subnet.id}}"
        routes: 
          - dest: 0.0.0.0/0
            gateway_id: "{{NATGW_out.nat_gateway_id}}"
      register: privRT_out

    - set_fact:
        vpc_id: "{{vpcout.vpc.id}}"
        pubsub1id: "{{pubsub1_out.subnet.id}}"
        pubsub2id: "{{pubsub2_out.subnet.id}}"
        pubsub3id: "{{pubsub3_out.subnet.id}}"
        privsub1id: "{{privsub1_out.subnet.id}}"
        privsub2id: "{{privsub2_out.subnet.id}}"
        privsub3id: "{{privsub3_out.subnet.id}}"
        igwid: "{{igw_out.gateway_id}}"
        pubRTid: "{{pubRT_out.route_table.id}}"
        NATGWid: "{{NATGW_out.nat_gateway_id}}"
        privRTid: "{{privRT_out.route_table.id}}"
        cacheable: yes

    - name: Create variables file for vpc output
      copy: 
        content: "vpc_id: {{vpcout.vpc.id}}\npubsub1id: {{pubsub1_out.subnet.id}}\npubsub2id: {{pubsub2_out.subnet.id}}\npubsub3id: {{pubsub3_out.subnet.id}}\nprivsub1id: {{privsub1_out.subnet.id}}\nprivsub2id: {{privsub2_out.subnet.id}}\nprivsub3id: {{privsub3_out.subnet.id}}\nigwid: {{igw_out.gateway_id}}\npubRTid: {{pubRT_out.route_table.id}}\nNATGWid: {{NATGW_out.nat_gateway_id}}\nprivRTid: {{privRT_out.route_table.id}}"
        dest: vars/output_vars

```

Create a folder ```vars```, in it create the following files

bastion_setup:

```
bastion_ami: ami-024e6efaf93d85776
region: us-east-2
MYIP: 0.0.0.0/0
```

vpc_septup:
```
vpc_name: "Vprofile-vpc"

# VPC range
vpcCidr: '172.20.0.0/16'

# Subnet Range
PubSub1Cidr: 172.20.1.0/24
PubSub2Cidr: 172.20.2.0/24
PubSub3Cidr: 172.20.3.0/24
PrivSub1Cidr: 172.20.4.0/24
PrivSub2Cidr: 172.20.5.0/24
PrivSub3Cidr: 172.20.6.0/24

# Region name 
region: "us-east-2"

zone1: us-east-2a
zone2: us-east-2b
zone3: us-east-2c

state: "present"
```

Push all of these to Github

2; Login to aws account and create a control machine

```
Name: Control-Machine
SecGrp:
    name: control-mach-sg
    os: ubuntu 22.04
    rules: allow ssh from my ip    
userdata:
    #!/bin/bash
    sudo apt update
    sudo apt install software-properties-common -y
    sudo add-apt-repository --yes --update ppa:ansible/ansible
    sudo apt install ansible -y
    sudo apt install python3-boto3 -y

```

3; SSH into the control machine and clone the project repository

```
git clone https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible.git
cd Project_11-IAAC-with-Ansible
```

4; Build the infrastructure, run the code
```
ansible-playbook vpc-setup.yml
```
The console will exicute the different plays to build the setup
![Ansible playbook creates vpc with setup](https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible/blob/main/images/ansible%20playbook%20run.png)

The playbook will create a new file called output_vars in the vars/ folder. Copy the contents to our local machine and create the same file in the vars folder and paste the copied content. Push save the project and push to github. Now to launch the EC2 instances within our setup, run the following

```
ansible-playbook bastion-instance.yml
```

Now we can check the list of services created
![](https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible/blob/main/images/vpc%20creation.png)
![](https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible/blob/main/images/subnets.png)
![](https://github.com/Ndzenyuy/Project_11-IAAC-with-Ansible/blob/main/images/running%20instances.png)

