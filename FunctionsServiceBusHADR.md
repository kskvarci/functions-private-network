# Service Bus - Competing Consumers - HA / DR

## TOC

[Overview](#Overview)

[Virtual Network Foundation](#Virtual-Network-Foundation)

[Deploying a base Function App](#Deploying-a-base-Function-App)

[Monitoring Setup](#Monitoring-Setup)

[Triggers and Bindings](#Triggers-and-Bindings)  

## Overview

Azure functions are  simple to get up and running in their default configuration. Things get significantly more complex when integrating into private networks where the flow of traffic is controlled more granularly. Below is a diagram showing a fully locked down implementation for a single region in Azure. We'll walk through simulating this in a test environment.  


## Networking Walk-Through (Single Region)
![](images/networkingsingle.PNG)
### Producer Flow
1. A producer application (1) in the Workload Subnet is set up to send messages to a Service Bus Queue running in Namespace A (2).  

2. Namespace A (2) has been configured with a private endpoint so that it is only accessible via a private endpoint (6). E.G. there is no Internet facing access to this namespace.  

3. When the producer application (1) connects to the Namespace (2), it first issues a DNS query to obtain the namespace's IP address.  

4. The VNet that the producer application (1) is running in has been configured with a custom DNS setting that causes VMs running in that VNet to use a DNS forwarder (3) in the Hub VNet for DNS resolution.  

5. The DNS forwarder is set up to forward queries for private zones to the internal resolver (4). The internal resolver (4) returns the IP address of the private endpoint (6) instead of the public IP for the namespace as the VNet it's running in has been linked to a private zone (5) specifically configured for Private Link on Namespace A (2) .  

6. Now that the app (1) has the private endpoint (6) IP, it is able to connect to the Service Bus Queue in Namespace A (2) so that it can send messages.  

### Consumer Flow
1. A consumer function (7) is running in an App Service Plan. It is configured with Regional VNet Integration and is connected to a integration subnet (8) in the Spoke VNet. All egress from the function is sent down the VNet integration using the WEBSITE_VNET_ROUTE_ALL application setting.  

2. Code running in the function is set up to use a Service Bus trigger and is constantly polling the Service Bus Queue in the namespace (2).  

3. When connecting to the queue, the function first needs to issue the same DNS query that the producer app issued. The query for the namespace IP is sent down the Regional VNet integration (8) and into the integration subnet.  

4. The VNet that the function has been integrated into is configured with a custom DNS entry that causes the function to send it's DNS query to the fowarder in the hub (3).  

5. The DNS forwarder is set up to forward queries for private zones to the internal resolver (4). The internal resolver (4) returns the IP address of the private endpoint (6) instead of the public IP for the namespace as the VNet it's running in has been linked to a private zone specifically configured for Private Link on Service Bus (5).   

6. Now that the function app (7) has the private endpoints (6) IP it is able to connect to the Service Bus Queue in Namespace A (2) through the Regional VNet integration (8) and private endpoint (6) to read messages when they arrive. 

## Cross-Region redundancy for non-HTTP functions triggering on Service Bus
![](images/networkingDR.PNG)
1. Service Bus itself will be setup for Geo Replication across two regions. This causes the primary namespace entities ( no data ) to be replicated to a secondary namespace. In this example we're replicating From Namespace A in East US 2 to Namespace B in Central US.  

2. Producer apps send messages to the Primary Namespace ( Namespace A to start ) using a DNS alias that is specified when geo-replication is configured. This is a literal DNS alias that redirects clients to Namespace A's DNS name.  

3. Consumer Applications are set up to connect to the primary and secondary namespace directly in each region. This create an active / passive setup for consumers in which the consumers in the secondary region are effectively watching a queue that has an identical set of entities to the primary namespace but zero messages under normal operating conditions.

4. In the case of a failure, the Namespaces are failed over. At this time, the alias is automatically re-pointed to Namespaces B's DNS name.  

5. Messages coming in through producers begin to flow into Namespace B in Central US which is now the primary namespace.

6. Consumers in Central US now begin picking up messages from Namespace B.



## Networking Walk-Through (Cross-Region Redundancy)
![](images/networkingmulti.PNG)
