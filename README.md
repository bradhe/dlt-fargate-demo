# Launch dlt in Fargate :balloon:

If you wrap your dlt job in a container, you can easily launch it as a periodic
job in Fargate. This CloudFormation template serves as a basis for that.

## Prerequisits

### Dependencies

You need the AWS CLI installed on your machine and configured with the
appropriate credentials for creating/manging secrets and launching
CloudFormation stacks.

### DockerHub credentials

The template assumes that your container is stored in DockerHub and uses AWS
Secrets Manager to store/get access to the relevant credentials at runtime. To
get your DockerHub credentials into AWS Secret Manager, you can use the
following AWS CLI commands.


```
$  aws secretsmanager create-secret \
    --name dev/DockerHubSecret \
    --description "Docker Hub Secret" \
    --secret-string '{"username":"Test1","password":"abc123"}' \
    --region us-east-1
{
    "ARN": "arn:aws:secretsmanager:us-east-1:0123456789:secret:dev/DockerHubSecret2-QFrQ4R",
    "Name": "dev/DockerHubSecret",
    "VersionId": "ea50d146-b184-42d9-9c30-6e22ed9be780"
}
```

We'll refer to this credential via it's ARN later on. 

### VPC information

You'll need the VPC ID as well as two Subnet IDs from your EC2 environment.
These are used by CloudFormation for task placement.

## How to launch

After your container is in DockerHub, you can launch this template using the
following command:

```
$ aws cloudformation create-stack \
    --stack-name dlt-fargate-1 \
    --template-body file://./template.yaml \
    --parameters \
        ParameterKey=ImageName,ParameterValue=dockerorg/dockerimage:laters \
        ParameterKey=VPCID,ParameterValue=vpc-12345 \
        ParameterKey=SubnetA,ParameterValue=subnet-12345 \
        ParameterKey=SubnetB,ParameterValue=subnet-67890 \
        ParameterKey=DockerHubCredentials,ParameterValue=arn:aws:secretsmanager:us-east-1:0123456789:secret:dev/DockerHubSecret2-QFrQ4R \
    --region us-east-1 \
    --capabilities CAPABILITY_IAM
```

After the stack is launched (you can use the console to monitor), you can
invoke the job with the following command.

*Note:* You _will_ need one of the subnet IDs as well as the security group ID
that is an output from the CloudFormation stack. Notice the
`awsvpcConfiguration` parameter.

```
aws ecs run-task \
    --launch-type FARGATE \
    --cluster dlt-fargate-1-cluster \
    --task-definition arn:aws:ecs:us-east-1:0123456789:task-definition/dlt-fargate-1-job \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-12345],securityGroups=[sg-abcd12345],assignPublicIp=ENABLED}" \
    --count 1 \
    --region us-east-1
```

# Todo

If you want to contribute to this, here are a few things you could do.

- [ ] An example dlt script and Dockerfile
- [ ] Use CloudWatch to invoke the job periodically
- [ ] I think we could reduce invokation complexity in general.
