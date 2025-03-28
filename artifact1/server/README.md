# Helidon OCI Archetype Server using MP

This module demonstrates a simple Hello World JAX-RS based application and is composed of the following:

1. Server-side source code that will be generated using OpenApi tool based on an OpenApi specification document in the form
   of [api.yaml](../spec/api.yaml) as an input. The generated source code will be located in
   target.generated-sources.server.src.main.java.group1.server and will have the following components:
   * A JAX-RS base interface
   * A model source code that will represent a JSON object
2. A server application that will implement the generated JAX-RS base interface to add a simple hello world
greet logic. Furthermore, it will also include OCI SDK integration with OCI Logging and Metrics.
3. It also showcases how to use generated [client](../client) to test this server application.
   
## OCI SDK integration
Helidon provides easy integration with the OCI SDK by injecting SDK clients. This server application provides a sample
demonstration for both the OCI Monitoring and Logging SDK in src.main.java.group1.server.GreetResource.java with simple annotations like below:
```java
    @Inject
    public GreetResource(Monitoring monitoringClient, Logging loggingClient, ...)
```
Authentication can be configured and is explained in [Application Configuration](#application-configuration)
section with the use of `oci.auth-strategy` property in [microprofile-config.properties](src/main/resources/META-INF/microprofile-config.properties).
If not specified, it will default to `auto` which will cycle through all the authentication types until one succeeds.

To enable this feature, the `helidon-integrations-oci-sdk-cdi` artifact needs to be 
added as a dependency in the application's [pom.xml](pom.xml):
```xml
        <!-- Enabled OCI SDK integration  -->
        <dependency>
            <groupId>io.helidon.integrations.oci.sdk</groupId>
            <artifactId>helidon-integrations-oci-sdk-cdi</artifactId>
            <scope>runtime</scope>
        </dependency>
```
Along with this dependency, the OCI SDK common library and the OCI service library that's going to be used also
needs to be added. For example, to integrate with OCI logging, below is the example of the dependency declaration:
```xml
        <dependency>
            <groupId>com.oracle.oci.sdk</groupId>
            <artifactId>oci-java-sdk-common</artifactId>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>com.oracle.oci.sdk</groupId>
            <artifactId>oci-java-sdk-loggingingestion</artifactId>
            <scope>compile</scope>
        </dependency>
```

## Custom Logs Monitoring
This Server application can also be configured to demonstrate OCI Custom Logs Monitoring using Unified Monitoring Agent 
(not part of Helidon). It was added in this project to demonstrate logging processor (collect and send to logging service) capabilities.
 
To set this up, please see the details in [Creating Custom Logs](https://docs.oracle.com/en-us/iaas/Content/Logging/Concepts/custom_logs.htm#creating_custom_logs).
See [OCI IAM Setup for Custom Logs](#oci-iam-setup-for-custom-logs) section below as well. After this exercise, it is expected that 
a Log Group and Log OCI resources had been created. Please take note of the custom log that got created as its used in this configuration
setup as it will be used in the next section. Furthermore, Helidon logging also needs to be set up so that the log data can be persisted
into a file that was designated to be the log file that the Unified Monitoring Agent will process. Below is an example of the configuration
that needs to be set up in [logging.properties](src/main/resources/logging.properties).

```properties
handlers=io.helidon.common.HelidonConsoleHandler, java.util.logging.FileHandler
java.util.logging.FileHandler.pattern=%h/helidon_log/helidon.log
java.util.logging.FileHandler.formatter=java.util.logging.SimpleFormatter
```

The use of `java.util.logging.FileHandler` as a handler and defining the `java.util.logging.FileHandler.pattern` will
make the application save all the log data into ${HOME}/helidon_log/helidon.log which will then be monitored and
processed by the Unified Monitoring Agent.

### OCI IAM Setup for Custom Logs
1. Using User Principal authentication
   1. Create oci config by generating
      [API Signing Key](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm) for the IAM user who
      will be responsible for publishing logs and metrics.
   2. Create a Group (ex.: `LogMetricPublisher`) and add the User to it. See
      [Managing Groups](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managinggroups.htm#) for more
      information.
   3. Create a Policy using the group from the previous step to allow metrics to be published. For logging, use the
      compartment that was designated as the logging compartment. More details can be found here:
      https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/monitoringpolicyreference.htm#Details_for_Monitoring
      * Metrics Policy:
      ```
      Allow group LogMetricPublisher to use metrics in compartment LogMetricCompartment
      ```
      * Logging Policy:
      ```
      Allow group LogMetricPublisher to use log-content in compartment LogMetricCompartment
      ```
2. Using Instance Principal authentication on an OCI compute instance. More details can be found in
   [Calling Services from an Instance](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm).
   1. Create a dynamic group for the compartment where the Compute instance will be created.
      ```
      Any {instance.compartment.id = 'ocid1.compartment.oc1..mycompartmentocid'}
      ```
   2. Create policies to allow use (read/write) of metrics and logging.
      * Metrics Policy:
      ```
      Allow dynamic-group helidondev to use metrics in compartment LogMetricCompartment
      ```
      * Logging Policy:
      ```
      Allow dynamic-group dynamicgroup1 to use log-content in compartment LogMetricCompartment
      ```

## Helidon MP Metrics to OCI
Helidon generated and recorded metrics can be transmitted to the
[OCI Monitoring service](https://docs.oracle.com/en-us/iaas/Content/Monitoring/Concepts/monitoringoverview.htm) by 
adding the following configuration in 
[src/main/resources/META-INF/microprofile-config.properties](src/main/resources/META-INF/microprofile-config.properties):

```properties
# OCI compartment ID where the metrics will be sent
ocimetrics.compartmentId=<your monitoring compartment id>
# OCI metric namespace
ocimetrics.namespace=<your monitoring namespace e.g. helidon_oci>
# Delay in seconds before the 1st metrics transmission to OCI takes place.  Defaults to 1 second if not specified.
ocimetrics.initialDelay=30
# Interval in seconds between metrics transmission to OCI. Defaults to 60 seconds if not specified.
ocimetrics.delay=180
# Filter only the scopes that will be sent to OCI. This is optional and will default to all scopes if not specified.
ocimetrics.scopes.0=base
ocimetrics.scopes.1=vendor
ocimetrics.scopes.2=application
# Enable or disable metric transmission to OCI. Defaults to true if not specified.
ocimetrics.enabled=true
```

Alternatively, this can also be configured in yaml format and added to [src/main/resources/application.yaml](src/main/resources/application.yaml):

```yaml
ocimetrics:
  # OCI compartment ID where the metrics will be sent
  compartmentId: <your monitoring compartment id>
  # OCI metric namespace
  namespace: <your monitoring namespace e.g. helidon_oci>
  # Delay in seconds before the 1st metrics transmission to OCI takes place.  Defaults to 1 second if not specified.
  initialDelay: 30
  # Interval in seconds between metrics transmission to OCI. Defaults to 60 seconds if not specified.
  delay: 180
  # Filter only the scopes that will be sent to OCI. This is optional and will default to all scopes if not specified.
  scopes: [base, vendor, application]
  # Enable or disable metric transmission to OCI. Defaults to true if not specified.
  enabled: true
```

Along with the above configuration setting, the `helidon-oci-integrations-metrics` artifact needs to be added as a 
dependency in the application's [pom.xml](pom.xml):

```xml
        <!-- Allows Helidon Metrics to be pushed to OCI Monitoring Service  -->
        <dependency>
            <groupId>io.helidon.oci.integrations</groupId>
            <artifactId>helidon-oci-integrations-metrics</artifactId>
        </dependency>
```

For more details about the Helidon metrics, please check out the [Helidon MP Metrics Guide](https://helidon.io/docs/v2/index.html#/mp/guides/05_metrics) 
particularly the following relevant sections:
1. `Controlling Metrics Behavior` - has various options to manage the behaviour of the helidon metrics
2. `Application-Specific Metrics Data` - allows application-specific metrics to be created using CDI.

As part of the example, some of the methods in src.main.java.group1.server.GreetResource.java
where annotated with Application-Specific metrics such as @Counted, @Metered and @Timed as shown below:
```java
    @Counted(name = "getDefaultMessage", absolute = true)
    @Override
    public GreetResponse getDefaultMessage() {
        LOGGER.info("getDefaultMessage() is invoked");
        publishMetricAndLog("getDefaultMessage");
        return createGreetResponse("World");
    }
    
    @Metered(absolute = true, unit = MetricUnits.MILLISECONDS)
    @Override
    public GreetResponse getMessage(String name) {
        LOGGER.info("getMessage(" +  name + ") is invoked");
        publishMetricAndLog("getMessage");
        return createGreetResponse(name);
    }

    @Timed(name = "updateGreeting", absolute = true, unit = MetricUnits.MILLISECONDS)
    @Override
    public void updateGreeting(@Valid @NotNull GreetUpdate body) {
        LOGGER.info("updateGreeting(greeting: " +  body.getGreeting() + ") is invoked");
        greetingProvider.setMessage(body.getGreeting());
        publishMetricAndLog("updateGreeting");
    }
```

## Health Checks
Helidon supports built-in and custom liveness and readiness health checks. Refer to 
[MicroProfile Health](https://helidon.io/docs/v2/#/mp/health/01_introduction) for an overview and how to set up this
feature. For more details on how to use it including creating custom health checks, check out 
[Helidon MP Health Check Guide](https://helidon.io/docs/v2/#/mp/guides/04_health). 
src.main.java.group1.server.GreetLivenessCheck.java and src.main.java.group1.server.GreetReadinessCheck.java had been added
in the application to demonstrate custom liveness and readiness check, respectively. Liveness can be retrieved via
`/health/live` and readiness via `/health/ready`. Check out [Run and exercise the application](#run-and-exercise-the-application)
for examples of retrieving health checks statuses.

## Application Configuration:
The application configuration can be customized in 
[src/main/resources/META-INF/microprofile-config.properties](src/main/resources/META-INF/microprofile-config.properties).
and currently has the following parameters:
1. `app.greeting` - The greeting word and currently set to `Hello`
2. `server.host` - Host IP address of the server application and currently set to accept all at `0.0.0.0`
3. `server.port` - Host port of the server application and currently set to `8080`
4. `metrics.rest-request.enabled` - Enables rest request metrics and currently set `true`. Please see [Controlling REST.request Metrics](https://helidon.io/docs/v2/#/mp/guides/05_metrics#controlling-rest-request-metrics) for more information.
5. `oci.monitoring.compartmentId` - Compartment Id used for the sending custom metrics. Please see [Publishing Custom Metrics](https://docs.oracle.com/en-us/iaas/Content/Monitoring/Tasks/publishingcustommetrics.htm#Publishing_Custom_Metrics) for more details.
6. `oci.monitoring.namespace` - Monitoring namespace configured for sending metrics. Please see [Publishing Custom Metrics](https://docs.oracle.com/en-us/iaas/Content/Monitoring/Tasks/publishingcustommetrics.htm#Publishing_Custom_Metrics) for more information.
7. `oci.logging.id` - Custom log id configured. Please see [Custom Logs Monitoring](#Custom-Logs-Monitoring) for more information.
8. `oci.auth-strategy` - Allows OCI client authentication mechanism to be specified and can either be an individual
value or a list of authentication types separated by comma. If specified as a list, it will cycle through each until a 
successful authentication is reached. This is currently set to `config_file,instance_principals,resource_principal` 
which means that `config_file` will be tried first, then falls back to `instance_principals` if it fails, and 
eventually falls back to `resource_principals` if `instance_principals` fails. The following are the different types
of OCI client authentication:
   * `config_file` - Uses user authentication set in ~/.oci/config
   * `config` - Sets user authentication in the helidon config, i.e. microprofile-config.properties
   * `instance_principals` - Uses the compute instance as the authentication and authorization principal. Please see 
[Calling Services from an Instance](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/callingservicesfrominstances.htm)
for more details.
   * `resource_principal` - Very similar to instance principal auth but uses OCI resources and services as the 
authentication and authorization principal such as serverless functions. Please see 
[About Using Resource Principal to Access Oracle Cloud Infrastructure Resources](https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/resource-principal-about.html#GUID-7EED5198-2C92-41B6-99DD-29F4187CABF5) for more information.

Values of parameters in microprofile-config.properties (e.g., dev, test, prod) can also be overridden in a variety of 
ways:
1. Using `config profiles` is a good way of running or testing multiple scenarios that require different property values. 
Please see [Helidon Config Profiles Example](https://github.com/oracle/helidon/tree/master/examples/config/profiles)
for more details. This allows users to create a new profile in another file with the format
`microprofile-config<-ProfileName>.properties` and can then be switched to become the active profile by adding 
`-dconfig.profile=ProfileName` when running the application. For example, in the section 
[Run and exercise the application](#run-and-exercise-the-application), there are 2 options of running the application 
by either using `-dconfig.profile=test` or `-dconfig.profile=prod` profile. Running with those profile options
will correspond to using specified properties in 
[microprofile-config-test.properties](src/main/resources/META-INF/microprofile-config-test.properties) or 
[microprofile-config-prod.properties](src/main/resources/META-INF/microprofile-config-prod.properties), respectively. 
Moreover, properties that are not defined in those profiles will default to the ones that already exist in 
[microprofile-config.properties](src/main/resources/META-INF/microprofile-config.properties).
2. Using [Environment Variables Mapping Rules](https://download.eclipse.org/microprofile/microprofile-config-2.0/microprofile-config-spec-2.0.html#default_configsources.env.mapping)
is another option that allows values of existing properties in the config file to be replaced. For example if you want 
to replace the configured value of `server.port` (currently set to `8080`), all you need to do is set the 
`SERVER_PORT` environment variable to the desired value:
   ```
   SERVER_PORT=8888 java -jar target/artifact1-server.jar
   ``` 

## System Requirements:
1. JDK 11+
2. mvn 3.8.3+
3. Helidon 2.5.4+

## Build and test
As mentioned in introduction, we are showcasing how to use generated client for testing. This requires to have [client module build first](../client/README.md#build) else you will get compilation errors in test.

1. Build without running a test
   ```bash
   mvn clean package -DskipTests
   ``` 
2. Build with run default test profile using mocked unit tests
   ```bash
   mvn clean package
   ```
3. Build and run `test` test profile that performs authentication for OCI services via .oci config file
   ```bash
   mvn clean package -Ptest
   ```
4. Build and execute `prod` test profile that performs OCI services authentication via instance
principal when running it in an oci compute instance.  
   ```bash
   mvn clean package -Pprod
   ```

## Run and exercise the application
1. Run the server application depending on the config profile. Below are the choices:
   * Default with no profile will use `config_file,instance_principals,resource_principal` authentication strategy
     ```bash
     java -jar target/artifact1-server.jar
     ```
   * `test` profile will use user principal via oci config file authentication
     ```bash
     java -Dconfig.profile=test -jar target/artifact1-server.jar
     ```
   * `prod` profile will use instance principal authentication
     ```bash
     java -Dconfig.profile=prod -jar target/artifact1-server.jar
     ```
2. Use curl to access the client application
   ```
   curl -X GET http://localhost:8080/greet
   {"message":"Hello World!","date":[2022,4,1]}

   curl -X GET http://localhost:8080/greet/Joe
   {"message":"Hello Joe!","date":[2022,4,1]}
   
   curl -X PUT -H "Content-Type: application/json" -d '{"greeting" : "Ola"}' http://localhost:8080/greet/greeting
   curl -X GET http://localhost:8080/greet
   {"message":"Ola World!","date":[2022,4,8]}
   ```
3. Use curl to access the health checks:
   ```
   $ curl -X GET  http://localhost:8080/health/live
   {"outcome":"UP","status":"UP","checks":[{"name":"CustomLivenessCheck","state":"UP","status":"UP","data":{"time":1646361485815}}]}
   $ curl -X GET  http://localhost:8080/health/ready
   {"outcome":"UP","status":"UP","checks":[{"name":"CustomReadinessCheck","state":"UP","status":"UP","data":{"time":1646361474774}}]}
   ```

## Build and run using docker
1. Build docker image
   ```
   docker build .. -t artifact1-server -f Dockerfile
   ```
2. Run the application
   * Run from local machine using User Principal authentication. This will mount `~/.oci` as a volume in the running 
     docker container.
     ```
     docker run --user helidon --rm -p 8080:8080 -v ~/.oci:/home/helidon/.oci:ro artifact1-server:latest
     ```
   * Run from an OCI instance using Instance Principal
     ```
     docker run --user helidon --rm -p 8080:8080  artifact1-server:latest
     ```
     
## Deploy on Kubernetes Cluster

If you don’t have access to a Kubernetes cluster, you can [install one](https://helidon.io/docs/latest/#/about/kubernetes) on your desktop. Then deploy the example:

Verify connectivity to cluster
```
kubectl cluster-info
kubectl get nodes
```

Make sure, your application has access to your OCI setup. One way, you can do so, if your kubernetes cluster is running locally, is by volume.

Create a volume pointing to your OCI configuration file
```yaml
      volumes:
        - name: oci-config
          hostPath:
            # directory location on host
            path: <Directory with oci config file>
```

Mount this volume as part of your application containers specification
```yaml
        volumeMounts:
        - mountPath: /home/helidon/.oci
          name: oci-config
```

Deploy the application to Kubernetes
```
kubectl apply -f app.yaml
kubectl get pods                    # Wait for artifact1-deploy pod to be RUNNING
```

The step above created a service that is exposed as node port. Lookup the service to find the port.
```
kubectl get service artifact1-svc
```

Note the PORTs. You can now exercise the application as you did before but use the second port number (the NodePort) instead of 8080. For example:
```
curl -X GET http://localhost:31431/greet
```

After you’re done, remove the application from Kubernetes
```
kubectl delete -f app.yaml
```

## Distribution
As part of the [build](#build-and-test), a zip file in
[target/artifact1-server-distribution.zip](target/artifact1-server-distribution.zip)
can be created that can be used to distribute/deploy to other machines such as an OCI VM instance. This zip file can be
securely copied over by tools like `scp (secure copy)` to another machine's directory, unzipped and
[run](#run-and-exercise-the-application).
1. Generate zip assembly
```bash
mvn clean package -Pzip
```
2. Upload to a remote machine using scp
```bash
$ scp target/artifact1-server-distribution.zip user@DESTINATION_HOST:/home/user
```
3. Unzip (will extract to `artifact1-server` directory) and run
```bash
$ unzip artifact1-server-distribution.zip
$ java -jar artifact1-server/artifact1-server.jar
```