# AWS VPC template

This VPC template is a complete CloudFormation template to build out a VPC network with public and private subnets in three AWS Availability Zones.

"Public" means subnets can receive traffic directly from the Internet. Traffic outbound from a public subnet usually goes through an Internet Gateway. "Private" means that a subnet cannot receive traffic directly from the Internet. Traffic outbound from a private subnet usually goes through a NAT Gateway.

## IP Address Layout

First, specify the IP Address CIDR block for your VPC. In this example, we are using the 10.0.0.0/8.

We have carved the example 10.0.0.0/8 address space into four /10 address spaces. A /10 address space is assigned to a given AWS Region.
This means, we could be in four different AWS Regions. In practice we will use only two Regions, US-West-2 and US-East-1. This will support up to 64 /16 VPCs in each Region.

| Location | Region    | IP CIDR      | Address Range              |
| -------- | --------- | ------------ | -------------------------- |
| Oregon   | us-west-2 | 10.0.0.0/10  | 10.0.0.1 - 10.63.255.255   |
| Virginia | us-east-1 | 10.64.0.0/10 | 10.64.0.1 - 10.127.255.255 |

### Example - VPC CIDR blocks

VPC CIDR blocks we're be using:

```text
Oregon (us-west-2):   10.0.0.0/16
Virginia (us-east-1): 10.64.0.0/16
```

**Note**: _Consult [VPC Sizing](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html#vpc-sizing-ipv4) for more info_.

## VPC Template Parameters

For deployment you three things:

- VPC CIDR block
- Three public
- Three private subnet CIDR blocks

You need to choose whether the VPC will support highly available NAT Gateways, or, by default, a more cost effective _single_ NAT Gateway and whether VPC Flow Logs will be enabled, or disabled (default behavior is disabled to reduce costs).

| Parameter               | Description                  | Example       |
| ----------------------- | ---------------------------- | ------------- |
| _VpcCidrParam_          | IPv4 CIDR block (/16 to /28) | 10.0.0.0/16   |
| _PublicAZASubnetBlock_  | AZ A public subnet block     | 10.0.32.0/20  |
| _PublicAZBSubnetBlock_  | AZ B public subnet block     | 10.0.96.0/20  |
| _PublicAZCSubnetBlock_  | AZ C public subnet block     | 10.0.160.0/20 |
| _PrivateAZASubnetBlock_ | AZ A private subnet block    | 10.0.0.0/19   |
| _PrivateAZBSubnetBlock_ | AZ B private subnet block    | 10.0.64.0/19  |
| _PrivateAZCSubnetBlock_ | AZ C private subnet block    | 10.0.128.0/19 |
| _HighlyAvailable_       | Highly Available NAT config  | true          |
| _EnableVpcFlowLogs_     | VPC Flow Logs                | true          |

To make it easier to specify these parameters on the command line, you can use the example Parameters files included in the `parameters/` directory.

## VPC Flow Logs

[VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html) is a feature that allows you to capture information about IP traffic going to and from network interfaces in your VPC. This template is configured to deliver this data to a CloudWatch Logs Group called `FlowLogs/<CloudFormation Stack Name>`. Enabling VPC Flow Logs will increase your monthly usage costs (see [VPC Flow Logs Pricing](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-pricing) and [CloudWatch Pricing](https://aws.amazon.com/cloudwatch/pricing/) pages).

Flow logs can help you with a number of tasks, eg.:

- Diagnosing overly restrictive security group rules
- Monitoring the traffic that is reaching your instance
- Determining the direction of the traffic to and from the network interfaces

VPC Flow Logs require an IAM role to give the VPC Flow Logs service permission to post to CloudWatch. This role only has permission to create and read CloudWatch log groups, log streams, and put log events.

## How to Deploy

### Prerequisites

- AWS CLI

### Validate/Lint Stack

```bash
aws cloudformation validate-template \
    --template-body file://vpc.yaml
```

### Deploy Stack

You will need to verify you have the appropriate parameters file for the AWS Region and account/environment you want to deploy to. See `./parameters/<region>/<acct>.json`. eg. `parameters/us-east-1/dev.json`.

Then run

```bash
aws cloudformation deploy \
    --template-file vpc.yaml \
    --stack-name main-vpc \
    --parameter-overrides file://parameters/us-east-1/dev.json
```

**Note**: \
_If you have the `EnableVpcFlowLogs` parameter set to `true`, you will need to add `--capabilities CAPABILITY_IAM` to the CLI command since enabling VPC Flow Logs requires creation of an IAM role to give the VPC Flow Logs service permission to write to CloudWatch Logs._

```bash
aws cloudformation deploy \
    --template-file vpc.yaml \
    --stack-name main-vpc \
    --parameter-overrides file://parameters/us-west-2/dev.json \
    --capabilities CAPABILITY_IAM
```

### Update Stack

Updates to the stack can also be done using the deploy command above.

## Template Outputs/Exports

AWS CloudFormation supports [exporting Resource names and properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html). You can import these [Cross-Stack References](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html) in other templates.

This VPC template exports the following values for use in other CloudFormation templates. Each export is prefixed with the **Stack Name**. For example, if you name the stack "main-vpc" when you launch it, the VPC's public route table will be exported as "_main-vpc-public-rtb_"

| Export                         | Description                                          | Example         |
| ------------------------------ | ---------------------------------------------------- | --------------- |
| _main-vpc-VpcId_               | VPC Id                                               | vpc-1234abcd    |
| _main-vpc-public-rtb_          | Public Route table Id (shared by all public subnets) | rtb-1234abcd    |
| _main-vpc-public-AZ-A-subnet_  | AZ A public subnet Id                                | subnet-1234abcd |
| _main-vpc-public-AZ-B-subnet_  | AZ B public subnet Id                                | ""              |
| _main-vpc-public-AZ-C-subnet_  | AZ C public subnet Id                                | ""              |
| _main-vpc-private-AZ-A-subnet_ | AZ A private subnet Id                               | subnet-abcd1234 |
| _main-vpc-private-AZ-B-subnet_ | AZ A private subnet Id                               | ""              |
| _main-vpc-private-AZ-C-subnet_ | AZ A private subnet Id                               | ""              |
| _main-vpc-private-AZ-A-rtb_    | Route table for private subnets in AZ A              | rtb-abcd1234    |
| _main-vpc-private-AZ-B-rtb_    | Route table for private subnets in AZ B              | ""              |
| _main-vpc-private-AZ-C-rtb_    | Route table for private subnets in AZ C              | ""              |

## Conclusion

That's it! Not terrible once you get used to `.yml` syntax and poke about in the AWS CLI a bit.

Thanks! \
\
/J
