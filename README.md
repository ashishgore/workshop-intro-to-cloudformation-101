# Intro to CloudFormation 101

This document outlines a workshop for introducing CloudFormation.

**Are participants expected to be a CloudFormation expert at the end of the
workshop?**

Absolutely not.

The goal of this workshop is to expose what CloudFormation is and does.

It is totally OK if any or all of these things are beyond you - *right now*.

After the workshop, perhaps through reflection it will start to make sense.

Down the road you might become more exposed to the terms and concepts described in this workshop and it might start to make more sense at
that point.

Just keep an open mind and everything will be 200 OK. It's more important to
know about things you don't know then to know everything. :)

## Overview

The following topics will be covered:

- [Brief History](#brief-history)
    + [The Manual Way](#the-manual-way)
    + [Configuration Management](#configuration-management)
    + [Infrastructure as Code](#infrastructure-as-code)
- [CloudFormation Templating Languages](#cloudformation-templating-languages)
    + [JSON](#json)
    + [YAML](#yaml)
    + [Alternatives](#alternatives)
- [CloudFormation Resources](#cloudformation-resources)
    + [Security Groups](#security-groups)
    + [Launch Configuration](#launch-configuration)
    + [Auto Scaling Group](#auto-scaling-group)
    + [EC2 Instance](#ec2-instance)
- [CloudFormation Features](#cloudformation-features)
    + [Parameters](#parameters)
    + [Resource Types](#resource-types)
    + [Functions](#functions)
- [Lab](#lab)

## Brief History

Before we jump into writing a CloudFormation template, let's review a brief
history of managing AWS infrastructure before CloudFormation.

### The Manual Way

![Automation - xkcd](https://imgs.xkcd.com/comics/automation.png)

The fallback for automation is just doing whatever it is manually.

Without the right tooling, automating a process can be time consuming because a
lot of time can be spent on building tools to assist with the automation (ie. writing a script).

Examples:

- Logging into the AWS console and manually provisioning servers.
- SSH into each server to apply system upgrades.

We could save some time by writing some scripts and it'll get the job done and this will generally work fine at a small scale.

The manual way becomes tedious and error prone when we start to introduce the
need to manage more systems in many different environments and need to create
copies of an environment that are all similarly configured (ie. dev/staging/
prod).

Tracking changes to a system can also be challenging as there might not be any
sort of audit trail to refer back to.

Pros:

- Fast

Cons:

- Tedious when repeating
- Error prone when repeating
- Hard to determine the state of the world

### Configuration Management

As we move away from the manual way, we might start to look at configuration
management tools.

Popular choices include:

- Chef
- Puppet
- Salt
- Ansible

Configuration management is all about maintaining consistency and tracking
changes. Configuration management also makes it easier to audit and maintain
compliance.

Essentially, configuration management allows you to declare the state of the world and it will update the targeted systems to match it.

Pros:

- Consistency
- Auditing
- Compliance

Cons:

- Not necessarily needed for containerized application deployments
- Rolling back is not free

### Infrastructure as Code

Infrastructure as code is all about having a single source of truth that serves
as a blueprint for what we want the infrastructure to look like.

Templates are written using a DSL that describes the desired resources and their relationships.

Examples:

- CloudFormation
- Terraform

Pros:

- Consistency
- Auditing
- Compliance
- Rollbacks*

Cons:

- Potentially can destroy a resource unintentionally
- Can be slower if it can't update resources in place

## CloudFormation Templating Languages

CloudFormation supports two flavors of templating JSON and YAML.

### JSON

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Single EC2 Instance",
  "Resources": {
    "MyEC2Instance": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "ImageId": "ami-79fd7eee",
        "KeyName": "testkey",
        "InstanceType": "t2.micro"
      }
    }
  }
}
```

At the bare minimal, the JSON template contains top level keys for
`AWSTemplateFormatVersion`, `Description`, and `Resources`.

Inside `Resources` are uniquely named keys that are mapped to specific AWS
resources.

### YAML

```yaml
---
AWSTemplateFormatVersion: '2010-09-09'
Description: Single EC2 Instance
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-79fd7eee
      KeyName: testkey
      InstanceType: t2.micro
```

CloudFormation has recently offered support for YAML offering a more human
option for writing templates. Whitespace has significance with YAML, rather
than use curly braces we can nest keys by indenting two spaces at each level.

### Alternatives

Generally, most CloudFormation users will probably not be writing their
templates in JSON by hand, but utilize some sort of scripting language to
generate the templates to JSON.

Such options include:

- [cfndsl](https://github.com/cfndsl/cfndsl) - Ruby
- [troposphere](https://github.com/cloudtools/troposphere) - Python

But ultimately, we can use whatever we want as long it generates valid JSON or
YAML.

## CloudFormation Resources

A CloudFormation template represents a *stack*, and the components within the
*stack* are known as *resources*.

In this workshop, as a class we will compose a static template. It will cover the creation of a auto scaling group with an associated security group(s). We assume a VPC and subnets already exist.

If you are lost, it is OK. This part of the workshop is more about doing.

The initial template should look like this:

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Workshop stack.",
  "Resources": {}
}
```

```yaml
---
AWSTemplateFormatVersion: '2010-09-09'
Description: Workshop stack.
Resources:
```

The following resources below, will be nested under the parent level
`Resources` key.

### Security Groups

> A security group acts as a virtual firewall that controls the traffic for one or more instances. When you launch an instance, you associate one or more security groups with the instance. You add rules to each security group that allow traffic to or from its associated instances. You can modify the rules for a security group at any time; the new rules are automatically applied to all instances that are associated with the security group. When we decide whether to allow traffic to reach an instance, we evaluate all the rules from all the security groups that are associated with the instance.

**KEY FACT:**

> Security groups are stateful — if you send a request from your instance, the response traffic for that request is allowed to flow in regardless of inbound security group rules. Responses to allowed inbound traffic are allowed to flow out, regardless of outbound rules.

For example, if you have a EC2 instance that needs to talk to MySQL, the
instance only needs to have a egress (outbound) rule to 3306. The MySQL server
would need a ingress (inbound) rule for 3306.

**Let's add a resource to our template:**

```json
  "WorkshopServerSg": {
    "Type": "AWS::EC2::SecurityGroup",
    "Properties": {
      "GroupDescription": "Workshop server security group.",
      "SecurityGroupIngress": [
        {
          "Description": "HTTP (all).",
          "IpProtocol": "tcp",
          "FromPort": 80,
          "ToPort": 80,
          "CidrIp": "0.0.0.0/0"
        }
      ],
      "SecurityGroupEgress": [
        {
          "Description": "HTTP (all).",
          "IpProtocol": "tcp",
          "FromPort": 80,
          "ToPort": 80,
          "CidrIp": "0.0.0.0/0"
        },
        {
          "Description": "MySQL (all).",
          "IpProtocol": "tcp",
          "FromPort": 3306,
          "ToPort": 3306,
          "CidrIp": "0.0.0.0/0"
        }
      ],
      "VpcId": "vpc-123456"
    }
  }
```

```yaml
  WorkshopServerSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Workshop server security group.
      SecurityGroupIngress:
      - Description: HTTP (all).
        IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      SecurityGroupEgress:
      - Description: HTTP (all).
        IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - Description: MySQL (all).
        IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        CidrIp: 0.0.0.0/0
      VpcId: vpc-123456
```

**NOTE:** By default a security group will allow all outbound traffic unless
you specify a rule.

**NOTE:** For simplicity we are being very lax by setting the `CIDR` to
`0.0.0.0/0`. In production environments it would be more strict.

Here, we have defined a `AWS::EC2::SecurityGroup` resource and have named it
`WorkshopServerSg`. We gave the group a description, and assigned it ingress
and egress rules. We also associate it to a VPC.

#### References

- [Security Groups for Your VPC](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html)
- [AWS::EC2::SecurityGroup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html)
- [EC2 Security Group Rule Property Type](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-rule.html)

### Launch Configuration

> A launch configuration is a template that an Auto Scaling group uses to launch EC2 instances. When you create a launch configuration, you specify information for the instances such as the ID of the Amazon Machine Image (AMI), the instance type, a key pair, one or more security groups, and a block device mapping. If you've launched an EC2 instance before, you specified the same information in order to launch the instance.

**KEY FACT**:

> You can specify your launch configuration with multiple Auto Scaling groups.

> However, you can only specify one launch configuration for an Auto Scaling group at a time, and you can't modify a launch configuration after you've created it.

A launch configuration is immutable and can be assigned to multiple auto scaling groups, but a auto scaling group can only use one launch configuration.

**Let's add a resource to our template:**

```json
  "WorkshopLaunchConfig": {
    "Type": "AWS::AutoScaling::LaunchConfiguration",
    "Properties": {
      "KeyName": "key-123456",
      "ImageId": "ami-123456",
      "SecurityGroups": [],
      "InstanceType": "t2.micro"
    }
  }
```

```yaml
  WorkershopLaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      KeyName: key-123456
      ImageId: ami-123456
      SecurityGroups: []
      InstanceType: t2.micro
```

In our launch configuration, we've added a key name (for ssh), the AMI we want
to use, and the instance type.

#### References

- [Launch Configurations](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchConfiguration.html)
- [AWS::AutoScaling::LaunchConfiguration](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-as-launchconfig.html)

### Auto Scaling Group

> An Auto Scaling group contains a collection of EC2 instances that share similar characteristics and are treated as a logical grouping for the purposes of instance scaling and management. For example, if a single application operates across multiple instances, you might want to increase the number of instances in that group to improve the performance of the application, or decrease the number of instances to reduce costs when demand is low. You can use the Auto Scaling group to scale the number of instances automatically based on criteria that you specify, or maintain a fixed number of instances even if a instance becomes unhealthy. This automatic scaling and maintaining the number of instances in an Auto Scaling group is the core functionality of the Amazon EC2 Auto Scaling service.

A auto scaling group (asg) is collection of EC2 instances. The benefit of using
a ASG (even if its just limited to one instance) - is that is can provide self-
healing. If the cluster size drops due to the instance ending up in a undesired
state, the ASG can automatically remove the unhealthy instance and spawn a new
instance to replace it.

**Let's add a resource to our template:**

```json
  "WorkshopAsg": {
    "Type": "AWS::AutoScaling::AutoScalingGroup",
    "Properties": {
      "DesiredCapacity": "0",
      "LaunchConfigurationName": {
        "Ref": "WorkshopLaunchConfig"
      },
      "MaxSize": "1",
      "MinSize": "0",
      "VPCZoneIdentifier": [
        "subnet-123456"
      ]
    }
  }
```

```yaml
WorkshopAsg:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    DesiredCapacity: '0'
    LaunchConfigurationName:
      Ref: WorkshopLaunchConfig
    MaxSize: '1'
    MinSize: '0'
    VPCZoneIdentifier:
    - subnet-123456
```

Here we create a auto scaling group, where we set the desired capacity to 0 (do not create any instance), we use the ref function that points to the
`WorkshopLaunchConfig`, set the max size to 1 and the min size to 0, and we
set the subnets.

#### References

- [Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)

### EC2 Instance

> Amazon Elastic Compute Cloud (Amazon EC2) provides scalable computing capacity in the Amazon Web Services (AWS) cloud. Using Amazon EC2 eliminates your need to invest in hardware up front, so you can develop and deploy applications faster. You can use Amazon EC2 to launch as many or as few virtual servers as you need, configure security and networking, and manage storage. Amazon EC2 enables you to scale up or down to handle changes in requirements or spikes in popularity, reducing your need to forecast traffic.

We don't need to do anything here because our auto scaling group will create
EC2 instances for us. This is section is only here for reference.

#### References

- [What is Amazon EC2?](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

## CloudFormation Features

To keep things simple, we will only cover a few features CloudFormation
templates supports.

#### References

- [What is AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
- [Template Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-reference.html)

### Parameters

> Parameters enable you to input custom values to your template each time you create or update a stack.

Why are parameters useful? It allows us to reuse templates across different
environments, as we can pass in parameters to specify customizations.

`Parameters` is a top level key, where it maps a parameter name to a parameter
object.

**Here's an example:**

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "Workshop stack.",
  "Parameters": {
    "Project": {
      "Type": "String",
      "Description": "This is a string!",
      "Default": "my_project_name"
    },
    "SecurityGroups": {
      "Type": "List<AWS::EC2::SecurityGroup::Id>",
      "Description": "Security groups",
      "Default": ["sg-123456"]
    }
  }
}
```

```yaml
---
AWSTemplateFormatVersion: '2010-09-09'
Description: Workshop stack.
Parameters:
  Project:
    Type: String
    Description: This is a string!
    Default: my_project_name
```

Review the references to learn more about parameters!

#### References

- [Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)

### Resource Types

Review the references to see the resources CloudFormation supports creating.

Custom resources can be created through Lambda Functions.

This section is for reference only.

#### References

- [AWS Resource Types Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

### Functions

CloudFormation also supports several functions, such as being able to reference
another resource we've created, or splitting a list, joining, a list, etc.

Review the references to see them all.

For this workshop, we will focus on `Ref`. You saw this function back when we
were writing the snippet for the auto scaling group. We used a `Ref` function
to pass in the name of the launch configuration.

```json
      ...
      "LaunchConfigurationName": {
        "Ref": "WorkshopLaunchConfig"
      },
```

```yaml
    ...
    LaunchConfigurationName:
      Ref: WorkshopLaunchConfig
```

#### References

- [Intrinsic Function Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)
- [Ref](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)

## Lab

Write a CloudFormation template that does the following:

- `AWSTemplateFormatVersion` defined to `2010-09-09`.
- `Description` defined to `Lab stack.`.
- Accepts a parameter called `DesiredCapacity`, that is a `Number`, and defaults to 0.
- Accepts a parameter called `MinSize`, that is a `Number`, and defaults to 0.
- Accepts a parameter called `MaxSize`, that is a `Number`, that defaults to 1.
- Accepts a parameter called `AmiId`, that is a `AWS::EC2::Image::Id`, that defaults to `ami-f2d3638a `.
- Accepts a parameter called `InstanceType`, that is a `String`, that defaults to `t2.micro`. It should restrict the options to `["t2.micro", "t2.medium"]`.
- Accepts a parameter called `VpcId`, that is a `AWS::EC2::VPC::Id`.
- Accepts a parameter called `DefaultKeyPair` that is a `AWS::EC2::KeyPair::KeyName`.
- Accepts a parameter called `SubnetIds`, that is a `List<AWS::EC2::Subnet::Id>`.
- Creates a security group named `LabSg` which has:
  + Egress rule to MySQL (3306) on CIDR block `0.0.0.0/0`.
  + Egress rule to HTTP (80) on CIDR block `0.0.0.0/0`.
  + Egress rule to HTTPS (443) on CIDR block `0.0.0.0/0`.
  + Ingress rule to HTTP (80) on CIDR block `0.0.0.0/0`.
  + Ingress rule to HTTP (443) on CIDR block `0.0.0.0/0`.
- Create a launch configuration named `LabLaunchConfig` which:
  + References `DefaultKeyPair`
  + References `AmiId`
  + References `LabSg`
  + References `InstanceType`
- Create a auto scaling group named `LabAsg` which:
  + References `DesiredCapacity`
  + References `LabLaunchConfig`
  + References `MaxSize`
  + References `MinSize`
  + References `SubnetIds`
