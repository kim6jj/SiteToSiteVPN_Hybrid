# SiteToSiteVPN_Hybrid
Implementing a Dynamic BGP based, HA Site-to-Site VPN using Adrian Cantril's lab (learn.cantril.io)
- Simulates both a Hybrid AWS and On-Prem environment both utilizing AWS
- Deploy the AWS then OnPrem environments using the CloudFormation templates
    - AWS-Side - 2 subnets, 2 EC2 instances, a Transit GW, VPC attachement and default route pointing at the Transit GW
    - On-Prem Environment - 1 public subnet, 2 private subnets - public subnet has 2 Ubuntu + strongSwan + Free VPN endpoints
![stage4](stage3.JPG)

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
    - update 'ipsec.secrets' file (Remember two tunnels per VPN connection on AWS side)
        - Connection1_Tunnel1_OutsideIP Connection1_Tunnel1_AWS_OutsideIP : PSK "Preshared Key Tunnel 1" 
        - Connection1_Tunnel2_OutsideIP Connection1_Tunnel2_AWS_OutsideIP : PSK "Preshared Key Tunnel 2" 
    - update script ipsec-vti.sh - for each AWS-VPC-GW, configure appropriate IPs WITH subnet mask CIDR (ex. x.x.x.x/30)
        - VTI_Localaddr = Onprem Inside IP for Tunnel 1 + 2
        - VTI_Remoteaddr == AWS Inside IP for Tunnel 1 + 2

- After filling the information in, we will copy the IPsec files over to the /etc directory where it'll be read by the StrongSwan software (cp ipsec* /etc)
- Make the .sh script file executable (chmod +x /etc/ipsec-vti.sh) and restart strongswan software (systemctl restart strongswan)
    - To verify, can run an 'ifconfig' and see if we have vti1 and vti2 showing which would indiciate that the tunnel interfaces are up and active to AWS
- Repeat for OnPrem Router 2 (R2)
    - Tunnel will show down in the AWS console as we don't have BGP connectivity yet but the IPSEC tunnel should show as UP

# Stage 4 (Establish BGP sessions across IPSec tunnels and confirm IP connectivity between Simulate OnPrem and AWS side)
- Now configure BGP to run over the top of the 4 IPSec tunnels we have created
- Go into root/bash again for EC2 instances of OnPrem Router 1 then Router 2, in /home/ubuntu/demo_assets/ there will be a script file 'ffrouting-install.sh'
    - make it executable (chmod +x ffrouting-install.sh) and execute (./ffrouting-install.sh) while in directory
- Up to this point, the AWS side (10.16.0.0/16) and OnPrem 192.168.10.0 - 192.168.12.0/24 are still segmented and have no awareness of one another

- Connect back into OnPrem R1 via session manager
    - In order to configure BGP, we need to go into the shell of FFRouting by typing in the command 'vtysh'
    - 'conf t'                                                    - go into configuration mode
    - 'frr defaults traditional'                                  - reflects default profile adhering to IETF standards
    - 'router bgp 65016'                                          - customer side BGP ASN from earlier
    - 'neighbor <connection1_tunnel1_aws_bgp_ip> remote-as 64512' - configure relationship between onprem asn and aws asn used by transit gateway
    - 'neighbor <connection1_tunnel2_aws_bgp_ip> remote-as 64512' - configure same for second tunnel which uses different AWS endpoint **For HA**
    - 'no bgp ebgp-requires-policy'                               - turns off requirement of incoming/outgoing filters applied to eBGP sessions (RFC8212); disable to make our isntance of BGP work with AWS
    - 'address-family ipv4 unicast'                               - 
    - 'redistribute connected'                                    - redistribute any networks it is aware of
    - 'exit-address-family'
    - 'exit'
    - 'exit'
    - 'wr'                                                        - write/save to memory the configuration
    - 'exit' to exit shell then 'sudo reboot' to reboot the instance/OnPrem Router 1
 
 - verify by going back to AWS console, should see both tunnels UP with BGP routes being advertised
    - under Transit Gateway Route Tables, there should be routes dynamically learned via OnPrem Routers to the Transit gateway via BGP through the VPN connections
    - can log back into the onPrem routers and issue a 'route' command or get more in detail by logging back into FFrouting via 'vtysh' and issuing a 'show ip route' should show BGP learned routes via its vti1 and 2 tunnel (HA and redundancy)

- Should now be able to log into either side AWS or OnPrem instances created from the CloudFormation Stack and be able to ping and have reachability to either side.

     
    
