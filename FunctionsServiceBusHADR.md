# Serverless Producer / Consumer Pattern in a Locked Down Network
Many cloud native applications are expected to handle a large number of requests. Rather than process each request synchronously, a common technique is for the application to pass them through a messaging system to another service (a consumer service) that handles them asynchronously. This strategy helps to ensure that the business logic in the application isn't blocked while the requests are being processed. For full details on this pattern see the [following article](https://docs.microsoft.com/en-us/azure/architecture/patterns/competing-consumers).

On Azure, the primary enterprise messaging service is [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview).

[Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/#:~:text=Azure%20Functions%20Documentation.%20Azure%20Functions%20is%20a%20serverless,code%20in%20response%20to%20a%20variety%20of%20events.) offers a convenient compute platform from which to implement a producer consumer pattern with relatively little underlying infrastructure management.

Azure Functions and Service Bus are relatively simple to get up and running in their default configurations. Things get significantly more complex when implementing them in environments with stringent security requirements that dictate more aggressive network perimeter security and segmentation.
  
This document explains key considerations for deploying Azure Functions alongside Service Bus in a fully locked down environment using technologies including regional VNet Integration for functions, private endpoints for Service Bus and a variety of network security controls including Network Security Groups and Azure Firewall.

We will also provide guidance on how to achieve redundancy across multiple regions while retaining the same security posture.
## TOC
- [Architecture](#Architecture)
- [Scalability Considerations](#Scalability-Considerations)
- [High Availability Considerations](#High-Availability-Considerations)
- [Disaster Recovery](#Disaster-Recovery)
- [Security Considerations](#Security-Considerations)
- [Cost Considerations](#Cost-Considerations)
- [DevOps Considerations](#DevOps-Considerations) 
## Architecture
### Virtual Network Foundation
#### Requirements
- asdf
- asdf
- asdf
- asdf  
#### Implementation
![](images/networking-foundation.png)
This guide assumes that you are deploying a solution into a networking environment with the following characteristics:

- [Hub and Spoke](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology)  network architecture.   

- The hub VNet (1) is used for hosting shared services like Azure Firewall and providing connectivity to an on-premises networks. In a real implementation, the hub network would be connected to an on-premises network via ExpressRoute, S2S VPN, etc (2). In this reference implementation, we will leave this out.  

- The spoke network (3) is used for hosting business workloads. In this case we're integrating our Function App to a dedicated subnet ("Integration Subnet") that sits within the spoke network. We'll use a second subnet ("Workload Subnet") for hosting other components of the solution.  

- The Hub is peered to Spoke.  

- In many locked down environments [Forced tunneling](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-forced-tunneling-rm) is in place. E.G. routes are being published over ExpressRoute via BGP that override the default 0.0.0.0/0 -> Internet system route in all connected Azure subnets. The effect is that there is **no** direct route to the internet from within Azure subnets. Internet destined traffic is sent to the VPN/ER gateway. We can simulate this in a test environment by using restrictive NSGs and Firewall rules to prohibit internet egress. We'll route any internet egress traffic to Azure firewall (4) using a UDR where it can be filtered and audited.  

- Generally, custom DNS is configured on the spoke VNet settings. DNS forwarders in the hub are referenced. These forwarders  provide conditional forwarding to the Azure internal resolver and on premises DNS servers as needed. In this reference implementation we'll deploy a simple BIND forwarder (5) into our hub network that will be configured to forward requests to the Azure internal resolver. 
#### Deploy
1. Create a resource group
	```
	```
2. Deploy the base VNets and Subnets (ARM Template)
	```
	```
3. Deploy and Configure the Integration Subnet for Regional VNet Integration (ARM Template)
	```
	```
4. Deploy and configure Azure Firewall (ARM Template)
	```
	```

___
[Deploy steps 2-3 in one step.](link_url)
___

[top ->](#TOC)    
### Azure Service Bus
#### Requirements
- Predictable / Consistent Performance
- Direct integration to and accessibility from private networks
- No accessibility from public networks

#### Implementation
![](images/networking-servicebus.png)  
The base-level resource for Azure Service Bus is a Service Bus Namespace. Namespaces contain the entities that we will be working with ( Queues, Topics and Subscriptions ).

When creating a namespace in a single region, there are relatively few decisions you'll need to make. You will need to specify the name, pricing tier, initial scale and redundancy settings, subscription, resource group and location.

In our reference implementation we will be deploying a Premium namespace. This is the tier that is recommended for most production workloads due to it's performance characteristics. In addition, the Premium tier supports VNet integration which allows us to isolate the namespace to a private network. This is key to achieving our overall security objectives.   
  
- We'll create a namespace A in East US 2 (1)  

- The namespace will be set up with a private endpoint (2) in the spoke workload subnet. We will configure access restrictions (per-namespace firewall) on the namespace such that the endpoint will be the only method one can use to connect to the namespace. This effectively takes the namespace off the internet.  

- A private DNS zone (3), requisite A record and VNet link will be created such that queries originating from any VNet that is configured to use our central DNS server will resolve the namespace name to the IP of the private endpoint and not the public IP. This is done via split horizon DNS. E.G. externally, the namespace URLs will continue to resolve to the public IP's which will be inaccessible due to the access restriction configuration. Internally the same URLs will resolve to the IP of the private endpoint.    
#### Deploy
1. Create the Namespace
	```
	```
2. Enable Private Link
	```
	```
3. Link the Private DNS Zone
	```
	```
4. Create a test queue and topic
	```
	```
___
[Deploy steps 1-4 in one step.](link_url)
___
[top ->](#TOC) 

### Azure Functions Producer / Consumer
#### Requirements
#### Implementation
#### Deploy
1. tbd
	```
	```
2. tbd
	```
	```
3. tbd
	```
	```
4. tbd
	```
	```
___
[Deploy steps 1-4 in one step.](link_url)
___
[top ->](#TOC)  
## Scalability Considerations
## High Availability Considerations
## Disaster Recovery
## Security Considerations
## Cost Considerations
## DevOps Considerations



































