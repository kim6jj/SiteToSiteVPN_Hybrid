# SiteToSiteVPN_Hybrid
Implementing a Dynamic BGP based, HA Site-to-Site VPN using Adrian Cantril's lab (learn.cantril.io)
- Simulates both a Hybrid AWS and On-Prem environment both utilizing AWS
- Deploy the AWS then OnPrem environments using the CloudFormation templates
    - AWS-Side - 2 subnets, 2 EC2 instances, a Transit GW, VPC attachement and default route pointing at the Transit GW
    - On-Prem Environment - 1 public subnet, 2 private subnets - public subnet has 2 Ubuntu + strongSwan + Free VPN endpoints
![image](https://user-images.githubusercontent.com/1181741/148135490-05a9c1c5-7ebc-4537-b7ff-0321d88cc86c.png)

# 1 Click Install

- [AWS Side](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://cf-templates-30zc7z9turk3-us-east-1.s3.amazonaws.com/20220042DZ-S2SVPN-AWS.yaml&stackName=AWS)
- [ONPREM-Simulated](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://cf-templates-30zc7z9turk3-us-east-1.s3.amazonaws.com/S2SVPN-ONPREM.yaml&stackName=ONPREM)

# Stage 1 (What the CloudFormation template above creates)
- Configure VPC + 2 Subnets (each with single EC2 instance) - Transit GW created/attached to AWS Side VPC with default routes pointing to TGW
- Security groups allowing connections from IP range used OnPrem
- OnPrem simulated - public network with 2 VPN appliances (Router 1+2), Running Ubuntu 18 with LTS StrongSwan (IPSec) with FRR endpoints(For BGP Routing). two subnets (192.168.10.0/24 and 192.168.11.0/24)

# 
