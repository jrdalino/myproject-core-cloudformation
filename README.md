# Infrastructure as Code AWS - Cloudformation

## This template will create:
- An Amazon VPC - a network environment that contains four subnets (two public and two private) in the 10.0.0.0/16 private IP space, as well as all the needed Route Table configurations. The subnets for this network are created in separate AWS Availability Zones (AZ) to enable high availability across multiple physical facilities in an AWS Region. Learn more about how AZs can help you achieve High Availability here.
- Two NAT Gateways (one for each public subnet, also spanning multiple AZs) - allow the containers we will eventually deploy into our private subnets to communicate out to the Internet to download necessary packages, etc.
- A DynamoDB VPC Endpoint - our microservice backend will eventually integrate with Amazon DynamoDB for persistence
- A Security Group - Allows your docker containers to receive traffic on port 8080 from the Internet through the Network Load Balancer.
- IAM Roles - Identity and Access Management Roles are created.

## Step 1: Modify the Core Template
```
$ cd ~/environment/myproject-product-restapi
$ mkdir aws-cfn
$ vi ~/environment/myproject-product-restapi/aws-cfn/customvpc.yml
```
```
---
AWSTemplateFormatVersion: '2010-09-09'
Description: This stack deploys the core network infrastructure and IAM resources
             to be used for a service hosted in Amazon ECS using AWS Fargate.

Mappings:
  # Hard values for the subnet masks. These masks define
  # the range of internal IP addresses that can be assigned.
  # The VPC can have all IP's from 10.0.0.0 to 10.0.255.255
  # There are four subnets which cover the ranges:
  #
  # 10.0.0.0 - 10.0.0.255
  # 10.0.1.0 - 10.0.1.255
  # 10.0.2.0 - 10.0.2.255
  # 10.0.3.0 - 10.0.3.255
  #
  # If you need more IP addresses (perhaps you have so many
  # instances that you run out) then you can customize these
  # ranges to add more
  SubnetConfig:
    VPC:
      CIDR: '10.0.0.0/16'
    PublicOne:
      CIDR: '10.0.0.0/24'
    PublicTwo:
      CIDR: '10.0.1.0/24'
    PrivateOne:
      CIDR: '10.0.2.0/24'
    PrivateTwo:
      CIDR: '10.0.3.0/24'

Resources:
  # VPC in which containers will be networked.
  # It has two public subnets, and two private subnets.
  # We distribute the subnets across the first two available subnets
  # for the region, for high availability.
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      EnableDnsSupport: true
      EnableDnsHostnames: true
      CidrBlock: !FindInMap ['SubnetConfig', 'VPC', 'CIDR']

  # Two public subnets, where a public load balancer will later be created.
  PublicSubnetOne:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 0
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PublicOne', 'CIDR']
      MapPublicIpOnLaunch: true
  PublicSubnetTwo:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 1
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PublicTwo', 'CIDR']
      MapPublicIpOnLaunch: true

  # Two private subnets where containers will only have private
  # IP addresses, and will only be reachable by other members of the VPC
  PrivateSubnetOne:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 0
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PrivateOne', 'CIDR']
  PrivateSubnetTwo:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 1
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PrivateTwo', 'CIDR']

  # Setup networking resources for the public subnets.
  InternetGateway:
    Type: AWS::EC2::InternetGateway
  GatewayAttachement:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref 'VPC'
      InternetGatewayId: !Ref 'InternetGateway'
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref 'VPC'
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: GatewayAttachement
    Properties:
      RouteTableId: !Ref 'PublicRouteTable'
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref 'InternetGateway'
  PublicSubnetOneRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetOne
      RouteTableId: !Ref PublicRouteTable
  PublicSubnetTwoRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetTwo
      RouteTableId: !Ref PublicRouteTable

  # Setup networking resources for the private subnets. Containers
  # in these subnets have only private IP addresses, and must use a NAT
  # gateway to talk to the internet. We launch two NAT gateways, one for
  # each private subnet.
  NatGatewayOneAttachment:
    Type: AWS::EC2::EIP
    DependsOn: GatewayAttachement
    Properties:
        Domain: vpc
  NatGatewayTwoAttachment:
    Type: AWS::EC2::EIP
    DependsOn: GatewayAttachement
    Properties:
        Domain: vpc
  NatGatewayOne:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayOneAttachment.AllocationId
      SubnetId: !Ref PublicSubnetOne
  NatGatewayTwo:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayTwoAttachment.AllocationId
      SubnetId: !Ref PublicSubnetTwo
  PrivateRouteTableOne:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref 'VPC'
  PrivateRouteOne:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableOne
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayOne
  PrivateRouteTableOneAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableOne
      SubnetId: !Ref PrivateSubnetOne
  PrivateRouteTableTwo:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref 'VPC'
  PrivateRouteTwo:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableTwo
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayTwo
  PrivateRouteTableTwoAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTableTwo
      SubnetId: !Ref PrivateSubnetTwo

  # VPC Endpoint for DynamoDB
  # If a container needs to access DynamoDB (coming in module 3) this
  # allows a container in the private subnet to talk to DynamoDB directly
  # without needing to go via the NAT gateway.
  DynamoDBEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: "*"
            Principal: "*"
            Resource: "*"
      RouteTableIds:
        - !Ref 'PrivateRouteTableOne'
        - !Ref 'PrivateRouteTableTwo'
      ServiceName: !Join [ "", [ "com.amazonaws.", { "Ref": "AWS::Region" }, ".dynamodb" ] ]
      VpcId: !Ref 'VPC'

  # The security group for our service containers to be hosted in Fargate.
  # Even though traffic from users will pass through a Network Load Balancer,
  # that traffic is purely TCP passthrough, without security group inspection.
  # Therefore, we will allow for traffic from the Internet to be accepted by our
  # containers.  But, because the containers will only have Private IP addresses,
  # the only traffic that will reach the containers is traffic that is routed
  # to them by the public load balancer on the specific ports that we configure.
  FargateContainerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Access to the fargate containers from the Internet
      VpcId: !Ref 'VPC'
      SecurityGroupIngress:
          # Allow access to NLB from anywhere on the internet
          - CidrIp: !FindInMap ['SubnetConfig', 'VPC', 'CIDR']
            IpProtocol: -1

# These are the values output by the CloudFormation template. Be careful
# about changing any of them, because of them are exported with specific
# names so that the other task related CF templates can use them.
Outputs:
  CurrentRegion:
    Description: REPLACE_ME_REGION
    Value: !Ref AWS::Region
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'CurrentRegion' ] ]
  CurrentAccount:
    Description: REPLACE_ME_ACCOUNT_ID
    Value: !Ref AWS::AccountId
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'CurrentAccount' ] ]
  VPCId:
    Description: REPLACE_ME_VPC_ID
    Value: !Ref 'VPC'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'VPCId' ] ]
  PublicSubnetOne:
    Description: REPLACE_ME_PUBLIC_SUBNET_ONE
    Value: !Ref 'PublicSubnetOne'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'PublicSubnetOne' ] ]
  PublicSubnetTwo:
    Description: REPLACE_ME_PUBLIC_SUBNET_TWO
    Value: !Ref 'PublicSubnetTwo'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'PublicSubnetTwo' ] ]
  PrivateSubnetOne:
    Description: REPLACE_ME_PRIVATE_SUBNET_ONE
    Value: !Ref 'PrivateSubnetOne'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'PrivateSubnetOne' ] ]
  PrivateSubnetTwo:
    Description: REPLACE_ME_PRIVATE_SUBNET_TWO
    Value: !Ref 'PrivateSubnetTwo'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'PrivateSubnetTwo' ] ]
  FargateContainerSecurityGroup:
    Description: REPLACE_ME_SECURITY_GROUP_ID
    Value: !Ref 'FargateContainerSecurityGroup'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'FargateContainerSecurityGroup' ] ]
```

## Step 2: ECS Roles
```
---
AWSTemplateFormatVersion: '2010-09-09'
Description: This stack deploys the IAM Role

Resources:
  # This is an IAM role which authorizes ECS to manage resources on your
  # account on your behalf, such as updating your load balancer with the
  # details of where your containers are, so that traffic can reach your
  # containers.
  EcsServiceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - ecs.amazonaws.com
            - ecs-tasks.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: /
      Policies:
      - PolicyName: ecs-service
        PolicyDocument:
          Statement:
          - Effect: Allow
            Action:
              # Rules which allow ECS to attach network interfaces to instances
              # on your behalf in order for awsvpc networking mode to work right
              - 'ec2:AttachNetworkInterface'
              - 'ec2:CreateNetworkInterface'
              - 'ec2:CreateNetworkInterfacePermission'
              - 'ec2:DeleteNetworkInterface'
              - 'ec2:DeleteNetworkInterfacePermission'
              - 'ec2:Describe*'
              - 'ec2:DetachNetworkInterface'

              # Rules which allow ECS to update load balancers on your behalf
              # with the information sabout how to send traffic to your containers
              - 'elasticloadbalancing:DeregisterInstancesFromLoadBalancer'
              - 'elasticloadbalancing:DeregisterTargets'
              - 'elasticloadbalancing:Describe*'
              - 'elasticloadbalancing:RegisterInstancesWithLoadBalancer'
              - 'elasticloadbalancing:RegisterTargets'

              # Rules which allow ECS to run tasks that have IAM roles assigned to them.
              - 'iam:PassRole'

              # Rules that let ECS interact with container images.
              - 'ecr:GetAuthorizationToken'
              - 'ecr:BatchCheckLayerAvailability'
              - 'ecr:GetDownloadUrlForLayer'
              - 'ecr:BatchGetImage'

              # Rules that let ECS create and push logs to CloudWatch.
              - 'logs:DescribeLogStreams'
              - 'logs:CreateLogStream'
              - 'logs:CreateLogGroup'
              - 'logs:PutLogEvents'

            Resource: '*'

  # This is a role which is used by the ECS tasks. Tasks in Amazon ECS define
  # the containers that should be deployed togehter and the resources they
  # require from a compute/memory perspective. So, the policies below will define
  # the IAM permissions that our Mythical Mysfits docker containers will have.
  # If you attempted to write any code for the Mythical Mysfits service that
  # interacted with different AWS service APIs, these roles would need to include
  # those as allowed actions.
  ECSTaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service: [ecs-tasks.amazonaws.com]
          Action: ['sts:AssumeRole']
      Path: /
      Policies:
        - PolicyName: AmazonECSTaskRolePolicy
          PolicyDocument:
            Statement:
            - Effect: Allow
              Action:
                # Allow the ECS Tasks to download images from ECR
                - 'ecr:GetAuthorizationToken'
                - 'ecr:BatchCheckLayerAvailability'
                - 'ecr:GetDownloadUrlForLayer'
                - 'ecr:BatchGetImage'

                # Allow the ECS tasks to upload logs to CloudWatch
                - 'logs:CreateLogStream'
                - 'logs:CreateLogGroup'
                - 'logs:PutLogEvents'
              Resource: '*'

            - Effect: Allow
              Action:
                # Allows the ECS tasks to interact with only the MysfitsTable
                # in DynamoDB
                - 'dynamodb:Scan'
                - 'dynamodb:Query'
                - 'dynamodb:UpdateItem'
                - 'dynamodb:GetItem'
              Resource: 'arn:aws:dynamodb:*:*:table/ModernAppTable*'

  # An IAM role that allows the AWS CodePipeline service to perform it's
  # necessary actions. We have intentionally left permissions on this role
  # that will not be used by the CodePipeline service during this workshop.
  # This will allow you to more simply use CodePipeline in the future should
  # you want to use the service for Pipelines that interact with different
  # AWS services than the ones used in this workshop.
  ModernAppBackendCodePipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ModernAppBackendCodePipelineServiceRole
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - codepipeline.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: ModernAppBackend-codepipeline-service-policy
        PolicyDocument:
          Statement:
          - Action:
            - codecommit:GetBranch
            - codecommit:GetCommit
            - codecommit:UploadArchive
            - codecommit:GetUploadArchiveStatus
            - codecommit:CancelUploadArchive
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:GetObject
            - s3:GetObjectVersion
            - s3:GetBucketVersioning
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:PutObject
            Resource:
            - arn:aws:s3:::*
            Effect: Allow
          - Action:
            - elasticloadbalancing:*
            - autoscaling:*
            - cloudwatch:*
            - ecs:*
            - codebuild:*
            - iam:PassRole
            Resource: "*"
            Effect: Allow
          Version: "2012-10-17"

  # An IAM role that allows the AWS CodeBuild service to perform the actions
  # required to complete a build of our source code retrieved from CodeCommit,
  # and push the created image to ECR.
  ModernAppBackendCodeBuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ModernAppBackendCodeBuildServiceRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          Effect: Allow
          Principal:
            Service: codebuild.amazonaws.com
          Action: sts:AssumeRole
      Policies:
      - PolicyName: "ModernAppBackend-CodeBuildServicePolicy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
            - "codecommit:ListBranches"
            - "codecommit:ListRepositories"
            - "codecommit:BatchGetRepositories"
            - "codecommit:Get*"
            - "codecommit:GitPull"
            Resource:
            - Fn::Sub: arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:ModernAppBackendRepository
          - Effect: "Allow"
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            Resource: "*"
          - Effect: "Allow"
            Action:
            - "s3:PutObject"
            - "s3:GetObject"
            - "s3:GetObjectVersion"
            - "s3:ListBucket"
            Resource: "*"
          - Effect: "Allow"
            Action:
            - "ecr:InitiateLayerUpload"
            - "ecr:GetAuthorizationToken"
            Resource: "*"
            
# These are the values output by the CloudFormation template. Be careful
# about changing any of them, because of them are exported with specific
# names so that the other task related CF templates can use them.
Outputs:
  EcsServiceRole:
    Description: REPLACE_ME_ECS_SERVICE_ROLE_ARN
    Value: !GetAtt 'EcsServiceRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'EcsServiceRole' ] ]
  ECSTaskRole:
    Description: REPLACE_ME_ECS_TASK_ROLE_ARN
    Value: !GetAtt 'ECSTaskRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'ECSTaskRole' ] ]
  CodeBuildRole:
    Description: REPLACE_ME_CODEBUILD_ROLE_ARN
    Value: !GetAtt 'ModernAppBackendCodeBuildServiceRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'ModernAppBackendCodeBuildServiceRole' ] ]
  CodePipelineRole:
    Description: REPLACE_ME_CODEPIPELINE_ROLE_ARN
    Value: !GetAtt 'ModernAppBackendCodePipelineServiceRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'ModernAppBackendCodePipelineServiceRole' ] ]
```

## Step 3: Create the Stack
```
$ aws cloudformation create-stack \
--stack-name MyProjectVPCStack \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://~/environment/calculator-backend/aws-cfn/my-project-stack-vpc.yml
```

```
$ aws cloudformation create-stack \
--stack-name MyProjectIAMRolesStack \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://~/environment/calculator-backend/aws-cfn/my-project-stack-iam-roles.yml
```

## Step 4: Verify completion of Stack Creation "StackStatus": "CREATE_COMPLETE"
```
$ aws cloudformation describe-stacks \
--stack-name MyProjectStackVPC
```

```
$ aws cloudformation describe-stacks \
--stack-name MyProjectStackIAMRoles
```
