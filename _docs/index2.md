
# Infoblox Anycast DNS in Multipod ACI

## Purpose  

This article is a way for me to both document my experience migrating Anycast DNS from our Legacy 7K environment using IP SLAs to ACI using L3outs and BGP to advertise the Anycast Host routes.

IP SLAs are a great way to track services and take action based on changes to those services.  Depending on the version of software your core is running, there are limitations on what can be monitored.  In our environment, we used the ICMP probe to monitor the physical Infoblox node and drop routes if it were to go down. The biggest fault of this set up is that IP SLA runs on the supervisor, so it is susceptible to dropping packets during peak times or when getting scanned by your security team.  The other drawback we faced is that after tuning, we could only get timers down to a 4s threshold before the tracked routes were pulled.

### Topics Discussed

- BFD - Bi-Directional Fordwarding Detection
- Anycast
- ACI L3OUT Configuration
- ACI BGP Peering (iBGP)
- ACI Transit Routing

##  Assumptions

- You already have a running fabric with spines, leafs, and at least two pods.
	- Connectivity to external WAN is established
	- Connectivity to IPN and other pods is established
- You have a working ACI Fabric running **3.0+**
	- Most of the configuration and workflows will work with 2.2+, but screenshots will probably not match up with earlier versions.
- You have a working Infoblox grid running **NIOS v8.3+**
	- BFD was introduced in NIOS v8.3 but was buggy.  Recommendation is to run at least NIOS v8.5
- You are using ***physical*** Infoblox appliances that are joined to your grid
	- While using VMs will work, there are extra steps and considerations to consider that are beyond the scope of this document.
- For this exercise we will use the following IP addresses:
	- Infoblox Information:
		- Anycast DNS Address: `172.172.172.172`
		- BGP ASN: 65500 [This will be the same as your fabric's underlay ASN]
		- Infoblox HA VIP: `10.50.50.50/24` [used for BGP peering, Router ID of Infoblox]
		- Infoblox Node1
			- LAN1:  `10.50.50.51/24` [local address of HA Node1, assumed to be operational Primary]
			- Infoblox Node1 HA: `10.50.50.61/24` [HA Address, used for heartbeats]
		- Infoblox Node2
			-  LAN1:`10.50.50.52/24` [local address of HA Node2, assumed to be operational Secondary]
			- Infoblox Node2 HA: `10.50.50.62/24` [HA Address, used for heartbeats]
	- ACI Leaf Information:
		- ACI Underlay ASN: `65500` [This should match ASN in Infoblox]
		- Leaf1 Peering Addresses: `10.50.50.2/24`
		- Leaf2 Peering Addresses: `10.50.50.3/24`
		- Leaf1/2 Secondary Addresses: `10.50.50.1/24` [This is configured on the Logical Interface Profile of L3OUT so that each leaf can respond as 10.50.50.1 GW address

## Example Topology

[@@TODO: There will be some graphics here]

## Outline of work

At a high level, the steps needed are:
1. *INFOBLOX*: 
	1. Create BFD Template
	1. Create Loopback (Anycast) interfaces
	1.  Enable Listening on Anycast addresses for Services (DNS, TFTP, NTP, etc)
2. *ACI*:
	1. Create External Routed Domain and VLAN Pool
	1. Create a new L3OUT
		1. Create Node Profile(s)
		2. Create Logical Interface Profile(s)
		3. Create BGP Timer profile
		4. Create BFD Profile
		5. Create BGP Peer Connectivity Profile
	1.  Use External Route Control EPGs to allow Anycast addresses to be advertised on other L3OUTs (Transit Routing)

## Do Some Things

### Infoblox
 
 1. Create BFD Profile in Infoblox
	1. Navigate to `Grid > Members > BFD Templates [right side]`
		1. Name: `BFD-ACI`
		2. Choose an Authentication Type (or none)
	    2. Choose BFD interval settings.  In my environment, I chose to stick with Infoblox's defaults of 100ms/100ms/3 Multiplier to take into account when you make a DNS change that requires a service restart.  ACI's default is 50ms/50ms/3.  
		    > **What you choose here must match the config in ACI**  
		    
1. Create Loopback Interfaces
	1. Navigate to `Grid > Members` and for each member that you want to use Anycast:
		1.  `Edit the Member`
		2. Navigate to `Network`
			1. Set up Loopback Addresses (`172.172.172.172`).  For DNS, you'll want at least two addresses to stick with DNS nameserver standards.  You can also create more to have an anycast address per service.  Like one for TFTP anycast, one for NTP anycast, etc.  Having one addresses for everything is just fine, too.
	1. Navigate to `Anycast`
	    1. BGP Configuration:
	        1. **ASN**: [Ex. 65500] This should be the ASN of your ACI Underlay
	        1. **BGP Timers**: Default [ 4 / 16 / 32 ].  Since we will be using BFD, the default timers should suffice for the majority of environments.
	    1. BGP Neighbor Configuration:
		    1. ***NOTE*:** Infoblox HA Pair and BGP works  in an Active/Standby mode.  Only the Active member of an HA pair will establish a BGP and BFD session with the neighbor routers.  You will want to pair with *both* ACI Leafs' local address. (@@section for topology). 
	        1. For each neighbor [10.50.50.2, 10.50.50.3]
	            - **Remote ASN**: [Ex. 65500] This ASN needs to match the ASN above, since we want to peer iBGP
	            - **Enable BFD**, select the profile you created earlier [*BFD-ACI*]

### ACI 

#### Create a New Route Domain

> It is best practice to create a new Domain, VLAN Pool, and AAEP for each L3OUT or groups of L3OUTS

You can then use this domain across your pods/sites for where ever you have Anycast DNS.

1. Navigate to `Fabric > Access Policies > Physical and External Domains > External Routed Domains`
    1. Name: `DOM-ANYCAST`
    1. AAEP: `AAEP-ANYCAST`
    1. VLAN Pool: `VLPOOL-ANYCAST`
	    1. VLANS: `100-110`

#### Create EXPORT-ROUTE-CONTROL External EPG

> I created a specific L3EPG in our WAN facing L3out specifically for transit routing.
> Routes from IB will be distributed into MP-BGP of the Fabric, and then redistributed into EIGRP and out to WAN.  

Because the routes from Anycast DNS will belong to it's own L3out, we must use transit routing to allow these routes to be advertised out to your WAN

1. Navigate to `Tenants > [_TENANT_] > Networking > External Routed Networks > [_L3OUT_WAN_] > Networks`
    1. Create `EXPORT-ROUTE-CONTROL` network
        1. Description: `Required for Transit routing. Used to allow routes learned from other L3OUTs to be disributed out this L3OUT.`
        1. Subnets:
            1. For each network, scope will be set to **only** `Export Route Control Subnet`
            1. This will need to be the /32 routes of your anycast addresses AND the the network that the Infoblox local addresses live in - [172.172.172.172/32 , 10.50.50.0/24]


### Create New L3-Out for Infoblox Network
> This creates a psuedo-SVI on the leaf switches so that it can respond as the gateway as well as for a peering point.  We will peer the *Infoblox VIP* address with ACI Leaf's *local* SVI address.  This is due to ACI sourcing BFD packets from the SVI and not a loopback.

1. Navigate to `Tenants > _Tenant_ > Networking > External Routed Networks`
    1. Settings
        1. Name: `<Name of Anycast L3OUT`
        1. VRF: `<VRF of your tenant>`
        1. External Routed Domain: [DOM-ANYCAST] `<Domain you created earlier>` 
        1. Place tick in ***BGP***
    1. Create Logical Node Profile:
        1. [@@TODO check this] Name: `LNP-LEAF1` (I like to make a LNP per Leaf, you can combine nodes into one profile if you'd like)
            1. Create BFD Protocol Profile
                1. Name: `BGP-INFOBLOX`
                    1. Settings: 4/16/32, Disable Graceful Restart Helper
            1. Create Logical Interface Profile
                1. Name: `LIP-LEAF1`
                    1. Create SVI for each port that appliance is connected to on this leaf [N101/Eth1-2]
                        1. **NOTE:**  Each LAN1 and HA port must have a Path, otherwise HA pair will show broken in Grid Manager.
                        1. Settings:
                            1. Encap: [100] `<Vlan from Vlan Pool created earlier>`
                            1. Encap Scope: `VRF`
                            1. Mode: `802.1P`
	1. Repeat for other Nodes if not combining Interface Profiles


### References
- [Cisco ACI L3 Networking Documentation](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/2-x/L3_config/b_Cisco_APIC_Layer_3_Configuration_Guide/b_Cisco_APIC_Layer_3_Configuration_Guide_chapter_010100.html)
- [Infoblox BGP Configuration](https://docs.infoblox.com/display/NAG8/Chapter+24+Configuring+IP+Routing+Options)