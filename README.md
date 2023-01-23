# QUARKUS - TEMPORAL EXTENSION

With this extension you can easily implement a temporal workflow in your quarkus project.
Review this [demo](https://github.com/mostafacs/quarkus-temporal-demo) for a quick start.
## How to use ?

1- Add extension dependency to your maven POM file.

* Quarkus 2.4.2 , JDK 11 and Native Image Build Support (graalvm-ce-java11-21.2.0)

 ```xml
<dependency>
    <groupId>com.sellware.quarkus-temporal</groupId>
    <artifactId>temporal-client</artifactId>
    <version>2.0.2</version>
</dependency>
```

* Quarkus 1.x , JDK 8 ( Native image build not supported)

 ```xml
<dependency>
    <groupId>com.sellware.quarkus-temporal</groupId>
    <artifactId>temporal-client</artifactId>
    <version>1.13.1.7</version>
</dependency>
```

2- Updated netty-shaded on quarkus-bom

```xml
 <dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-bom</artifactId>
            <version>${quarkus.platform.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Only necessary while quarkus does not bump the netty-all lib. -->
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty-shaded</artifactId>
            <version>1.39.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

3- Add temporal server url config in `application.properties`

```properties
quarkus.temporal.service.url=localhost:7233
quarkus.temporal.service.secure=true
```

4- Add configuration file named `workflow.yml` to resources folder

* Field `name` in annotation `@TemporalWorkflow` used to load workflow configurations
* Field `name` in annotation `@TemporalActivity` used to load activities configurations

#### Example

```yml
defaults:
  workflowExecutionTimeout: 20 # in minutes default is 60 minute if not set
  workflowRunTimeout: 15 # in minutes default is 60 minute
  workflowTaskTimeout: 14 # in minutes default is 60 minute

  activityScheduleToStartTimeout: 60 # in minutes default is 60 minute
  activityScheduleToCloseTimeout: 60 # in minutes default is 60 minute
  activityStartToCloseTimeout: 60 # in minutes default is 60 minute
  #heartbeat timeout must be shorter than START_TO_CLOSE timeout
  activityHeartBeatTimeout: 5 # in minutes default is 60 minute
  activityRetryInitInterval: 1 # in minutes default is 5 minute
  activityRetryMaxInterval: 1 # in minutes default is 1 minute if not set
  activityRetryBackOffCoefficient: 1.0  # default is 1.0
  activityRetryMaxAttempts: 5  # default is 1 attempt

# override defaults per workflow
workflows:
  test:
    executionTimeout: 2
    runTimeout: 2
    taskTimeout: 2
    activities:
      test:
        scheduleTostartTimeout: 10
        scheduleTocloseTimeout: 10
        #startTocloseTimeout: 20
        #heartbeatTimeout: 2
        #retryInitInterval: 1
        #retryMaxInterval: 1
        #retryMaxAttempts: 1
```

### Declare your Temporal Activities:

```java
    @ActivityInterface
    public interface TestActivity {
    
        String hello();
    }
```

```java
    // name used to get the activity configurations from workflow.yml
    @TemporalActivity(name="test")
    public class TestActivityImpl implements TestActivity {
    
        // you can inject your services here.
 
        @Override
        public String hello() {
            return "I'm hello activity";
        }
    }
```

### Declare your Temporal Workflows:

```java
    @WorkflowInterface
    public interface TestWorkflow {
    
        @WorkflowMethod
        void run();
    }
```

```java
    @TemporalWorkflow(queue = "testQueue", name="test")
    public class TestWorkflowImpl implements TestWorkflow {
    
        @TemporalActivityStub
        TestActivity testActivity;
    
        @Override
        public void run() {
            System.out.println(testActivity.hello());
            Workflow.sleep(10000);
            System.out.println("Workflow <<1>> completed");
        }
    }
```

### Run your workflow:

```java

    @Path("/temporal-client")
    @ApplicationScoped
    public class TemporalClientController {
    
        @Inject
        WorkflowBuilder workflowBuilder;
    
        @GET
        public String hello() {
            TestWorkflow testWorkflow = workflowBuilder.build(TestWorkflow.class, "test123");
            WorkflowClient.execute(testWorkflow::run);
            return "workfow started";
        }
    }

```
