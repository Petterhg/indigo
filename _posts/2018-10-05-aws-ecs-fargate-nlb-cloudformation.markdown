---
title: "Deploying a scalable Flask API using AWS CloudFormation, Fargate and Python - Part 2"
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

{% highlight python %}
from troposphere.ec2 import RouteTable, Route, SubnetRouteTableAssociation, EIP, NatGateway
from troposphere import Ref, GetAtt, Template


def create_routing(t: Template, vpc_id, igw):

    eip_1a = t.add_resource(EIP('NatEip1a', Domain="vpc"))
    eip_1b = t.add_resource(EIP('NatEip1b', Domain="vpc"))

    public_route_table = t.add_resource(RouteTable(
        'PublicRouteTable',
        VpcId=vpc_id
    ))

    t.add_resource(Route(
        'PublicDefaultRoute',
        RouteTableId=Ref(public_route_table),
        DestinationCidrBlock='0.0.0.0/0',
        GatewayId=igw
    ))

    t.add_resource(SubnetRouteTableAssociation(
        'PublicRouteAssociation1b',
        SubnetId=Ref('SubnetPublicEuWest1b'),
        RouteTableId=Ref(public_route_table),
        DependsOn='PublicRouteTable'
    ))

    t.add_resource(SubnetRouteTableAssociation(
        'PublicRouteAssociation1a',
        SubnetId=Ref('SubnetPublicEuWest1a'),
        RouteTableId=Ref(public_route_table),
        DependsOn='PublicRouteTable'
    ))

    nat_1a = t.add_resource(NatGateway(
        'Nat1a',
        AllocationId=GetAtt(eip_1a, 'AllocationId'),
        SubnetId=Ref('SubnetPublicEuWest1a'),
        DependsOn='NatEip1a'
    ))

    nat_1b = t.add_resource(NatGateway(
        'Nat1b',
        AllocationId=GetAtt(eip_1b, 'AllocationId'),
        SubnetId=Ref('SubnetPublicEuWest1b'),
        DependsOn='NatEip1b'
    ))

    private_route_table_1a = t.add_resource(RouteTable(
        'PrivateRouteTable1a',
        VpcId=vpc_id
    ))

    t.add_resource(Route(
        'NatRoute1a',
        RouteTableId=Ref(private_route_table_1a),
        DestinationCidrBlock='0.0.0.0/0',
        NatGatewayId=Ref(nat_1a),
        DependsOn='Nat1a'
    ))

    t.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociation1a',
        SubnetId=Ref('SubnetEuWest1a'),
        RouteTableId=Ref(private_route_table_1a),
    ))

    private_route_table_1b = t.add_resource(RouteTable(
        'PrivateRouteTable1b',
        VpcId=vpc_id
    ))

    t.add_resource(Route(
        'NatRoute1b',
        RouteTableId=Ref(private_route_table_1b),
        DestinationCidrBlock='0.0.0.0/0',
        NatGatewayId=Ref(nat_1b),
    ))

    t.add_resource(SubnetRouteTableAssociation(
        'PrivateRouteAssociation1b',
        SubnetId=Ref('SubnetEuWest1b'),
        RouteTableId=Ref(private_route_table_1b),
    ))
{% endhighlight %}
Nothing we haven't seen before here. So let's continue right away to creating our NLB.
This time, as mentioned, it will be placed inside the private subnets. This practically means
that no one can reach this load balancer without our specific permission. And that's
exactly what we want in this scenario, since I have endpoints that do internal work, and
then some that serves content to the public.

{% highlight python %}
from troposphere import Ref, Template, Output, Export, GetAtt, Join
from troposphere.elasticloadbalancingv2 import LoadBalancer, TargetGroup, Listener, Action, Matcher


def create_load_balancer(t: Template, vpc_id=None):
    load_balancer = t.add_resource(LoadBalancer(
        "LoadBalancer",
        Scheme="internal",
        Subnets=[Ref('SubnetEuWest1a'), Ref('SubnetEuWest1b')],
        Type='network'
    ))

    target_group = t.add_resource(TargetGroup(
        "TargetGroup",
        Name="TargetGroup",
        HealthCheckIntervalSeconds="30",
        HealthCheckProtocol="HTTP",
        HealthyThresholdCount="3",
        HealthCheckPort="80",
        HealthCheckPath="/",
        Matcher=Matcher(HttpCode="200-399"),
        Port="80",
        Protocol="TCP",
        UnhealthyThresholdCount="3",
        TargetType="ip",
        VpcId=vpc_id,
    ))

    t.add_resource(Listener(
        "Listener",
        Port="80",
        Protocol="TCP",
        LoadBalancerArn=Ref(load_balancer),
        DefaultActions=[Action(
            Type="forward",
            TargetGroupArn=Ref(target_group)
        )],
        DependsOn=['LoadBalancer', 'TargetGroup']
    ))

    t.add_output(Output(
        'LoadbalancerDNS',
        Description='The DNS of the internal load balancer',
        Export=Export(
            name=Join('-', (Ref('AWS::StackName'), 'PmmLoadbalancerDNS'))
        ),
        Value=GetAtt(load_balancer, 'DNSName')
    ))
{% endhighlight %}
Maybe pointing out the obvious, but setting the scheme to `internal` in the LoadBalancer 
resource is what makes sure the NLB doesn't get assigned a public IP. The target group
is used to specify where the load balancers target resources are placed. Here we also need to specify
the endpoint where the LB makes health checks, and how often these occur. I recommend setting the health
check interval pretty high in case of a NLB. This might sound idiotic, but the NLB actually does not guarantee
that this interval is held, and can easily send out 50 requests per health check interval, maybe swamping
your app.  
The Listener is what actually forwards the traffic to the target group. Here you set the port for incoming traffic,
in our case 80. All traffic hitting the LB URL on port 80 will be routed forward to the specified target group.  

Ok next up is the API Gateway. Here we will only add one endpoint, which will be the only one that is exposed
publicly. Because of some limitations in the troposphere library, I have divided the api gateway definition into a python file
and a swagger file to get the full capacity of AWS.

{% highlight python %}
from troposphere import Ref
from troposphere.apigateway import (
    VpcLink,
    RestApi,
    EndpointConfiguration,
    Deployment,
    Stage,
    BasePathMapping,
)

import yaml
import os


def create_api_gateway(t, domain_name=None, stage_name=None):
    t.add_resource(VpcLink(
        "ApiGatewayVpcLink",
        Description="Links Rest Api with Load balancer",
        Name="ApigatewayToNLB",
        TargetArns=[Ref('LoadBalancer')],
        DependsOn="LoadBalancer"
    ))

    with open(os.path.join(os.path.dirname(__file__), 'api_gateway.yml'), 'r') as rest_api_spec:
        rest_api = t.add_resource(RestApi(
            'ApiGateway',
            EndpointConfiguration=EndpointConfiguration(Types=['EDGE']),
            Body=yaml.load(rest_api_spec),
            DependsOn='ApiGatewayVpcLink'
        ))

    deployment = t.add_resource(Deployment(
        f'ApiGatewayDeployment{stage_name}',
        RestApiId=Ref(rest_api),
        DependsOn='ApiGateway',
    ))

    stage = t.add_resource(Stage(
        f'ApiGatewayStage{stage_name}',
        StageName=stage_name,
        RestApiId=Ref(rest_api),
        DeploymentId=Ref(deployment),
    ))

    t.add_resource(BasePathMapping(
        'BasePathMapping',
        DomainName=domain_name,
        Stage=Ref(stage),
        RestApiId=Ref(rest_api),
        DependsOn=[rest_api, stage]
    ))
{% endhighlight %}
As we can see here, I basically define the whole gateway from a file `api_gateway.yml` which we will look
at now. The only thing worth noticing here, is the VPC Link. This is basically a service that allows the public
API to route traffic into the private subnet, ie linking the gateway with the internal NLB, defined earlier.

{% highlight yaml %}
swagger: 2.0
info:
  title: person-identifier
basePath: /
schemes:
  - http
paths:
  /{person}/{firstname}:
    get:
      parameters:
        - name: person
          in: path
          required: true
          type: string
        - name: firstname
          in: path
          required: true
          type: string
      x-amazon-apigateway-integration:
        type: http_proxy
        connectionType: VPC_LINK
        connectionId:
          Ref: ApiGatewayVpcLink
        httpMethod: GET
        requestParameters:
          integration.request.path.person: method.request.path.person
          integration.request.path.firstname: method.request.path.firstname
        uri:
          Fn::Join: ["", ["http://", {"Fn::GetAtt": [LoadBalancer, 'DNSName']}, "/{person}/{firstname}"]]
{% endhighlight %}
This is where the real work is done. Basically, we are saying "If you are not doing a GET to find a persons
first name, you are going to get a Permission Denied". The `http_proxy` method allows us to route the
raw request data through the gateway into the NLB and the flask app. This is somewhat a miss use of the API
Gateway, since it's a pretty powerful routing tool that can assign and verify headers, encode/decode to binary
etc.  
Finally, the Fargate task and Service looks just like in the [previous post][1], so please copy paste from there
to get the full working example!


[1]: https://petterhg.github.io/aws-ecs-fargate-alb-cloudformation/
[2]: https://github.com/aws-samples/aws-cidr-finder
