# Deploy Hyper-V on an Amazon EC2 Bare Metal instance

This repository contains a sample CloudFormation template to deploy Hyper-V on an Amazonn EC2 Bare Metal instance. It also creates the supporting network components including VPCs, subnets and security groups. Instructions are also provided (below) to configure Hyper-V networking and launch a Windows Server 2019 guest VM. 

***Note:*** If you are following the blog 'Using AWS Elastic Disaster Recovery to protect Hyper-V workloads on-premise', this CloudFormation template deploys an infrastructure that simulates an on-premise Hyper-V environment. Once deployed, you can follow the blog instructions to set up AWS Elastic Disaster Recovery, install the replication agent, initiate a drill and perform a failback.

## Deployed architecture
![drs-blog-solution-github](https://github.com/aws-samples/disaster-recovery-for-on-premise-hyperv-using-drs/assets/91114681/62be7eb1-0425-46eb-be3a-e17921433117)

The CloudFormation template will set up the following components:
- Three VPCs
  - 'blog-primary-vpc'
  - 'blog-staging-vpc'
  - 'blog-recovery-vpc'
- Three subnets
  - 'blog-primary-public-subnet'
  - 'blog-staging-public-subnet'
  - 'blog-recovery-public-subnet'
- Internet Gateways, Security Groups and a DRS IAM replication agent user
- A Hyper-V bare metal instance in the ‘blog-primary-vpc’ to simulate the on-premise workload

***Note: You will be billed for the AWS resources created if you create a stack from this template.***

Steps to configure the required networking on the Hyper-V bare metal instance and launching a Windows Server 2019 guest VM are provided below. Once complete, you can then follow the blog to set up AWS Elastic Disaster Recovery, install the replication agent and server, and perform a failover to a recovered Windows 2019 instance.

### Pre-requisites
For this walkthrough, you should have the following prerequisites: 
- An [AWS account](https://aws.amazon.com/resources/create-account/ "Create account")
- An Amazon EC2 key pair (required for authentication). For more details, see [Amazon EC2 key pairs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html "EC2 key pairs")

### Deploy the infrastructure CloudFormation Template
1.	Log in to the AWS Management Console and open the ‘CloudFormation’ service.
2.	Create the infrastructure stack using the hyperv-drs-blog-setup.yaml template
3.	Note the ‘DRSIAMAccessKeyId’ and ‘DRSIAMSecretAccessKey’ values in the ‘Outputs’ tab for the created stack as these will be used as part of the replication agent installation.

### Configure Hyper-V Networking
1. Connect to the Hyper-V bare metal instance using Remote Desktop.
2. Create an internal virtual switch to act as a NAT gateway for any new Hyper-V guest Virtual Machines (VM). Using PowerShell, run the following:

`New-VMSwitch -SwitchName “Hyper-VSwitch” -SwitchType Internal`

3. Determine which network interface is associated with the virtual switch by running:

`Get-NetAdapter`  

In our example, the Get-NetAdapter command shows that the Hyper-V virtual switch has an ifIndex value of 15:
![drs-blog-get-adapter](https://github.com/aws-samples/disaster-recovery-for-on-premise-hyperv-using-drs/assets/91114681/8eec3857-c659-4c98-9b89-0a25feaf4c23)

4. Run the following PowerShell command to configure the Hyper-V Virtual Ethernet adapter with the NAT gateway IP address. This IP address is used as default gateway (Router IP) for the guest VMs. The following command sets the IP address 192.168.0.1 with a subnet mask 255.255.255.0 on the Interface (Interface Index 15):

`New-NetIPAddress -IPAddress 192.168.0.1 -PrefixLength 24 -InterfaceIndex 15`

5. Run the following PowerShell command to create a NAT virtual network using the range of 192.168.0.0/24:

`New-NetNat -Name MyNATnetwork -InternalIPInterfaceAddressPrefix 192.168.0.0/24`

6. Now the environment is ready for the guest VMs to have outbound communication with other resources through the host NAT. To configure DHCP for guest VMs to assign IPs automatically, run the following:

`Install-WindowsFeature -Name 'DHCP'-IncludeManagementTools`

7. Run the following PowerShell command to configure the DHCP scope and specify a range from the subnet that you determined earlier. In this example, we use the range from 192.168.0.10 to 192.168.0.20:

`Add-DhcpServerv4Scope -Name GuestIPRange -StartRange 192.168.0.10 -EndRange 192.168.0.20 -SubnetMask 255.255.255.0 -State Active`

8. To configure the DHCP server to bind on the Hyper-V virtual interface, choose Control Panel > Administrative Tools > DHCP. Add the localhost as a server.
9. Once the server has been added, right-click on the server and choose ‘Add or Remove Bindings’ to confirm that the binding to virtual interface is selected.
10. Configure the DHCP server options for ‘Router’ and ‘DNS Server’, where the router IP is the NAT gateway IP created earlier, and the DNS server IP is the Amazon provided DNS server in your VPC. In our example, the ‘blog-primary-public-subnet’ CIDR is 10.0.0.0/24, so the IP address of the DNS server is the base of the VPC network range plus two (which is 10.0.0.2). Run the following command:

`Set-DhcpServerV4OptionValue -DnsServer 10.0.0.2 -Router 192.168.0.1`

### Launch a Windows Server 2019 guest VM on Hyper-V
1.	Download the image (VHD File format) from https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019
2.	Open the Hyper-V Manager and create a new Virtual Machine (VM). The blog parameters used for the ‘New Virtual Machine Wizard’ are:
    - Name: Windows-2019-VM
    - Choose the generation of this virtual machine: Generation 1
    - Startup memory: 10240 MB
    - Connection: Hyper-VSwitch
    - Use an existing virtual hard disk: C:\<path to file>\17763.737.amd64fre.rs5_release_svc_refresh.190906-2324_server_serverdatacentereval_en-us_1.vhd
3.	Start and connect to the Virtual Machine

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

