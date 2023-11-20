---
layout: post
title: Infrastructure as Code - setting initial AWS cdk stack - VPC
---
In this initial technical post, I will detail how I set up my VPC stack and the challenges I encountered. Please do not consider this as the definitive approach, and feel free to reach out if you identify any bugs, potential improvements, or if you'd simply like to discuss it.

As this is the first post about CDK, let's begin by setting it up from scratch. Assuming you have npm installed, run:

```bash
npm install -g aws-cdk
```

Once aws-cdk is installed, create a new directory for CDK and run:
```bash
cdk init app --language=python
```

This will generate the basic AWS CDK setup. If you want to follow my configuration, update your requirements.txt:
```plaintext
aws-cdk-lib>=2.84.0,<3.0.0
constructs>=10.0.0,<11.0.0
aws_cdk.aws_lambda_python_alpha
aws-cdk.aws-glue-alpha>=2.84.0a0
```

Now, follow the instructions in README.md.

Once that's done, we can proceed to create our first stack - VPC. Remove the example stack created by the CDK installer and clean up the app.py file:
```python
import aws_cdk as cdk

app = cdk.App()

app.synth()
```

In the main repo, create a 'lib' folder and a file 'vpc_stack.py'. Let's start with a basic VPC:
```python
from aws_cdk import (
    aws_ec2 as ec2,
    Stack,
)
from constructs import Construct

class VpcStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        self.custom_vpc = ec2.Vpc(
            self, 'RDS_VPC',
            cidr='10.0.0.0/16',
            max_azs=1,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    cidr_mask=26,
                    name='isolatedSubnet',
                    subnet_type=ec2.SubnetType.PRIVATE_ISOLATED,
                ),
                ec2.SubnetConfiguration(
                    cidr_mask=26,
                    name='publicSubnet',
                    subnet_type=ec2.SubnetType.PUBLIC,
                )
            ],
        )
```
Let's break down what's going on over there:

- cidr='10.0.0.0/16': This sets the CIDR (Classless Inter-Domain Routing) block. A CIDR block is a range of IP addresses identified by a common prefix. For example, 10.0.0.0/16 is a CIDR block representing all IP addresses from 10.0.0.0 to 10.0.255.255.
- max_azs=2: Sets our VPC to be running in two availability zones, providing high availability and fault tolerance. However, for development purposes, one is enough.
- subnet_configuration: We define two subnets - private isolated and public. In the private isolated subnet, we will host RDS and our Lambdas, while in the public subnet, our web server will reside.

Now, let's define security groups for our VPC:
```python
self.lambda_security_group = ec2.SecurityGroup(
    self, 'LambdaSecurityGroup',
    vpc=self.custom_vpc,
    allow_all_outbound=True,
    security_group_name='LambdaSecurityGroup',
)
self.lambda_security_group.add_ingress_rule(peer=ec2.Peer.ipv4('0.0.0.0/0'), connection=ec2.Port.tcp(80))

self.ingress_security_group = ec2.SecurityGroup(
        self, 'IngressSecurityGroup',
        vpc=self.custom_vpc,
        allow_all_outbound=False,
        security_group_name='IngressSecurityGroup',
    )
self.ingress_security_group.add_ingress_rule(peer=ec2.Peer.ipv4('10.0.0.0/16'), connection=ec2.Port.tcp(5432))

self.egress_security_group = ec2.SecurityGroup(
    self, 'EgressSecurityGroup',
    vpc=self.custom_vpc,
    allow_all_outbound=False,
    security_group_name='EgressSecurityGroup',
)
self.egress_security_group.add_egress_rule(peer=ec2.Peer.any_ipv4(), connection=ec2.Port.tcp(80))
```
Here's a breakdown of their roles:
- LambdaSecurityGroup:
    * Purpose: This security group is associated with Lambda functions within the VPC. Configuration:
      * Allows all outbound traffic (allow_all_outbound=True).
      * Permits inbound traffic on TCP port 80 from any IPv4 address (0.0.0.0/0).
- IngressSecurityGroup:
    * Purpose: Configured for inbound traffic to specific resources in the VPC. Configuration:
      * Restricts outbound traffic (allow_all_outbound=False).
      * Permits inbound traffic on TCP port 5432 (commonly used for databases) from the VPC's CIDR range (10.0.0.0/16).
- EgressSecurityGroup:
    * Purpose: Configured for outbound traffic from specific resources in the VPC. Configuration:
      * Restricts outbound traffic (allow_all_outbound=False).
      * Allows outbound traffic on TCP port 80 to any IPv4 address (Peer.any_ipv4()).
 These security groups provide a level of isolation and control over the communication flow within the VPC, ensuring that Lambda functions, specific inbound resources, and outbound traffic adhere to the defined security policies.


Now, add interface endpoints to allow Lambdas to communicate with AWS SNS and AWS Secrets Manager:
```python
self.custom_vpc.add_interface_endpoint(
    'SecretsManagerEndpoint',
    service=ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
    private_dns_enabled=True,
    security_groups=[self.lambda_security_group],
)

self.custom_vpc.add_interface_endpoint(
    'SnsEndpoint',
    service=ec2.InterfaceVpcEndpointAwsService.SNS,
    private_dns_enabled=True,
    security_groups=[self.lambda_security_group],
)
```

I encountered an issue with my EventEmitter Lambda where the problem was not obvious. In that Lambda, using boto3, I created an SNS client and called publish on it. After deploying the stack, the Lambda timed out. Initially, the Lambda had a 3-second timeout, so I increased it to 30 seconds. It still timed out without any exception. After some investigation, I found out that it hangs on the SNS client publish invocation. The problem is, if you do not add an interface endpoint pointing to the SNS service, you will not get any error, but the Lambda will just hang infinitely, as the client waits for a response from SNS, which was blocked by the VPC.

You can find the full code, which may be slightly different as it evolves, [here](https://github.com/adam-drag/stock-mate/tree/main/cdk).