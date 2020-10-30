# Testing For Supportability

## Contents
- [Intro](#intro)
- [TL;DR](#tl-dr)
- [Observability](#observability)
- [Breaking Stuff](#breaking-stuff)
- [Fixing Stuff (Quickly)](#fixing-stuff--quickly-)
  * [Write Good Logs](#write-good-logs)
  * [Test Your Logs](#test-your-logs)
  * [Test Your Monitoring](#test-your-monitoring)
  * [Test Your Metrics](#test-your-metrics)
- [Operational Acceptance Testing Is Too Late](#operational-acceptance-testing-is-too-late)
- [Other Ways To Improve Supportability](#other-ways-to-improve-supportability)
  * [Game Days](#game-days)
  * [Chaos Engineering](#chaos-engineering)
- [Wrap Up](#wrap-up)

## Intro
Operations are now becoming a key part of the life of a development team. Development teams are often tasked with 
supporting code that they deploy to production.  This is even more common when those applications are non-critical to 
the organisation.

Applications are often deployed to production without testing that the application can be supported.  But that support
will eventually need to come from today's dev team or one in the future.

I am going to describe what I think are the key activities for a team to perform to help them prove that their
code is **Supportable**.

Throughout I refer to testers not as a profession but as the person performing a software verification activity.  She
or he may well be a Developer, a Security Consultant, a Technical Analyst or a Tester.  All people involved in the
production of code should in the business of quality assurance.  We are all testers. 

## TL;DR
Applications must be tested to prove that they can be supported. To prove that, testers should be trying to break their 
applications.

The tester should be able to tell what broke and why from observing the application.  If not, changes are needed to
improve the non-functional characteristics of the system.

## Observability
One of the most important aspects of an application is the ability to know what is going on inside it at runtime.

In a cloud native world, observability is harder as logging into the server and inspecting the logs is often not 
possible. Servers are often ephemeral and so log data can be lost when the server is destroyed.

In both cloud and traditional on-premise environments, access to servers is restricted. Logs must be shipped to a log 
aggregator such as [Splunk](https://www.splunk.com/ "Splunk") or the 
[ELK Stack](https://www.elastic.co/what-is/elk-stack "ELK Stack") for indexing, visualisation, reporting and alerting.

Application Performance Monitoring (APM) systems like [Dynatrace](https://www.dynatrace.com/) and 
[New Relic](https://newrelic.com/) are critical today.  They allow you to visualise application behaviour, predict
capacity breaches and raise problems to page on-call engineers. Cloud platform providers also provide tools for you to 
visualise your workload inside their platforms - [CloudWatch on AWS](https://aws.amazon.com/cloudwatch/) and 
[Azure Monitor](https://docs.microsoft.com/en-us/azure/azure-monitor/).  Some APMs can also trigger actions to resolve 
issues without manual intervention.

There are Open Source players in this space that allow you to build dashboards from your application metrics and logs.
[Grafana](https://grafana.com/ "Grafana") along with [Prometheus](https://prometheus.io/ "Prometheus") provide a
powerful stack for you to view your applications at runtime. 

Newer systems like [Grafana Loki](https://grafana.com/docs/loki/ "Grafana Loki") provide a more cost-effective log
aggregation option where you pay for storage based on what you use. Tools like [Istio](https://istio.io/) provide 
visibility of your microservice architectures and traffic flows. Also new standards like 
[Open Telemetry](https://opentelemetry.io/) are making it easier to build more tooling in this space than ever before. 

## Breaking Stuff
So how do we know that we have built an observable system? How do we know that when things break in production we will 
be able to determine the root cause?  How can we restore service in as short a time a possible?

The only way to know this is to try to break your systems in various ways.  Then prove that when things do go wrong the 
tester is able to easily see why. 

## Fixing Stuff (Quickly)
To reduce the [Mean Time To Repair](https://en.wikipedia.org/wiki/Mean_time_to_repair), whomever reacts to the incident 
must be able to find the root cause as fast as possible..

If you have an APM system, they can tell you which downstream system is broken.  Your logs should also do that too.  

APMs don't generally give you details of internal application logic failures.  They might show you a spate of 401/403
responses but they can't generally give more detail than that.  

For example your logs must show that bearer tokens are being rejected due to key signing incompatibility. Or highlight 
that business logic is failing due to incorrect data assumptions.

Sometimes fixing things is as simple as "turning it off and on again".  But sometimes it is much harder and visibility 
into what the system is doing is your most important tool.

### Write Good Logs
Writing good logs is a MUST in modern development.  Ensure that log data are available to those that need.  

Follow these guidelines to ensure that you have consistency across different systems:
* Use a standard logging framework like [log4j](https://logging.apache.org/log4j) or [serilog](https://serilog.net/).
* Use a standard for your log messages in all systems. The Open Telemetry project from the CNCF is incubating an open
  logging standard.  Until then, find another (e.g. [timber.io](https://github.com/timberio/log-event-json-schema)) or 
  invent your own.    
* Correlation IDs are key and must be propagated between systems.  For example create a UUID when your systems first
  get a request for work. Pass it to downstream systems as metadata and make sure it is present in every log event in
  every system. 
* Ensure that you log all exceptions (they shouldn't be leaked out to consumers of your system via the UI or API).  
  This is often the easiest way to pinpoint the failing line of code.   
* Use a Log Aggregator like Splunk or ELK.  Ensure that everyone is trained in how to use it and that they have access 
  to all the data they need.
* Log all your events in a machine parsable format (e.g. JSON) so that they can be indexed by your Log Aggregator.
* Sanitise your sensitive data to ensure that personally identifiable information is not logged.  This includes
  personal details, passwords and credit card numbers etc.  
* Use log levels effectively.  Ensure that log levels can be dynamically increased to aid troubleshooting or decreased 
  to reduce noise.
  
A good example of a log event taken from [timber.io](https://github.com/timberio/log-event-json-schema) is:
```
{
  "dt": "2016-12-01T02:23:12.236543Z",           // Consistent ISO8601 dates with nanosecond precision
  "level": "info",                               // The level of the log
  "message": "POST /checkout for 192.321.22.21", // Human readable message
  "context": { ... },                            // Context data shared across log line
  "event": { ... }                               // Structured representation of this log event
}
```
`Note that this JSON content should all be contained on a single line. It is only pretty printed here for readability.`

There are many good guides on good logging practices but if you start with the principles above you will go far.

If your logs don't follow these basic principles then talk to your product owner or engineering lead to start to 
address this. 

### Test Your Logs
As with everything else in software, you should test your logs. This is often seen as a subjective activity but you 
can also objectively test and measure your logs events.

In unit tests, use mocking tools like [Mockito](https://site.mockito.org/ "Mockito") to verify that your Logger is 
called the expected number of times.  Verify the correct arguments are passed.

The following snippet shows an example of verifying that exception and warning events are both logged when an exception 
occurs:

```java
public class MetricsAnalyserTest {

    @InjectMocks
    private MetricsAnalyser metricsAnalyser;

    @Mock
    private Logger logger;
 
    @Test
    public void thatWarningAndErrorEventsAreLoggedAsExpectedWhenAnExceptionOccurs() throws Exception {
        // Arrange
        Metrics invalidMetrics = MetricsBuilder.anInvalidMetrics().build();
        doThrow(InvalidMetricsException.class).when(metricsAnalyser).analyse(invalidMetrics);

        // Act
        try {
            metricsAnalyser.analyse(invalidMetrics);
            fail("Exception expected");
        } catch (Exception ex) {
            assertThat(ex, instanceOf(InvalidMetricsException.class));
        }
        
        // Assert
        // Verify errors events are recorded 
        verify(logger).error(isA(InvalidMetricsException.class));
        // Verify warning events are recorded
        verify(logger).warn("Invalid metrics data was supplied");
    }
}
```

In integration tests, use an in-memory log appender to capture your logs.  Verify they contain the expected contents.
Here is a [good example](https://www.baeldung.com/junit-asserting-logs).

In functional or component tests, verifying of logging is tricky.  Your system is deployed to a real environment where 
capturing log data is harder. 

Here the tester should stress the system and look at it's non-functional outputs.  They should prove that they can see 
what went wrong.  

Examples of stressing include:
* Providing large data sets or disallowed character sets.
* Providing invalid security credentials.
* Make downstream systems unavailable to your application.
* Change system integration configuration to make downstream API calls timeout. 
* Push enough load into the system for it to start failing.

In scenarios like these you should see failure responses.  They should provide the minimal amount of information to the 
client about what went wrong. But the system logs should give you full details of the failure.

### Test Your Monitoring
The other critical observability component stack is the Application Performance Monitoring system.  But you can't 
blindly trust that it meets your needs out of the box. You also need to verify that:
* The metrics you need to support the application are easily visible.
* You can build dashboards in non-prod environments to support your testing.  Also to ensure that you know what you can 
  visualise in production.
* You can run performance tests against your applications and view resource usage stats. 
* You can verify that errors are visualised as expected and problems are raised.
* Any extra metadata used in your APM is being applied as expected. 

### Test Your Metrics
Open source tools like Prometheus can extract system metrics like response times or CPU statistics.  If you use this 
then you must test that your application is shipping it's metrics. Also that they are made visible somewhere, such as 
in a Grafana dashboard.

The same verification for vendor monitoring solutions for applies to open source ones.

## Operational Acceptance Testing Is Too Late
Many companies rely on the Operational Acceptance Testing (OAT) phase to test for supportability.  But in my view that 
is too late.

Testers should be looking at the operational aspects of their runtime systems as early as possible.  Defects and changes
are more expensive the closer to production you get.  So reduce the cost of changing or fixing non-functionals by 
testing them early.

## Other Ways To Improve Supportability
Game Days and Chaos Engineering are newer techniques to improve the supportability of your systems.

### Game Days
Game Days are the gamification of MTTR. They can help your organisation to learn how to deal with multi-system failures. 
Also identify areas in your architecture where fragility and reduced supportability exists.  

This can lead you to find low-hanging fruit to fix up to increase your resiliency.  Or find critical flaws in your 
architecture before they manifest in production.
    
### Chaos Engineering
Chaos Engineering is practice of injecting failures and issues into Production.  As such it is a riskier and more
advanced approach to ensuring the overall resiliency of your distributed systems.  

Care needs to be taken to minimise the blast radius of automated chaos activity. Systems in this type of environment
must be designed for resiliency and auto-recovery from failures.  

It goes without saying that if you were running this in production you will be doing the same in non-production. You 
should push continuous traffic into your systems to mimic production load. Take the same approach to operational 
support that you would have in production - keep MTTR low.   

## Wrap Up
If you are building distributed systems today, you should be striving for high resiliency and automated recovery. If 
you are testing them you should be able to prove these characteristics.

But systems cannot always be built to include automated recovery and all systems will inevitably break.  When
they do it is imperative to be able to find and fix the issue as quickly as possible.  The **Mean Time To Repair** is
your key metric to drive down to improve your customer's experience. 

To do that you must test for supportability to prove that when things do go wrong the support engineer (developer, 
tester, SRE or ops specialist) can find and fix it as soon as possible.

It may often not be possible to run chaos testing test in production for various reasons. But as the guardians of
quality we should think like this during the development life cycle.  

- We should look for new and interesting ways to break our software to see how it behaves.  

- We should improve observability when we cannot see into our systems.  

- We should explore and poke at the edges of our systems and not focus only on functionality.

Importantly we need to feed the results of that exploration back into the development backlog.  This allows us to 
improve the quality and security of our software and to provide a better experience for our customers.

**Non-functional characteristics of a system are as important as the functional ones.  They must also be rigorously 
tested.**

After all, would you want to support a system in production that was silent and unfathomable?
