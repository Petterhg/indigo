---
title: "WIP: Deploying a scalable Flask API using AWS CloudFormation, Fargate and Python - Part 2"
layout: post
date: 2018-10-05
image: /assets/images/markdown.jpg
headerImage: false
tag:
- aws
- ecs
- cloudformation
- nlb
- dmz
- python
star: true
category: blog
author: petter
description: A demo on a private ECS Fargate cluster deployed using Troposphere
---
## What we will do
Okey so in the [previous post][1] I deployed a Flask API with ECS/Fargate and load balanced it using an application load
balancer. Now we are going to through an alternative approach that solves the problem of only exposing _some_ endpoints
in your Flask app to the internet, while having others for internal purposes. Now, there are many ways you could do this that are maybe smarter even then the way presented here. You could for
example split up your Flask app in two or set up rules on the ALB listener.  
But for the purpose of awesomeness and variation we will now implement an API Gateway and switch over to a network
load balancer.

So the setup scenario for this post is: We've got a Flask app that exposes some endpoints.
What these endpoints do is still irrelevant, we are just concerned with exposing these services, and now we also 
want to pick and choose what endpoints that are accessible from the world wide web (WWW). We also have _other_ AWS 
services in the same VPC that needs access to these private endpoints. So now, let's just assume we already have VPC
up and running and in this we have databases, lambdas, etc. 
<img src="/assets/images/aws/AWS-network-NLB.jpg">

## Let's get "coding"!
This time I will go a little bit more modal and pythonic in the file structure. I will use _multiple_ files passing
the template object around between functions. I know. Heavy stuff.  
Ok let's start with the "main" file that will be called:

{% highlight python %}
from troposphere import Template, Tags
from troposphere.ecs import Cluster
from troposphere.ec2 import SecurityGroup, SecurityGroupRule
from troposphere.s3 import Bucket
import boto3

from repository import get_repository
from network import create_subnets
from routing import create_routing
from load_balancer import create_load_balancer
from api_gateway import create_api_gateway
from ecs_task import create_task
from service import create_service


def define_stack():

    t = Template()

    repository_name = "my_flask_repo"
    repository_uri, image_latest = get_repository(repository_name)

    ec2_client = boto3.client('ec2')
    igw = ec2_client.describe_internet_gateways()['InternetGateways'][0]['InternetGatewayId']
    vpc = ec2_client.describe_vpcs(Filters=[{'Name': 'isDefault', 'Values': ['true']}])['Vpcs'][0]
    vpc_id = pmm_vpc['VpcId']
    vpc_cidr_block = pmm_vpc['CidrBlockAssociationSet']
    subnet_cidr_sizes = ['24', '24', '24', '24']
    create_subnets(t, vpc_id=vpc_id, vpc_cidr_block=vpc_cidr_block, subnet_cidr_sizes=subnet_cidr_sizes)

    create_routing(t, vpc_id, igw)

    t.add_resource(SecurityGroup(
        "FargateSecurityGroup",
        GroupDescription="NLB Security Group",
        VpcId=vpc_id,
        Tags=Tags(Name='NlbSecurityGroup'),
        SecurityGroupIngress=[
            SecurityGroupRule(
                IpProtocol="-1",
                CidrIp="0.0.0.0/0"
            )]
    ))

    create_load_balancer(t, vpc_id=vpc_id)

    stage_name = 'v1'
    create_api_gateway(t, stage_name=stage_name)

    t.add_resource(Cluster(
        "ECSCluster",
        ClusterName='ECSCluster'
    ))

    image = f"{repository_uri}:{image_latest}"
    create_task(t, bucket_uri=bucket_uri, image=image)

    create_service(t)

    return t
{% endhighlight %}

As you can see above, I now use the [CidrFindr][2] project that I talked about in the previous post in order to 
find available subnets in the already existing VPC. As you can see as, well I make good use of the `boto3` module 
that lets me query AWS for resources.  
Okey, let's go from top to bottom. This time, we will get the repository URI and the latest image from an already
existing repo hosting our app:

{% highlight python %}
import boto3


def get_repository(repository_name):

    container_registry = boto3.client('ecr')
    repository_uri = container_registry.describe_repositories(
        repositoryNames=[repository_name])['repositories'][0]['repositoryUri']
    images = container_registry.describe_images(repositoryName=repository_name)
    image_latest = sorted(images['imageDetails'], key=lambda k: k['imagePushedAt'],
                              reverse=True)[0]['imageTags'][0]
    
    return repository_uri, image_latest
{% endhighlight %}

Now we move on to creating the subnets for our application architecture. As you can see from the image, it's going
to be the same setup as the last time - two public ones and two private ones. And we are still routing traffic
from the private subnets with NAT's placed in the public subnets. 

{% highlight python %}
import boto3
import Cidrfindr.cidrfindr as cidrfindr
import Cidrfindr.lambda_utils as lambda_utils
from troposphere import Tags, Template
from troposphere.ec2 import Subnet


def create_subnets(t: Template, vpc_id=None, vpc_cidr_block=None, subnet_cidr_sizes=None):

    parsed_sizes = tuple(map(lambda_utils.parse_size, subnet_cidr_sizes))

    try:
        vpc_cidrs = [cidr_block_association["CidrBlock"] for cidr_block_association in vpc_cidr_block]
    except Exception as e:
        raise e

    ec2_client = boto3.client('ec2')
    vpc_existing_cidrs = [subnet["CidrBlock"] for subnet in
                          ec2_client.describe_subnets(Filters=[{"Name": "vpc-id", "Values": [vpc_id]}])["Subnets"]]
    findr = cidrfindr.CidrFindr(networks=vpc_cidrs, subnets=vpc_existing_cidrs)
    try:
        subnet_cidrs = [findr.next_subnet(size) for size in parsed_sizes]
    except cidrfindr.CidrFindrException as e:
        raise e

    t.add_resource(Subnet(
        "SubnetEuWest1a",
        AvailabilityZone='eu-west-1a',
        VpcId=vpc_id,
        CidrBlock=subnet_cidrs[0],
        Tags=Tags(Name='SubnetEuWest1a')
    ))

    t.add_resource(Subnet(
        "SubnetEuWest1b",
        AvailabilityZone='eu-west-1b',
        VpcId=vpc_id,
        CidrBlock=subnet_cidrs[1],
        Tags=Tags(Name='SubnetEuWest1b')
    ))

    t.add_resource(Subnet(
        "SubnetPublicEuWest1a",
        AvailabilityZone='eu-west-1a',
        VpcId=vpc_id,
        MapPublicIpOnLaunch=True,
        CidrBlock=subnet_cidrs[2],
        Tags=Tags(Name='SubnetPublicEuWest1a')
    ))

    t.add_resource(Subnet(
        "SubnetPublicEuWest1b",
        AvailabilityZone='eu-west-1b',
        VpcId=vpc_id,
        MapPublicIpOnLaunch=True,
        CidrBlock=subnet_cidrs[3],
        Tags=Tags(Name='SubnetPublicEuWest1b')
    ))

{% endhighlight %}

As you can see above, we are now just passing on the template object `t` into this function and continue adding on
resources. One of the best and worse features of high-level languages I guess. As long as you know what objects are
mutable and not.  
Anyways, let's not waste time hating on Python syntax. Next up - routing! 


[1]: https://petterhg.github.io/aws-ecs-fargate-alb-cloudformation/
[2]: https://github.com/aws-samples/aws-cidr-finder
