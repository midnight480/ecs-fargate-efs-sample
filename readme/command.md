# 00-Prerequisites


```
aws --version
export WOF_AWS_REGION=ap-northeast-1
export WOF_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
export WOF_ECS_CLUSTER_NAME=ecs-fargate-wordpressexport WOF_CFN_STACK_NAME=WordPress-on-Fargate
```

# 01-Create base infrastructure

```
wget https://https://github.com/midnight480/ecs-fargate-efs-sample/main/Cfn/wordpress-ecs-fargate.yaml

aws cloudformation create-stack \
  --stack-name $WOF_CFN_STACK_NAME \
  --region $WOF_AWS_REGION \
  --template-body file://wordpress-ecs-fargate.yaml
```

Note:
L.817 `MasterUserPassword` 

```
aws cloudformation wait stack-create-complete \
  --stack-name $(aws cloudformation describe-stacks  \
    --region $WOF_AWS_REGION \
    --stack-name $WOF_CFN_STACK_NAME \
    --query 'Stacks[0].StackId' --output text) \
  --region $WOF_AWS_REGION
```

```
export WOF_EFS_FS_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='EFSFSId'].OutputValue" \
  --output text)

export WOF_EFS_AP=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='EFSAccessPoint'].OutputValue" \
  --output text)
  
export WOF_RDS_ENDPOINT=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='RDSEndpointAddress'].OutputValue" \
  --output text)
  
 export WOF_VPC_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='VPCId'].OutputValue" \
  --output text)
  
export WOF_PUBLIC_SUBNET0=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet0'].OutputValue" \
  --output text)
  
export WOF_PUBLIC_SUBNET1=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1'].OutputValue" \
  --output text)
  
export WOF_PRIVATE_SUBNET0=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet0'].OutputValue" \
  --output text)

export WOF_PRIVATE_SUBNET1=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet1'].OutputValue" \
  --output text)
  
export WOF_ALB_SG_ID=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='ALBSecurityGroup'].OutputValue" \
  --output text)
  
export WOF_TG_ARN=$(aws cloudformation describe-stacks  \
  --region $WOF_AWS_REGION \
  --stack-name $WOF_CFN_STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='WordPressTargetGroup'].OutputValue" \
  --output text)
```

# 02-ECS-Task

```
cat > wp-task-definition.json << EOF
{   "networkMode": "awsvpc", 
    "containerDefinitions": [
        {
            
            "portMappings": [
                {
                    "containerPort": 8080,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "mountPoints": [
                {
                    "containerPath": "/bitnami/wordpress",
                    "sourceVolume": "wordpress"
                }
            ],
            "name": "wordpress",
            "image": "bitnami/wordpress",
            "environment": [
                {
                    "name": "MARIADB_HOST",
                    "value": "${WOF_RDS_ENDPOINT}"
                },
                {   
                    "name": "WORDPRESS_DATABASE_USER",
                    "value": "admin"
                },
                {   
                    "name": "WORDPRESS_DATABASE_PASSWORD",
                    "value": "supersecretpassword"
                },
                {   
                    "name": "WORDPRESS_DATABASE_NAME",
                    "value": "wordpress"
                },
                {   
                    "name": "PHP_MEMORY_LIMIT",
                    "value": "512M"
                },
                {   
                    "name": "enabled",
                    "value": "false"
                },
                {   
                    "name": "ALLOW_EMPTY_PASSWORD",
                    "value": "yes"
                }
            ]
        }
    ],
    "requiresCompatibilities": [ 
        "FARGATE" 
    ],
    "cpu": "1024", 
    "memory": "3072", 
    "volumes": [
        {
            "name": "wordpress",
            "efsVolumeConfiguration": {
                "fileSystemId": "${WOF_EFS_FS_ID}",
                "transitEncryption": "ENABLED",
                "authorizationConfig": {
                    "accessPointId": "${WOF_EFS_AP}",
                    "iam": "DISABLED"
                }
            }
        }
    ],
    "family": "wof-tutorial"
}
EOF
```

Note:
`WORDPRESS_DATABASE_PASSWORD`

```
WOF_TASK_DEFINITION_ARN=$(aws ecs register-task-definition \
--cli-input-json file://wp-task-definition.json \
--region $WOF_AWS_REGION \
--query taskDefinition.taskDefinitionArn --output text)s
```

# 03-Run WordPress

```
aws ecs create-cluster \
  --cluster-name $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION
```

```
WOF_SVC_SG_ID=$(aws ec2 create-security-group \
  --description Svc-WordPress-on-Fargate \
  --group-name Svc-WordPress-on-Fargate \
  --vpc-id $WOF_VPC_ID --region $WOF_AWS_REGION \
  --query 'GroupId' --output text)

##Accept traffic on port 8080
aws ec2 authorize-security-group-ingress \
  --group-id $WOF_SVC_SG_ID --protocol tcp \
  --port 8080 --source-group $WOF_ALB_SG_ID \
  --region $WOF_AWS_REGION
```

# 04-Create ECS Service

```
aws ecs create-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --service-name wof-efs-rw-service \
  --task-definition "${WOF_TASK_DEFINITION_ARN}" \
  --load-balancers targetGroupArn="${WOF_TG_ARN}",containerName=wordpress,containerPort=8080 \
  --desired-count 2 \
  --platform-version 1.4.0 \
  --launch-type FARGATE \
  --deployment-configuration maximumPercent=100,minimumHealthyPercent=0 \
  --network-configuration "awsvpcConfiguration={subnets=["$WOF_PRIVATE_SUBNET0,$WOF_PRIVATE_SUBNET1"],securityGroups=["$WOF_SVC_SG_ID"],assignPublicIp=DISABLED}"\
  --region $WOF_AWS_REGION
  
#Wait until there two running tasks
watch aws ecs describe-services \
  --services wof-efs-rw-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --query 'services[].runningCount' 
```

# 05-Admin Dashboard

```
echo "http://$(aws elbv2 describe-load-balancers \
  --names wof-load-balancer --region $WOF_AWS_REGION \
  --query 'LoadBalancers[].DNSName' --output text)/wp-admin/"
```

Note:
Username `user`
Password `bitnami`

# 06-Test data persistence

```
aws ecs update-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --service wof-efs-rw-service \
  --task-definition "$WOF_TASK_DEFINITION_ARN" \
  --desired-count 0 
```

```
watch aws ecs describe-services \
  --services wof-efs-rw-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --query 'services[].runningCount'
```

```
aws ecs update-service \
  --cluster $WOF_ECS_CLUSTER_NAME \
  --region $WOF_AWS_REGION \
  --service wof-efs-rw-service \
  --task-definition "$WOF_TASK_DEFINITION_ARN" \
  --desired-count 2
```

```
aws application-autoscaling \
  register-scalable-target \
  --region $WOF_AWS_REGION \
  --service-namespace ecs \
  --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 4
```

```
cat > scaling.config.json << EOF
{
     "TargetValue": 75.0,
     "PredefinedMetricSpecification": {
         "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
     },
     "ScaleOutCooldown": 60,
    "ScaleInCooldown": 60
}
EOF
```

```
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service \
  --policy-name cpu75-target-tracking-scaling-policy \
  --policy-type TargetTrackingScaling \
  --region $WOF_AWS_REGION \
  --target-tracking-scaling-policy-configuration file://scaling.config.json
```

```
wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
./hey_linux_amd64 -z 20m  <WordPress URL>
```

# 05-Cleanup

```
aws application-autoscaling delete-scaling-policy --policy-name cpu75-target-tracking-scaling-policy --service-namespace ecs --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service  --scalable-dimension ecs:service:DesiredCount --region $WOF_AWS_REGION 
aws application-autoscaling deregister-scalable-target --service-namespace ecs --resource-id service/${WOF_ECS_CLUSTER_NAME}/wof-efs-rw-service  --scalable-dimension ecs:service:DesiredCount --region $WOF_AWS_REGION
aws ecs delete-service --service wof-efs-rw-service --cluster $WOF_ECS_CLUSTER_NAME --region $WOF_AWS_REGION --force 
aws ec2 revoke-security-group-ingress --group-id $WOF_SVC_SG_ID --region $WOF_AWS_REGION --protocol tcp --port 8080 --source-group $WOF_ALB_SG_ID 
aws ec2 delete-security-group --group-id $WOF_SVC_SG_ID --region $WOF_AWS_REGION 
aws ecs delete-cluster --cluster $WOF_ECS_CLUSTER_NAME --region $WOF_AWS_REGION
aws cloudformation delete-stack --stack-name $WOF_CFN_STACK_NAME --region $WOF_AWS_REGION
```
