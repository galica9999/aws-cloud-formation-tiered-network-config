# aws-cloud-formation-tiered-network-config

This repo contains an example template that can be used to create a tiered network in AWS using cloud formation.

It has a master template that calls nested stacks for each portion of the network.

It also has a single file that can be used instead of the nested setup. This readme will go over how to use the nested-stack and how to make it your own.

# What is CloudFormation

CloudFormation is an automation and expandability service provided by AWS. It allows users to specify resources within AWS in either a JSON or YAML file and then have it created in AWS. For example, if someone wants to make a VM, they can specify in the file they want a VM with a certain OS. Then upload the file into CloudFormation and CloudFormation will read the file and create the VM with no further input needed from the user.

It can be used to update something after it has been created as well. With our previous VM example, say we want to add the VM to a certain subnet. We can add a subnet assignment section to the same file and CloudFormation will see a change has been made to the file and perform an update. It will skip the VM section since no change was made, but it will do the subnet assignment since that is new.

The processes above are called _stacks_. So, each time a resource is created, it is made within a stack. Within each stack we can make what is called a nested stack. This is when we reference a separate configuration file within another one. We do not need to use nested stacks, but it makes reading the configurations easier and reduces development time of the stacks. This is because we are not using one large file that could be error prone. It allows us to develop each section separately and not have a single point of failure.

This guide will break down how we achieved this setup and how it works. As we get to a new portion of the file, we will go over how we share resources within files, how they are called, and what is happening.

# What will be created with this setup?

With these configuration files, we will be building out a 3-tiered architecture setup using a nested stack setup. The architecture will look so:

### insert photo here once it’s been diagramed

The architecture consists of a web tier that lives in a public subnet, an application tier that is in a private subnet, and a database tier that is in another private subnet. The web tier has an application load balancer that will balance requests between multiple hosts within the subnet. Then each host in the web application tier will reach out to a load balancer for the application tier. This load balancer will do the same thing as the web balancer but for the application hosts. Then the application hosts will reach out the database without a load balancer. The end goal of this setup is to have a running web application utilizing this architecture.

# Nested stack configuration files

To access the nested stack files, you can go to the nested stack directory in this repo. Within that directory there are 8 files in total. The _masterTemplate.json_ is the template you will use to run all the subsequent json files in this folder.

Each configuration file is numbered in the order that the master template calls them:

- 1-network-setup
- 2-db-setup
- 2-swc-setup
- 3-app-launch-config
- 4-app-asg-lb
- 5-web-launch-config
- 6-web-asg-lb
- masterTemplate

Each number for each file is the step at which they take place in the master template. You can see that there are two step **2** files. One for the database and one for stealthwatch. This means that they do not rely on another portion of the configuration to be finished before another one can start. All of the rest that continue the numbering depend on one of the others before it.

# Master Template Overview

The master template has three main sections in it.

1. Description
2. Mappings
3. Parameters
4. Resources

## Description

The description is a blurb of what the template is for so that anyone reading it will be able to tell its use.

## Mappings

Mappings are used to define what resource should be used in which region of AWS. For example: an AWS VM(AMI) will have separate IDs depending on the region it is in. We can use the mappings to say "use this AMI if in us-east-1, or use this AMI if in ap-northeast-1".

For each region we define, we want to specify the virtulization type and the AMI ID. The virtualization type is generally decided by what the AMI was created for, but sometimes you can have multiple types for one AMI. This is why we have to specify which type we want to look for.

```json
"Mappings": {
    "RegionMap": {
      "us-east-1": { "HVM": "ami-00e87074e52e6c9f9" },
      "us-west-1": { "HVM": "ami-0bdb828fd58c52235" },
      "eu-west-1": { "HVM": "ami-08d2d8b00f270d03b" },
      "ap-southeast-1": { "HVM": "ami-0adfdaea54d40922b" },
      "ap-northeast-1": { "HVM": "ami-0ddea5e0f69c193a4" }
    }
  }
```

## Parameters

The parameters are values that are to be passed into the template when it is run in CloudFormation. Parameters can be a name for S3 Bucket (file storage) or a cidr block for a subnet and many other things can be passed into it. Parameters look like so within the file:
![image](https://user-images.githubusercontent.com/10239022/114605903-924d3f80-9c68-11eb-9ad6-7e9337578c54.png)

These same parameters that are defined in the file will show up when you attempt to create a stack in CloudFormation like so:
![image](https://user-images.githubusercontent.com/10239022/114731629-e7905c00-9d0f-11eb-87af-dbff6055f3c3.png)

We do not need to use parameters, but it makes the setup more dynamic and allows the use of the files without needing to manually change the fields in the file each time we want something to be different.

In our parameters section we have a multitude of inputs. There are inputs for the subnets, availability zones, names for the database and S3 bucket, types for the instances (t2.mirco/m2.medium)., and a username/password for the database.

We will go over these parameters since most of them are repeated with different names:

- CidrBlock
- DBInstanceID
- DBName
- DBInstanceClass
- DBAllocatedStorage
- DBUsername
- DBPassword
- AppAMItype
- appKeyName
- S3BucketName
- ExternalID
- AvailabilityZone1

Within each parameter we go over, we will put the parameter in a code block and break it down. For the AMI type parameters, we will shorten them because they are very large.

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

- **AllowedPattern** - This contains a regex to allow certain text to be entered into the input.
- **Default** - This is used to make have the input laod with a default value. In this case it is a whole subnet.
- **Description** - This contains an explanation of what the input is used for or what should be input.
- **Type** - _REQUIRED_ - This field is required and determines how it will be interpreted as a parameter. SInnce it is of type strinng, it will be an input box when looking at the parameter section in CloudFormation. For other parameters it could be a dropdown, but this is determined by the type.

Since we have gone over AllowedPattern, Default, and Description we will not go over them again unless there is a major change in them. But for the most part the values will change, but the concept will remain the same.

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

- **MinLength** - This ensures the field is not left empty.
- **MaxLength** - The ensures the field does not have too many characters.
- **ConstraintDescription** - This is used to tell a user that there are requirements when creating this field.
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

- **NoEcho** - This field means the password will not be seen while typing it for security reasons.
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

- **AllowedValues** - This field is always a list and will create a selectable dropdown of values that can be used in this parameter.

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

- **Type** - The type is of an AWS resource. It is looking for any keys that have been created within EC2 and creates a dropdown to show us the keys we can select.
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

- **Type** - The type is of an AWS resource. It is looking for any availability zones that are available within EC2 and creates a dropdown to show us the zones we can select.
</p>
</details>

## Resources

The resources section is where we define the different resources we want to create within AWS. In this section we do not directly make the resources, we instead call out to another configuration file and that is the file that holds the creation information. This is what will create our nested stacks.

We will be going over the 7 resources that we are creating in this master template:

- networkSetup
- DBSetup
- SWCSetup
- appLaunchConfig
- AppAsgLb
- webLaunchConfig
- WebAsgLb

We will go over each of these resources as they are defined in the master template and then each resource will have their own section. Most of these won;t too different from the masterTemplate, so we will only go over the if anything is different and how some of the functions are being used.

#### networkSetup

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. Then the nested template will create everything related to the base network.
      
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

- **Type** - The type is of an AWS resource. This specific type of CloudFOrmation::Stack is what tells Cloudformation to look for another template and pass the parameters down to that template.
- **Properties** - It defines what will be called for the template and holds the parameters to be passed down. These properties will generally only have two sections: TemplateURL and Parameters.  
   _ **TemplateURL** - This is URL location of the template we want to call. It can be a github link or anything else that is publicly accesible. But generally we host them in AWS S3 buckets. If it is in a buvket, we do not need to make the file public.  
   _ **Parameters** - This holds the values of what will be sent to the template to be used for resource creation. This field works the same way as the parameters for the masterTemplate. \* **{"Ref":"nameOfResource"}** - For each of the parameters, we are using a reference assignment. Ths means we are pulling the value that was input into the parameter field called CidrBlock. It works the same way for the other parameters we create a reference for.

</p>
</details>

#### DBSetup

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the DBSetup template and passes down parameters for the template to use. The nested template will then setup the database.
      
```json
"DBSetup": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["networkSetup"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/2-db-setup.json",
        "Parameters": {
          "DBInstanceID": { "Ref": "DBInstanceID" },
          "DBName": { "Ref": "DBName" },
          "DBInstanceClass": { "Ref": "DBInstanceClass" },
          "DBAllocatedStorage": { "Ref": "DBAllocatedStorage" },
          "DBUsername": { "Ref": "DBUsername" },
          "DBPassword": { "Ref": "DBPassword" },
          "dbCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.dbSG"]
          },
          "DBCloudformationSubnetA": {
            "Fn::GetAtt": ["networkSetup", "Outputs.dbSubnetA"]
          },
          "DBCloudformationSubnetB": {
            "Fn::GetAtt": ["networkSetup", "Outputs.dbSubnetB"]
          }
        }
      }
    }
```

- **DependsOn** - This field is used to make sure one resource isn't created until another is finished. In this case, the DBSetup is going to wait for the NetworkSetup to complete and then it will start. This field, can take either one input that does not need to be in a list, or it can take multiple inputs if it needs to wait on multiple resources to finish up.
- **{"Fn::GetAtt":["nameOfResource","Outputs.nameOfOutput"]}** - The Fn::GetAtt is used to get attributes of a resource. For the dbCoudformationSG and subnets, we need to get a resource that was created in another template. To do that we define thigs called _Outputs_ (we will go over outputs in more detail inn the nested template section). Those outputs have a name we define and we are able to use them by usinng the Fn:GetAtt function. The function takes a list with two attributes. The first is the resource you want to look at, in this case it is networkSetup, and the second is the output we want. Then these attributes are passed down to the nest template beinng called in this resource.

</p>
</details>

#### SWCSetup

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the stealthwatch template and passes down parameters for the template to use. 
      
```json
"SWCSetup": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["networkSetup"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/2-swc-setup.json",
        "Parameters": {
          "VPCID": {
            "Fn::GetAtt": ["networkSetup", "Outputs.StackVPC"]
          },
          "S3BucketName": {
            "Ref": "S3BucketName"
          },
          "ExternalID": {
            "Ref": "ExternalID"
          }
        }
      }
    }
```

</p>
</details>

#### appLaunchConfig

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the application launch config template and passes down parameters for the template to use. 
      
```json
"appLaunchConfig": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["DBSetup"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/3-app-launch-configs.json",
        "Parameters": {
          "AppAMItype": { "Ref": "AppAMItype" },
          "AppAMI": {
            "Fn::FindInMap": ["RegionMap", { "Ref": "AWS::Region" }, "HVM"]
          },
          "appKeyName": { "Ref": "appKeyName" },
          "appCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.appSG"]
          },
          "webCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.webSG"]
          },
          "DB": {
            "Fn::GetAtt": ["DBSetup", "Outputs.dbEndpoint"]
          },
          "DBPassword": {
            "Ref": "DBPassword"
          },
          "DBUsername": {
            "Ref": "DBUsername"
          }
        }
      }
    }
```

- **Fn::FindInMap** - This function is used to search a map defined in the mappings section of the template. We can have multiple mappings in the mapping section but for our template we only needed one for determining which AMIs to use for each region. In this example we are searching the _RegionMap_ for the region we are in, which is defined by the AWS::Region field, and then saying to grab the virtualization type of HVM for that AMI ID.

</p>
</details>

#### AppAsgLb

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use.
      
```json
"AppAsgLb": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["appLaunchConfig"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/4-app-asg-lb.json",
        "Parameters": {
          "appLaunchConfig": {
            "Fn::GetAtt": ["appLaunchConfig", "Outputs.appLaunchConfig"]
          },
          "insideCloudformationSubnetA": {
            "Fn::GetAtt": ["networkSetup", "Outputs.insideSubnetA"]
          },
          "insideCloudformationSubnetB": {
            "Fn::GetAtt": ["networkSetup", "Outputs.insideSubnetB"]
          },
          "cloudformationVPC": {
            "Fn::GetAtt": ["networkSetup", "Outputs.StackVPC"]
          },
          "appCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.appSG"]
          },
          "webCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.webSG"]
          },
          "appLaunchConfigVersion": {
            "Fn::GetAtt": ["appLaunchConfig", "Outputs.appLaunchConfigVersion"]
          }
        }
      }
    }
```

</p>
</details>

#### webLaunchConfig

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"webLaunchConfig": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["AppAsgLb"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/5-web-launch-configs.json",
        "Parameters": {
          "route53DNS": {
            "Ref": "route53DNS"
          },
          "WebAMItype": { "Ref": "WebAMItype" },
          "webAMI": {
            "Fn::FindInMap": ["RegionMap", { "Ref": "AWS::Region" }, "HVM"]
          },
          "webKeyName": { "Ref": "webKeyName" },
          "appCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.appSG"]
          },
          "webCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.webSG"]
          },
          "appALBDNS": {
            "Fn::GetAtt": ["AppAsgLb", "Outputs.appLoadBalancerDNS"]
          }
        }
      }
    }
```

</p>
</details>

#### WebAsgLb

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"WebAsgLb": {
      "Type": "AWS::CloudFormation::Stack",
      "DependsOn": ["webLaunchConfig"],
      "Properties": {
        "TemplateURL": "https://cloudformation--json-templates.s3-ap-northeast-1.amazonaws.com/6-web-asg-lb.json",
        "Parameters": {
          "webLaunchConfig": {
            "Fn::GetAtt": ["webLaunchConfig", "Outputs.webLaunchConfig"]
          },
          "outsideCloudformationSubnetA": {
            "Fn::GetAtt": ["networkSetup", "Outputs.outsideSubnetA"]
          },
          "outsideCloudformationSubnetB": {
            "Fn::GetAtt": ["networkSetup", "Outputs.outsideSubnetB"]
          },
          "cloudformationVPC": {
            "Fn::GetAtt": ["networkSetup", "Outputs.StackVPC"]
          },
          "appCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.appSG"]
          },
          "webCloudformationSG": {
            "Fn::GetAtt": ["networkSetup", "Outputs.webSG"]
          },
          "webLaunchConfigVersion": {
            "Fn::GetAtt": ["webLaunchConfig", "Outputs.webLaunchConfigVersion"]
          }
        }
      }
    }
```

</p>
</details>

# Nested Templates

All of these nested templates are the same resources that are being called within the masterTemplate. It starts with the network setup, then database and stealthwatch, and so on. We will gover each of those configuration files explainn what each does as we get to it. Most of the concepts will be repeated from the masterTemplate but there are some things we will need to go over when we get there.

We will go over each file in the order that they are called:

1. 1-network-setup
2. 2-db-setup
3. 2-swc-setup
4. 3-app-launch-configs
5. 4-app-asg-lb
6. 5-web-launch-configs
7. 6-web-asg-lb

## 1-network-setup

<details>
      <summary>Drop Down</summary><p>
The network setup template is the first template we need to create our architecture and is the longest file of them all.  It is the template that will be used to create the VPC, subnets, gateways, routes, and security groups. In this template, the parameters section replciates what is already in the masterTemplate.  This allows us to use the teamplate by itself without the masterTemplate to call it. We will focus on the resources section and output section here.
    
As a note, all of these parameters should match what the masterTemplate has when calling it. Since that is how the masterTemplate parameters will be passed to this one.  If the template was used standalone then we would still have those inputs. Since it is being called by a masterTemplate, these parameters will be overwritten with what the masterTemplate sends down to it.

We will break up the file into more readable chunks for each section and resource.

### Resources

#### VPC

<details>
      <summary><strong>#### VPC</strong></summary><p>
This resource creates the VPC our architecture will live in. 
      
```json
"VPC": {
      "Type": "AWS::EC2::VPC",
      "Properties": {
        "CidrBlock": { "Ref": "CidrBlock" },
        "EnableDnsSupport": "true",
        "EnableDnsHostnames": "true",
        "InstanceTenancy": "default",
        "Tags": [{ "Key": "Name", "Value": { "Ref": "AWS::StackName" } }]
      }
    }
    
```

- **Type** - The type is of the AWS resouce VPC. Meaning it will create a VPC with the properties we specify.
- **Properties** - The properties are the values that will be used to specify how the VPC will be created and features it will have enabled, or will support.
- **CidrBlock** - We are referencing the CidrBlock parameter. It was defined in the masterTemplate but can be used here since we have defined it in this template as well.
- **EnableDnsSupport** - This is set true because we want our VPC to allow DNS to be used in our network.
- **EnableDnsHostname** - This is set to true because we want our VPC to allow hostnames to be assigned within our network.
- **InstanceTenancy** - This detemines if we will use dedicated host for our instances or not. The default does not use dedicated instances. If we wanted dedicated ones, then we could change this be "dedicated"
- **Tags** - We are giving this VPC a tag of Name. This will allow us to find out VPC faster since the name always shows up in the left most column.

</p>
</details>

#### outsideSubnetA

<details>
      <summary>Drop Down</summary><p>
This resource creates one of the outside subnets we are using in the VPC.  This process is the same for the inside subnets and DB subnets.  The only thing that changes is the reference to the Availability Zone, CidrBlock and if it should give out public IP addresses.
We have two inside, outside, and DB subnets that are created with this same process.
      
```json
"outsideSubnetA": {
      "Type": "AWS::EC2::Subnet",
      "Properties": {
        "MapPublicIpOnLaunch": true,
        "VpcId": { "Ref": "VPC" },
        "CidrBlock": { "Ref": "OutsideNetA" },
        "AvailabilityZone": { "Ref": "AvailabilityZone1" }
      }
    }
    
```

- **Type** - The type of this resource is EC2::Subnet. This means it will be using the EC2::Subnet api in cloud formation to create a subnet with the properties we specify.
- **MapPublicIpOnLaunch** - This is set to true, because we want our AMIs to get public IP addresses when they are created in the public subnet.
- **VpcId** - We are referencing the VPC so cloud formation knows what VPC to put our subnet in.
- **CidrBlock** - The cidrblock is what ip address block we want to assign to this subnet. In this case we are assigning a /24 subnet that is being referenced from the parameters of the template. This will change for each of the subsequent subnets.
- **AvailabilityZone** - The availability

</p>
</details>

#### insideSubnetA

<details>
      <summary>Drop Down</summary><p>
This resource creates one of the inside subnets we are using in the VPC. In this one we can see that the MapPublicIP option is missing because we do not want our inside net to be publicly accessible.
      
```json
"insideSubnetA": {
      "Type": "AWS::EC2::Subnet",
      "Properties": {
        "VpcId": { "Ref": "VPC" },
        "CidrBlock": { "Ref": "InsideNetA" },
        "AvailabilityZone": { "Ref": "AvailabilityZone1" }
      }
    }
    
```

</p>
</details>

#### DBSubnetA

<details>
      <summary>Drop Down</summary><p>
This resource creates one of the Db subnets we are using in the VPC. In this one we can see that the MapPublicIP option is missing because we do not want our inside net to be publicly accessible.
      
```json
"DBSubnetA": {
      "Type": "AWS::EC2::Subnet",
      "Properties": {
        "VpcId": { "Ref": "VPC" },
        "CidrBlock": { "Ref": "DBNetA" },
        "AvailabilityZone": { "Ref": "AvailabilityZone1" }
      }
    }
    
```

</p>
</details>

#### webSG

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"webSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "GroupDescription": "Allow http and port 3000 to web servers",
        "VpcId": { "Ref": "VPC" },
        "SecurityGroupIngress": [
          {
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIp": "0.0.0.0/0"
          },
          {
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 3000,
            "CidrIp": "0.0.0.0/0"
          }
        ]
      }
    }
    
```

</p>
</details>

#### appSG

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"appSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "DependsOn": ["outsideSubnetB", "outsideSubnetA"],
      "Properties": {
        "GroupDescription": "Allow http to app server",
        "VpcId": { "Ref": "VPC" },
        "SecurityGroupIngress": [
          {
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIp": { "Ref": "OutsideNetA" }
          },
          {
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIp": { "Ref": "OutsideNetB" }
          },
          {
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIp": { "Ref": "OutsideNetA" }
          },
          {
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIp": { "Ref": "OutsideNetB" }
          }
        ]
      }
    }
    
```

</p>
</details>

#### dbSG

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"dbSG": {
      "Type": "AWS::EC2::SecurityGroup",
      "DependsOn": ["insideSubnetA", "insideSubnetB"],
      "Properties": {
        "GroupDescription": "Allow mysql access to db",
        "VpcId": { "Ref": "VPC" },
        "SecurityGroupIngress": [
          {
            "IpProtocol": "tcp",
            "FromPort": 3306,
            "ToPort": 3306,
            "CidrIp": { "Ref": "InsideNetA" }
          },
          {
            "IpProtocol": "tcp",
            "FromPort": 3306,
            "ToPort": 3306,
            "CidrIp": { "Ref": "InsideNetB" }
          }
        ]
      }
    }
    
```

</p>
</details>

#### IG

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"IG": {
      "Type": "AWS::EC2::InternetGateway",
      "Properties": {
        "Tags": [{ "Key": "Name", "Value": "cfIG" }]
      }
    }
    
```

</p>
</details>

#### AttachGateway

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"AttachGateway": {
      "Type": "AWS::EC2::VPCGatewayAttachment",
      "Properties": {
        "VpcId": { "Ref": "VPC" },
        "InternetGatewayId": { "Ref": "IG" }
      }
    }
    
```

</p>
</details>

#### OutsideRT

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"OutsideRT": {
      "Type": "AWS::EC2::RouteTable",
      "Properties": {
        "VpcId": { "Ref": "VPC" }
      }
    }
    
```

</p>
</details>

#### myRoute

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"myRoute": {
      "Type": "AWS::EC2::Route",
      "DependsOn": "IG",
      "Properties": {
        "RouteTableId": { "Ref": "OutsideRT" },
        "DestinationCidrBlock": "0.0.0.0/0",
        "GatewayId": { "Ref": "IG" }
      }
    }
    
```

</p>
</details>

#### insideRoute

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"insideRoute": {
      "Type": "AWS::EC2::Route",
      "DependsOn": "NATGW",
      "Properties": {
        "RouteTableId": { "Ref": "InsideRT" },
        "DestinationCidrBlock": "0.0.0.0/0",
        "NatGatewayId": { "Ref": "NATGW" }
      }
    }
    
```

</p>
</details>

#### RTSubnetAssocA

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"RTSubnetAssocA": {
      "Type": "AWS::EC2::SubnetRouteTableAssociation",
      "Properties": {
        "SubnetId": { "Ref": "outsideSubnetA" },
        "RouteTableId": { "Ref": "OutsideRT" }
      }
    }
    
```

</p>
</details>

#### NATGW

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"NATGW": {
      "Type": "AWS::EC2::NatGateway",
      "DependsOn": "EIP",
      "Properties": {
        "AllocationId": { "Fn::GetAtt": ["EIP", "AllocationId"] },
        "SubnetId": { "Ref": "outsideSubnetA" }
      }
    }
    
```

</p>
</details>

#### EIP

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json
"EIP": {
      "DependsOn": "AttachGateway",
      "Type": "AWS::EC2::EIP",
      "Properties": {
        "Domain": "vpc"
      }
    }
    
```

</p>
</details>

#### stuff2

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use. 
      
```json

````

</p>
</details>

#### stuff

<details>
      <summary>Drop Down</summary><p>
This resource calls out to the networkSetup template and passes down parameters for the template to use.

```json


````

</p>
</details>

### Outputs

```json

  "Outputs": {
    "StackVPC": {
      "Description": "The ID of the VPC",
      "Value": { "Ref": "VPC" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-VPCID" }
      }
    },
    "outsideSubnetA": {
      "Description": "The ID of outside subnet A",
      "Value": { "Ref": "outsideSubnetA" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-OutsideNetA" }
      }
    },
    "outsideSubnetB": {
      "Description": "The ID of outside subnet B",
      "Value": { "Ref": "outsideSubnetB" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-OutsideNetB" }
      }
    },
    "insideSubnetA": {
      "Description": "The ID of inside subnet A",
      "Value": { "Ref": "insideSubnetA" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-InsideNetA" }
      }
    },
    "insideSubnetB": {
      "Description": "The ID of inside subnet B",
      "Value": { "Ref": "insideSubnetB" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-InsideNetB" }
      }
    },
    "dbSubnetA": {
      "Description": "The ID of db subnet A",
      "Value": { "Ref": "DBSubnetA" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-dbNetA" }
      }
    },
    "dbSubnetB": {
      "Description": "The ID of db subnet B",
      "Value": { "Ref": "DBSubnetB" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-dbNetB" }
      }
    },
    "webSG": {
      "Description": "The ID of webSG",
      "Value": { "Ref": "webSG" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-webSG" }
      }
    },
    "appSG": {
      "Description": "The ID ofappSG",
      "Value": { "Ref": "appSG" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-appSG" }
      }
    },
    "dbSG": {
      "Description": "The ID of dbSG",
      "Value": { "Ref": "dbSG" },
      "Export": {
        "Name": { "Fn::Sub": "${AWS::StackName}-dbSG" }
      }
    }
  }

```

</p>
</details>

## 2-db-setup

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>

## 2-swc-setup

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>

## 3-app-launch-configs

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>

## 4-app-asg-lb

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>

## 5-web-launch-configs

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>

## 6-web-asg-lb

<details>
      <summary>Drop Down</summary><p>

```json

```

</p>
</details>
