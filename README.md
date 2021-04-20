# aws-cloud-formation-tiered-network-config
This repo contains an example template that can be used to create a tiered network in AWS using cloud formation. 

It has a master template that calls nested stacks for each portion of the network.  

It also has a single file that can be used instead of the nested setup.  This readme will go over how to use the nested-stack and how to make it your own.

# What is CloudFormation
CloudFormation is an automation and expandability service provided by AWS.  It allows users to specify resources within AWS in either a JSON or YAML file and then have it created in AWS.  For example, if someone wants to make a VM, they can specify in the file they want a VM with a certain OS.  Then upload the file into CloudFormation and CloudFormation will read the file and create the VM with no further input needed from the user.  

It can be used to update something after it has been created as well.  With our previous VM example, say we want to add the VM to a certain subnet.  We can add a subnet assignment section to the same file and CloudFormation will see a change has been made to the file and perform an update.  It will skip the VM section since no change was made, but it will do the subnet assignment since that is new.

The processes above are called *stacks*.  So, each time a resource is created, it is made within a stack.  Within each stack we can make what is called a nested stack.  This is when we reference a separate configuration file within another one.  We do not need to use nested stacks, but it makes reading the configurations easier and reduces development time of the stacks. This is because we are not using one large file that could be error prone. It allows us to develop each section separately and not have a single point of failure.

This guide will break down how we achieved this setup and how it works.  As we get to a new portion of the file, we will go over how we share resources within files, how they are called, and what is happening.

# What will be created with this setup?
With these configuration files, we will be building out a 3-tiered architecture setup using a nested stack setup.  The architecture will look so:
### insert photo here once it’s been diagramed

The architecture consists of a web tier that lives in a public subnet, an application tier that is in a private subnet, and a database tier that is in another private subnet.  The web tier has an application load balancer that will balance requests between multiple hosts within the subnet.  Then each host in the web application tier will reach out to a load balancer for the application tier.  This load balancer will do the same thing as the web balancer but for the application hosts.  Then the application hosts will reach out the database without a load balancer.  The end goal of this setup is to have a running web application utilizing this architecture.

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

Each number for each file is the step at which they take place in the master template.  You can see that there are two step **2** files.  One for the database and one for stealthwatch.  This means that they do not rely on another portion of the configuration to be finished before another one can start.  All of the rest that continue the numbering depend on one of the others before it.  

# Master Template Overview
The master template has three main sections in it.  
1. Description
2. Parameters
3. Resources

## Description
The description is a blurb of what the template is for so that anyone reading it will be able to tell its use.

## Parameters
The parameters are values that are to be passed into the template when it is run in CloudFormation. Parameters can be a name for S3 Bucket (file storage) or a cidr block for a subnet and many other things can be passed into it.  Parameters look like so within the file:
![image](https://user-images.githubusercontent.com/10239022/114605903-924d3f80-9c68-11eb-9ad6-7e9337578c54.png)

These same parameters that are defined in the file will show up when you attempt to create a stack in CloudFormation like so:
![image](https://user-images.githubusercontent.com/10239022/114731629-e7905c00-9d0f-11eb-87af-dbff6055f3c3.png)

We do not need to use parameters, but it makes the setup more dynamic and allows the use of the files without needing to manually change the fields in the file each time we want something to be different.

In our parameters section we have a multitude of inputs. There are inputs for the subnets, availability zones, names for the database and S3 bucket, types for the instances (t2.mirco/m2.medium)., and a username/password for the database.

We will go over these parameters since most of them are repeated with different names:
* CidrBlock
* DBInstanceID
* DBName
* DBInstanceClass
* DBAllocatedStorage
* DBUsername
* DBPassword
* AppAMItype
* appKeyName
* S3BucketName
* ExternalID
* AvailabilityZone1

Within each parameter we go over, we will put the parameter in a code block and break it down.  For the AMI type parameters, we will shorten them because they are very large.



#### CidrBlock
<details>
      <summary>Drop Down</summary>
<p>CidrBlock is used to define the subnet to be used for the VPC being created.  It is repeated for each smaller subnet that will be used within the template(OutsideNet, InsideNet, DBNet). 
      
```json
"CidrBlock": {
      "AllowedPattern": "((\\d{1,3})\\.){3}\\d{1,3}/\\d{1,2}",
      "Default": "10.0.0.0/16",
      "Description": "VPC CIDR Block (eg 10.0.0.0/16)",
      "Type": "String"
    }
```

The CidrBlock parameter has four parameters.  
* **AllowedPattern** - This contains a regex to allow certain text to be entered into the input.
* **Default** - This is used to make have the input laod with a default value.  In this case it is a whole subnet.
* **Description** - This contains an explanation of what the input is used for or what should be input.
* **Type** - *REQUIRED* - This field is required and determines how it will be interpreted as a parameter.  SInnce it is of type strinng, it will be an input box when looking at the parameter section in CloudFormation. For other parameters it could be a dropdown, but this is determined by the type.

Since we have gone over AllowedPattern, Default, and Description we will not go over them again unless there is a major change in them.  But for the most part the values will change, but the concept will remain the same.
</p>
</details>


#### DBInstanceID
<details>
      <summary>Drop Down</summary><p>
This field is used to name the identifier when it is created. The identifier is used as the true name of the database when referncing it.
      
```json
"DBInstanceID": {
      "Default": "mydbinstance",
      "Description": "My database instance",
      "Type": "String",
      "MinLength": "1",
      "MaxLength": "63",
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "ConstraintDescription": "Must begin with a letter and must not end with a hyphen or contain two consecutive hyphens."
    }
```

* **MinLength** - This ensures the field is not left empty.
* **MaxLength** - The ensures the field does not have too many characters.
* **ConstraintDescription** - This is used to tell a user that there are requirements when creating this field.
</p>
</details>

#### DBName
<details>
      <summary>Drop Down</summary><p>
This field is used to name the database.  Its used as an easy to find name for us to use.
      
```json
"DBName": {
      "Default": "mydb",
      "Description": "My database",
      "Type": "String",
      "MinLength": "1",
      "MaxLength": "64",
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "ConstraintDescription": "Must begin with a letter and contain only alphanumeric characters."
    }
  ```

</p>
</details>

#### DBInstanceClass
<details>
      <summary>Drop Down</summary><p>
This field is used to determine the class type to be used.  This determine show much ram and cpu it will have on hand.
      
```json
"DBInstanceClass": {
      "Default": "db.m5.large",
      "Description": "DB instance class",
      "Type": "String",
      "ConstraintDescription": "Must select a valid DB instance type."
    }
  ```

</p>
</details>

#### DBAllocatedStorage
<details>
      <summary>Drop Down</summary><p>
This field will be used to determine the size of the database.
      
```json
"DBAllocatedStorage": {
      "Default": "50",
      "Description": "The size of the database (GiB)",
      "Type": "Number",
      "MinValue": "5",
      "MaxValue": "1024",
      "ConstraintDescription": "must be between 20 and 65536 GiB."
    }
  ```
  
* **Type** - The type for this parameter is Number instead of string.  This means it will only take a number (0-9).
</p>
</details>

#### DBUsername
<details>
      <summary>Drop Down</summary><p>
This field is to name the user that will be used to access the database.
      
```json
"DBUsername": {
      "Description": "Username for MySQL database access",
      "Type": "String",
      "MinLength": "1",
      "MaxLength": "16",
      "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
      "ConstraintDescription": "must begin with a letter and contain only alphanumeric characters."
    }
```

</p>
</details>

#### DBPassword
<details>
      <summary>Drop Down</summary><p>
This field is used to create a password for the database user being created.
      
```javascript
"DBPassword": {
      "NoEcho": "true",
      "Description": "Password MySQL database access",
      "Type": "String",
      "MinLength": "8",
      "MaxLength": "41",
      "AllowedPattern": "[a-zA-Z0-9]*",
      "ConstraintDescription": "must contain only alphanumeric characters."
    }
```

* **NoEcho** - This field means the password will not be seen while typing it for security reasons.
</p>
</details>

#### AppAMItype
<details>
      <summary>Drop Down</summary><p>
This field is used to determine the AMI type.  It determines how much ram and cpu is allocated to it.  
      
```javascript
"AppAMItype": {
      "Description": "Instance type for app server",
      "Type": "String",
      "Default": "t2.micro",
      "AllowedValues": [
        "t1.micro",
        "t2.nano",
        "t2.micro",
        "t2.small",
        "t2.medium",
        "t2.large"
        ]
      }
```

* **AllowedValues** - This field is always a list and will create a selectable dropdown of values that can be used in this parameter.

</p>
</details>

#### appKeyName
<details>
      <summary>Drop Down</summary><p>
This field allows us to select a keypair that will be used to ssh into the AMIs after they have been created.
      
```json
"appKeyName": {
      "Description": "EC2 KeyPair to enable SSH access to the instance",
      "Type": "AWS::EC2::KeyPair::KeyName"
    }
```

* **Type** - The type is of an AWS resource.  It is looking for any keys that have been created within EC2 and creates a dropdown to show us the keys we can select.
</p>
</details>

#### S3BucketName
<details>
      <summary>Drop Down</summary><p>
This field allows us to name our S3 bucket that will be used to store our logs.
      
```json
"S3BucketName": {
      "Type": "String",
      "Description": "Name the S3 bucket to be created to store VPC flow logs.",
      "AllowedPattern": "[a-z0-9-]*"
    }
```

</p>
</details>

#### ExternalID
<details>
      <summary>Drop Down</summary><p>
This field lets us input the external ID used to connect Stealth Watch to our VPC.
      
```json
"ExternalID": {
      "Type": "String",
      "Description": "The Stealthwatch cloud Observable ID."
    }
```

</p>
</details>

#### AvailabilityZone1
<details>
      <summary>Drop Down</summary><p>
This field allows us to choose the availability zone to be used for each subnet.
      
```json
"AvailabilityZone1": {
      "Description": "The AvailabilityZone to use for the first subnet",
      "Type": "AWS::EC2::AvailabilityZone::Name"
    }
```

* **Type** - The type is of an AWS resource.  It is looking for any availability zones that are available within EC2 and creates a dropdown to show us the zones we can select.
</p>
</details>

## Resources
The resources section is where we define the different resources we want to create within AWS.  In this section we do not directly make the resources, we instead call out to another configuration file and that is the file that holds the creation information.  This is what will create our nested stacks.

We will be going over the 7 resources that we are creating in this master template:
* networkSetup
* DBSetup
* SWCSetup
* appLaunchConfig
* AppAsgLb
* webLaunchConfig
* WebAsgLb

We will go over each of these resources as they are defined in the master template and then each resource will have their own section.

#### networkSetup
<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. The creation of the resources is handled within that template itself.
      
```json
"networkSetup": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/1-network-setup.json",
        "Parameters": {
          "CidrBlock": { "Ref": "CidrBlock" },
          "AvailabilityZone1": { "Ref": "AvailabilityZone1" },
          "AvailabilityZone2": { "Ref": "AvailabilityZone2" },
          "OutsideNetA": { "Ref": "OutsideNetA" },
          "OutsideNetB": { "Ref": "OutsideNetB" },
          "InsideNetA": { "Ref": "InsideNetA" },
          "InsideNetB": { "Ref": "InsideNetB" },
          "DBNetA": { "Ref": "DBNetA" }
        }
      }
    }
```

* **Type** - The type is of an AWS resource.  This specific type of CloudFOrmation::Stack is what tells Cloudformation to look for another template and pass the parameters down to that template.
* **Properties** - It defines what will be called for the template and holds the parameters to be passed down. These properties will generally only have two sections: TemplateURL and Parameters.  
      * **TemplateURL** - This is URL location of the template we want to call.  It can be a github link or anything else that is publicly accesible.  But generally we host them in AWS S3 buckets.  If it is in a buvket, we do not need to make the file public.  
      * **Parameters** - This holds the values of what will be sent to the template to be used for resource creation.  
          * **{"Ref":"CidrBlock"}** - This field works the same way as the parameters for the masterTemplate. For each of the parameters, we are using a reference assignment.  Ths means we are pulling the value that was input into the parameter field called CidrBlock.  It works the same way for the other parameters we create a reference for.

The parameters that are being defined in this resource need to be the same name as what is defined in the separate template file too. This is how the parameters get passed to the nested template.  

</p>
</details>

