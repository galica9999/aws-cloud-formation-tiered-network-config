# aws-cloud-formation-tiered-network-config
This repo contains an example template that can be used to created a tiered network in AWS using cloud formation. 

It has a master template that calls nested stacks for each portion of the network.  

It also has a single file that can be used instead of the nested setup.  This readme will go over how to use the nested-stack and how to make it your own.

# What is CloudFormation
CloudFormation is automation and expandability service profided by AWS.  It allows users to specify resources within AWS in either a JSON or YAML file and then have it created in AWS.  For example, if someone wants to make a VM, they can specify in the file they want a VM with a certain OS.  Then upload the file into CloudFormation and CloudFormation will read the file and create the VM with no further input needed from the user.  

It can be used to update something after it has been created as well.  With our previous VM example, say we want to add the VM to a certain subnet.  We can add a subnet assignment section to the same file and CloudFormation will see a change has been made to the file and perform an update.  It will skip the VM section since no change was made, but it will do the subnet assignment since that is new.

# Nested Stack Configuration Files
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
The parameters are values that are to be passed into the template when it is run in cloudformation.  The parameters looks like so within the file:
![image](https://user-images.githubusercontent.com/10239022/114605903-924d3f80-9c68-11eb-9ad6-7e9337578c54.png)

These same parameters that are defined in the file will show up when you attempt to create a stack in CloudFormation like so:
![image](https://user-images.githubusercontent.com/10239022/114731629-e7905c00-9d0f-11eb-87af-dbff6055f3c3.png)

