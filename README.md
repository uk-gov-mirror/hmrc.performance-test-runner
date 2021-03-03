
# performance-test-runner




This is a wrapper around the [Gatling](http://gatling.io/) load testing framework, 
with preconfigured injection steps, protocols and assertions.


### Adding to your performance test

#### Compatible versions

Library Version | Scala Version | gatling-version*          | gatling-sbt plugin
--------------- | ------------- | ------------------------- | ------------------ 
4.x.x           | 2.12          | 3.4.2                     | 3.2.1
3.x.x           | 2.11, 2.12    | 2.2.5, 2.3.1              | 2.2.0, 2.2.2

Gatling version refers to the version of the below Gatling dependencies:
- gatling-test-framework
- gatling-charts-highcharts

Add the below dependencies:

```scala
"uk.gov.hmrc"          %% "performance-test-runner"   % "x.x.x" % Test,
"io.gatling"            % "gatling-test-framework"    % "x.x.x" % Test,
"io.gatling.highcharts" % "gatling-charts-highcharts" % "x.x.x" % Test
```

Add the below plugin:
```
addSbtPlugin("io.gatling" % "gatling-sbt" % "x.x.x")
```

### Implement your first simulation

##### Step 1: Implement the requests to your pages

```scala
import uk.gov.hmrc.performance.conf.ServicesConfiguration

import io.gatling.core.Predef._
import io.gatling.http.Predef._

object HelloWorldRequests extends ServicesConfiguration {

  val baseUrl = baseUrl("hello-world-frontend")

  def navigateToLoginPage =
    http("Navigate to Login Page")
      .get(s"$baseUrl/login")
      .check(status.is(200))


  def submitLogin = {
    http("Submit username and password")
      .post(s"$baseUrl/login": String)
      .formParam("userId", "${username}")
      .formParam("password", "${password}")
      .check(status.is(303))
      .check(header("Location").is("/home"))
  }

  def navigateToHome =
    http("Navigate To Home")
      .get(s"$baseUrl/home")
      .check(status.is(200))

}
```

##### Step 2: Setup the simulation

```scala
import uk.gov.hmrc.performance.simulation.ConfigurationDrivenSimulations
import HelloWorldRequests._

class HelloWorldSimulation extends ConfigurationDrivenSimulations {

  setup("login", "Login") withRequests (navigateToLoginPage, submitLogin)

  setup("home", "Go to the homepage") withRequests navigateToHome

  runSimulation()
}
```

##### Step 3. Configure the journeys 

journeys.conf

```
journeys {

  hello-world-1 = {
    description = "Hello world journey 1"
    load = 9.1
    feeder = data/helloworld.csv
    parts = [
      login,
      home
    ]
    run-if = ["label-A"]
  }
  
  hello-world-2 = {
    description = "Hello world journey 2"
    load = 6.2
    feeder = data/helloworld.csv
    parts = [
      login
    ]
    run-if = ["label-B"]
  }

}
```

##### Step 4. Configure the tests

application.conf

```
runLocal = true

baseUrl = "http://helloworld-service.co.uk"

perftest {
  rampupTime = 1
  constantRateTime = 5
  rampdownTime = 1
  loadPercentage = 100
  journeysToRun = [
    hello-world-1,
    hello-world-2
  ],
  labels = "label-A, label-B"   #optional
  percentageFailureThreshold = 5

}

}
```

##### Step 4. Configure the user feeder

helloworld.csv

```csv
username,password
bob,12345678
alice,87654321
```

In your csv you can use placeholders:<br>
`${random}` is replaced with a random int value<br>
`${currentTime}` is replaced with the current time in milliseconds<br>
`${range-X}` is replaced by a string representation of number made of X digits. 
The number is incremental and starts from 1 again when it reaches the max value. 
For example ${range-3} will be replaced with '001' the first time, '002' the next and so on. 

This allows you to create a csv with only one record but an infinite number of random values 
```csv
username,password
my-${random}-user,12345678
```

### Run a smoke test

To run a smoke test through all journeys, with one user only, set the following.

```
sbt -Dperftest.runSmokeTest=true -Djava.io.tmpdir=${WORKSPACE}/tmp test
```

### Run the test

To run the full performance test execute

```
sbt -Djava.io.tmpdir=${WORKSPACE}/tmp test
```


### More about setting up the simulation 
#### Using a pause 

```scala
import io.gatling.core.action.builder.PauseBuilder
import uk.gov.hmrc.performance.simulation.ConfigurationDrivenSimulations
import HelloWorldRequests._

import scala.concurrent.duration._

class HelloWorldSimulation extends ConfigurationDrivenSimulations {
  
  val pause = new PauseBuilder(1 milliseconds, None)
  
  setup("login", "Login") withActions(navigateToLoginPage, pause, submitLogin)

  runSimulation()
}
```

### More about the journey configuration.

`description` will be assigned to the journey in the test report

`load` is the number of journeys that will be started every second

`feeder` is the relative path to the csv feeder file. More [here](http://gatling.io/docs/2.1.7/session/feeder.html#csv-feeders)

`parts` is the list of parts that combined create your journey

`run-if` is a list of labels. Runs this journey only if a label from this list is passed in the `labels` parameter of application.conf

`skip-if` is a list of labels. Skips this journey if a label from this list is passed in the `labels` parameter of application.conf

You can have as many journeys as you like in journeys.conf, the simulation will start and run them all together.


### More about the services configuration.

Contains the name of the service and the port when running locally. Read the services-local.conf file for more details and examples.


### More about application.conf

`runLocal` boolean value to run test locally. Default value is true.

`baseUrl` is the default url for every service. Read the application.conf file for more details and examples.

`rampupTime` is the time in minutes for the simulation to inject the users with a linear ramp up

`constantRateTime` is the time in minutes for the simulation to inject users at a constant rate

`rampdownTime` is the time in minutes for the simulation to inject the users with a linear ramp down

`loadPercentage` is the percentage of the load for the journeys. Read the application.conf file for more details and examples.

`journeysToRun` contains the journeys that will be executed. Leave it empty if you want to run all the journeys

`labels` optional string containing a comma-separated list of test labels. Read the application.conf file for more details.

`percentageFailureThreshold` optional int. Read the application.conf file for more details.

## Scalafmt
This repository uses [Scalafmt](https://scalameta.org/scalafmt/), a code formatter for Scala. The formatting rules configured for this repository are defined within [.scalafmt.conf](.scalafmt.conf).

To apply formatting to this repository using the configured rules in [.scalafmt.conf](.scalafmt.conf) execute:

 ```
 sbt scalafmtAll scalafmtSbt
 ```

To check files have been formatted as expected execute:

 ```
 sbt scalafmtCheckAll scalafmtSbtCheck
 ```

[Visit the official Scalafmt documentation to view a complete list of tasks which can be run.](https://scalameta.org/scalafmt/docs/installation.html#task-keys)


### License

This code is open source software licensed under the [Apache 2.0 License]("http://www.apache.org/licenses/LICENSE-2.0.html").
    
