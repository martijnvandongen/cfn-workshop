# Exercise 1

![](/assets/img-vpc-simple-wb.svg)

### VPC

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
  InternetGateway:
    Type: AWS::EC2::InternetGateway
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: "vpc-11cc22ee33"
      InternetGatewayId: "igw-11223344"
```

### Subnet

Hint: don't forget to give both Subnets a different AZ and different Cidr Block. Choose from 10.0.0.0/24, 10.0.1.0/24. [Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet.html)

```yaml
  Subnet:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: "eu-west-1a"
      CidrBlock: "10.0.0.0/24"
      VpcId: "vpc-11cc22ee33"
```

Bonus: Use `!Sub` to avoid hardcoded Availability Zones. [Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)

```
!Sub '${AWS::Region}-${AWS::AccountId}' # generates: "eu-west-1-123456789012"
```

### RouteTable

> One way to protect your VPC is to leave the main route table in its original default state (with only the local route), and explicitly associate each new subnet you create with one of the custom route tables you've created. This ensures that you must explicitly control how each subnet's outbound traffic is routed.

Therefore, you can't use the Default Route Table of a VPC. [Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html)

```yaml
  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: "vpc-11223344"
```

### Route

Add a route to the previous Route Table.

[Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html)

```yaml
  PublicRouteToInternetGateway:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
       RouteTableId: "rtb-11aabb33cc"
       DestinationCidrBlock: "0.0.0.0/0"
       GatewayId: "igw-11223344"
```

### Associate Route Table to a Subnet

A subnet is by default associated to the default route table in a VPC. To explicitly associate, use the following example. You can't use the Default Route Table in CloudFormation. [Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html)

```yaml
  PublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: 
      RouteTableId: "rtb-11aabb33cc"
      SubnetId: "subnet-11223344"
```

### Add Parameters

Now add parameters "CidrBlock", "SubnetCidrBlockA", "SubnetCidrBlockB", instead of hard coding the CidrBlock in the template. [Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)

```yaml
Parameters:
  ExampleParameter:
    Type: String
Resources:
  ExampleResource:
    Type: AWS::Example::Resource
    Properties:
      ExampleName: !Ref ExampleParameter
```
