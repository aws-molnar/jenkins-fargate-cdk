# Jenkins on Fargate with CDK

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=engel80_jenkins-fargate-cdk&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=engel80_jenkins-fargate-cdk) [![Lines of Code](https://sonarcloud.io/api/project_badges/measure?project=engel80_jenkins-fargate-cdk&metric=ncloc)](https://sonarcloud.io/summary/new_code?id=engel80_jenkins-fargate-cdk)

## Introduction

![Architecture](./screenshots/jenkins-arch.png?raw=true)

For the full stack, refer to the https://github.com/engel80/jenkins-sonarqube-fargate-cdk project.

## Objectives

. Build a Jenkins Master/Slave on Fargate with CDK

## Table of Contents

1. Deploy VPC stack
2. Deploy ECS Fargate cluster stack
3. Deploy IAM Role stack
4. Deploy ECR and CodeCommit repository stack
5. Deploy ECS Fargate Service stack
6. Set password from CloudWatch Logs
7. Run Jenkins builds

## Example

Jenkins version: *v2.346.2*

## Prerequisites

```bash
npm install -g aws-cdk@2.32.1
npm install -g cdk-ecr-deployment@2.5.5

# install packages in the root folder
npm install
cdk bootstrap
```

Use the `cdk` command-line toolkit to interact with your project:

* `cdk deploy`: deploys your app into an AWS account
* `cdk synth`: synthesizes an AWS CloudFormation template for your app
* `cdk diff`: compares your app with the deployed stack
* `cdk watch`: deployment every time a file change is detected

## CDK Stack

|   | Stack                          | Time To Complete |
|---|--------------------------------|------|
| 1 | VPC                            | 3m 30s (optional)     |
| 2 | ECS Fargate cluster            | 50s  |
| 3 | IAM roles                      | 1m   |
| 4 | ECR and CodeCommit repository  | 1m   |
| 5 | ECS Fargate Service and ALB    | 4m   |
|   | Total                          | 7m (10m 30s with a new VPC) |

## Steps

Use the [deploy-all.sh](./deploy-all.sh) file if you want to deploy all stacks without prompt at a time.

### Step 1: VPC

Deploy a new VPC:

```bash
cd vpc
cdk deploy
```

[vpc/lib/vpc-stack.ts](./vpc/lib/vpc-stack.ts)

The VPC ID will be saved into the SSM Parameter Store(`/jenkins-fargate-cdk/vpc-id`) to refer from other stacks.

To use the existing VPC, use the `-c vpcId` context parameter or create SSM Parameter:

```bash
aws ssm put-parameter --name "/jenkins-fargate-cdk/vpc-id" --value "{existing-vpc-id}" --type String 
```

### Step 2: ECS cluster

```bash
cd ../ecs-jenkins-cluster
cdk deploy 

# or define your VPC id with context parameter
cdk deploy -c vpcId=<vpc-id>
```

SSM parameter:

* /jenkins-fargate-cdk/vpc-id

Cluster Name: [config.ts](./config.ts)

[ecs-jenkins-cluster/lib/jenkins-cluster-stack.ts](./ecs-jenkins-cluster/lib/jenkins-cluster-stack.ts)

### Step 3: IAM Role

Create the ECS Task Execution role and default Task Role.

* AmazonECSFargateTaskExecutionRole
* ECSFargateDefaultTaskRole including a policy for ECS Exec

```bash
cd ../ecs-iam-role
cdk deploy 
```

[ecs-iam-role/lib/ecs-iam-role-stack.ts](./ecs-iam-role/lib/ecs-iam-role-stack.ts)

### Step 4: ECR and CodeCommit repository

```bash
cd ../ecr-codecommit
cdk deploy --outputs-file ./cdk-outputs.json
cat ./cdk-outputs.json | jq .
```

### Step 5: ECS Service

Crearte a Fargate Service, Auto Scaling, ALB, and Log Group.

```bash
cd ../ecs-jenkins-service
cdk deploy --outputs-file ./cdk-outputs.json
cat ./cdk-outputs.json | jq .
```

e.g.,

```json
{
  "ecs-jenkins-fargate-dev": {
    "TaskDefinition": "jenkins-task",
    "LogGroup": "jenkins",
    "ALB": "alb-jenkins-123456789.ap-northeast-2.elb.amazonaws.com",
    "Service": "arn:aws:ecs:ap-northeast-2:123456789:service/jenkins-fargate-dev/jenkins"
  }
}
```

SSM parameters:

* /jenkins-fargate-cdk/vpc-id
* /jenkins-fargate-cdk/cluster-securitygroup-id
* /jenkins-fargate-cdk/task-execution-role-arn
* /jenkins-fargate-cdk/default-task-role-arn

[ecs-jenkins-service/lib/ecs-jenkins-service-stack.ts](./ecs-jenkins-service/lib/ecs-jenkins-service-stack.ts)

**IMPORTANT**

If the ECS cluster was re-created, you HAVE to deploy after cdk.context.json files deletion with the below:

`find . -name "cdk.context.json" -exec rm -f {} \;`

### Step 6: Unlock Jenkins with password

![cloudformation-output](./screenshots/cloudformation-output.png?raw=true)

Connect to Jenkins ALB and Unlock Jenkins with password. You can find the password on CDK console and CloudWatch Logs stream:

![unlock-jenkins](./screenshots/unlock-jenkins.png?raw=true)

![pawwrod](./screenshots/jenkins-password-cdk-console.png?raw=true)

To connect into Jenkins container, refer to the [ecs-exec.md](./ecs-exec.md) page.

## Clean Up

[clean-up.sh](./clean-up.sh)

## Structure

```text
├── build.gradle
├── deploy-all.sh
├── clean-up.sh
├── config.ts
├── package.json
├── tsconfig.json
├── app
│   ├── Dockerfile
│   └── build.sh
├── ecr-codecommit
│   ├── bin
│   │   └── index.ts
│   ├── cdk.json
│   └── lib
│       └── ecr-codecommit-stack.ts
├── ecs-iam-role
│   ├── bin
│   │   └── index.ts
│   ├── cdk.json
│   └── lib
│       └── ecs-iam-role-stack.ts
├── ecs-jenkins-cluster
│   ├── bin
│   │   └── index.ts
│   ├── cdk.json
│   ├── jest.config.js
│   └── lib
│       └── jenkins-cluster-stack.ts
├── ecs-jenkins-service
│   ├── bin
│   │   └── index.ts
│   ├── cdk.json
│   └── lib
│       └── jenkins-fargate-stack.ts
└── vpc
    ├── bin
    │   └── index.ts
    ├── cdk.json
    └── lib
        └── vpc-stack.ts
```

## Reference

* [DockerHub - jenkins](https://hub.docker.com/_/jenkins)

* [GitHub - jenkins](https://github.com/SonarSource/jenkins)

### CDK Lib

* [ECS](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs-readme.html)

* [ECR Assets](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecr_assets-readme.html)

* [IAM](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam-readme.html)

* [SSM](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ssm-readme.html)

### IAM Role & Policy

* [Task Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)

* [Exec Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html)
