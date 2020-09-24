## Implement API Management in front of any HTTP endpoints.
___

As part of the architecture there were a few Azure Functions that were serving HTTP requests. These functions were using a Consumption plan and when using the Consumption plan the resources can be taken out of memory when not in use. The result of this is the next request to the function incurs the cost of warming up the process. Since the warm up can take an indeterminate amount of time then the user/client may see an error or some negative affect. The soltuion became to migrate these functions to a Azure Function Premium plan. The premium plan guarantees at least one instance of every function app to be warmed up at all times.

When the attempt was made to migrate the functions to the premium plan it was determined that it was not as simple as telling the function app that it should now be running on the new plan. A new function app was going to need to be created on the new plan and have the code deployed to it. This does not seem like a big deal on the surface, but you can not have more than one Azure Function with the same name as they are given a url that includes that name and the urls cannot be duplicated. This meant that there would be a period of down time while the cutover was made. The downtime could have been avoided if the function was behind API Management. In that scenario both function apps could have been available and a small configuration change in API Management would have immediately started sending traffic to the new function app and stopped sending traffic to the deprecated one, no down time needed.

## Be careful with Consumption Plans
___

### Spin Down
  As mentioned above when using a Consumption plan with a function app the application can be taken out of memory when idle for too long. This can cause the next client to access that function to incur the cost of warming up the function app. This could mean that the client is waiting an indeterminate amount of time while the warm-up processing is happening.

### Scaling

  Using the Consumption Plan has a nice built in feature in that it will autoscale based on the number of events that are being sent against it. This can be convenient as sometimes the scaling features can be difficult to tweek and get performing as needed.

  Since the plan is handling this scaling for you it does not expose any of the logic for editing the scaling algorithm. This is an all or nothing setup. If more control is needed around when and how much to scale then a different plan will need to be chosen. Also, if it is necessary to always have at least one instance of the function app warmed up or the execution time of the functions will be more than 5 minutes, then an Azure Function Premium Plan is recommended.

### Cost
  At first glance it may seem like the Consumption Plan is the right choice since you get 1 million executions for free each month. Although this may be the case in many scenarios, calculating the expected cost of the system should be done to ensure that the cost is minimized.

  Azure Functions on the Consumption Plan are billed on 2 metrics, Total Executions and Execution Time(GB-s). Both of these values are calculated on the Azure Function executions of the entire subscription not just the function itself. Likewise all free grants of executions and execution time are for the entire subscription.

  Planning ahead will save time and hassle later if a migration is needed. Here is a [link](https://azure.microsoft.com/en-us/pricing/details/functions/) to the pricing page which explains the calculation.

## Use Topics wherever possible
___

  Inevitably when creating a system with a microservice architecture there will be some services that are triggered based on work that eventually needs to be done. This is usually accomplished by adding events to a queue so that it can be processed at the systems leisure.
  
  If at any point the system requires that more than one action take place based on that event then the result is to chain that action into the process of the original action. This is not desirable due to the desire to create a separation of concerns.

  Topics give you the ability to have a single publisher of events and multiple subsribers to the event feed. New subscriber feeds can be created at any point without any change to the original publishing location.

## Application Insights
___

Application Insights can be invaluable when attempting to diagnose a problem with a piece of the system. The barrier to entry for gettning Application Insights recording the logs and events is very low.

Coupling this with the addition of alerting to signal someone if there is an anomaly in the process and this becomes an invaluable tool. Gone are the days of weeding through huge log files to help flush out a bug.
