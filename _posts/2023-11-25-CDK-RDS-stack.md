---
layout: post
title: Infrastructure as Code - creating RDS stack with initializing lambda
---
Infrastructure as Code - creating RDS stack with initializing lambda
This is the second post in the series, below link to the fist one, just in case you missed it:infrastructure-as-code-setting-initial-aws-cdk-stack-vpc-rds-lambda-sns

---

In the first post, we configured AWS CDK and created our first stack with VPC, security groups, and defined endpoints for both AWS Secrets Manager and AWS SNS. Now, we will move on and create a new stack with AWS RDS and a Lambda, which will initialize the RDS.
Let's not waste time and start from creating the RDS:
```
from aws_cdk import (
    aws_ec2 as ec2,
    Stack,
    aws_rds as rds,
    aws_secretsmanager as secretsmanager)
from constructs import Construct

from lib.vpc_stack import VpcStack

class RdsStack(Stack):
    def __init__(self, scope: Construct, vpc: VpcStack, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)
        self.db_name = "YourRdsDatabaseName"
        self.db_secret = secretsmanager.Secret(
            self, "DBSecret",
            generate_secret_string=secretsmanager.SecretStringGenerator(
                secret_string_template='{"username":"dbmaster"}',
                generate_string_key="password",
                exclude_characters="\"@/\\"
            )
        )
        self.db_instance = rds.DatabaseInstance(
            self, "YourRdsDatabaseId",
            engine=rds.DatabaseInstanceEngine.postgres(
                version=rds.PostgresEngineVersion.of("12", "12.4")
            ),
            instance_type=ec2.InstanceType.of(
                ec2.InstanceClass.BURSTABLE2, ec2.InstanceSize.MICRO
            ),

            credentials=rds.Credentials.from_secret(self.db_secret),
            multi_az=False,
            allocated_storage=20,
            max_allocated_storage=100,
            vpc=vpc.custom_vpc,
            security_groups=[vpc.egress_security_group, vpc.ingress_security_group],
            vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_ISOLATED)
        )
```
Let's breakdown this part and analyse bit by bit to understand what we've done:
1. SecretManager - we use AWS SecretManager to generate a password to our database:
    - secret_string_template- Sets the template for the secret string. It's a JSON string with a fixed username ("dbmaster"). The password will be generated dynamically.
    - generate_string_key="password" - Specifies that the value for the "password" key in the secret string will be dynamically generated.
exclude_charatcers - Sets the characters that should be excluded from the generated password.

2. RDS Database instance - here we are going to create an instance of the RDS database with given config:
    - engine- Sets the type of the database, in our case it's PostgreSQL and define it's version
    - instance_type - This sets the instance type on which the database will run, I chose MICRO from the T2 family to stick to the free tier with an awesome burstable option. T2 instances accumulate CPU credits when they operate below their baseline performance level. When your workload requires more CPU resources than the baseline, the instance can consume these accumulated credits to burst above the baseline for a period of time. However, if the accumulated credits are exhausted, the instance's performance is limited to the baseline level.
    - credentials - Sets the credentials to the database, just look how awesome it integrates with Secret returned by SecretManager.
multi_az - This sets if our DB should have a standby replica in a different Availability Zone for high availability and automatic failover. However, it's important to remember that this standby replica is not used for read scaling; its purpose is to provide a redundant copy of the primary database for failover purposes.
    - allocated_storage - Represents the amount of storage initially provisioned for your database instance.
max_allocated_storage - Represents the maximum storage capacity that your database instance can scale up to.
vpc - Defines in which VPC our RDS will be deployed - we set it to the VPC created in our VpcStack
    - security_groups - This code configures the security groups for our RDS instance by assigning the ingress and egress security groups defined in our VpcStack. The ingress security group manages inbound traffic to the RDS and permits it on the default PostgreSQL port, which is 5432. On the other hand, the egress security group governs outbound traffic, allowing it to be sent back on the HTTP port 80.
    - vpc_subnets -Determines the subnets within the VPC where the RDS instance will be available. Since the RDS is not intended to be reachable from outside the VPC, we set it to PRIVATE_ISOLATED subnets, ensuring that it remains isolated within the private network.

Now, I encourage you to get back to the terminal and try running cdk synth to make sure everything is set correctly. I do it often because after a big change, you might get confused about what caused the issue.

---

Basically, this post could conclude at this point, as the above code successfully creates the RDS instance. However, let's take it a step further and add a Lambda function to initialize the database by creating the schema.
Now, let's begin by creating the Lambda in CDK, first we need to define policies and roles:
CRUD policy to allow lambda interact with the RDS:
```
rds_data_crud_policy = iam.PolicyDocument(
    statements=[
        iam.PolicyStatement(
            actions=[
                "rds-data:BatchExecuteStatement",
                "rds-data:BeginTransaction",
                "rds-data:CommitTransaction",
                "rds-data:ExecuteSql",
                "rds-data:RollbackTransaction",
            ],
            resources=[self.db_instance.instance_arn],
        ),
    ],
)
```
NetworkInterfacePolicy -this is required because our lambda will be deploy in a VPC:
```
ni_policy = iam.PolicyDocument(
    statements=[
        iam.PolicyStatement(
            actions=[
                "ec2:CreateNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DeleteNetworkInterface"
            ],
            resources=["*"],
        ),
    ],
)
```
The last policy allows the Lambda function to access RDS credentials from Secrets Manager:
```
iam.PolicyDocument(
    statements=[
        iam.PolicyStatement(
            actions=["secretsmanager:GetSecretValue"],
            resources=[
                f"arn:aws:secretsmanager:{self.region}:{self.account}:secret:{self.db_secret.secret_name}*"
            ],
        )],
)
```
Finally, we are ready to create an IAM role of our lambda:
```
db_initializer_lambda_role = iam.Role(
    self, "DbInitializerRole",
    assumed_by=iam.ServicePrincipal("lambda.amazonaws.com"),
    managed_policies=[
        iam.ManagedPolicy.from_aws_managed_policy_name("service-role/AWSLambdaBasicExecutionRole"),
    ],
    inline_policies={
        "CRUDPolicy": rds_data_crud_policy,
        "NetworkInterfacePolicy": ni_policy,
        "SecretsManagerPolicy": secret_manager_policy,
    },
)
```
Now we are ready to create our first lambda which will be deployed within our custom VPC with access to the RDS:
```
aws_lambda_python_alpha.PythonFunction(
    self, "DbInitializer",
    runtime=_lambda.Runtime.PYTHON_3_9,
    entry="../db_initializer",
    index="app.py",
    handler="lambda_handler",
    vpc=vpc.custom_vpc,
    role=db_initializer_lambda_role,
    environment={
        "DB_HOST": self.db_instance.db_instance_endpoint_address,
        "DB_PORT": self.db_instance.db_instance_endpoint_port,
        "DB_SECRET_NAME": self.db_secret.secret_name,
        "DB_NAME": self.db_name,
    },
    security_groups=[vpc.lambda_security_group],
    timeout=Duration.minutes(1),
)
```
Before we move on, note how I create the Lambda function. I use aws_lambda_python_alpha.PythonFunction because it not only uploads our code to the Lambda function in AWS but also installs and packs all dependencies from requirements.txt.

Now, let's breakdown this part:
- runtime - This sets the runtime version of Python
- entry - The path to the directory with the Lambda sources. If you are using a relative path, as I did, remember that it's relative to the app.py where you run the CDK command.
- index - entry file name of your lambda, AWS Lambda will search for this file and attempt to run the method defined in the handler
handler - defines the method name in index file - this will be invoked by AWS Lambda
- vpc - This parameter specifies the VPC in which the Lambda function will be deployed. Typically, it may not be required; however, to enable the Lambda function to access the RDS while maintaining the RDS's hidden configuration within the VPC, it becomes necessary.
- role - Assigns role with persmissions which we have just created.
environment - Here, we pass system environment variables containing information about the DB address in our network, the database name, and the secret name. This allows us to retrieve this information during runtime.
- security_groups - Assigns our lambda_security_group create in the VpcStack.
- timeout- The timeout, by default, is set to 3 seconds; however, the time required by this Lambda largely depends on our schema definition. I believe 1 minute is more than enough. Still, it might require adjustments if you want to perform some time-consuming migrations.

Once again, I encourage you to get back to the terminal and try running cdk synth to make sure everything is set correctly.
Now we have CDK ready to deploy, but we miss something quite important - the Lambda code ;)

---

This post is starting to get a bit long, so let's quickly implement a generic Lambda. It will read the schema from schema.sql, connect with our RDS, and execute all SQL statements. It should rollback all changes if any of them fail.

You can do as I did and create a folder next to the cdk folder db_initlizer otherwise you need to adjust the entry value. In the folder create requirements.txt:
```
psycopg2-binary==2.9.1
boto3
```
Now, let's create schema.sql. I will use an example from my project. You don't need to drop the schema on every initialization, but I found it useful during development, as I often changed the schema, and all persisted data was test data.
```
DROP SCHEMA IF EXISTS your_schema CASCADE;

CREATE SCHEMA IF NOT EXISTS your_schema;

CREATE TABLE IF NOT EXISTS your_schema.product (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    avg_weekly_usage DECIMAL(10, 2) DEFAULT 0,
    description TEXT,
    safety_stock DECIMAL(10, 2) DEFAULT 0,
    max_stock DECIMAL(10, 2) DEFAULT 0
);
```
Once that's done we can finally move to the lambda code:
```
import json
import logging
import os
import sys
import traceback
from datetime import datetime

import boto3
import psycopg2

now = datetime.now()
timestamp = datetime.timestamp(now)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

def get_logger(name):
    desired_log_level = logging.DEBUG

    logger = logging.getLogger(name)
    logger.setLevel(desired_log_level)

    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setLevel(desired_log_level)

    stdout_handler.setFormatter(formatter)

    logger.addHandler(stdout_handler)
    return logger

logger = get_logger(__name__)

def database_exists(conn, db_name):
    try:
        with conn.cursor() as cursor:
            cursor.execute("SELECT 1 FROM pg_database WHERE datname = %s;", (db_name,))
            return cursor.fetchone() is not None
    except Exception as e:
        logger.error(f"Error checking if database exists {e}.")
        raise
    finally:
        conn.commit()

def create_database(conn):
    try:
        conn.autocommit = True
        with conn.cursor() as cursor:
            cursor.execute(f"CREATE DATABASE {os.environ['DB_NAME']};")
        conn.commit()
        logger.info(f"Database {os.environ['DB_NAME'] created successfully.")
    except Exception as e:
        logger.error(f"Error creating database: {e}")
        raise
    finally:
        conn.commit()

def get_logger(name):
    desired_log_level = logging.DEBUG

    logger = logging.getLogger(name)
    logger.setLevel(desired_log_level)

    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setLevel(desired_log_level)

    stdout_handler.setFormatter(formatter)

    logger.addHandler(stdout_handler)
    return logger

def lambda_handler(event, context):
    try:
        logger.info("Starting lambda")
        secret_string = pull_secret_string()
        secret_dict = json.loads(secret_string)
        username = secret_dict.get("username")
        password = secret_dict.get("password")
        logger.info("Successfully pulled db credentials")

        create_db_if_not_exists(password, username)
        run_sql_statements(password, username)
        return {
            'statusCode': 200,
            'body': 'RDS schema initialization successful!'
        }
    except Exception as e:
        traceback.print_stack()
        return {
            'statusCode': 500,
            'body': f'RDS schema initialization failed. Error: {str(e)}'
        }

def pull_secret_string():
    secret_name = os.environ.get("DB_SECRET_NAME")
    client = boto3.client('secretsmanager')
    logger.info(f"Pulling secret: {secret_name}")
    try:
        response = client.get_secret_value(SecretId=secret_name)
    except Exception as e:
        print(f"Error retrieving secret: {e}")
        raise e
    secret_string = response['SecretString']
    return secret_string

def run_sql_statements(password, username):
    sql_file_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), "schema.sql")
    conn = psycopg2.connect(
        host=os.environ['DB_HOST'],
        port=os.environ.get('DB_PORT', '5432'),
        user=username,
        password=password,
        database=os.environ['DB_NAME']
    )
    logger.info(f"Connected to db on {os.environ['DB_HOST']}")

    try:
        cursor = conn.cursor()
        with open(sql_file_path, "r") as sql_file:
            sql_commands = sql_file.read()

        logger.info(f"Executing all SQL statements from schema.sql")
        cursor.execute(sql_commands)

        logger.info("Committing statements")
        conn.commit()
    except Exception as e:
        logger.error(f"Error executing SQL statements: {e}")
        conn.rollback()
        raise
    finally:
        cursor.close()
        conn.close()

def create_db_if_not_exists(password, username):
    default_conn = psycopg2.connect(
        host=os.environ['DB_HOST'],
        port=os.environ.get('DB_PORT', '5432'),
        user=username,
        password=password,
        database='postgres'
    )
    if not database_exists(default_conn, os.environ['DB_NAME']):
        create_database(default_conn)
    default_conn.close()
```

A quick breakdown of what's happening in the above code:
1. AWS Lambda will invoke lambda_handler
2. Pull credentials from the AWS SecretManager using boto client
2. Create database if it doesn't exists by calling create_db_if_not_exists
4. Executes all sql statements in our schema.sql in run_sql_statements
