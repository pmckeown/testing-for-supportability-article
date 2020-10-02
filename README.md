# Testing for Supportability

## Intro
Operations are now becoming a key part of the life of a development team.  Teams are often asked to support the code
that they deploy into the production environment, especially when those applications are deemed non-critical to the
core business of the organisation.

Cloud native applications that run in other people's computers are often deployed to production without thoroughly
verifying that the application _can_ be supported properly, by the dev team of today or the team that inherits the
application in the future. 

Here I try to lay out what I think are the key activities for a team to perform to help them prove that their code is
**Supportable**.

## We are all testers
Note that throughout this article I refer to testers not as a profession, but as the person performing an activity.
She or he may well be a developer, a security consultant, a technical analyst or even a tester, the point is that
all people involved in the production of code should in the business of quality assurance - we are all testers. 

## TL;DR
Let me cut to the chase and state what must now be obvious - applications must be tested to prove that they can be
supported. To prove that, testers should be trying to break their applications through Exploratory Testing.  When it
breaks, if the tester cannot tell easily from observing the application, then there is work to be done.

## Observability
One of the most important aspects of a cloud native application is the ability to know what is going on inside the
application at runtime.  Gone are the days of ops teams logging into the server and inspecting the logs, now logs
are expected to be shipped automatically to a log aggregation system such as 
[Splunk](https://www.splunk.com/ "Splunk") or the [ELK](https://www.elastic.co/what-is/elk-stack "ELK Stack") stack
for indexing, visualisation, reporting and alerting.

Also gone are the days of Sys Admins using low-level tools like `top` and `Task Manager` for understanding the
resource usage of the servers running the applications.  Servers may not be accessible and you may be running
virtualised containers on virtual servers on hardware you don't own.  Nowadays Application Performance Monitoring
systems like [Dynatrace](https://www.dynatrace.com/) and [New Relic](https://newrelic.com/) are critical for you to
be able to visualise application behaviour, predict capacity breaches and page on-call engineers when $h!t hits the
fan.  

There are also Open Source players in this space that allow you to build dashboards from your application metrics and
logs.  [Grafana](https://grafana.com/ "Grafana") along with [Prometheus](https://prometheus.io/ "Prometheus") provide a
powerful stack for you to view your applications at runtime with and newer systems like 
[Grafana Loki](https://grafana.com/docs/loki/ "Grafana Loki") provide a more cost-effective log aggregation option
where you pay for storage based on what you use.

## Breaking stuff

## OAT Phase too late

## Real-world Examples
* Log why a request was denied, but not the details of the credential that was presented 