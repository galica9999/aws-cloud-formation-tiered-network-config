# aws-cloud-formation-tiered-network-config
This repo contains an example template that can be used to created a tiered network in AWS using cloud formation. 

It has a master template that calls nested stacks for each portion of the network.  

It also has a single file that can be used instead of the nested setup.  This readme will go over how to use the nested-stack and how to make it your own.

# What is CloudFormation
CloudFormation is an automation and expandability service profided by AWS.  It allows users to specify resources within AWS in either a JSON or YAML file and then have it created in AWS.  For example, if someone wants to make a VM, they can specify in the file they want a VM with a certain OS.  Then upload the file into CloudFormation and CloudFormation will read the file and create the VM with no further input needed from the user.  

It can be used to update something after it has been created as well.  With our previous VM example, say we want to add the VM to a certain subnet.  We can add a subnet assignment section to the same file and CloudFormation will see a change has been made to the file and perform an update.  It will skip the VM section since no change was made, but it will do the subnet assignment since that is new.

The processes above are called *stacks*.  So each time a resource is created, it is made within a stack.  Within each stack we can make what is called a nested stack.  This is when we reference a separate configuration file within another one.  We do not need to use nested stacks, but it makes reading the configurations easier and reduces development time of the stacks. This is because we are not using one large file that could be error prone. It allows us to develop each section separately and not have a single point of failure.

This guide will break down how we achieved this setup and how it works.  As we get to a new portion of the file, we will go over how we share resources within files, how they are called, and what is happening.

# What will be created with this setup?
With these configuration files, we will be building out a 3 tiered architecture setup using a nested stack setup.  The architecture will look so:
### insert photo here once its been diagramed

The architecure consists of a web tier that lives in a public subnet, an application tier that is in a private subnet, and a database tier that is in another private subnet.  The web tier has an application load balancer that will balance requests between multiple hosts within the subnet.  Then each host in the web applicaiton tier will reach out to a load balancer for the application tier.  This laod balancer will do the same thing as the web balancer but for the application hosts.  Then the application hosts will reach out the database without a load balancer.  The end goal of this setup is to have a running web application utilizing this architecture.

# Nested stack configuration files
To access the nested stack files, you can go to the nested stack directory in this repo.  Within that directory there are 8 files in total.  The *masterTemplate.json* is the template you will use to run all the subsequent json files in this folder.  

Each configuration file is numbered in the order that the master template calls them:
* 1-network-setup
* 2-db-setup
* 2-swc-setup
* 3-app-launch-config
* 4-app-asg-lb
* 5-web-launch-config
* 6-web-asg-lb
* masterTemplate

Each number for each file is the step at which they take place in the master template.  You can see that there are two step **2** files.  One for the database and one for stealthwatch.  This means that they do not rely on another portion of the configuration to be finished before another one can start.  All of the rest that continue the numbering depend on onf of the others before it.  

## Master Template Overview
The master template has three main sectionns in it.  
1. Description
2. Parameters
3. Resources

#### Description
The description is a blurb of what the template is for so that anyone reading it will be able to tell its use.

#### Parameters
The parameters are values that are to be passed into the template when it is run in cloudformation. Paramenters can be a name for S3 Bucket(file storage) or a cidr block for a subnet and many other things can be passed into it.  Parameters look like so within the file:
![image](https://user-images.githubusercontent.com/10239022/114605903-924d3f80-9c68-11eb-9ad6-7e9337578c54.png)

These same parameters that are defined in the file will show up when you attempt to create a stack in CloudFormation like so:
![image](https://user-images.githubusercontent.com/10239022/114731629-e7905c00-9d0f-11eb-87af-dbff6055f3c3.png)

We do not need to use parameters, but it makes the setup more dynamic and allows the use of the files without needing to manually change the fields in the file each time we want something to be different.

In our prameters section we have a multitude of inputs.  

