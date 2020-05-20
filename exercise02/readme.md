# Exercise 2

![](/assets/img-vpc-adv-wb.svg)

1. Copy the template below
2. Fix all TASKs
3. Deploy the template

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
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Sub "${AWS::Region}a"
# TASK 1: Replace the question marks with the !Select and !Cidr functions
#         Hint: What if we search on "Cidr" in this template...?
      CidrBlock: "???"
  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Sub "${AWS::Region}b"
# TASK 2: Replace the question marks with the !Select and !Cidr functions
#         Hint: What if we search on "Cidr" in this template...?
      CidrBlock: "???"
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PublicRouteToInternetGateway:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
       RouteTableId: !Ref PublicRouteTable
       DestinationCidrBlock: "0.0.0.0/0"
       GatewayId: !Ref InternetGateway
  PublicSubnetRouteTableAssociationA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: 
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnetA
  PublicSubnetRouteTableAssociationB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: 
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnetB
  EIPA:
    Type: AWS::EC2::EIP
    DependsOn: VPCGatewayAttachment
    Properties:
      Domain: vpc
  NatGatewayA:
    Type: AWS::EC2::NatGateway
    Properties:
# TASK 3: In which Subnet do we want to deploy this resource?
#     SubnetId: ???

# TASK 4: Where can I find the AllocationId of the EIP?
#     AllocationId: ???

# TASK 5:  Copy the NAT Gateway EIPA and NatGatewayA resources and
#          make sure Public Subnet B also gets an EIP and NatGateway

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Sub "${AWS::Region}a"
      CidrBlock:
        "Fn::Select":
          - 2
          - "Fn::Cidr":
              - !GetAtt VPC.CidrBlock
              - 16
              - 8
  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Sub "${AWS::Region}b"
      CidrBlock:
        "Fn::Select":
          - 3
          - "Fn::Cidr":
              - !GetAtt VPC.CidrBlock
              - 16
              - 8
  PrivateRouteTableA:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PrivateRouteTableB:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PrivateSubnetRouteTableAssociationA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: 
      RouteTableId: !Ref PrivateRouteTableA
      SubnetId: !Ref PrivateSubnetA
  PrivateSubnetRouteTableAssociationB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: 
      RouteTableId: !Ref PrivateRouteTableB
      SubnetId: !Ref PrivateSubnetB
  
  # TASK 6: Add a route in PrivateRouteTableA and PrivateRouteTableA
  #         to the Nat Gateways in the same Availability Zone
  #         Hint 1: Copy the Route used for Public Subnets
  #         Hint 2: Destination is not a GatewayId but a Nat...

```