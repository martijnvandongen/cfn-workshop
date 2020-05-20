# Cheat Sheet

## Gluing resources

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties: {}
      # VpcId: <-- can't find anything like this in the documentation!
      #            so there should be something like this:
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
```

## Deployment Order

```yaml
Resources:
  VPC: # 1
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
  InternetGateway: # 1
    Type: AWS::EC2::InternetGateway
  VPCGatewayAttachment: # 2
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
  # ...
  # 3
  PublicRouteToInternetGateway:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment # <-- Wait until the GW is attached to the VPC
    Properties:
      # ...
      GatewayId: !Ref InternetGateway
```

## Parameters

```yaml
Parameters:
  CidrBlock:
    Type: String
    DefaultValue: "10.0.0.0/16"
  Environment:
    Type: String
    DefaultValue: "Prod"
    Options: ["Prod", "Test"]
    Description: "Prod deploys 3 AZs, Test only 1"
```

## Mappings

```yaml
Mappings:
  Indexes:
    AZs:
      a: 0
      b: 1
      c: 2
```


## Conditions

```yaml
Condition:
  IsProd: !Equals [ !Ref Environment, "prod"]
  IsNotProd: !Not [ !Equals [ !Ref Environment, "prod"] ]
```

```yaml
Resource:
  ExampleResource:
    Type: AWS::Example::Resource
    Condition: IsProd
    Properties:
      IOPS: !If
        - IsHA
        - "1000"  # isHA = true
        - "10"    # isHA = false
```

## Export / Import 

```yaml
Outputs:
  VPCID:
    Value: !Ref VPC
    Export: !Sub "${AWS::StackName}-VPCID"
```

```yaml
Parameters:
  NetworkStack:
    Type: String
    Default: "network"
Resource:
  ExampleResource:
    Type: AWS::Example::Resource
    Properties:
      VpcId:
        "Fn::ImportValue":
          !Sub "${NetworkStack}-VPCID"
```

```yaml
Outputs:
  Subnets:
    Export: !Sub "${AWS::StackName}-Subnets"
    Value:
      "Fn::Join":
        - ","
        - - !Ref SubnetA
          - !Ref SubnetB
          - !Ref SubnetC
```

```yaml
Resources:
  ExampleResource:
    Type: AWS::Example::Resource
    Properties:
      Subnets:
        "Fn::Split":
          - ","
          - "Fn::ImportValue":
              !Sub "${NetworkStackName}-Subnets"
```

## !Ref, !GetAtt, !Sub Syntax

```yaml
ListOfFunctions:
  RefFunctions:
    - !Ref someParam
    - Ref: someParam
  GetAttFunctions:
    - !GetAtt logicalNameOfResource.attributeName
    - "Fn::GetAtt": [logicalNameOfResource, attributeName]
    - !GetAtt
      - resource
      - attribute_name
  RefAndGetAttFunctions:
    - !Sub "${someParam}-${logicalNameOfResource.attributeName}"
```

## Advanced !Sub

```yaml
Resource:
  ExampleResource:
    Type: AWS::Example::Resource
    Properties:
      VpcId: !Ref "VPC"
      VpcArn:
        "Fn::Sub":
          - "arn:aws:ec2:${region}:${account}:vpc/${vpc}"
          - region: !Ref "AWS::Region"
            account: !Ref "AWS::AccountId"
            vpc: !Ref "VPC"
```

## Other Functions

```yaml
Functions:
  - IOPS: !If [ isProd, 100, 1 ]
  - SnapshotId: !If [ useSnapshot, !Ref "SnapshotName", !Ref "AWS::NoValue" ]
  - AZ1: !GetAZs ""                        # ["eu-west-1a", "eu-west-1b", ...]
  - AZ2: !GetAZs "us-east-1"               # ["us-east-1a", "us-east-1b", ...]
  - What: !Join ["-",["this","that"]]      # "this-that"
  - Fruit: !Split [",", "apple,banana"]    # ["apple", "banana"]
  - AZ3: !Select [1, !GetAZs "eu-west-1"]  # "eu-west-1a
```

## Cidr

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
  SubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock:
        "Fn::Select":
          - 0
          - "Fn::Cidr":
              - "10.0.0.0/16" # Step 1: VPC Cidr Block
              - 256           # Step 3: 2^(24-16) = 256 Count
              - 8             # Step 2: 32 - 24 = 8 Cidr Bits
```

```yaml
Parameters:
  CidrBlock:
    Type: String
Mappings:
  Indexes:
    AZs: {"A": 0, "B": 1, "C": 2}
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref CidrBlock
  SubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock:
        "Fn::Select":
          - !Select [ !FindInMap ["Indexes", "AZs", "A"], !GetAZs "" ]
          - "Fn::Cidr":
              - !GetAtt VPC.CidrBlock
              - 256
              - 8
```

## Best Practice #1: AMI

```yaml
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'

Resources:
  Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !Ref 'LatestAmiId'
```

## Best Practice #2: Input for instance type

```yaml
Parameters:
  EnvSize:
    Type: String
    Default: "prod"
    AllowedValues: 
      - "prod"
      - "test"
      - "dev"
Conditions:
  isProd: !Equals [EnvSize, "prod"]
Resources:
  Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !If [isProd, "m5.xlarge", "t2.micro"]
```

## Resource Protection

```yaml
Resources:
  RDSDBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Retain       # Retain / Delete / Snapshot
    Properties:
      BucketName: "fixedbucketname"
```

## Stack Policies

```json
{
  "Statement" : [
    {
      "Effect" : "Allow",
      "Action" : "Update:*",
      "Principal": "*",
      "Resource" : "*"
    },
    {
      "Effect" : "Deny",
      "Action" : "Update:*",
      "Principal": "*",
      "Resource" : "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

## UpdatePolicy

```yaml
AutoScalingGroup:
  Type: 'AWS::AutoScaling::AutoScalingGroup'
  Properties:
    #...
  UpdatePolicy:
    AutoScalingRollingUpdate:
      MinInstancesInService: 1
      MaxBatchSize: 2
      WaitOnResourceSignals: true
      PauseTime: 'PT10M'
```

## CreationPolicy

```yaml
AutoScalingGroup:
  Type: 'AWS::AutoScaling::AutoScalingGroup'
  Properties:
    #...
  CreationPolicy:
    ResourceSignal:
      Count: 1
      Timeout: PT10M
    AutoScalingCreationPolicy:
      MinSuccessfulInstancesPercent: 10
```

## cfn-init & cfn-signal

```yaml
  AutoScalingLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      #...
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            yum update -y
            /opt/aws/bin/cfn-init --configsets webserver --verbose --stack ${AWS::StackName} --resource AutoScalingLaunchConfiguration --region ${AWS::Region}
            /opt/aws/bin/cfn-signal --exit-code $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          webserver:
            #...
```

## Custom Provider

```python
import cfnresponse
import boto3

def handler(event, context):
    responseData = {}
    ResponseStatus = cfnresponse.SUCCESS
    s3bucketName = event['ResourceProperties']['S3Bucket']
    if event['RequestType'] == 'Create':
        responseData['Message'] = "Resource creation successful!"
    elif event['RequestType'] == 'Update':
        responseData['Message'] = "Resource update successful!"
    elif event['RequestType'] == 'Delete':
        # what's happening in the following 3 lines?
        s3 = boto3.resource('s3')
        bucket = s3.Bucket(s3bucketName)
        bucket.objects.all().delete()
        responseData['Message'] = "Objects Deleted"
    cfnresponse.send(event, context, ResponseStatus, responseData)
```

## Custom Resource

```text
Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
  DeleteAllS3Objects:
    Type: Custom::DeleteAllS3Objects
    Properties:
      ServiceToken: !GetAtt DeleteAllS3ObjectsCustomProviderFunction.Arn
      S3Bucket: !Ref S3Bucket
  DeleteAllS3ObjectsCustomProviderFunction:
    Type: AWS::Lambda::Function
    Properties:
      #...
```

## SAM

```text
Transform: AWS::Serverless-2016-10-31
Resources:
  ServerlessFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: python3.6
      Tracing: Active
      Timeout: 120
      ReservedConcurrentExecutions: 30
      Layers:
        - Ref: MyLayer
      InlineCode: |
        def handler(event, context):
          print("Hello, world!")
      Policies:
        - AWSLambdaExecute
        - DynamoDBCrudPolicy:
            tableName: !Ref DynamoDBTable
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /path
            Method: get
```