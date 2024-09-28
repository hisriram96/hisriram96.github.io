---
layout: post
title: "Configure BGP in an Ubuntu VM using BIRD"
author: Sriram H. Iyer
---

## What is BIRD?

[The BIRD Internet Routing Daemon](https://bird.network.cz/) is an open-source project for configuring dynamic IP routing in Linux, FreeBSD, and UNIX based Operating Systems.

BIRD allows the configuration of multiple routing protocols, such as RIP, OSPF, and BGP, on a server or virtual machine, transforming it into a router or, more precisely, a route server.

In this example, we will configure BIRD with multiple BGP sessions in an Ubuntu VM, including sessions to an Azure Route Server and two Virtual Network Gateways over VPN. The same configuration can be applied to RedHat and other distributions.

## Pre-requisites

Since we will be deploying resources in Azure, you would need an Azure subscription to follow through. Please note that Virtual Network Gateways of "VpnGw1" SKU are free for first 12 months for 750 hours, Azure Route Server, and Azure Bastion are not free services as stated in https://azure.microsoft.com/en-in/pricing/purchase-options/azure-account?icid=azurefreeaccount#free-services if you are using [Azure free account](https://azure.microsoft.com/en-in/pricing/offers/ms-azr-0044p/).

## Network Architecture

We would be deploying three VNets. Two VNets would have Virtual Network Gateways and the third VNet would have an Azure Route Server and Azure Bastion for accessing the VM securely.

![architectural-diagram](https://raw.githubusercontent.com/hisriram96/blog/main/_pictures/Azure-deployment-strongSwan-and-BGP.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2FAzure-deployment-strongSwan-and-BGP.json)

## Enable IP forwarding

Before proceeding with configuration of VPN and BGP, we must enable *IP forwarding* in the guest OS (Ubuntu) of the VM. The Azure NIC has been enabled with IP forwarding in the ARM template.

Please note that the commands to enable *IP forwarding* could differ depending on the distro.

```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sed -i 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf
sudo sysctl -p
```

Example:

<img width="857" alt="image" src="https://github.com/user-attachments/assets/37d1e4c1-4087-4029-9a44-73e3939b40d6">

## Installing strongSwan and BIRD

We need to install strongSwan for configuring VPN to the Virtual Network Gateways and BIRD for configuring BGP.

```bash
sudo apt-get update
sudo apt-get install strongswan -y
sudo apt-get install strongswan-pki -y
sudo apt-get install libstrongswan-extra-plugins -y
sudo apt-get install bird -y
```

Example:

<img width="920" alt="image" src="https://github.com/user-attachments/assets/2bae30a8-3c88-4ea8-9111-eb2dd7436a14">

## Configuring VPNs using StrongSwan to Virtual Network Gateways

We need to establish VPNs to the Virtual Network Gateways so that we could configure BGP over these VPNs.

The step by step process for configuring VPN using stringSwan is explained in this blog: [Configure IPsec VPN in an Ubuntu VM using strongSwan](https://blog.hisriram.com/2023/12/13/strongSwan-IPsec.html).

We would configure two IPsec tunnels to the Virtual Network Gateway by editing the `ipsec.conf` file.

```bash
sudo vi /etc/ipsec.conf
```

Contents of the ```ipsec.conf``` file.

```bash
config setup
      charondebug="all"
      uniqueids=yes
conn vnetgatewaytunnel1
      type=tunnel
      left=<Private_IP_address_of_the_VM>
      leftsubnet=<Addess_space_of_the_VNet_of_VM>
      right=<Public_IP_address_of_Virtual_Network_Gateway_1>
      rightsubnet=<Addess_space_of_the_VNet_of_Virtual_Network_Gateway_1>
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
conn vnetgatewaytunnel2
      type=tunnel
      left=<Private_IP_address_of_the_VM>
      leftsubnet=<Addess_space_of_the_VNet_of_VM>
      right=<Public_IP_address_of_Virtual_Network_Gateway_2>
      rightsubnet=<Addess_space_of_the_VNet_of_Virtual_Network_Gateway_2>
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

<img width="347" alt="image" src="https://github.com/user-attachments/assets/d0348a44-f837-4fc3-b97f-2e5f1f6e3723">

We would the configure pre-shared key for VPNs in `ipsec.secrets` file.

```bash
sudo vi /etc/ipsec.secrets
```

Contents of the ```ipsec.secrets``` file.

```bash
<Private_IP_address_of_the_VM> <Public_IP_address_of_Virtual_Network_Gateway_1> : PSK "<pre-shared_key>"
<Private_IP_address_of_the_VM> <Public_IP_address_of_Virtual_Network_Gateway_2> : PSK "<pre-shared_key>"
```

Please make sure that the same pre-shared key is configured on VM and both Virtual Network Gateways. Otherwise, VPNs will be down.

Example:

<img width="367" alt="image" src="https://github.com/user-attachments/assets/af0f0b9e-87ab-4852-a307-ecbbaea3afdc">

We would then restart the strongSwan process.

```bash
sudo systemctl restart ipsec
sudo systemctl status ipsec
```

Example:

<img width="1115" alt="image" src="https://github.com/user-attachments/assets/07b12324-c27b-47c0-b645-14247e68605c">

We could verify if the VPNs are established to both Virtual Network Gateways by executing commands below:

```bash
sudo ipsec status
```

Example:

<img width="713" alt="image" src="https://github.com/user-attachments/assets/23b91bd1-cebf-4f8d-b058-34a76f61a2ea">

## Identifying the BGP peer IP addresses

We will configure BIRD with four BGP sessions in the VM. Two BGP peerings will be towards the Virtual Network Gateways, and the other two BGP peerings will be towards two instances of the Azure Route Server.

The Virtual Network Gateways will have their own BGP peering addresses, visible in the configuration blade of the Azure Portal. We will need these BGP peering addresses to configure BGP in the VM.

Example:

<img width="587" alt="image" src="https://github.com/user-attachments/assets/96997dd3-86c4-4430-b247-f75a2c625c13">

The Azure Route Server is a managed service that enables dynamic routing in the Virtual Network using BGP. We will establish BGP peerings from our VM and advertise routes learned from the Virtual Network Gateways to the Azure Route Server. The Azure Route Server will then propagate these routes within the Virtual Network, eliminating the need for user-defined routes on each subnet.

The Azure Route Server will have two instances, each with its own BGP peering IP address. These IP addresses will be visible in the overview blade of the Azure Portal.

Example:

<img width="1185" alt="image" src="https://github.com/user-attachments/assets/b48955f3-6324-46ab-b8f5-fc6b63fdf790">

## Configuring BGP using BIRD

Now that we know what are the BGP peer IPs, we could proceed with BGP configuration in our VM by editing the `/etc/bird/bird.conf` file.

```bash
sudo vi /etc/bird/bird.conf
```

Contents of the `/etc/bird/bird.conf` file.

```bash
router id <Private_IP_address_of_the_VM>;
protocol kernel {
      scan time 60;
      import none;
      export all;
}
protocol device {
      scan time 60;
}
protocol direct {
      interface "eth0";
}
protocol static {
      route <Private_IP_address_of_Virtual_Network_Gateway_1>/32 via <Private_IP_address_of_the_VM>;
      route <Private_IP_address_of_Virtual_Network_Gateway_2>/32 via <Private_IP_address_of_the_VM>;
}
protocol bgp vnetgateway1 {
      router id <Private_IP_address_of_the_VM>;
      local <Private_IP_address_of_the_VM> as 65523;
      neighbor <Private_IP_address_of_Virtual_Network_Gateway_1> as 65521;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import all;
      export all;
      enable route refresh on;
}
protocol bgp vnetgateway2 {
      router id <Private_IP_address_of_the_VM>;
      local <Private_IP_address_of_the_VM> as 65523;
      neighbor <Private_IP_address_of_Virtual_Network_Gateway_2> as 65522;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import all;
      export all;
      enable route refresh on;
}
protocol bgp azurerouteserverinstanceprimary {
      router id <Private_IP_address_of_the_VM>;
      local <Private_IP_address_of_the_VM> as 65523;
      neighbor <Private_IP_address_of_Azure_Route_Server_primary_instance> as 65515;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import filter {
          if net = <Addess_space_of_the_VNet_of_VM> then reject;
          else accept;
      };
      export all;
      enable route refresh on;
}
protocol bgp azurerouteserverinstancesecondary {
      router id <Private_IP_address_of_the_VM>;
      local <Private_IP_address_of_the_VM> as 65523;
      neighbor <Private_IP_address_of_Azure_Route_Server_secondary_instance> as 65515;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import filter {
          if net = <Addess_space_of_the_VNet_of_VM> then reject;
          else accept;
      };
      export all;
      enable route refresh on;
}
```

Example:

<img width="246" alt="image" src="https://github.com/user-attachments/assets/fb31caf9-b29e-4a5f-b596-fa5d669a7216">

The BGP configuration in the BIRD configuration file `/etc/bird/bird.conf` is specified in the below code block.

```bash
protocol bgp uniquebgpinstance {
...
}
```

Since we have configured 4 BGP sessions, we have 4 code blocks for BGP protocol.

In our BIRD configuration file, we have two BGP sessions towards the Virtual Network Gateways. The IP addresses 10.0.1.30 and 10.0.2.30 are BGP peer IP addresses of the Virtual Network Gateways and 10.0.0.4 is the IP address of our VM. We have used ASNs 65521 and 65522 in the Virtual Network Gateways and ASN 65523 for our VM. The `import all` and `export all` in the BBP code block allows learning and advertising all routes by VM.

```bash
protocol bgp vnetgateway1 {
      router id 10.0.0.4;
      local 10.0.0.4 as 65523;
      neighbor 10.0.1.30 as 65521;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import all;
      export all;
      enable route refresh on;
}
protocol bgp vnetgateway2 {
      router id 10.0.0.4;
      local 10.0.0.4 as 65523;
      neighbor 10.0.2.30 as 65522;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import all;
      export all;
      enable route refresh on;
}
```

In addition to the BGP code block, we have configured static routes to BGP peer IP addresses 10.0.1.30 and 10.0.2.30 of the Virtual Network Gateways. This is because we do not have any route in kernel to these IP address as we have configured policy-based VPN.

```bash
protocol static {
      route 10.0.1.30/32 via 10.0.0.4;
      route 10.0.2.30/32 via 10.0.0.4;
}
```

In our architecture, we have deployed an Azure Route Server in the same Virtual Network as our VM to advertise routes learned by the Virtual Network Gateways. We would configure BGP sessions to both instances of the Azure Route Server using the BGP peer IP addresses 10.0.0.36 and 10.0.0.37.

In order to avoid duplicate routes, we would *reject* the address space 10.0.0.0/24 advertised by Azure Route Server as it is of the Virtual Network where our VM is deployed.

```bash
protocol bgp azurerouteserverinstanceprimary {
      router id 10.0.0.4;
      local 10.0.0.4 as 65523;
      neighbor 10.0.0.36 as 65515;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import filter {
          if net = 10.0.0.0/24 then reject;
          else accept;
      };
      export all;
      enable route refresh on;
}
protocol bgp azurerouteserverinstancesecondary {
      router id 10.0.0.4;
      local 10.0.0.4 as 65523;
      neighbor 10.0.0.37 as 65515;
      multihop;
      next hop self;
      keepalive time 60;
      hold time 180;
      import filter {
          if net = 10.0.0.0/24 then reject;
          else accept;
      };
      export all;
      enable route refresh on;
}
```

We would be exporting all BGP routes to the kernel of our VM.

```bash
router id 10.0.0.4;
protocol kernel {
        scan time 60;
        import none;
        export all;
}
```

We would restart the BIRD daemon for the configuration to take effect.

```bash
sudo systemctl restart bird
sudo systemctl status bird
```

Example:

<img width="637" alt="image" src="https://github.com/user-attachments/assets/c00ecd51-048c-43f1-a1ec-5d6141ff3edd">

## Verifying BGP configuration

We could verify the BGP neighborship state by executing the command `sudo birdc show protocols`.

Example:

<img width="548" alt="image" src="https://github.com/user-attachments/assets/0fcd7ab5-c39f-478b-918b-dfaab7a3b4db">

We could verify routes learned by BGP by executing the command `show birdc show route`.

<img width="657" alt="image" src="https://github.com/user-attachments/assets/4065d711-08f1-4584-9500-fd22d0099eb5">

We could verify if the BGP learned routes are visible in the kernel by executing the command `ip route` or `netstat -rn`.

<img width="526" alt="image" src="https://github.com/user-attachments/assets/c62fd991-54d5-4663-81ab-117020581ed6">
