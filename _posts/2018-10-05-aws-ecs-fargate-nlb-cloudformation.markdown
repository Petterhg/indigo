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

[1]: https://petterhg.github.io/aws-ecs-fargate-alb-cloudformation/
