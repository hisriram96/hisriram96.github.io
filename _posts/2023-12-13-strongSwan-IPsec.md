---
layout: post
title: "Configure IPsec VPN in an Ubuntu VM using strongSwan"
author: Sriram H. Iyer
---

## What is strongSwan?

[strongSwan](https://docs.strongswan.org/docs/5.9/howtos/introduction.html) is an open-source package for IPsec VPN. You could use strongSwan for configuring VPN in Linux, UNIX, and BSD Operating Systems.

In this example, we will configure an IPsec VPN in an Ubuntu VM. However, same configuration can be applied for RedHat and other distos.

Please note that the IPsec VPN is setup between an Azure VM and Azure Virtual Network Gateway which are in different VNets and there is no connecitivity between VNets in any manner. So, before proceeding with rest of the Blog, please create Azure VMs and Virtual Network Gateway in different VNets with public IPs. You could perform this excerise in AWS as well by creating EC2 instances and Virtual Private Gateway in different VPCs.

## Network Architecture

Before we configure IPsec VPN using strongSwan, we need to deploy Azure VMs with Public IPs which are in different VNets and there is no connecitivity between VNets in any manner. You could perform this excerise in AWS as well by creating two EC2 instances in different VPCs.

![network-diagram](https://raw.githubusercontent.com/hisriram96/blog/main/_pictures/strongSwan-network-diagram.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2FstrongSwan-azure-deployment.json)

## Enable IP forwarding

Before configuring IPsec VPN, we must make sure that our VMs act as routers.

Router can accept and forward packets which are not destined to itself. This behaviour of router is called *IP forwarding*. However, servers/VMs would discard the packets which have destination IP differnt from the its own.

This is default behaviour for servers/VMs and we can change it. We can configure the OS of the server/VM to accept the packets which have destination IP differnt from the its own interface IP. This configuration differs depending on the OS.

In Ubuntu distros, you could enable IP forwarding by modifying the contents of ```/etc/sysctl.conf```. One of the easy way to do it is by using ```sed``` command as below.

```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf
sudo sysctl -p
```

Example:

<img width="724" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/9a2080c1-957a-4ef6-a455-f04e3899cee0">

Since we deployed Azure VMs, we must also enable IP forwarding in the vNIC of the Azure VMs.

Please note that enabling IP forwarding in the vNIC of the Azure VM is an additional step as it does not enable IP forwarding in the guest OS so we still need to enable IP forwarding in Ubuntu OS.

<img width="1184" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/056e8a7a-7d29-47d3-a540-95a1bfca7e1c">


## Configuring strongSwan

With our network infrastructure ready and IP forwarding enabled in the OS and in VNIC, we could proceed with the configuration of IPsec VPN.

1. Install strongSwan.

   ```bash
   sudo apt-get update
   sudo apt-get install strongswan -y
   sudo apt-get install strongswan-pki -y
   sudo apt-get install libstrongswan-extra-plugins -y
   ```

   Example:
   
   <img width="725" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/8a52568b-694a-4b9c-90bc-3aa6b84927d7">

3. Configure IPsec VPN by editing the ipsec.conf file.

   ```bash
   sudo vi /etc/ipsec.conf
   ```

   Contents of the ```ipsec.conf``` file.

   ```bash
   config setup
		   charondebug="all"
		   uniqueids=yes
   conn tunnel21
		   type=tunnel
		   left=<Private_IP_address_of_the_VM>
		   leftsubnet=<Local_IP_prefix>
		   right=<VPN_peer_IP_address>
		   rightsubnet=<Remote_IP_prefix>
		   keyexchange=ikev2
		   keyingtries=%forever
		   authby=psk
		   ike=aes256-sha256-modp1024!
		   esp=aes256-sha256!
		   keyingtries=%forever
		   auto=start
		   dpdaction=restart
		   dpddelay=45s
		   dpdtimeout=45s
		   ikelifetime=28800s
		   lifetime=27000s
		   lifebytes=102400000
   ```

   Example:

   <img width="287" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/708c0939-4f6e-4236-9ea7-d5b3c2fa4a35">
   
5. Configure pre-shared key for VPN in ipsec.secrets file.

   ```bash
   sudo vi /etc/ipsec.secrets
   ```

   Contents of the ```ipsec.secrets``` file.

   ```bash
   <Private_IP_address_of_the_VM> <VPN_peer_IP_address> : PSK "<pre-shared_key>"
   ```

   Please make sure that the same pre-shared key is configured on both IPsec peers. Otherwise, VPN will be down.

   Example:

   <img width="328" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/f9f23e3b-4bc3-401b-b66a-45b83cbab3f9">

7. Restart the strongSwan process.

   ```bash
   sudo systemctl restart ipsec
   sudo systemctl status ipsec
   ```

   Example:

   <img width="901" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/a604f765-08a1-4a44-8904-0219863772e3">

## Verification of IPsec

You could verify if the IPsec is established by executing the command ```sudo ipsec status```.

You could perform stop and start operations using command ```sudo ipsec stop``` and started using ```sudo ipsec start``` commands. Please refer to the [man page](https://manpages.ubuntu.com/manpages/noble/en/man8/ipsec.8.html) of ```ipsec``` command.

In case the IPsec doess not establish, you could troubleshoot with the help of IPsec logs by using the command below.

```bash
sudo cat /var/log/syslog | grep "ipsec"
```

<link rel="alternate" type="application/rss+xml"  href="{{ site.url }}/feed.xml" title="{{ site.title }}">
