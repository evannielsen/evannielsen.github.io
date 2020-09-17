## Implement API Management in front of any HTTP endpoints.
___

As part of the architecture there were a few Azure Functions that were serving HTTP requests. These functions were using a Consumption plan and when using the Consumption plan the resources can be taken out of memory when not in use. The result of this is the next request to the function incurs the cost of warming up the process. Since the warm up can take an indeterminate amount of time then the user/client may see an error or some negative affect. The soltuion became to migrate these functions to a Azure Function Premium plan. The premium plan guarantees at least one instance of every function app to be warmed up at all times.

When the attempt was made to migrate the functions to the premium plan it was determined that it was not as simple as telling the function app that it should now be running on the new plan. A new function app was going to need to be created on the new plan and have the code deployed to it. This does not seem like a big deal on the surface, but you can not have more than one Azure Function with the same name as they are given a url that includes that name and the urls cannot be duplicated. This meant that there would be a period of down time while the cutover was made. The downtime could have been avoided if the function was behind API Management. In that scenario both function apps could have been available and a small configuration change in API Management would have immediately started sending traffic to the new function app and stopped sending traffic to the deprecated one, no down time needed.

## Be careful with Consumption Plans
___

### Spin Down
  As mentioned above when using a Consumption plan with a function app the application can be taken out of memory when idle for too long. This can cause the next client to access that function to incur the cost of warming up the function app. This could mean that the client is waiting an indeterminate amount of time while the warm-up processing is happening.

### Scaling

  - Can they scale properly?

### Cost
  - How much money are you really spending?

## Use Topics wherever possible
___

  - Makes it easier to disperse messages later

## Overuse Application Insights
___
