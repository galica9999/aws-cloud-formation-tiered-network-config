# aws-cloud-formation-tiered-network-config
This repo contains an example template that can be used to created a tiered network in AWS using cloud formation. 

It has a master template that calls nested stacks for each portion of the network.  

It also has a single file that can be used instead of the nested setup.  This readme will go over how to use the nested-stack and how to make it your own.

#Nested Stack Configuration Files
To access the nested stack files, you can go to the nested stack directory in this repo.  Within that directory there are 8 files in total.  The *masterTemplate.json* is the template you will use to run all the subsequent json files in this folder.  

Each configuration file is numbered in the order that the master template calls them:
* 1-network-setup
* 2-db-setup
* 2-swc-setup
* 3-app-launch-config
* 4-app-asg-lb
* 5-web-launch-config
* 6-web-asg-lb

Each number for each file is the step at which they take place in the master template.  You can see that there are two step **2** files.  One for the database and one for stealthwatch.  This means that they do not rely on another portion of the configuration to be finished before another one can start.  All of the rest that continue the numbering depend on onf of the others before it.  




