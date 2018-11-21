# ServiceFabricBoilerplate


### ServiceFabricExecution

#### RunAsync
When [RunAsync](https://docs.microsoft.com/en-us/dotnet/api/microsoft.servicefabric.services.runtime.statelessservice.runasync?view=azure-dotnet) is executed the cancellationtoken must be honored for graceful shutdown. Exceptions that escape the function tells the run time different things. To ease usage, the common exception handling is wrapped in a utility function under ServiceFabricExecution.RunAsync. 

#### WhileAsync
It's common that you'll do some processing or continues work in RunAsync, introducing some kind of while loop. ServiceFabricExcution.WhileAsync is a utility function for creating that kind of loop, with execption handling and logging.

		protected override Task RunAsync(CancellationToken cancellationToken)
			=> ServiceFabricExecution.RunAsync(ct => ServiceFabricExecution.WhileAsync(token => DoWorkAsync(token), "DoWork", ct, logger), logger);


### CommunicationListeners

#### EventhubCommunicationListener

The communicationlistner for event hubs is based on the convetion that the number partitions of the service matches the number of partitions of the hub.

It uses the statemanager to track checkpoints and a telemetry client for dependecy tracking.

Due to the [service lifecycle](https://www.kickstarter.com/projects/1890003488/jagged-alliance-the-board-game/posts/2317992?ref=bggforums the listener is created, and a private field assigned.

        protected override IEnumerable<ServiceReplicaListener> CreateServiceReplicaListeners()
            => new[]
            {
                new ServiceReplicaListener(ctx =>
                {
                    _eventhubsCommunicationListener =
                        new EventhubsCommunicationListener(ctx, StateManager, Partition, _logger, _telemetryClient, _receiverConfiguration);
                    return _eventhubsCommunicationListener;
                })
            };

Later in RunAsync we are going to use the same instance to pass it a worker/handlers.

        protected override Task RunAsync(CancellationToken cancellationToken)
            => ServiceFabricExecution.RunAsync(ct =>
                    _eventhubsCommunicationListener.OnEventsAsync(
                        ct, _handleEvents, MaxMessageCount, WaitTime),
                cancellationToken,
                _logger);

#### OnEventsAsync

The OnEventsAsync method handles retriveing events and execution the handler. As a pump. This internaly uses the WhileAsync utility.

#### Partitions

If the event hub has 4 partitions, we need to match that with our service.

    <Parameter Name="yourservice"_PartitionCount" DefaultValue="4" />

Its important that the partition is int64.

        <UniformInt64Partition PartitionCount="[yourservice_PartitionCount]" LowKey="0" HighKey="yourservice_HighKey]" />

#### IHostedService

In some scenarios the communication listeners acts like an HostedService. In a future where Service Fabric might use GenricHost, HostedService might be a better solution for reading logs/queues.

[GenericHost in Service Fabric](https://github.com/Microsoft/service-fabric-aspnetcore/issues/48)

[CoherentSolutions.Extensions.Hosting.ServiceFabric](https://github.com/coherentsolutionsinc/aspnetcore-service-fabric-hosting)