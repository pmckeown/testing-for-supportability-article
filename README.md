# Testing For Supportability

## Intro
Operations are now becoming a key part of the life of a development team.  Teams are often asked to support the code
that they deploy into the production environment, especially when those applications are deemed non-critical to the
core business of the organisation.

Applications are often deployed to production without thoroughly verifying that the application _can_ be supported
properly, by the dev team of today or the team that inherits the application in the future. 

I am going to describe what I think are the key activities for a team to perform to help them prove that their
code is **Supportable**.

## We Are All Testers
Throughout this article I WILL refer to testers not as a profession, but as the person performing an activity.
She or he may well be a Developer, a Security Consultant, a Technical Analyst or a Tester, the point is that
all people involved in the production of code should in the business of quality assurance - we are all testers. 

## TL;DR
Let me cut to the chase - applications must be tested to prove that they can be supported. To prove that, testers
should be trying to break their applications through Exploratory Testing.  When it breaks, if the tester cannot easily 
tell why from observing the application, then there is work to be done to improve the non-functional characteristics of
the system.

## Observability
One of the most important aspects of an application is the ability to know what is going on inside it at runtime.
In a cloud native world, observability is harder as logging into the server and inspecting the logs is often simply
not possible. Servers are often ephemeral and so log data can be lost when the server is
destroyed.  In cloud and traditional on-premise environments the developers who know most about the system usually
don't have access to the servers to be able to view logs. Logs must be shipped automatically to a log aggregation
system such as [Splunk](https://www.splunk.com/ "Splunk") or the 
[ELK](https://www.elastic.co/what-is/elk-stack "ELK Stack") stack for indexing, visualisation, reporting and alerting.

With cloud-hosted systems, administrators can't use low-level tools like `top` and `Task Manager` for understanding the
resource usage of the servers running the applications.  Servers may not be accessible and you may be running
virtualised containers on virtual servers on hardware you don't own. Application Performance Monitoring (APM)
systems like [Dynatrace](https://www.dynatrace.com/) and [New Relic](https://newrelic.com/) are critical for you to
be able to visualise application behaviour, predict capacity breaches and raise problems to page on-call engineers when 
things go wrong.  Cloud platform providers also make tools for you to visualise your workload inside their platforms,
such as [CloudWatch on AWS](https://aws.amazon.com/cloudwatch/) and 
[Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/).

There are also Open Source players in this space that allow you to build dashboards from your application metrics and
logs.  [Grafana](https://grafana.com/ "Grafana") along with [Prometheus](https://prometheus.io/ "Prometheus") provide a
powerful stack for you to view your applications at runtime with and newer systems like 
[Grafana Loki](https://grafana.com/docs/loki/ "Grafana Loki") provide a more cost-effective log aggregation option
where you pay for storage based on what you use. Tools like Istio provide visibility of your service architectures
and traffic flows and new standards like [Open Telemetry](https://opentelemetry.io/) are making it easier to build more 
tooling in this space than ever before. 

## Breaking Stuff
So how do we know that we have built an observable system?  How do we know that when things inevitably break in
production that we will be able to determine the issue to restore service in as short a time a possible?

My advice is to try to break your systems in various ways and prove that when things do go wrong the tester is able
to easily see why. 

## Fixing Stuff (Quickly)
To reduce the [Mean Time To Repair](https://en.wikipedia.org/wiki/Mean_time_to_repair), whomever is paged to react to
the incident needs to be able to find out the reason for the degradation of service or full outage - and find out fast.

If you have an APM system, they can easily tell you which downstream system is broken but your logs should also do
that too.  APMs don't generally give you details of internal application logic failures.  They might show you a
spate of 401/403 responses but they can't generally give more details that that.  You logs are key and must clearly
show that (for example) the bearer tokens being presented are rejected due to key signing incompatibility as the
latest public key has not been fetched.  They can also make it obvious that some business logic is failing due to
data assumptions that are not true in the current environment.

Sometimes fixing things is a simple as "turning it off and on again" but sometimes it is much harder and visibility
into what the system is doing is your most important tool.

### Writing Good Logs
Writing good logs is a MUST in modern development.  Follow these guidelines to ensure that you have consistency
across different systems and that logs are available to those that need it:
* Use a standard logging framework like [log4j](https://logging.apache.org/log4j) or [serilog](https://serilog.net/).
* Use a standard for your log messages in all systems. The Open Telemetry project from the CNCF is incubating an open
  logging standard.  Until then, find another (e.g. [timber.io](https://github.com/timberio/log-event-json-schema)) or 
  invent your own.    
* Correlation IDs are key and must be propagated between systems.  For example create a UUID when your systems first
  get a request for work, pass it to downstream systems as metadata and make sure it is present in every log event in
  every system. 
* Ensure that you log all exceptions (they shouldn't be leaked out to consumers of your system via the UI or API) 
  as this is the easiest way to pinpoint the failing line of code.  Sometimes your APM can pinpoint this too.  
* Use a Log Aggregator like Splunk or ELK, ensure that everyone is properly trained in how to use it and that they
  have access to all of the data they need.
* Log all of your events in a machine parsable format (e.g. JSON) so that they can be easily indexed by your Log
  Aggregator.
* Sanitise your sensitive data to ensure that personally identifiable information is not logged.  This includes
  personal details, passwords and credit card numbers etc.  
* Use logging levels effectively and ensure that log levels can (ideally dynamically) be increased to aid
  troubleshooting and decreased to reduce noise.   

There are many good guides on good logging practices but if you start with the principles above you will go far.

### Test Your Logs
As with everything else in software, you should test your logs.  This is often seen an a subjective activity but you
can also objectively test and measure your logs events.

In unit tests, use mocking tools like [Mockito](https://site.mockito.org/) to verify that your Logger is called the
expected number of times with the necessary arguments.  

The following snippet shows an example where the logging behaviour is verified to log the exception that occurred and
business language warning event that was also written:

```java
public class MetricsAnalyserTest {

    @InjectMocks
    private MetricsAnalyser metricsAnalyser;

    @Mock
    private Logger logger;
 
    @Test
    public void thatWarningAndErrorEventsAreLoggedAsExpectedWhenAnExceptionOccurs() throws Exception {
        doThrow(Exception.class).when(metricsAnalyser).analsye(any(Metrics.class));

        try {
            metricsAnalyser.analyse(MetricsBuilder.anInvalidMetrics().build());
        } catch (Exception ex) {
            assertThat(ex, instanceOf(InvalidMetricsException.class));
        }
        
        // Verify errors events are recorded 
        verify(logger).error(any(InvalidMetricsException.class));
        // Verify warning events are recorded
        verify(logger).warn(any(String.class));
    }
}
```

In integration tests, use an in-memory log appender to capture your logs and verify they contain the expected contents.
Here is a [good example](https://www.baeldung.com/junit-asserting-logs).

In functional/component tests, testing logging is harder as you system is deployed to a real environment and mocking or 
capturing data is harder.  This is where the tester should be stressing the system and looking at it's non-functional
outputs to verify that they can see what went wrong.  

Examples of stressing include:
* Providing large data sets or disallowed character sets.
* Providing invalid security credentials.
* Make downstream systems unavailable to your application.
* Change system integration configuration to make downstream API calls timeout. 
* Push enough load into the system for it to start failing.

### Test Your Monitoring
The other critical component of your application's observability stack is the Application Performance Monitoring system
but you can't just trust that it meets your needs.  You also need to verify that:
* The metrics you need to support the application are easily visible.
* You can build dashboards in non-prod environments to support your testing and to ensure that you know that you can 
  in production.
* Run load tests into your applications and view resource usage stats in your APM. 
* Verify that errors are visualised as expected and problems are raised.
* Test that any extra metadata used in your APM is being applied as expected. 

### Test Your Metrics
If you use open source solutions like Prometheus to extract system metrics like response times, database query times,
memory and CPU statistics, then you must test that your application is configured correctly to ship its metrics and
they are made visible somewhere, such as in a Grafana dashboard.  The same verification needs apply to this open
source stack as the paid-for vendor solutions for Monitoring.   

## OAT - Too Little Too Late
Many companies rely on the Operational Acceptance Testing (OAT) phase of the software delivery life cycle to perform
this type of testing activity but in my view that is too late.  Testers should be looking at these operational
aspects of their runtime systems as early in the life cycle as possible to (like everything else) reduce the cost of
making changes or fixing defects in non-functional characteristics. 

## Other Ways To Improve Supportability
There are other techniques to improve the overall supportability of your system landscape:

### Game Days
Game Days are the gamification of MTTR.  They can help your organisation to learn how to deal with multi-system failures
and identify areas in your architecture where fragility and reduced supportability exists.  

This can lead you to find low-hanging fruit to fix up cheaply to increase your overall resiliency and provide a
better experience for your users.  Or perhaps even find critical flaws in your architecture before they manifest in
your production environment.
    
### Chaos Engineering
Chaos Engineering is the automation of Game Days and running that in Production.  As such it is a riskier and more
advanced approach to ensuring the overall resiliency of your distributed systems.  Care needs to be taken to minimise
the blast radius of automated chaos activity. Systems in this type of environment must be designed for resiliency and
auto-recovery from failures.  It goes without saying that if you were running this in production you will be doing to
same in non-production and with (ideally) continuous load into your systems to mimic production load and the same
approach to operational support that you would have in production.   

## Wrap Up
Designing and building distributed systems featuring high resiliency and automated recovery should be what we are all
striving to build and test in today's market.  

It may often not be possible to run chaos testing test in production for various reasons.  Compliance or risk aversion
may stop an organisation from going to this extreme for example.

However as testers and the guardians of quality in the systems that we build, we should think in this way during the
development life cycle and look for new and interesting ways to break our software and to see how it behaves.

Feeding the results of that exploratory testing back into the development backlog allows us continually improve the 
quality of our software and more importantly provide a better experience for our customers. 

Non-functional characteristics of a system are just as important as the functional ones and must also be rigorously
tested.  

After all, would you want to support a system in production that was silent and unfathomable?
