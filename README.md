# Azure Functions in a Private Network

## TOC

[Overview](#Overview)

[Virtual Network Foundation](#Virtual-Network-Foundation-/-Assumptions)

[Base Functionality](#Base-Functionality)

[Monitoring Setup (App Insights)](#Monitoring-Setup-(App-Insights))

[Triggers](#Triggers)

## Overview

Azure functions are  simple to get up and running in their default configuration. Things get significantly more complex when integrating into private networks where the flow of traffic is controlled more granularly. Below is a diagram showing a fully locked down implementation for a single region in Azure. We'll walk through simulating this in a test environment.

![](images\All.PNG)

## Virtual Network Foundation / Assumptions

This guide assumes that you're trying to deploy Azure Functions into a networking environment with the following characteristics:

- The general architecture is [Hub and Spoke.](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology) 
- The hub us used for hosting shared services like Azure Firewall and providing connectivity to an on-premises networks. In a real implementation, the hub network would be connected to an on-premises network via ExpressRoute, S2S VPN, etc. In our test environment, we wont bother with this. It is however possible to simulate an on premises network by using a third "on-premises" VNet and a VPN connection to the hub if desired.
- The spoke network is used for hosting business workloads. In this case we're integrating our Function App to a subnet that sits within the spoke network.
- The Hub is peered to Spoke.
- In many locked down environments [Forced tunneling](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-forced-tunneling-rm) is in place. E.G. routes are being published over ExpressRoute via BGP that override the default 0.0.0.0/0 -> Internet system route in Azure subnets. The effect is that there is **no** direct route to the internet from within Azure subnets. Internet destined traffic is sent to the VPN/ER gateway. We can simulate this in a test environment by using restrictive NSGs and Firewall rules to prohibit internet egress.
- Generally, custom DNS is configured on the spoke VNet. DNS forwarders in the hub are used to provide conditional forwarding to the Azure internal resolver and on premises DNS servers as needed. For our testing we'll just use the default Azure DNS resolvers.

------

To deploy a simple hub and spoke network for testing you can use [this ARM template](templates\base-network\azuredeploy.json).

------

[top ->](#TOC)

## Deploying a base Function App

1. Create a subnet in your spoke called "integration-subnet". This subnet will be use for your Function App's regional VNet Integration.

2. Set up service endpoints. To make this simple, we'll enable endpoints in the integration subnet for all common services that the function host might need to access via bindings, triggers or normal operations. These include: *Microsoft.Storage*, *Microsoft.AzureCosmosDB*, *Microsoft.EventHub*, *Microsoft.ServiceBus* and *Microsoft.Web*.

3. Create an NSG named "integration-subnet-nsg" and apply it to "integration-subnet".

4. Add a rule to the NSG to deny all traffic from VirtualNetwork to Internet on all ports (*). Set the priority to 500. This will force us to explicitly allow all outbound flows. Setting the rule with a priority of 500 gives us room to create rules above it to open targeted flows.

5. Add NSG rules for the above services on "integration-subnet-nsg" . Add outbound rules to allow traffic to the above services using each services service tag: *Storage*, *AzureCosmosDB*, *EventHub*, *ServiceBus* and *AppService*. Make sure these rules all sit above the internet deny rule. E.G < 500 from a priority perspective.

   ------

   Use this [ARM template](templates\integration-subnet\azuredeploy.json) to create and configure the above subnet.

   ------

   

6. Deploy an App Service Plan. P1V2 is used in this example.

7. Create a Function App on the above created App Service Plan. If you accept the defaults this will create a storage account and an App Insights resource along with the function app.

8. On the storage account for the function app, configure access restrictions to let traffic in only from the integration subnet ("integration-subnet"). This should be available for selection as you've enabled the service endpoint for storage. Also make sure to add in any IP's needed to allow the portal to hit the storage account. This might be your proxy egress IP's or your ISPs IP if you're testing from home.

9. Enable [Regional VNet Integration](https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet#regional-vnet-integration) on the Function App (referencing the subnet created in step 1 - "integration-subnet"). Selecting the subnet to be used for Regional VNet Integration via the portal will also mark it as delegated to Microsoft.Web/serverFarms. If you're not using the portal you need to delegate the subnet in advance of setting it as the integration subnet.

10. Ensure **WEBSITE_VNET_ROUTE_ALL** is set to 1 on the function apps configuration settings. This will force all outbound traffic down the regional VNet Integration.

[top ->](#TOC)

## Monitoring Setup (App Insights)

It's best to get monitoring working before we start deploying functions. The functions host will be default be instrumented with App Insights. Connectivity to App Insights however will be broken in this architecture until we give the function host which is running on the function app a path to the public App Insights endpoints. In the future we can solve this problem by using a private endpoint for Azure Monitor. Today however we need to filter this traffic with Azure Firewall.

1. Deploy Azure Firewall into the Hub VNet. Firewall needs to go into a subnet called AzureFirewallSubnet.
2. Create a User Defined Route (UDR) w/ a route sending address prefix 0.0.0.0/0 to next hop of type "Virtual Appliance" referencing the internal IP of the firewall.
3. Assign the route to the integration subnet.
4. Add a new outbound rule to "integration-subnet-nsg" to allow VirtualNetwork to the service tag of AzureMonitor on port 443. Make it priority 498.
5. Set up application rules for rt.services.visualstudio.com on 443, dc.services.visualstudio.com on 443
6. Make sure that you have an NSG rule to allow outbound traffic out of the integration subnet. You can reference the AzureMonitor service tag as a destination. Use 443 as a destination port.

[top ->](#TOC)

## Triggers and Bindings (via Portal)

At this point we should have a function app up and running with monitoring. Let's walk through creating a few hello world functions with some of the more common triggers / bindings. We'll walk through creating these through the portal for now realizing that in practice functions will be developed on development workstations and likely deployed via pipelines.

[HTTP Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp)

1. Open your function app in the portal and go to Functions -> Functions
2. Click + Add to create a new function. Choose HTTP Trigger.
3. Leave Authorization level set to Function and click Create Function.
4. On the functions page click "code and test". Click "Get Function URL" and trigger the function a few times.
5. Verify that monitoring is working by clicking "Monitor". Check to be sure invocations show up. Click on Logs, open Live Metrics and make sure it functions. Please note it takes a few minutes for invocations to show in monitor.

[Azure Blob Storage Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=csharp)

1. Open your function app in the portal and go to Functions -> Functions
2. Click + Add to create a new function. Choose Azure Blob Storage Trigger.
3. Use the default AzureWebJobsStorage  Storage account connection. This is the function apps storage account. Keep the default path of samples-workitems/{name}
4. Open your function apps storage account. Click on containers. Create a new container called sample-workitems.
5. Upload any file into the new container.
6. Verify that monitoring is working by clicking "Monitor". Check to be sure invocations show up. Click on Logs, open Live Metrics and make sure it functions. Please note it takes a few minutes for invocations to show in monitor.

[Azure Cosmos DB Trigger](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-cosmosdb-v2)

1. Create an Azure Cosmos DB account. 
2. Configure access restrictions on the Cosmos DB account to allow traffic in from the integration subnet ( for which you have already configured the service endpoint ) and any IP's associated with the portal. This might be your proxy egress IP's or your ISP IP if testing from home.
3. Open Data Explorer and create a new container called "functionscontainer" with a db called "functionsdb". Set a partition key of /test.
4. Open your function app in the portal and go to Functions -> Functions
5. Click + Add to create a new function. Choose Cosmos DB Trigger.
6. Create a new Cosmos DB Connection referencing the newly created Cosmos Account.
7. Fill in the db and collection name: "functionsdb" and "functionscontainer".
8. Allow for the automatic creation of the leases collection.
9. Go back to your Cosmos account. Open data explorer, open your container. Add a new item.
10. Go back to your function. Verify that monitoring is working by clicking "Monitor". Check to be sure invocations show up. Click on Logs, open Live Metrics and make sure it functions. Please note it takes a few minutes for invocations to show in monitor.

[top ->](#TOC)