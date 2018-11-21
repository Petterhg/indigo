---
title: "Deploying a scalable Flask API using AWS CloudFormation, Fargate and Python - Part 1"
layout: post
date: 2018-09-28
headerImage: false
tag:
- aws
- ecs
- cloudformation
- alb
- python
star: true
category: blog
author: petter
description: A demo on a private ECS Fargate cluster deployed using Troposphere
---
## What we will do
So the setup scenario for this post is: We've got a Flask app that exposes some endpoints.
What these endpoints do is irrelevant for now, we are just concerned with letting other services,
people on the internet etc. access these endpoints without compromising on security. Furthermore,
we are expecting this Flask service to handle a varied load, so we want to be able to load balance it
over multiple containers and in the future also attach an auto-scaler to handle all the peak loads for us.<br/>

So in this demo I will show how to deploy a complete AWS Elastic Container Service, using CloudFormation. 
ECS is like AWS own "Managed Kubernetes", and CloudFormation is their in-house Terraform project.
Basically, the way we are going to achieve this, is by creating a stack of resources in a Python file
that we link together, using referrals and dependencies within the script. The end architecture we want to 
accompish here is:
- ECS cluster deployed over two private subnets, where all tasks are sealed off from the internet
- Public subnets that routes traffic from the private subnets
- An application level load balancer that sends requests to the ECS tasks in a Round Robin manner

<img src="/assets/images/aws/AWS-network-ALB.jpg" alt="Smiley face">

## Let's get "coding"!
So not to scare anyone off, but I'm just going to start with showing the shitstorm of modules that
we are going to import for this template:
{% highlight python %}
1 import boto3
2 import os
3 import yaml
4 from troposphere import Template, Ref, GetAtt, Tags, Join, Output, FindInMap
5 from troposphere.iam import Role, PolicyType, Policy
6 from troposphere.ecs import (Service, TaskDefinition, ContainerDefinition, PortMapping,
7                              Cluster, NetworkConfiguration, AwsvpcConfiguration,
8                              DeploymentConfiguration, Environment)
9 from troposphere.ecs import LoadBalancer as ecsLoadBalancer
10 from troposphere.ecr import Repository
11 from troposphere.ec2 import (SecurityGroup, SecurityGroupRule, SecurityGroupIngress, VPC, Subnet,
12                              InternetGateway, VPCGatewayAttachment, Route, RouteTable,
13                              SubnetRouteTableAssociation, EIP, NatGateway) 
14 from troposphere.elasticloadbalancingv2 import (LoadBalancer, TargetGroup, Matcher,
15                                                Listener, ListenerRule, Action, Condition)
16 from troposphere.rds import DBSubnetGroup, DBInstance
17 from troposphere.s3 import Bucket
18 from troposphere.apigateway import (RestApi, VpcLink, Stage, Deployment,
19                                     Resource, Method, Account, MethodSetting, Integration,
20                                     EndpointConfiguration)
{% endhighlight %}

Remember to breath... There are about 500 lines of code left.

Ok, so lets start defining our stack. First off, this is a container service, and we are going to run
our own private images. So that means we need our own registry. For this I choose to go with the 
native AWS ECR, but you could of course deploy your own Docker container registry as a task in the cluster,
which is slightly more hardcore.
{% highlight python %}
23 def define_stack(ctx):
24     
25     t = Template()
26     
27     # --- REPOSITORY SETUP AND IMAGE PULL ---
28     repository_name = 'my_repo_name'
29     container_registry = boto3.client('ecr')
30     
31     repository_uri = container_registry.describe_repositories(
32         repositoryNames=[repository_name])['repositories'][0]['repositoryUri']
33     if not repository_uri:
34         t.add_resource(Repository(
35             'ECR',
36             RepositoryName='flask',
37             RepositoryPolicyText={
38                 "Version": "2008-10-17",
39                 "Statement": [
40                     {   
41                         "Effect": "Allow",
42                         "Action": [
43                                 "ecr:GetAuthorizationToken",
44                                 "ecr:BatchCheckLayerAvailability",
45                                 "ecr:GetDownloadUrlForLayer",
46                                 "ecr:GetRepositoryPolicy",
47                                 "ecr:DescribeRepositories",
48                                 "ecr:ListImages",
49                                 "ecr:DescribeImages",
50                                 "ecr:BatchGetImage",
51                                 "ecr:InitiateLayerUpload",
52                                 "ecr:UploadLayerPart",
53                                 "ecr:CompleteLayerUpload",
54                                 "ecr:PutImage"
55                         ],
56                         "Resource": "*"
57                     }
58                 ]
59             }
60         ))
61     
62     try:
63         images = container_registry.describe_images(
64             repositoryName=repository_name)
65         image_latest_tag = sorted(pmm_images['imageDetails'],
66                                       key=lambda k: k['imagePushedAt'], reverse=True)[0]['imageTags'][0]
67     except Exception as e:
68         raise e
{% endhighlight %}

So now that we have a registry (which sort of becomes a side show in this setup), let's move on to defining
our network stack. A Virtual Private Cloud (VPC) is just a resource isolation thing in AWS, the equivalent
of a Resource Group in Azure. Note here that we are generating two private and two public subnets within this 
VPC, and we are spreading them around Europe to have a higher availability in the cluster. Btw, if you think
it's lame to hardcode the subnet CIDR's check out [this project][2].
{% highlight python %}
70     # --- NETWORK SETUP WITH PRIVATE AND PUBLIC SUBNETS ---
71     vpc = t.add_resource(VPC(
72         "Vpc",
73         CidrBlock="172.31.0.0/16",
74     ))
75     
76     subnet_eu_west_1a = t.add_resource(Subnet(
77         "SubnetEuWest1a",
78         AvailabilityZone='eu-west-1a',
79         VpcId=Ref(vpc),
80         CidrBlock='172.31.112.0/24',
81         Tags=Tags(Name='SubnetEuWest1a')
82     ))  
83     
84     subnet_eu_west_1b = t.add_resource(Subnet(
85         "SubnetEuWest1b",
86         AvailabilityZone='eu-west-1b',
87         VpcId=Ref(vpc),
88         CidrBlock='172.31.113.0/24',
89         Tags=Tags(Name='SubnetEuWest1b')
90     ))  
91     
92     subnet_public_eu_west_1a = t.add_resource(Subnet(
93         "SubnetPublicEuWest1a",
94         AvailabilityZone='eu-west-1a',
95         VpcId=Ref(vpc),
96         MapPublicIpOnLaunch=True,
97         CidrBlock='172.31.114.0/24',
98         Tags=Tags(Name='SubnetPublicEuWest1a')
99     ))  
100     
101     subnet_public_eu_west_1b = t.add_resource(Subnet(
102         "SubnetPublicEuWest1b",
103         AvailabilityZone='eu-west-1b',
104         VpcId=Ref(vpc),
105         MapPublicIpOnLaunch=True,
106         CidrBlock='172.31.115.0/24',
107         Tags=Tags(Name='SubnetPublicEuWest1b')
108     )) 
{% endhighlight %}

Ok, next up is making sure that the private subnets can send data _out_ on the internet. This will let
the Flask app send responses back, and the ECS Task to "answer" the load balancers health checks. To do 
this we have to create routing tables for both the private and public subnets.  
So in AWS, you need to manually create an Internet Gateway and then, attach this gateway to the VPC in order for any
resources within the VPC to have internet access. Then we setup the public subnets routing tables, 
sort of like scripting your _iptables_ if you like. Finally we have to associate the routing tables with our
subnets. Observe here the `DependsOn` variable - On deployment, CloudFormation will create resources in
whatever order it may seem fit, if not otherwise instructed. 
{% highlight python %}
110     # --- INTERNET GATEWAY, PUBLIC ROUTING ---
111 
112     igw = t.add_resource(InternetGateway('InternetGateway', ))
113 
114     net_gw_vpc_attachment = t.add_resource(VPCGatewayAttachment(
115         "NatAttachment",
116         VpcId=Ref(vpc),
117         InternetGatewayId=Ref(igw),
118         DependsOn=['InternetGateway']
119     ))  
120         
121     public_route_table = t.add_resource(RouteTable(
122         'PublicRouteTable',
123         VpcId=Ref(vpc)
124     
125     public_route = t.add_resource(Route(
126         'PublicDefaultRoute',
127         RouteTableId=Ref(public_route_table),
128         DestinationCidrBlock='0.0.0.0/0',
129         GatewayId=Ref(igw), 
130         DependsOn='NatAttachment'
131     ))  
132     
133     public_route_association1b = t.add_resource(SubnetRouteTableAssociation(
134         'PublicRouteAssociation1b',
135         SubnetId=Ref(subnet_public_eu_west_1b),
136         RouteTableId=Ref(public_route_table),
137         DependsOn='PublicRouteTable'
138     ))  
139     
140     public_route_association1a = t.add_resource(SubnetRouteTableAssociation(
141         'PublicRouteAssociation1a',
142         SubnetId=Ref(subnet_public_eu_west_1a),
143         RouteTableId=Ref(public_route_table),
144         DependsOn='PublicRouteTable'
145     )) 
 {% endhighlight %}
 
 
Now let's make some NAT's for the public subnets. But first off, we want to give these NAT gateways a static
IP, with the AWS Elastic IP resource. When that is done, we can create the routing tables for the private
subnets and make sure their traffic is sent through these NAT's.
 {% highlight python %}                                                                                          X 
147     # --- EIP, NAT, PRIVATE ROUTING ---
148     eip1a = t.add_resource(EIP(
149         'NatEip1a',
150         Domain="vpc",
151         DependsOn='NatAttachment'
152     ))
153 
154     eip1b = t.add_resource(EIP(
155         'NatEip1b',
156         Domain="vpc",
157         DependsOn='NatAttachment'
158     ))
159 
160     nat1a = t.add_resource(NatGateway(
161         'Nat1a',
162         AllocationId=GetAtt(eip1a, 'AllocationId'),
163         SubnetId=Ref(subnet_public_eu_west_1a),
164         DependsOn='NatAttachment'
165     ))
166 
167     nat1b = t.add_resource(NatGateway(
168         'Nat1b',
169         AllocationId=GetAtt(eip1b, 'AllocationId'),
170         SubnetId=Ref(subnet_public_eu_west_1b),
171         DependsOn='NatAttachment'
172     ))
173 
174     private_route_table_1a = t.add_resource(RouteTable(
175         'PrivateRouteTable1a',
176         VpcId=Ref(vpc)
177     ))  
178     
179     t.add_resource(Route(
180         'NatRoute1a',
181         RouteTableId=Ref(private_route_table_1a),
182         DestinationCidrBlock='0.0.0.0/0',
183         NatGatewayId=Ref(nat1a),
184     ))
185     
186     private_route_association_1a = t.add_resource(SubnetRouteTableAssociation(
187         'PrivateRouteAssociation1a',
188         SubnetId=Ref(subnet_eu_west_1a),
189         RouteTableId=Ref(private_route_table_1a),
190     ))
191     
192     private_route_table_1b = t.add_resource(RouteTable(
193         'PrivateRouteTable1b',
194         VpcId=Ref(vpc)
195     ))
196     
197     t.add_resource(Route(
198         'NatRoute1b',
199         RouteTableId=Ref(private_route_table_1b),
200         DestinationCidrBlock='0.0.0.0/0',
201         NatGatewayId=Ref(nat1b),
202     ))
203     
204     private_route_association_1b = t.add_resource(SubnetRouteTableAssociation(
205         'PrivateRouteAssociation1b',
206         SubnetId=Ref(subnet_eu_west_1b),
207         RouteTableId=Ref(private_route_table_1b),
208     ))
 {% endhighlight %}


Yet another part of setting up your AWS cluster is to create the security groups. I guess this is what would
be the actual _iptables_ scripting, since here we set the in- and outbound traffic rules for the resources
defined in those security groups. 
{% highlight python %}                                                                                          X 
210     # --- SECURITY GROUPS ---
211     fargate_container_security_group = t.add_resource(SecurityGroup(
212         "FargateSecurityGroup",
213         GroupDescription="PMM ALB Security Group",
214         VpcId=Ref(vpc), 
215         Tags=Tags(Name='FargateSecurityGroup'),
216     ))
217 
218     public_LB_SG = t.add_resource(SecurityGroup(
219         "PublicLBSG",
220         GroupDescription="Public LB SG",
221         VpcId=Ref(vpc), 
222         Tags=Tags(Name='PublicLBSG'),
223         SecurityGroupIngress=[
224             SecurityGroupRule(
225                 IpProtocol="-1",
226                 CidrIp="0.0.0.0/0",
227             )]
228     ))
229 
230     alb_public_ingress_security_group = t.add_resource(SecurityGroupIngress(
231         "AlbPublicIngressSecurity",
232         Description="Public ALB Ingress Security",
233         GroupId=Ref(fargate_container_security_group),
234         SourceSecurityGroupId=Ref(public_LB_SG),
235         IpProtocol='-1'
236     ))
 {% endhighlight %}
 
So with networking and security groups for the cluster established we can move on to something more fun, 
the load balancer. I mentioned that this is an Application Load Balancer (as in layer 7 of the OSI model),
which means it routes on HTTP requests. This load balancer is going to take requests from resources both
on the internet and in the VPC and round robin them onto the Flask containers.  
Furthermore, we are going to create a Target group (where the LB will send the requests), a Listener (that 
looks for incoming traffic and forwards it to the Target group) and a Rule for the Listener. Rules
allows us to specify actions dependant on parameters and request paths sent from the client. 
{% highlight python %}                                         
239     # --- PUBLIC LOAD BALANCER, TARGET GROUP, LISTENER & RULES ---
240     public_LB = t.add_resource(LoadBalancer(
241         "PublicALB",
242         Scheme="internet-facing",
243         Subnets=[Ref(subnet_public_eu_west_1a), Ref(subnet_public_eu_west_1b)],
244         SecurityGroups=[Ref(public_LB_SG)],
245         DependsOn='NatAttachment',
246         Type='application'
247     ))
248 
249     target_group_private = t.add_resource(TargetGroup(
250         "TargetGroupPrivate",
251         HealthCheckIntervalSeconds="30",
252         HealthCheckProtocol="HTTP",
253         HealthCheckTimeoutSeconds="10",
254         HealthyThresholdCount="2",
255         HealthCheckPort="80",
256         HealthCheckPath="/",
257         Matcher=Matcher(HttpCode="200-499"),
258         Port="80",
259         Protocol="HTTP",
260         UnhealthyThresholdCount="3",
261         TargetType="instance",
262         VpcId=Ref(vpc)
263     ))
264 
265     listener_public = t.add_resource(Listener(
266         "ListenerPublic",
267         Port="80",
268         Protocol="HTTP",
269         LoadBalancerArn=Ref(public_LB),
270         DefaultActions=[Action(
271             Type="forward",
272             TargetGroupArn=Ref(target_group_private)
273         )],
274         DependsOn="PublicALB"
275     ))
276 
277     listener_public_rule = t.add_resource(ListenerRule(
278         "ListenerRulePublic",
279         ListenerArn=Ref(listener_private),
280         Actions=[Action(
281             Type="forward",
282             TargetGroupArn=Ref(target_group_private)
283         )],
284         Conditions=[Condition(
285             Field="path-pattern",
286             Values=["*"]
287         )],
288         Priority=1,
289         DependsOn="PublicALB"
290     ))
{% endhighlight %}

This part is just boring. In order for certain services to have access to certain resources in AWS,
you need to assign the resource an AWS ROLE. That's all I'm going to say about it, the rest you can
read in the AWS documentation.
{% highlight python %}
292     # --- ROLES AND POLICIES ---
293     ecs_role = t.add_resource(Role(
294         "EcsService",
295         Path='/',
296         Policies=[Policy(
297             PolicyName='root',
298             PolicyDocument={
299                 'Version': '2012-10-17',
300                 'Statement': [{
301                     "Effect": "Allow",
302                     "Action": [
303                         "ecr:GetAuthorizationToken",
304                         "ecr:BatchCheckLayerAvailability",
305                         "ecr:GetDownloadUrlForLayer",
306                         "ecr:BatchGetImage",
307                         "ecr:DescribeRepositories",
308                         "ecr:ListImages",
309                         "ecr:DescribeImages",
310                         "logs:CreateLogStream",
311                         "logs:PutLogEvents",
312                         'ec2:AttachNetworkInterface',
313                         'ec2:CreateNetworkInterface',
314                         'ec2:CreateNetworkInterfacePermission',
315                         'ec2:DeleteNetworkInterface',
316                         'ec2:DeleteNetworkInterfacePermission',
317                         'ec2:Describe*',
318                         'ec2:DetachNetworkInterface',
319                         'elasticloadbalancing:DeregisterInstancesFromLoadBalancer',
320                         'elasticloadbalancing:DeregisterTargets',
321                         'elasticloadbalancing:Describe*',
322                         'elasticloadbalancing:RegisterInstancesWithLoadBalancer',
323                         'elasticloadbalancing:RegisterTargets'
324                     ],
325                     "Resource": "*"
326                 }]
327             }
328         )
329         ],
330         AssumeRolePolicyDocument={
331             'Statement': [{
332                 'Effect': 'Allow',
333                 'Principal': {'Service': ['ecs.amazonaws.com']},
334                 'Action': ["sts:AssumeRole"]
335             }]
336         }
337     ))
338 
339     task_execution_role = t.add_resource(Role(
340         "TaskExecutionRole",
341         Path='/',
342         Policies=[Policy(
343             PolicyName='root',
344             PolicyDocument={
345                 'Version': '2012-10-17',
346                 'Statement': [{
347                     "Effect": "Allow",
348                     "Action": [
349                         "ecr:GetAuthorizationToken",
350                         "ecr:BatchCheckLayerAvailability",
351                         "ecr:GetDownloadUrlForLayer",
352                         "ecr:BatchGetImage",
353                         "ecr:DescribeRepositories",
354                         "ecr:ListImages",
355                         "ecr:DescribeImages",
356                         "logs:CreateLogStream",
357                         "logs:PutLogEvents"
358                     ],
359                     "Resource": "*"
360                 }]
361             }
362         )
363         ],
364         AssumeRolePolicyDocument={
365             'Statement': [{
366                 'Effect': 'Allow',
367                 'Principal': {'Service': ['ecs-tasks.amazonaws.com']},
368                 'Action': ["sts:AssumeRole"]
369             }]
370         }
371     ))
372 
373     t.add_resource(PolicyType(
374         "FargateExecutionPolicy",
375         PolicyName="fargate-execution",
376         PolicyDocument={'Version': '2012-10-17',
377                         'Statement': [{
378                             'Action': [
379                                 'ecr:GetAuthorizationToken',
380                                 'ecr:BatchCheckLayerAvailability',
381                                 'ecr:GetDownloadUrlForLayer',
382                                 'ecr:BatchGetImage',
383                                 'logs:CreateLogStream',
384                                 'logs:PutLogEvents'
385                             ],
386                             'Resource': ['*'],
387                             'Effect': 'Allow'},
388                         ]},
389         Roles=[Ref(task_execution_role)],
390     ))
 {% endhighlight %}
 
Finally we can define our ECS cluster, the Service and the Task. In ECS, each cluster can have multiple
services, and each service can have multiple Tasks, and each Task can have multiple containers. Now you know.
We only have one Service, but we have a desired Task count of 2. And each Task is one instance of the Flask 
app. 
{% highlight python %}                                                                                          X 
457     # --- ECS Cluster, Task and Service ---
458     cluster = t.add_resource(Cluster(
459         "Cluster",
460         ClusterName='Cluster'
461     ))
462 
463     task_definition = t.add_resource(TaskDefinition(
464         "FlaskTask",
465         ContainerDefinitions=[ContainerDefinition(
466             Image=f"my_repo_flask_image_uri:tag",
467             Command=['/bin/bash', '/usr/src/app/docker-entrypoint.sh'],
468             Cpu=512,
469             Memory=1900,
470             MemoryReservation=1024,
471             DisableNetworking=False,
472             PortMappings=[PortMapping(ContainerPort=80)],
473             Name='flask',
474             User='root',
475             WorkingDirectory='/my/flask/dir'
476         )],
477         Cpu='1024',
478         Family='pmm',
479         Memory='2GB',
480         NetworkMode='awsvpc',
481         RequiresCompatibilities=['FARGATE'],
482         ExecutionRoleArn=Ref(task_execution_role)
483     ))
484 
485     t.add_resource(Service(
486         'Service',
487         Cluster=Ref(cluster),
488         DesiredCount=2,
489         DeploymentConfiguration=DeploymentConfiguration(
490             MaximumPercent="200",
491             MinimumHealthyPercent="75"
492         ),
493         TaskDefinition=Ref(task_definition),
494         LaunchType='FARGATE',
495         LoadBalancers=[
496             ecsLoadBalancer(
497                 "PrivateServiceLoadbalancer",
498                 ContainerName='pmm',
499                 ContainerPort=80,
500                 TargetGroupArn=Ref('TargetGroupPrivate')
501             )
502         ],
503         NetworkConfiguration=NetworkConfiguration(
504             AwsvpcConfiguration=AwsvpcConfiguration(
505                 Subnets=[Ref(pmm_subnet_eu_west_1a), Ref(pmm_subnet_eu_west_1b)],
506                 SecurityGroups=[Ref(fargate_container_security_group)],
507                 AssignPublicIp='DISABLED'
508             )
509         ),
510         DependsOn=["ListenerPublic"]
511     ))
512     return t
 {% endhighlight %}

 
Finally we can deploy our stack by running this code below in a _main_ or whatever
{% highlight python %}
cf = boto3.client('cloudformation', 'eu-west-1')
template = define_stack(ctx)
cf.update_stack(
            StackName=CF_STACK_NAME,
            TemplateBody=template.to_json(),
            Capabilities=['CAPABILITY_IAM'],
        )
{% endhighlight %}


[2]: https://github.com/aws-samples/aws-cidr-finder
