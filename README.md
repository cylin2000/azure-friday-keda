# Azure Friday - Azure Serverless on Kubernetes with KEDA

Demo used in Azure Friday on Azure Serverless on Kubernetes with KEDA episode.

![Azure Friday](./media/azure-friday-logo.png)

## Scenario

We have an application that processes orders from a queue and schedules a shipment for every order.

![Scenario](./media/scenario.png)

To coop with the load, KEDA is used to automatically scale out/in based on the current workload and to optimize for cost.

## How to run the demo

**Prerequisites**
- Create an Azure Service Bus namespace with `orders` & `shipments` queues
- Install KEDA in your Kubernetes cluster ([info](https://keda.sh/docs/2.0/deploy/))

**Deploying the Azure Friday application**
- Provide base64 encoded secrets in `deploy\deploy-app.yml` with your Service Bus connection strings for our app
- Deploy the application with `kubectl apply -f deploy\deploy-app.yml`

**Autoscaling our .NET Core Orders worker**
- Provide base64 encoded secrets in `deploy\deploy-autoscaling-orders.yml` with your Service Bus connection strings for our autoscaling
- Deploy the application with `kubectl apply -f deploy\deploy-autoscaling-orders.yml`

**Autoscaling our Shipments Azure Function**
- Deploy the application with `kubectl apply -f deploy\deploy-autoscaling-shipments.yml`

## License

This demo is shared under MIT license but is mainly a fork of [kedacore/sample-dotnet-worker-servicebus-queue](https://github.com/kedacore/sample-dotnet-worker-servicebus-queue) which provides a walkthrough to scale .NET Core workloads with KEDA and includes various Service Bus authentication examples.


## 代码调用分析

App 使用了 .net 的 host 作为外壳启动 Service https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host?tabs=appbuilder , https://github.com/cylin2000/DotNetHostTest

Src --> AzureFriday.Orders --> Program.cs Line 38 (启动后台服务: 这个服务是 RunAsync 启动后一直等待，除非收到控制台 Ctrl+C 或者 外层 Docker stop/kill 等) 
                           --> Extensions --> IserviceCollectionExtensions.cs Line 15 使用扩展启动 OrdersQueueProcessor 
                           --> OrdersQueueProcessor --> 继承自父类 QueueWorker Line 27 启动 Task.ExecuteAsync
                           --> QueueWorker Line 33  queueClient.RegisterMessageHandler(HandleMessage, HandleReceivedException);  queueClient 注册侦听 Service Bus 的消息，有消息就使用 HandleMessage 处理
                           --> 因为是异步的，所以后面的代码一直等待 queueClient 执行(Line 36: while (!stoppingToken.IsCancellationRequested))，直到外层传入 Cancellation
