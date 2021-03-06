# ServiceFabricBoilerplate

## Setup

When setting up a new Service Fabric Application project using the VS template, we use the naming convetion "ServiceFabric". Analogously, all other projects are named without the solution. Then we do a couple of changes to the generated files.

In the **.csproj* files we set the following;

    <AssemblyName>[Solution].[Project]</AssemblyName>
    <RootNamespace>[Solution].[Project]</RootNamespace>

In *ServiceManifest.xml* we change the program name

    <Program>[Solution].[Project].exe</Program>

In the *ApplicationManifest.xml* we change the the *ApplicationTypeName* to 

    ApplicationTypeName="[Solution].Type"

In the different *ApplicationParameters* files we change the name of the application

    Name="fabric:/[Solution]"

If we want to deploy different environment specific versions of the applications in a common cluster, we need to give the application different names for the different environments.

    Name="fabric:/[Solution]-[Enviroinment]"


## Web projects
In a web project we change the *applicationUrl* in *launchSettings.json* to the same host and port for both IIS and Web Project.
Next, we set the same port in the *ServiceManifest.xml*.

### Running locally outside the cluster
The web project is configured to be able to run stand alone outside a Service Fabric cluster. When we do that *launchSettings.json* is used instead of the *ServiceManifest.xml*. In *Program.cs* we configure the program differently based on if we are in a cluster or not.


## Configuration

The solution uses the following configuration providers 

* Environment
* Service Fabric
* Yaml
* Key Vault

The environment configuration can be set in either the *launchSettings.json* or the *ServiceManifest.xml*. We use the environment for the *ASPNETCORE_ENVIRONMENT* parameter.

The service fabric configuration is used to configure the keyvault provider and the environment.

In *Settings.xml*

    <Section Name="KeyVault">
      <Parameter Name="Name" Value="" />
      <Parameter Name="ClientId" Value="" />
      <Parameter Name="ClientSecret" Value="" />
    </Section>

In *ServiceManifest.xml*

    <EnvironmentVariables>
      <EnvironmentVariable Name="ASPNETCORE_ENVIRONMENT" Value=""/>
    </EnvironmentVariables>


Yaml is used for all configuration that isn't sensitive. We also use *appsettings.Development.yml* to configure the keyvault provider during development when we're running outside a Service Fabric cluster.

    keyvault:
      name:
      clientid:
      clientsecret:

The keyvault is used for all sensitive configuration e.g. connection strings. The keyvault doesn't support colon and instead one need to use double dashes.

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

Due to the [service lifecycle](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-lifecycle) the listener is created, and a private field assigned.

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