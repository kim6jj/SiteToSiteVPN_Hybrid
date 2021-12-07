# SiteToSiteVPN_Hybrid
Implementing a Dynamic BGP based, HA Site-to-Site VPN using Adrian Cantril's lab (learn.cantril.io)
- Simulates both a Hybrid AWS and On-Prem environment both utilizing AWS
- Deploy the AWS then OnPrem environments using the CloudFormation templates
    - AWS-Side - 2 subnets, 2 EC2 instances, a Transit GW, VPC attachement and default rpute pointing at the TGW
    - On-Prem Environment - 1 public subnet, 2 private subnets - public subnet has 2 Ubuntu + strongSwan + Free VPN endpoints
