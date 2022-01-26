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
- AWS Side VPC + 2 Subnets (each with single EC2 instance) - Transit GW created/attached to AWS Side VPC with default routes pointing to TGW
- Security groups allowing connections from IP range used OnPrem (Which is simulated on AWS)
- OnPrem simulated - public network with 2 VPN appliances (Router 1+2), Running Ubuntu 18 with LTS StrongSwan (IPSec) with FRR endpoints(For BGP Routing). two subnets (192.168.10.0/24 and 192.168.11.0/24)

- Create Customer Gateways (Which are logical representations of OnPrem Routers)
- Use the IP's for 'Router1 Public' and 'Router2 Public' after stacks have been created under the 'Outputs' tab (Derived from Outputs section in CloudFormation template: 
    - Outputs:<br/>
        Router1Public:<br/>
            Description: Public IP of Router1<br/>
            Value: !GetAtt Router1.PublicIp<br/>
        Router2Public:<br/>
            Description: Public IP of Router2<br/>
            Value: !GetAtt Router2.PublicIp<br/>
    
- Create OnPrem Router 1 and 2 using IPs that were output after stack creation, dynamic routing, and using a private BGP ASN of 65016 (can use any within range but demo is configured to use 65016)

# Stage 2 (AWS Side VPN Architecture)
- 2 VPN attachments for TGW using accelerated VPN endpoints (2x per connection) - transit back to AWS side gateway over AWS global network as well as creating VPN connections using IPSec tunnels to the ONPrem Router 1+2
- Create Transit Gateway Attachements (under VPC) For each OnPrem Router
    - using already existing A4L TGW from CloudFormation stack
    - attachment type = VPN (One side connecting to AWS and other connecting to the Customer GW we created before in Stage 1) - Dynamic routing option and enable acceleration
    - Should see these pending under 'Site-to-Site VPN Connections' section
    - Download the configuration file for each connection/ID (For use in Stage 3)

# Stage 3 (Establish IPSec tunnels by configuring OnPrem side)
- To extract Tunnel information from Stage 2
- We need the following information:
    - Router1+2 Private IP - found in OnPrem stack output, OnPrem BGP ASN and AWS BGP ASN (64512, 65016)
    - Connection 1 which is AWS side to On Prem R1 
    - PresharedKey, OnPrem Outside + Inside IP, AWS Outside + Inside IP, AWS BGP IP - 'Tunnel Interface Connection' - FOR BOTH TUNNELS 1 + 2
    - Connection 2 which is AWS side to On Prem R2
    - PresharedKey, OnPrem Outside + Inside IP, AWS Outside + Inside IP, AWS BGP IP - 'Tunnel Interface Connection' - FOR BOTH TUNNELS 1 + 2
- AWS Site-to-Site VPN has two tunnels per connection for resiliency 
    - First tunnel, outside, is an IPSec encrypted tunnel running over the Public internet with a preshared key (outside IPs) 
    - Inside tunnels, using inside IPs - BGP runs over the inside and data transfer

- Moving to the OnPrem Routers (EC2 instances - repeat for each OnPrem Router)
    - We will enter session manager via AWS and go into root permissions (sudo bash) for R1 instance
    - Change directory into demo_assests (cd /home/ubuntu/demo_assets)
    - should see ipsec.conf file (configuration of IPSec tunnels), ipsec.secrets (authentication information to authenticate with AWS) and ipsec-vti.sh (script file, enable/disable ipsec tunnels whenever system detects traffic) - *Files have been created beforehand with placeholder values
    - will update values and restart instances (R1 and R2) to bring tunnel up after editing above files
    - update 'ipsec.conf' file; placeholder values needing values are derived from earlier: R1 Private IP, OnPrem Outside IP, AWS Outside IP, etc
    - update 'ipsec.secrets' file
        - sss
     
    
