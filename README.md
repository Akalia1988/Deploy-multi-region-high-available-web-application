# Deploy-multi-region-high-available-web-application
Deploying multi-region high available web application in Azure with App Services, SQL Database, and Azure Front Door 
Blue/Green Deployment with Azure Front Door.

you can get 99.95% high availability for app service, if your app service plan running under standard or premium tier

Configure Azure SQL database
	
1. Setting up geo-replication for your Azure SQL database. This will replicate your databases to databases located in different regions (one or more). Note that the replication process happens asynchronously in the background. Therefore this will not affect to the performance of your application. However, it does take a few seconds to update your database changes to the secondary databases after you commit your transaction to the primary database.
	
  2. Deploy your application into one or more secondary regions and load-balance your HTTP traffics to those regions based on availability. If your application up and running in the primary region, traffic goes to the primary region. If your primary region not available, traffic goes to the secondary regions. You can use the priority routing method for this. There are few azure load balancers which provide priority load balancing options. Azure Traffic Manager, Azure Application Gateway, and Azure Front Door. In addition to the priority routing, these load balancers provide different routing methods such as path-based routing, weighted routing, Performance-based routing…etc. You can use these methods to improve the performance of your application.
	
  3. Use geo-replication for your Azure storage. You can set up the Read Access Geo Redundant Storage (RA-GRS) option for your storage account. This will keep 3 extra copies of your data in another region. It will look like below. 
  
  ![image](https://user-images.githubusercontent.com/58148717/103944345-1be4f280-50f9-11eb-9701-6690fe1d174e.png)
  
For Azure SQL failover groups we always can chose the paired regions. 
P.S Azure SQL hyperscale doesn’t support Geo replication so workaround would be that either setup the transactional replication or simply restore the existing database in different region.

Azure Front Door
Azure Front Door is a global, scalable entry-point that uses the Microsoft global edge network to create fast, secure, and widely scalable web applications. With Front Door, you can transform your global consumer and enterprise applications into robust, high-performing personalized modern applications with contents that reach a global audience through Azure.
	
we have to deploy a copy of our application into the secondary region and load balance incoming HTTP traffic among those two regions based on the availability. We can choose any load balancer which provides priority routing option. We can use Azure Front Door as a layer 7 load balancer which provides many routing options including priority routing. In addition to priority routing, Azure Front Door provides a rich set of features including, Web Application Firewall, SSL offloading, caching, custom domains….etc. You can read more about AFD and its feature from the official documentation. Azure Front Door comes with Azure Application Gateway Web Application Firewall. Therefore, you can protect your application from common web application vulnerabilities defined in OWASP
If the primary region becomes unavailable, traffic moves to the application with the second-highest priority. In general, this type of high availability setup is called Active-Passive hot standby.

![image](https://user-images.githubusercontent.com/58148717/103944537-6b2b2300-50f9-11eb-945e-53c52f903c90.png)

Implementation steps: 
Create a Resource Group for the primary and secondary region resources.
Create a primary database or use existing Azure SQL database
Create a failover group so database can be replicated in another Geo location

![image](https://user-images.githubusercontent.com/58148717/103944627-91e95980-50f9-11eb-9a91-85741762bf4f.png)

![image](https://user-images.githubusercontent.com/58148717/103944653-9d3c8500-50f9-11eb-9263-4aaf1fa4b2d8.png)

![image](https://user-images.githubusercontent.com/58148717/103944700-b04f5500-50f9-11eb-933e-15cdfb124141.png)

You can see above there are two endpoints:
Read/Write listener endpoint- This endpoint always connects your application to the primary database.
Read only listener endpoint-This will connect your application to the secondary database which is read-only. Using this option you can forward your read-only workload to the secondary database and can improve the performance of your application. However, it is recommended to use ApplicationIntent=ReadOnly; option in your read-only connection string.

Deploying web apps to Azure App Service
Before you publish the solution, change the AppRegion value of the appsettings.json, located in the root folder of the solution. This will help you to identify the web application that responds to your request.

![image](https://user-images.githubusercontent.com/58148717/103944897-f1e00000-50f9-11eb-87b5-89c6dc4bf5ea.png)

![image](https://user-images.githubusercontent.com/58148717/103944926-fc9a9500-50f9-11eb-97d7-7cbdc8cdf014.png)

Click on the publish button. This will deploy the application to your Azure app service located in the primary region.
Do the same thing for the secondary region app service as well. Before you publish the application into the secondary region, make sure that you have changed the AppRegion property of the appsettings.json to “Secondary”.


Blue/Green Deployment with Azure Front Door

The first step is to configure the frontend

![image](https://user-images.githubusercontent.com/58148717/104476970-6ac1da80-5586-11eb-8d47-b6ca5282b7ce.png)

The main item here as it relates to blue/green is “Session Affinity.” This determines whether the end user always gets routed to the same backend after first accessing the Front Door.Whether or not you enable this depends on your application, and the type of enhancements being rolled out. If it’s a major revision you will likely want to enable Session Affinity, so that if the user is initially routed to the new codebase she will continue to use it. If the enhancement is relatively minor, for example involving a single page with no dependencies on other parts of the application, you could potentially leave this disabled. If in doubt, enable Session Affinity.

Backend Pools

This is where you configure the host(s) that will end up satisfying your users’ web requests.
In this example, I just chose two relatively-random public endpoints. You would configure your existing backend website and the new one under test. I divided requests between my two on a 75% to 25% split. You would likely send more traffic to the current website than the new one, and then likely direct more to the new one as it proves itself to be bug free.

Here is a screenshot of the definition of one of the two backends. Note the three parameters circled which, with one other, will be discussed in relation to how a configured backend gets chosen by Azure Front Door for a given request.

![image](https://user-images.githubusercontent.com/58148717/104477455-beccbf00-5586-11eb-822a-f02d041b0cb3.png)

The following screenshot shows the two backends I configured as part of my backend pool. 
Note that the total of the “Weights” does not have to add up to 100 — I did that for clarity.

![image](https://user-images.githubusercontent.com/58148717/104477720-12d7a380-5587-11eb-9ef4-003379edc9f3.png)

For the purposes of a simple Blue/Green configuration, here is a summary on how you might configure these parameters.

Keep the backend under test disabled to start. Once you get all other parameters configured, you can enable it to start your Blue/Green testing.

The Priority of both backends should be the same. I recommend just leaving them at “1.”

As part of the Backend Pool configuration there is a section called “Load Balancing.” For the purposes of our discussion, the most important parameter here is “Latency sensitivity.” This value determines the highest number of milliseconds that a health probe can take for a given backend to be even considered for selection; setting it to zero disables the parameter entirely. I recommend you set it to a reasonably high value such as 500 (a half a second), meaning any backends responding in less than that time frame will be considered. The goal is to make sure both backends get used, as it’s possible they are in different data centers and present different latencies to the end user.

![image](https://user-images.githubusercontent.com/58148717/104478031-6b0ea580-5587-11eb-9e6e-196835be6385.png)

Any backends making it this far are finally selected based on weight.

Routing Rules

For the purposes of Blue/Green, the routing rules are relatively unimportant. What you select here depends more on how your web application works.

For more info: https://docs.microsoft.com/en-us/azure/frontdoor/front-door-route-matching













 











  
  
  
