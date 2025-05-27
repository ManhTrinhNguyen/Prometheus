# Prometheus

- [Introduction](#Introduction)

  - [Why use Prometheus ?](#Why-use-Prometheus-?)
 
  - [Prometheus Architecture](#Prometheus-Architecture)
 
  - [Pull Mechanism](#Pull-Mechanism)
 
  - [Configuring Prometheus](#Configuring-Prometheus)
 
  - [Alert Manager](#Alert-Manager)
 
  - [Data Storage](#Data-Storage)
 
  - [PromQL Query Language](#PromQL-Query-Language)
 
  - [Prometheus Characteristic](#Prometheus-Characteristic)
 
  - [Scale Prometheus using Prometheus Federation](#Scale-Prometheus-using-Prometheus-Federation)
 
  - [Prometheus with Docker and Kubernetes](#Prometheus-with-Docker-and-Kubernetes)

## Introduction

Prometheus was created to monitor highly dynamic container environment like Kubernetes, Docker Swarm etc....

However it can be used in the traditional non-container infrastructure, where I have just bare servers with application deployed directly on it 

#### Why use Prometheus ? 

Modern Devop is becoming more and more complex to handle manually, and therefore need automation . 

So typically I have multiple servers that run containerized applications, and there are 100 of different processes running on that infrastructure, and things are interconnected . So maintaining such setup to run smoothly and with application downtimes is very challenging . 

- Imagine having such a complex infrastructure with loads of servers distributed over many locations, and I have no insight of what is happening on hardware level or on application level like Errors, Response Latency, hardware down, overloaded, maybe running out of resources etc .

- In such complex infrastructure there are more thing that can go wrong . When I have tons of servers and applications deplpoyed, any one of them can crash and cause failure of other services . And when I have so many moving pieces and suddenly application becomes unavailables to users, I must quickly identify what exactly out of these hundred different things went wrong, and that could be difficult and time comsuming when debugging the system manually

- For example : 1 of specific server ran out of memory and kicked off a running container that was responsible for providing database sync between 2 database pods in a Kubernetes Cluster. That in turn caused 2 databases pods to fail that database was used by an authentication service that also stopped working bcs the database became unavailable and an application that depended on that authentication service couldn't authenticate users in UI anymore . Bu from a user perspective, all I see is error in the UI `Can't login`. How do I know what went wrong when I don't have any insight of what's going inside the cluster, I don't see that red line of the chain of events . I just see the error, so I start working backwards from there to find the cause and fix it . Check all the way to the initial container failure . What will make this searching the problems process more efficent would be to have a tool that constantly monitors whether services are running and alerts the maintainers as soon as one service crashes, so I know exactly what happened, Or even better, it identifies problems before they even occur and alerts the system Administrators responsible for that infrastructure to prevent that issue . For example In this case it would check regularly the status of memory usage on each server, and when on one of the servers it spikes over, for example, 70% for over an hour or keeps increasing notify about the risk that the memory on that server might soon run out

- Or let's consider another scenario where suddenly I stop seeing logs for my application bcs Elasticsearch doesn't accept any new logs bcs the server ran out of disk space or Elasticsearch reached the storage limit that was allocated for it . The Mornitor tools would check continuously the storage space and compare with the Elasticsearch consumption of space of storage, and it will see the risk and notify maintainers of the possible storage issue. And I can tell the mornitoring tool what critical point is when the alert should be triggered . For exmaple if I have a very important application that absolutely can have any log data loss, I might be very strict and want to take measure as soon as 50% or 60% capacity is reached . Or maybe I know adding more storage will take long bcs it is a bureaucratic process in my organization where I need approval of some IT department and several other people . Then maybe I also want to be notified earlier about the possible storage issue . So that I have more time to fix it

- Or Third scenario where application becomes too slow bcs one service breaks down and starts sending 100 of error messages in a loop across the network that creates high network traffic and slow down other services too . Having a tool that detects such spikes in network load, plus tells you which service is responsible for causing it, Can give me timely alert to fix the issue

 #### Prometheus Architecture 

At it core Prometheus has the main component called `Prometheus server` that does the actual monitoring work and is made up of three parts : 

<img width="600" alt="Screenshot 2025-05-27 at 10 51 58" src="https://github.com/user-attachments/assets/4e8f971c-7d83-4080-ae65-2412ed44d0ae" />

- `Time Series Database` : that stores all the metrics data like current CPU usage or a number of exceptions in an application

- `Data retrieval worker`: that is responsible for getting or pulling those metrics from application, services, servers and other target resources and storing them or pusing them into the Database .

- `Web Server or Server API`: that accept queries for that stored data. And that web server component or Server API is used to display the data in a dashboard UI, either through Prometheus dashboard or some other data visualization tool like Grafana .

So the Prometheus server mornitor a particular thing . Could be anything . Could be entire Linux/Window Server or It could be standalone Apache server, a single application or service like database and those thing that Prometheus monitors are called `Targets` . 

Each `Target` has `Units` of monitoring . 

<img width="600" alt="Screenshot 2025-05-27 at 11 03 47" src="https://github.com/user-attachments/assets/908e4324-09b0-431d-8a6e-b564b762aa6e" />

- For Linux Server target it could be a current CPU status, its memory usage, disk space usage etc ...

- For Application for example it could be a number of exceptions, number of requests or request duration .

And that `unit` I would like to monitor for a specific target is called a `Metric`

- `Metric` are what gets saved into Prometheus database component .

- Prometheus defines human readable text based format for these metrics . Metrics entries or data has `TYPE` and `HELP` attribute to increase its readability

- `HELP` is description just describe what the metrics is about

- `TYPE` is one of 3 `metrics` types

  - For metics about how many times something happended, like number of exeception that application had or number of request it has received there a `Counter Type`
 
  - Metric that can go both up and down is represented by a `Gauge`. For example what is the current value of CPI usage now ? Or what is the current capacity of disk space now ? Or what is the number concurrent requests at that given moment ?
 
  - And for tracking how long something took or how big, for example, a size of a request was there is a `Historgram Type`

How does Prometheus collect those `metrics` from the `targets` ? 

- Prometheus pulls metric data from the targets from an HTTP endpoint, which by default is hostaddress/metrics

- And for that to work , One targets must expose that `/metrics` endpoint, and two data available at `/metrics` endpoint must be in the format that Prometheus can understands

Some servers already exposing Prometheus endpoints, so I don't need extra work to gather metrics from them .

But many services don't have native Prometheus endpoints, so extra component is required to do that and this component is `Exporter`

- `Exporter` is basically a script or a service that fetches metrics from my `Target` and converts them in format Prometheus understands and exposes this converted data at its own `/metrics` endpoint, where Prometheus can scrape them.

- And Prometheus has a list of `exporters` for different services like MySQL, Elasticsearch, Linux Server, build tool, clould platforms, and so on

-  For example if I want to mornitor a Linux Server, I can download a Node  `exporter` tar file from Prometheus repository . I can untart and execute it and it will start converting the metrics of the Server and making them scrapable at its own `/metrics` enpoints . And then I can go and configure Prometheus to scrape that endpoint and this `exporters` are also available as Docker Images .

-  For example if I want to monitor my MySQL Container, in Kubernetes cluster, I can deploy a `sidecar`  container of MySQL `exporter` that will run inside the pod with MySQL container connect to it and start translating MySQL metrics for Prometheus and making them available at its own `/metrics` endpoint

-  Once I add MySQL exporter endpoint to Prometheus configuration, Promethues will start collecting those metrics and saving them in its database 

**Monitoring my own Application**

Let's say I want to see how many requests my application is getting at different times or how many exeptions are occurring, how many server resources my application is using etc ... 

For this use case there is Prometheus `client library` for different languages like Nodejs, Java etc...

Using those Library I can expose the `/metrics` scraping endpoint in my application and provide different metrics that are relevant for me on that endpoint 

This is the convenient way to tell the Infrastructure team to tell Developers emit metrics that are relevant to me and will collect and monitor them in our infrastructure . 

#### Pull Mechanism 

So mentioned that Prometheus pull these data from enpoints and that's actually an imporant characteristic of Prometheus 

Most monitoring systems like Amazon Cloudwatch or new Relic etc.. use a push system, meaning applications and servers are responsible for pushing their metric data to a centralized collection platform of that monitoring tool . So when I working with many micros Services and I have each service pushing the metrics ti the monitoring system it creates a high load of traffics within my infrastructure and my mornitoring can actually become my bottleneck . So I have monitoring which is great but I pay the price of overloading my infrastructure with constant push request from all the services and thus flooding the network. Plus I also have to install daemons on each of these targets to push the metrics to monitoring server, while prometheus requires just a scraping endpoint . And this way metrics can also be pulled by multiple Prometheus instances. 

And another advanrage of that is, using `pull`, Prometheus can easily detect whether services up and running 

- For example when it doesn't respond on the `pull` or when the endpoint isn't available . While with `push` if the services doesn't push any data or send its health status, it migh have many reason other than the service isn't running . It could be that network isn't working. The package get lost on the way or some other problem . So I don't really have an insigjt of what happended .

- But there are limited number of cases where a target that needs to be monitored run only for a short time . So they aren't around long enough to be scraped . Example it could be a batch job or scheduled job that say cleans up some old data or does backups etc... . For such jobs, Prometheus offers `pushgateway` component so these services can push their metrics directly to Prometheus database. But obviously using `pushgateway` to gather metric in Prometheus should be an exception

#### Configuring Prometheus

How does Prometheus know what to scrape and when ?

All that is configured in Prometheus `.yaml` configuration . So I define which targets Prometheus should scrape and at what interval . Prometheus then uses a service discovery mechanism to find those target endpoints . When I first download and start Prometheus I will see the sample config file with some default values in it 

For example :

<img width="600" alt="Screenshot 2025-05-24 at 14 12 55" src="https://github.com/user-attachments/assets/212e4409-a87a-46eb-9ef5-ac95b3346da8" />


- We have `global` config that defines scrape interval or how ofter Prometheus will scrape its targets. And I can override these for individual `targets`

- The `rule_files` block specifies location of any `rules` we want Prometheus server to load and the `rules` are basically either for aggregating metric values or creating alerts when some condition is met like CPU usage reached 80%. So Prometheus uses `rules` to create new time series entries and to generate alerts and the evaluation interval option in global config defines how often Prometheus will evaluate these `rules`

- Last block is `scrape_configs` controls what resources Prometheus monitors . This is where I define the `Targets`. Since Prometheus has its own metrics endpoints to expose its own data, it can monitor its own health . So in this default configuration, there is a single job called Prometheus, which scrapes the metrics exposed by the Prometheus Server . So it  has a single target at localhost 9090 and Prometheus expects metrics to be available on that `target` on a path of `/metric` which is a default path that is configured for that endpoint 

I also can define another endpoint to scrape through jobs, so I can create another job and for example override the `scrape_interval` from the global configuration and define target host address 

<img width="600" alt="Screenshot 2025-05-24 at 14 15 25" src="https://github.com/user-attachments/assets/cd63c8c3-ff72-42f3-9884-27ee5c442968" />

#### Alert Manager

Important point : 

First one is how des Prometheus actually trigger the alerts that are defined by `rules` and who receies them ?

- Prometheus has a compomnent called `alert manager` that is responsible for firing alers via different channels . It could be email it could be a Slack channel or some other notification client . So Prometheus server will read the alert `rules` and if the condition in the rules is met an alert gets fired through that configured channel

#### Data Storage

Second one this Prometheus data storage. Where does Promethus store all this data that it collects and then aggregates ? And how can other system access this  data 

Prometheus stores metrics data on disk so it includes a local on disk time series database, but also optionally intergrates with remote storage system . And the data is stored in a custom time series format . And bcs of that I can write Prometheus data directly into a relational database 

So once I have collected the metrics, Prometheus also lets me query the metrics data on targets through its server API using PromQL, query language 

#### PromQL Query Language 

I can use Prometheus dashboard UI to ask the Prometheus server via PromQL to show status of a particular target right now, or I can use more powerful data visualization tools like Grafana to display the data, which under the hood also use PromQL to get the data out of Prometheus 

Exmaple of PromQL query : 

- With this one here basically queries all HTTP status codes, execpt the ones in 400 range

- And this one basically does some sub query that for a period of 30 mins

But with Grafana instead of writing PromQL queries directly into the Prometheus server, I basically have Grafana UI where I can create dashboard that can then in the background use Prom QL to query the data 


#### Prometheus Characteristic

Prometheus Characteristic that it is designed to be reliable, even when other systems have an outage so that I can diagnose the problems and fix them 

So each Prometheus server is standalone and self-containing meaning it doesn't depend on network storage or other remote services . It meant to work when other parts of the infrastructure are broken and I don't need to set up extensive infrastructure to use it 

However it also has disadvantage that Prometheus can be difficult to scale . So when I have 100 of servers I might want to have multiple Prometheus servers that somewhere aggreate all these metrics and configuring that and scaling Prometheus in that way can actually be very difficult bcs of this charactieristic 

While using a single node is less complex and I can get started very easily it put a limit on the number of metric that can be monitored by Prometheus . 

So to work around that I either increase the capacity of the Prometheus server so we can store more metrics data . Or I limit the number of metrics that Prometheus collects from the applications to keep it down to only relevant one 

#### Scale Prometheus using Prometheus Federation 

Now if I have a set up of 1000 of nodes where we need to scale Prometheus, as an alternatvie we can build what's called `Prometheus Federation`. 

`Prometheus Federation` allow one promethues server to scrape data from another prometheus server . 

If we have tones of nodes and we need scalability, this way using Prometheus Federation we can achieve a scalable Prometheus monitoring set up

#### Prometheus with Docker and Kubernetes


In terms of Prometheus with Docker and Kubernetes, Prometheus is fully compatible with both 

Prometheus components are available as Docker image and therefore can easily be deployed in Kunernetes or other container environments 

And integrates greate with Kubernetes infrastructure, providing cluster node resource monitoring out of the box . Which mean once it's deployed on Kubernetes, it starts gathering metrics data on each Kubernetes node server without any extra configuration 









