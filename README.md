# Prometheus

- [Introduction](#Introduction)

  - [Why use Prometheus](#Why-use-Prometheus)
 
  - [Prometheus Architecture](#Prometheus-Architecture)
 
  - [Pull Mechanism](#Pull-Mechanism)
 
  - [Configuring Prometheus](#Configuring-Prometheus)
 
  - [Alert Manager](#Alert-Manager)
 
  - [Data Storage](#Data-Storage)
 
  - [PromQL Query Language](#PromQL-Query-Language)
 
  - [Prometheus Characteristic](#Prometheus-Characteristic)
 
  - [Scale Prometheus using Prometheus Federation](#Scale-Prometheus-using-Prometheus-Federation)
 
  - [Prometheus with Docker and Kubernetes](#Prometheus-with-Docker-and-Kubernetes)
 
- [Prometheus Stack](#Prometheus-Stack)

  - [Demo Overview](#Demo-Overview)
 
  - [Create EKS Cluster](#Create-EKS-Cluster)
 
  - [Deploy Micros Services Applications](#Deploy-Micros-Services-Applications)
 
  - [Deploy Prometheus Stack using Helm](#Deploy-Prometheus-Stack-using-Helm)
 
  - [Understanding Prometheus Stack Components](#Understanding-Prometheus-Stack-Components)
 
  - [Components inside Promethetheus Alertmanager Operator](#Components-inside-Promethetheus-Alertmanager-Operator)
 
- [Data Visualization with Prometheus UI](#Data-Visualization-with-Prometheus-UI)

  - [Decide what we want to monitor](#Decide-what-we-want-to-monitor)
 
  - [Prometheus UI](#Prometheus-UI)
 
  - [Introduction to Grafana UI](#Introduction-to-Grafana-UI)
 
  - [Create my own Dashboard](#Create-my-own-Dashboard)
 
  - [Resource Consumption of Cluster Nodes](#Resource-Consumption-of-Cluster-Nodes)
 
  - [Test Anomoly](#Test-Anomoly)
 
  - [Configure Users and Data Sources](#Configure-Users-and-Data-Sources)
 
- [Alert Rules in Prometheus](#Alert-Rules-in-Prometheus)

  - [Existing Alert Rules](#Existing-Alert-Rules)

- [Create own Alert Rules](#Create-own-Alert-Rules)

  - [First Rule](#First-Rule)
 
- [Alert Rule for Kubernetes](#Alert-Rule-for-Kubernetes)

  - [Write 2nd Alert Rule](#Write-2nd-Alert-Rule)
 
  - [Apply Alert Rules](#Apply-Alert-Rules)
 
  - [Test Alert Rule](#Test-Alert-Rule)
 
- [Alert Manager](#Alert-Manager)

  - [Firing State](#Firing-State)
 
  - [Alertmanager Configuration File](#Alertmanager-Configuration-File)
 
- [Configure Alertmanager with Email Receiver](#Configure-Alertmanager-with-Email-Receiver)

  - [Condfigure Email Notification](#Condfigure-Email-Notification)
 
  - [Apply the configurations](#Apply-the-configurations)
 
- [Trigger Alerts for Email Receiver](#Trigger-Alerts-for-Email-Receiver) 

## Introduction

Prometheus was created to monitor highly dynamic container environment like Kubernetes, Docker Swarm etc....

However it can be used in the traditional non-container infrastructure, where I have just bare servers with application deployed directly on it 

#### Why use Prometheus

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


## Prometheus Stack 

I will set up my Cluster on EKS and deploy Prometheus 

There are serveral moving parts and components in Prometheus monitoring stack 

How do we  go about deploying it in Kubernetes Cluseter:

1. Putting it together all the configuration files I need for all the parts . So for each component of the Prometheus monitoring stack, basically creating those YAML file . For Prometheus Stateful set, Alert Manager, Grafana deployments, all the config maps and secrets that I need ... and then going ahead and executing them in the right order bcs of the dependencies (Not Recommended)

2. Using Operator . Think of an operator as a manager of all Prometheus individual components that I create . So Stateful Set and Deployment will manage their pod replicas like restart them when they die, make sure they are accessible and so on . In the same way, Operator will keep an eye and manage the combination of Stateful set, Deployment and all the other components that make up Prometheus as one unit so that I don't have to manually manage those separate pieces . I will find an Operator for Prometheus and deploy it in the configuration file

3. Using Helm chart to deploy the Operator . Prometheus Operator has a Helm Chart that is maintained by the Helm community itself (Recommended)

  - So Helm will do the initial setup and operator will then manage the running Prometheus set up

#### Demo Overview 

First I will create a Cluster in EKS where I am going to deploy my Microservices application. And then I will deploy Prometheus monitoring stack that will monitor my Cluster and the Micros Application 

#### Create EKS Cluster 

I will use `Terraform` to create my EKS Cluster (https://github.com/ManhTrinhNguyen/Terraform-Exercise). With 2 Nodes

#### Deploy Micros Services Applications 

I will Deploy my Micorservices 

(https://github.com/ManhTrinhNguyen/Kubernetes-Practice/blob/main/Create-HelmChart-Microservices/charts/microservices/templates/deployments.yaml)

 Now I want to deploy Monitoring Stack that will monitor my Cluster and the Application running inside . 

 So while the applications are starting up in the default namespace, I can now deploy Prometheus monitoring stack using the HelmChart 

#### Deploy Prometheus Stack using Helm 

Docs (https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)

First I have to add Helm Repository : `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts` and `helm repo update`

We will install the whole monitoring stack in its own dedecated namespace . So it is not going to be mixed with our Micros Application . So we have a nice separation between them . 

Create namespace : `kubectl create namespace monitoring`

Install Prometheus Chart : `helm install [RELEASE_NAME] -n monitoring prometheus-community/kube-prometheus-stack`

Now the monitoring stack deployed I can check the Prometheus pod whether they are running `kubectl get all -n monitoring`

I can see a lot of K8s components get deployed . I will go through what applications got deployed that are part of the Prometheus Stack . And what is the Role of each Application in the Cluster so that I know basically what I am dealing with when I have Prometheus stack running in my Cluster 

#### Understanding Prometheus Stack Components 

So we have 2 Stateful Set:

- One of them is Prometheus itself This chained name : `prometheus-monitoring-kube-prometheus-prometheus`  . This is the actual core Prometheus Server based on the main Image

- Other Stateful set is the `AlertManager` which is another part of the stack 

We have 3 Deployments:

- We have `monitoring-kube-prometheus-operator` itself . This is the main one that actually  created `Prometheus` and `Alert Manager` Stateful sets

- And we have `monitoring-grafana` which is its own deployment

- And we have `monitoring-kube-state-metrics` is basically its own Helm Chart so it is a dependency of this Helm Chart that we just installed. So bascially this application what it does is it scrapes Kubernetes components metrics themselves. So it monitor the health of deployments. And stateful sets and pods inside the cluster and makes it available for Prometheus to scrape . Which is good bcs we get the K8 infrastructure monitoring out of the box with this setup

We have ReplicaSets created by Deployment . So it the same for Deployment 

We have 1 `DaemonSet` in this stack of node `exporter`

- First `DaemonSet` is a component which will run on every single Worker Node of Kubernetes .

- Node Exporter component of Prometheus basically what it does is it connect to the Server itself or it lays over the Server and it translates the Server metrics, Worker Node metrics like CPU usage, the load on that server . It transform them into Prometheus metric so they can be scraped  


Those Pods are just coming from the deployments and Stateful Sets

And Each Components has its own Services 

To put it in the big Picture We have set up the monitoring stack itself that will monitor different parts. In addition to that default I already get an out of the box monitoring configuration for my Kubernetes Cluster . So my Worker Nodes and statistics on the Worker Nodes are being monitored . And the Kubernetes components like pods and deployments and replicas, stateful set are also being monitored. And this configuration is there out of the box 

Where does this configuration actually come from ? `kubectl get configmap -n monitoring` . I have the configuration for all the all different parts they're also managed by the Operator . And this include all the information to how Prometheus monitoring stack will connect default metrics, scrape that information and do all this stuff . 

- I have the configuration files and I also the default rule file for Prometheus `prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0`

We also have `Secrets` for Grafana, for Prometheus, again, for different components. This will include the certificates . This will include the Username password for different UI parts and so on

Another thing was also created from this stack ae the CRDs . `kubectl get crd -n monitoring`

- A `CRDs` lets you create your own Kubernetes objects so Kubernetes can understand and manage custom things in your system — not just built-in objects.

- `CRDs` are `Custom Resource Definition` this is another concept in Kubernetes. Prometheus monitoring stack setup includes these custom resoruces definitions.    

#### Components inside Promethetheus Alertmanager Operator

First I will do `kubectl get  statefulset -n monitoring` . Then I will describe it `kubectl describe statefulset prometheus-monitoring-kube-prometheus-prometheus -n monitoring > prom.yaml`

Also I want to see what inside the `Alert Manager`: `kubectl describe statefulset prometheus-monitoring-kube-prometheus-alertmanager -n mornitoring > alert-manager.yaml`

Also I want to see the operator itself : `kubectl describe monitoring-kube-prometheus-operator -n monitoring > operator.yaml`

In `prom.yaml file` :

- We have `Prometheus container` and we have `Config Reloader`

- I can see what Images they based on and what version . I can also see which Port are they running and also Argument

- `Mounts` this is where Prometheus get all the configuration data . For example there is Promethues configuration file . There is a `rules` file and some `certs` . All this is mounted into Prometheus Pod so that it has access to it

  - `Configuration file` is where Prometheus defines what endpoints it should scrape so it has all the addresses of Applications that expose this `/metric` endpoint so it know where to get them from
 
  - `Rules` file basically defines different rules . For example it could be alerting rules that states that when something happen on the server or CPU usage spikes to a certain percentage then send out this email to some people
 
  - So those 2 are seperation configuration files and both of them are mounted into a Prometheus pod 

`Config-reloader` the `Hepler container or Sidecar` of main Prometheus, they help reoloading . 

- Example When configuration changes, these containers are responsible for reloading and letting Prometheus know what there are changes in the configuration

- Config reloader has access to Promethues configuration file `--config-file=/etc/prometheus/config/prometheus.yaml.gz`

- `--reload-url=http://127.0.0.1:9090/-/reload` This is a endpoint of  Prometheus

- `--config-file=/etc/prometheus/config/prometheus.yaml.gz` this is a mount path inside the pod. So each container can access this path

- And the `config reloader` also manages the `rule file `. So it has access to where the rule file is located 

Where does the configuration come from ? Where does the rule file come from ?

- That is also part of the stack . Out of the box I have the default Prometheus configuration file with default configuration and I get the default rules . And these are created as `config maps`

- Get the list of config map `kubectl get configmap -n monitoring`

- I can easily find that for example for the config file I have the `mount` path `--config-file=/etc/prometheus/config/prometheus.yaml.gz` and if I go to the `Mounts` I will see the `/etc/prometheus/config from config (rw)`

- The way the `Mount` work is I mount a volume inside the pod then the individual containers can actually mount those volumes inside the container path so they can access those mounts of the pods

- I will check the `Volumes` and this is a volume will mount inside the pod . I can see the `Config` one with Secret Type and the name of the Secret

  - To check to secret `kubectl get secret <name-of-secret> -o yaml > secret.yaml` I can see the `prometheus.yaml.gz` file encoded

- I can also see where rule file coming from .

  - `prometheus-prometheus-kube-prometheus-prometheus-rulefiles-0` this is a name of the rule file with type Configmap
 
  - `kubectl get configmap configmap-name -o yaml > configmap.yaml` to get the content of configmap 

Then we also have the `AlertManager` config file . We have all the configuration file the same as `Prom.yaml` file   

Then we also have an Operator which is included as part of the entire Prometheus stack which is the name of the container `kube-prometheus-stack` . 

- This is basically the orchestrator of whole Prometheus monitoring stack . This manages all the different moving part of that stacks. It loads all this configuration stuff and it orchestarte how everything work

- All these things are interconnected with each other   

Important to understand is add or adjust the alert rules or alert configuration . And second one is how to adjust the Prometheus configuration so I can add new endpoints for example for scraping 

## Data Visualization with Prometheus UI

#### Decide what we want to monitor 

With all the components, what do we want to achieve with it ? 

- Whenever something strange or unexpected happens in a cluster, either in the cluster components or applications themselves, so basically we want to observe any anomalies which will be an indicator that something is wrong in our cluster

- So when you are configuring monitoring for my application, I have to decide, what do I want to observe, so what anomalies I want to monitor in my cluster . Basically when I have CPU spikes in my Cluster, that would be an indicator that something is happening with one of the applications, so I need to react and fix that issue . Or maybe cluster is running out of storage . I want to observe whether my application is suddenly getting too much traffic, which could be an indicator that too many users are suddenly aware of my application or maybe my cluster or application is being hacked or maybe our cluster is getting too many unauthenticated requests and in such cases, I may want to react and basically analyze what is going on in the cluster and fix the issue, so my cluster doesn't break

How do I get this information using my monitoring stack that I just deployed in the Cluster 

- We need some visibility of the data that shows exactly there is a CPU spike or my application is getting too many connections too many requests or my cluster is running out of storage

- And we want to see bascially what data we have available from the cluster that we can observe . Like for CPU, RAM, are we monitoring the Cluster Nodes to get this information, to have this information about Cluster accesses, for application traffic, are we monitoring the application itself to have this information about the number of requests it is getting or for application accessibility itself, are we monitoring the Kubernetes components

- Basically what components are we monitoring and what kind of data we are getting from these components . So we need to see all this information somewhere in the human readable way visualized in a UI

#### Prometheus UI

To see my Prometheus UI -> In `Services` component there is  a Prometheus Service itself that is a `Service` that expose Prometheus UI `service/monitoring-kube-prometheus-prometheus` . 

- I will do `portforwarding` bcs this is a internal Service so I can access in localhost : `kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &`

- `&` this sign is to run in the background

What `Targets` is our Prometheus that is running in the cluster actually monitoring ? Where are we getting data from ? 

- Click in `Status` -> Choose `Target` -> I will see the list of components that Prometheus is already actively collecting data from

- If I want to monitor an application like Redis, for example and it is not on the list here (UI) then I would have to add that target to my Prometheus monitoring `target` list so that I start getting data from that

- The stack the we deployed already includes application that let Prometheus scrape data like node exporter, or the cube state metrics

- If we expand this we can also see some information about labels and the state, whether it's up or not, and the metrics endpoints that this application exposes inside the cluster `http://192.168.8.195:9093/metrics` (This is an internal endpoint)

Let's say I have the targets I want to monitor . Now I want to know for a target, do you acutally have the data you want to observe  for that specific target ? So you want to see whether you have the actual metrics like CPU usage, number of requests, etc that Prometheus is gathering from the `Target` 

- And I can actually look up the metrics on this main view of Prometheus UI . Either by just typing in the keyword `cpu`

- Also simply just getting the list of all the metrics that these different targets provide .

- So `metrics` is always one piece of data about a specific `target`, like number of total requests that the API server is getting, or how many container processes I have running, etc ...

- If I type in `CPU` I will have CPU information for each instance or node itself in the cluster, or maybe per namespace and so on . This is more of the low level view of your data and you can actually use it for debugging or troubleshooting if you are not getting the `metrics` you want or maybe the target is not accessible etc ...

- I can also see Prometheus Configuration in `Status -> Configuration` as well as Runtime and Build in `Status -> Runtime and Build` information about my Prometheus application

- Prometheus configuration is the concept of jobs . If we look inside the configuration file of Prometheus, we see that in the `scrape_config` we have a list of jobs . So we have a job with name and the following configuration and so on

- Job concept in Prometheus : If I go to `Targets` and I expand one of the `Target` that has 2 processes running, so API Server for example , we have something called `endpoint`. In Prometheus terms an `endpoint` you can scrape are called `Instances` . And the collection of those instance basically scrapes the same application in this case API Server, is called `Job`

  - Intance = An endpoint you can scrape .
    
  - Job = Collection of Instances with the same purpose
  
  - Each such `endpoint` metrics has `labels` that automatically get edit and injected into the metrics and one of those labels is a job name . So with Job name Prometheus knows how to group them together. So we have job API server and here as well, job API server . So those 2 belong to the same job

- I will copy the `monitoring-kube-prometheus-apiserver` and paste in the configuration . I will see the name of the job for the API server . And that means whenever we are searching for `metrics` and let's do  `apiserver_request_total` and execute in every metric you will see lots of `labels` and one of them always be a job name that acutally scaped this `metric` and I can also filter each metric by its job name  . And I can also filter each metric by its job name `apiserver_request_total{job="apiserver"}` . Basically if the same metric was scraped by multiple different jobs , You can actually filter that metrics results for specific job .

  - And in the addition to job `label`, every metric will also have an instance label representing the endpoint or the instance from which that metric was actually taken or scraped

  - I can filter for specific endpoint `apiserver_request_total{instance="192.168.153.237:443"}` to get the result of this specific endpoint

Prometheus UI can be very helpful in debugging, filter the metrics seeing the targets and jobs as well as the whole Prometheus configuration . However Prometheus UI is not an ideal tool for visualizing the data to see the actual anomalies, like to see how your CPU usage is over time or how much the or how much the number of requests my application is actually changing over time for that we need Grafana

## Introduction to Grafana UI

Grafana can access all the `metrics` that Prometheus have and then let us see them in nicely visualize way with diagrams, graphs, tables, etc ...  

I will do `kubectl port-forwarding service/monitoring-grafana -n monitoring 8080:80 &` to access Grafana UI in my localhost

The helm chart of Prometheus stack is configured with login credentials . 

- username : admin

- password: prom-operator

In Grafana I basically work with Dashboard . 

Dashboard is a page of visualizing your data and I have the list of dashboards 

If I go to Dashboard I will see a lists of Dashboard in Grafana which was configured by default in this helm chart that we deployed and the dashboards are actually grouped together or organized in folders

Now we have 1 folder which is `General` folder . We can also create new one 

And inside that folder I have bunch of dashboards, I can also create a new Dashboard insde an existing folder 

Lets look at Kubernetes / Compute Resources / Cluster Dashboard 

To go through how the dashboards are actually made up and I can create all of these myself as well, on a dashboard view I can have multiple rows, so if I add 2 rows here, I can see that a row is one grouping of different data . I can then group the data into each row based on what the data is, so we can have one row for CPU data another row for memory data and so on 

We can organize the data to be part of these rows . For examole I create to be used for `metrics` about memory I can move any relevant data here into `memory` row so that it's group together correctly

These diffent pieces of information here that we can display on their own or in rows are called `panels` 

`Panels` can be percentage values displayed, Or it could be a Graph, or it could be a table with multiple columns and different values 

In most cases for the common scenearios such as CPU and storage memory monitoring we have these out-of-box dashboard that you can use in a ready form 

Structure Summary : We have Folder that have multiple dashboard, the Dashboard is made up of rows which one or more `panels` inside . So that's how the structure of data can look like

We see that CPU and memory usages are are display in graphs where I see per `namespace` how much CPU or Memory  is used the Cluster . 

I also have table view tell me per `namespace` how many pods, the CPU usage per namespace or memory usage per `namepsace` 

When we observe these values, what we are looing for are anomalies, when there are spikes or out of ordinary behavior instead of this consistent average flow. 

- If I see sudden CPU or memory spikes here on these graphs, that would be an indicator that something werid is happening in the cluster and you might want to take a look at this . If spikes continues for a couples of seconds of minutes, then I would have to take some action to fix that issue in the Cluster. Otherwise the Application may not work anymore and my application may not be available to users

Let's say in this Cluster overall CPU and RAM usage view, I see that we have a spike in the CPU something is happening in the cluster . I want to drill down to what node and which Pod or which application specifically is responsible for that spike . So I have other dashboard as well where I can see more detailed information 

If I switch back to the list of dashboard and go inside compute resources node with the pod breakdown . We have multiple `panels` for CPU and Memory . 

- Bcs of the Spike we can see here a breakdown to different pods and how much CPU they are consuming or using in a table view or graph view here for specific node. We can switch to another Node bcs different pods are running on different Nodes

- This could help me really identify which application or which pod is causing that load on the cluster

- On the top I also have the time frame collection . By default I always see the data for the last hour but let's say if the anomaly happned overnight, I obviously want to go back in time and see the graph of CPU usage or number of requests or whatever in that time frame

- I can also select at specific time

- If I click inside in one of those panels and click edit, I will see a view with a PromQL, query in the background . This is basically the PromQL language . Which is basically a query language for Prometheus . And using PromQL, I can get data from Prometheus and then Grafana will use that data to visualize it in any form that I want like table or graph etc ...

#### Create my own Dashboard

If I want to create my own dashboard , then I will have to write PromQL queries here or alternatively, I can use this metrics browser which lets me select these metrics, as well as respective labels for that metrics and basically execute the query 

Alternatively I can pick any metrics from the list, which is the same as we saw in Prometheus UI 

It require me to basically know what metrics, labels to select, etc ...  


#### Resource Consumption of Cluster Nodes

Another dashboard that gives us a high level overview of resource consumption in the Cluster for each Cluster Node and that's going to be `Node Exporter / Nodes`.
 
This I can also see the CPU and Memory usage, Storage and network usage in the Cluster per node 

If every is a straight line it mean there are no anomalies here, everything just average . 

Different metrics information will be relevant for different IT teams, so for example someone who is operating the cluster or managing the servers underlying infrastructure of the cluster maybe interested more in the networking information and someone who is operating  the Kubernetes cluster itself and taking care of the applications inisde will be more interested into information about the workload inside the cluster or information about the individual about pod and application 

#### Test Anomoly

I will produce some anomaly in the cluster . We will trigger a CPU spikes so that we can see how it's visualized or displayed in the dashboard in Grafana and how we can then pingpoint that spike or where it's coming from exactly and we will that pretty easily by just executing a simple script that hits the endpoint of our online shop in a loop, so bascially 10000 requests to my online shop application and as a result we should see some spikes in the cpu usage 

I will deploy simple pod in my cluster, an image that executes curl command, so this will simulate an application inside the cluster making a bunch of requests to another application inside the cluster, so we will see the total load on the CPU . `kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm`

- `--image=radial/busyboxplus:curl`: This image can be use for using curl

- `-i` : In teractive mode

- `--tty`: Allocates a TTY (terminal interface) — makes your session feel like a normal shell

- ` --rm`: remove the pod when we done

The command above will give me a terminal inside that pod : 

- Now I am inside the busy box I will execute the simple script that will hit  the endpoint of our application 10000 times

- To create a script : `vi curl-test.sh`

```
for i in $(seq 1 10000)
do
  curl <my-endpoint> > test.txt
done
```

- `> test.txt` to avoid literring our command line interface with a bunch of outputs of html . So I pipe that whole thing into text file

- Make it executable `chmod +x curl-test.sh`

- Once my script curl endpoint is  done we see this CPU spike on the graph that basically lasted for about five minitues, the memory usage didn't change much during that time and if we go to another dashboard with the pod breakdown and I can see some of the different pods that have seen using the allocated resources on the graph here the catalog service and redis cart have spiked . So I can take a look at the break down per pod and see which individual pods are consuming CPU resources, like recommendation service and the Prometheus monitoring pod

- If select other Node the I will see the cart service, the front end pod and the curl test pod which was the one generating all the request and we have the spike as well

- Bottom line here is that I don't have to become an expert in reading all the graphs and analyzing what each one of them means, the goal is to decide what exactly I want to monitor in the Cluster and basically identify the behavior that I want to observe for anomalies, in this case like CPU spikes, Ram spikes, storage space remaining and so on  

#### Configure Users and Data Sources 

2 more things about Grafana 

First I have multiple team members who need access to the Grafana dashboards as an admin I can configure users in Grafana in the configuration view or server admin view or server admin view as well as teams and their access to the dashboards

And the second thing is how Grafana gets the data from Prometheus bcs we haven't configured Prometheus endpoint for Grafana, so how does Grafana know how to talk to Prometheus or on which endpoint it should talk to Prometheus 

 - That is configure on `Connections` and `Data Sources`

`Data Soruces` which have been automatically configured in this Helm Chart, so we have the default Prometheus source that points to the service name or the internal service endpoint of Prometheus server in the Cluster on port 9090

And then we have another source for the Alert Manager which is running on port 9093

Basically in the dashboards I can only visualize the data from the data sources that I have configured  and I can add new data sources to my Grafana, so I can basically visualize  data from different or multiple data . And Prometheus is just one of the many data soruce types that Grafana support 

We have different time series database , we also have loggin and tracing services as well as SQL database and cloud services . As well as SQL Databae and Cloud Services . Basically all of these databases or data services can be configured with Grafana . So I can fetch the data from the mand visualize them in dashboard in Grafana 

Grafana is actually a general purpose data visualization tool and Prometheus is just one of the many data sources that it works with and also based on the data source that I have configured for Grafana and that I am using to build my dashboard, if we go to explore, the queries will be totally different 

- From here if I had multiple data soruce I could choose one of them and based on which one I select here the query will look different

- PromQL is obviously a query language from Prometheus, if it was a MySQL or PostgreSQL database I have have a SQL query here

- For elastic data source I will have something specific for elastic search and so on 

## Alert Rules in Prometheus

Now we have dashboards in Grafana that shows us the animalies and lets us visualize the data that we are interested in, but in reality, me and my team members who are resposible for monitoring the cluster resources and that everything bascially run properly inside, will not be sitting in front of the screen and waiting for any anomaly in my data to happen 

Ideally when something happens in my Cluster some weird behavior or something suspicious, I want to be notified immediately per email or a Slack message or some other way that something happened that need my attention 

This part of monitoring, we will configure our monitoring stack to basically notify us whenever something unexpected or not desirable happens in the cluster (Notify CPU usage more than 50%, Pod can not start, or Application isn't available in the cluster)

1. Define what we want to be notified about (Alert Rules in Prometheus Server).

- Send notification - When CPU usage more than 50% , or When Pod cannot restart

2. Send notification (Configure Alert Manager) to an Email for that we will configure an application, which is part of Prometheus stacked called Alert Manager

Alert Manager will be the one will send out an alert describing what actually happended in the Cluster and to take some action 

Based on the Grafana dashboard we, we saw an average CPU usage in our cluster per Node . So we know what a normal usage for our service looks like from the dashboard 

Based on the data we can decide, normally CPU usage in our Cluster is pretty low so if at any point it exceeds 50%, we know something weird is going on in the cluster 

We will configure first Alert Rule that says alert when CPU usage over 50% . How do we actually create an alert rule or how does it acutally look like ? 

- By default when we install a Prometheus stack, we actually get some alert rules already out of the box

#### Existing Alert Rules 

This is Alert Rules that Prometheus created for us 

Back to Prometheus UI -> Click on Alert we will see a list of already configured alerts that we already have from this stack and they are grouped by the name. So we have `alermanager.rules` group . We have `config-reloaders` group and `etcd` group and so on ... And in each groups we have alert rules 

And most of these rules that I see are actually for Prometheus stack applications themselves . So we have a bunch of alert rules for the alert manager application. So whenever alert manager application fails to reload or it fails to send alers to configured notification channel or if the alert manager configuration is inconsistent, etc 

And then we have a bunch of rules for `etcd` which is a `Control Plane` service in Kubernetes . 

We also have alert rules, which are actually pretty helpful for us, which are grouped under `kubernetes- apps`. Basically these are the rules for Kubnernetes resources like pods, deploymentsm stateful set, etc ... 

- For example we have `KubePodCrashLooping` or `KubePodNotReady` rules . Bascially if that happens with any of the pod this alert will be triggered

- If stateful set replicas do not match . We have 3 Stateful Replicas configured, but there is only 2 replica pods running this alert will be trigger.

However, I see that most of these alerts are green and that means alerts are inactive or these conditions haven't been met in the Cluster . So nothing or no alerts gets fired  . So these alers are not happening or are not being fired in the Cluster that's what `Green Status` mean .

Whenever alert is `red`, it's in a firing state, which means that whatever that alert rule alert about has happended in the Cluster . `KubeSchedulerDown` and that's why rthe alert is firing 

`Firing` mean alert has happened or whatever the alert rule describe has happened And Prometheus basically fired that alert towards the Alert Manager so that Alert toward the `Alert Manager` so that Alert Manager can notify us and alert us about what has happened 

How does a syntax of alert rule configuration actually look like ? 

If I expend any of the alert rules, I will see the configuration syntac for that specific alert rule (This is the main configuration) 

- `name` Which is a descriptive of what we want to be alerted about

- `expr` The main part of any aler rule which is the `expression` .

  - `expr: max_over_time(alermanager_config_last_reload_successful{job="monitoring-kube-prometheus-alertmanager", namespace="monitoring"}[5m]) == 0`

  - If we know from programming languages, we have these logical expressions like if this and this equals whatever, then and have a logic there . That is basicaclly the same concept
 
  - The `expression` tells us if `Alert Manager` failed to reload, and then fire the Alert . And the `expression` here is defined in PromQL . This is actually a Prometheus query that expresses the logic of alert Manager failed to reload or alert manager failed to send alerts
 
  - PromQL is not a very intuitive or easy query language . Just need to understand the basic will obviously help to at least be able to read existing expressions if I am not able to create them
 
  - Break it down, we always have a metric inside an expression alert manager config last reload successful . that's a metric which is available for us `alermanager_config_last_reload_successful` in Prometheus UI . And then we have the `Filter` on that metric `{job="monitoring-kube-prometheus-alertmanager", namespace="monitoring"}` also in the same page . Filtering on these two attribute or two labels, since as we know each metric has a bunch of `labels` which are are highlighted . We just have one instance here but if we have 100 instance with these `labels`, but different values then we could basically filter them using this label key value pair
 
    - Then on that filtered `Metric`, we apply a function, a Prometheus function that we have available from Prometheus, which you can always look up in the Documentaiton (https://prometheus.io/docs/prometheus/latest/querying/functions/)
   
    - `max_over_time()` this tell us that we get the maximum value from the returned result max over time . And bcs we are asking for maximum value over time, we also have to specify that time that will be `[5m]` 5 minutes .
   
    - Once I exected this will give us a value of `1` . So in 5 mins of times, we are getting the maximum value of this metric or alert manager in the `namespace=monitoring`
   
    - The value is `1` that mean we have a successful relaod if it's   `0` then we don't have the successful reload in that recent time and that is `== 0`
   
    - If we don't have `1` but `0` it means alert manager was not able to successfully reload . So Alert Manager failed to reload and that will be an alert
   
    - But if we execute this filter here, we see empty query results, which means this did not happen in a cluster . Alert manager hasn't failted to reload that is why the alert is green

    - That's basically how each expression is put together . So we start, we always start with a basic metric like `alertmanager_notification_failed_total` etc... and  then we do Filter on it and function around it
   
    - Let's take the `alertmanager_notification_failed_total` metric and rebuild it . We have a couple of results  where each result has a different notification channel `integration` could be email, slack, discord .... or to my own custom endpoint `integration` . And we see that the values are `0`, so we don't have any failed notification . Again we have a filter here for `job` nmae and `namespace=monitoring` which i  basically the same for all of these entry
   
    - IF I copy this and add a filter `alertmanager_notifications_failed_total{job="monitoring-kube-prometheus-alertmanager", namespace="monitoring"}[5m]` and execute we should have the same result, bcs each metric entry has these `labels` with these values . With this `exp:(rate(alertmanager_notifications_failed_total{job="monitoring-kube-prometheus-alertmanager", namespace="monitoring"}[5m]))`is calculates an average rate of failed notifications for `alert manager` and comparts it to the total amount of notifications that alert manager send to basically calculate how many or how much `%` of the notification or alert have failed if this `> 1% or 0.01` then it gonna `fire` . It say hey we have too many fail notification here . We should notify someone
   
Another the question is we want to create lots of different alert rules, but some of them will be more important than others . So if the application is not  accessible that could be much more urgent to fix thatn if a request to an application is slow . For that in a Alert Rule we have a `critical` alert or just a `warning`

```
labels:
  serverity: critical 
```

- Each Alert rule will have a `serverity` labels .

- However we can add additional labels to each alert rule besides the `severity` labels to be able to identify and group together different alert rules in alert manager when we send it out

- Why would need to group together different rules by labels . We could have a use case where we want to send all the rules that are critical to select channel and all the alerts that are warning to email or we may add labels to alert rules that basically says which application it belong to, or which namespace it belongs to etc . And then say all the alerts that belong to `namespace=<something>` should be sent to this select channel, or all the alerts that have label with this application should be sent to this specific webhook URL . That's what label are for to target alerts rules specifically

Now when an Alert happens, Another thing we need to decide on is do we actually send the alert righ away, or do we wait for sometime to see maybe the thing that the Alert Rule is warining about basically solves itself 

- For example `AlertmanagerFailedReload` happen that alert manager application dies and it cannot reload . However because Kubernetes has all these self-healing, automatically restarting features, it could be that within next couple of minutesm the application manages to reload without our manual interaction .

- Basically when the alert happen, put it in the pending state but i am going to give you sometime to see whether cluster can heal itself and bascially solve this condition by itself . And we can define the time that we give to the alerts with `for: 10m` attribute . So for 10 mins we will see whether this condition resolves itself . If it hasn't resolved after 10 minutes, then fire an alert, bcs it means that cluster was not able to solve this issue by itself we need some manual interaction . Depending on Alert rule and what exactly is happening, we can set this value to 10 minnutes or 5 minutes or even lower . So that's what the `for` label for 

Finally when the alert acutally gets fired, so it's in firing state and it get sent to our email or Slack channel, What will the Alert actually say . It has to deliver a message that say . Hey this and this has happended, that's why I am alerting you . So that message contents are configured like this 

```
annotaions:
  description: Configuration has failed to load for {{$labels.namespace}}/{{$labels.pod}}
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload
  summary: Reloading an Alertmanager configurtion has failed 
```

- Where is `{{$labels.namespace}}/{{$labels.pod}}` come from ? We saw that each Alert `expression` is based on `metric` . The `metric` itself has a bunch of `labels`, including the container name, the pod name, namesapce and service event . We can access all this information in our alert rules

- `runbook_url` is basically points to a URL that explain the error or the issue that we are alerted about, as well as possible fix for this issue . (https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload) this one point to a Github repository for Kubernetes alert runbooks

- `summary`: summarize what iusse we are alerted about 

That basically is how alert look like. This will be a structure for any alert rule if I look through them, each one has the same attributes

## Create own Alert Rules

First I will be when the CPU usage exeeds 50% on the Kubernetes node, and the second one will be  when a pod cannot start (When we have apod not in a running state but Crashbackloop)

#### First Rule 

In the `/monitroing` folder . I will use this folder to create all my monitoring related configuration files . So that's where we will create our alert rule 

Create a file call `touch alert-rule.yaml` . And the structure we will going to have is going to be basically the same attribute , just change the value 

```
name: HostHighCPULoad
expr: 100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100) > 50
for: 2m
labels:
  severity: warning
  namespace: monitoring
annotations:
  description: "CPU load on Host is over 50%\n Value = {{ $value }}\n Instance = {{ $instance }}"
  runbook_url:
  summary: "CPU load on Host is over 50%"
```

- `expr: 100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100) > 50` . We have this metric `node_cpu_seconds_total` that we are starting with. It is a node CPU seconds total with a bunch of output where the difference is mainly in the `mode` . So we have different `mode` for our `node_cpu_seconds_total` and one of them is `idle` This mean the CPU is not being used . Bascially with `node_cpu_seconds_total{mode="idle"}` we are saying how much of the CPU is `idle`. The more CPU is `idle` the less we are using. However for that value, we want to acutally average per second over a period of 2 minutes `rate(node_cpu_seconds_total{mode='idle'}[2m])` this will give me a rate, so not the actual value of the `idle` CPU, the average rate over 2 mins intervals . And we have values for each `instance`, Each `instance` has 2 CPU or 2 Core . And bcs we want to measure the CPU usage per host, we have to group them by instance . We have to group them by `instance` . basically we want to see what is the average `idle` time of CPU per node by calculaitng average for both CPU cores for the same `instance`

  - `(avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m])` so for each `instance`, so this is one instance with 2 CPU cores, the average of these 2 values will be calculated, And for the second instance, average of these 2 value
 
  - Once I execute this, I will see the average for each instance of being `idle`
 
  - And to get the percentage value of average we will multiple this by 100 `(avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100)` . Then I will have an average `idle` time of the node, whihc mean our nodes are underutilized, basically `idle` most of the time . From this value we will get the acutual usage or utilization per instance . So total CPU usage is 100% and we basically will deduct this value from 100 and that will be it `100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100)`
 
  - Both instances are utilized by less than 10% on average . This is just a value . Now we need to acutally add logic to this expression to say, if this value is over 50% then fire an alert `100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100) > 50`

- Now When do we acutally trigger alert . Do we wait some time to let the CPU usage go down or do we trigger it right away

  - Let's say we decide, 50% isn't that high, so we can give it 2 mins to see whether it decreases again and goes below 50% . If it doesn't for 2 full minutes `for: 2m`, then we will trigger an alert
 
- If it only 50% I will give this a severity of warning `labels: severity: warning`

- Finally we have our message contents with description and summary

   - `summary: "CPU load on Host is over 50% "` : We can even grap the acutal value of what the CPU usage is bcs it could be 60%, or it could be 80% . Maybe want to notify our team meber exactly what that CPU load is, for that we can do `summary: "CPU load on Host is over 50%\n Value = {{ $value }}\n"` . This `value` variable available for us
 
   - For our alert we can remove the `runbook_url`. If we have a `runbook_url` whoch describes the issue and possible fix for that we can add that as well . Good practice to inlucde `runbook_url`
 
  - What we could also add in the message is the acutal instance that is affected, so we can say that's the value for isntance `description: "CPU load on Host is over 50%\n Value = {{ $value }}\n Instance = {{ $instance }}"`
 
- We can add addition labels to our alert rules, so i will add the label `namespace: monitoring` . Which we going to use later to target this specific alert rule inside the alert manager configuration 

## Alert Rule for Kubernetes 

How to configure Alert Rule in Prometheus 

A context or background how these works or how does Prometheus know about these Alert Rules, we can go to Prometheus UI -> Status -> Configuration -> Right under `rule_files:` we have a actual rule file `etc/prometheus/rules/prometheus*/*.yaml`. This is where all the Alert Rule are defined 

However we don't have to go there and adjust the `Configmap` or `Secret` or whatever this is and add our alert rule there and reload Prometheus so it picks up our new Alert Rule . We don't have to do that bcs we have Prometheus running in Kubernetes Cluster as a `Prometheus Operator` .

- `Prometheus Operator` let us create custom Kubernetes components which are defined by `CRDs`, to create alert rules, so that `Operator` will then go to Prometheus and say there is a new Alert Rule that you need to pick up or load in and add it to your configured list of Alert Rule and this means that if we didn't have Prometheus running inside Kubernetes Cluster using this Prometheus Operator, we would actually have to go to the Prometheus configuration file and whatever the rule files points to and actually add our rule in this exact structure to that file and then reload Prometheus

- `Prometheus Operator` extand Kubernetes API and let us create custom Kubernetes Resources which basically look like any other Kubernetes configuration file and in the background, Operator will then take our custom resource with the Alert Rule that we configured and tell Prometheus, go ahead and reload the configuration bcs there is a new alert rule (All of that happen in the background), which is a big conveience for us bcs we don't have to learn K8 configuration syntax and do stuff inside  


Let's see how that custom Kubernetes resource for creating Alert rule actually looks like: 

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
spec:
  groups:
  - name: main.rules
    rules:
    - alert: HostHighCpuLoad
      expr: 100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100) > 50
      for: 2m
      labels:
        severity: warning
        namespace: monitoring
      annotations:
        description: "CPU load on Host is over 50%\n Value = {{ $value }}\n Instance = {{ $instance }}"
        runbook_url:
        summary: "CPU load on Host is over 50%"
```

Bcs this is a API from Prometheus Operator, that extend Kubernetes API so this will be `apiVersion: monitoring.coreos.com/v1`

The resource, the custom resource name for Alert Rule in Prometheus stack is called `kind: PrometheusRule`

Then we will have `metadata` with `name: main-rules` and `namespace: monitoring` 

`Spec` Specification is different for each Kubernetes resource, whether it is a native resource like Pod, Deployment, Service or a custom resource (How to know what Attributes this custom Prometheus rule expects from us ? Where to see ?)(https://docs.redhat.com/en/documentation/openshift_container_platform/4.13/html/monitoring_apis/prometheusrule-monitoring-coreos-com-v1))

- In the docs I see the `spec` has `group` attribute

```
groups:
  - 
```

- This is a syntax for array in Yaml

- Inside `.spec.groups` we have `name` of the group . The group will look somehow like this `alertmanager.rules` . We have the way to group our Alert Rule logically

- Another attribute required is `rules` . This is also an array . All attribute I can see in the docs

  - I will take the Alert Rule that I created above And add that in

So we have our Prometheus rule, custom resource that we can create in Kubernetes just like we create any other Kubernetes resources and this is a way more convenient way of creating alert rules and adding them to Prometheus rather than working this config map and secret configuration, This gives us a really flexibility of creating and then removing the Alert rules without ever touching the Prometheus configuration file

!!! NOTE: If you go and change the rules file manually which you could actually do , you can grap the value here and you can adjust it and save it back . Prometheus operator will automatically rewrite the changes and basically remove what I have added it, it will automatically reload the rule file so my changes will be gone, so it means that we can't even adjust this manually even if we wanted to with Prometheus Operator, which is a good thing that it doesn't acutally allow from manual configuration changes 

#### Write 2nd Alert Rule 

 We can add another the one in the same Prometheus rule definition bcs as we saw rules is an array so it lets us create multiple rules with one resource 

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
  - name: main.rules
    rules:
    - alert: HostHighCpuLoad
      expr: 100 - (avg by(instance)) (rate(node_cpu_seconds_total{mode="idle"}[2m]) * 100) > 50
      for: 2m
      labels:
        severity: warning
        namespace: monitoring
      annotations:
        description: "CPU load on Host is over 50%\n Value = {{ $value }}\n Instance = {{ $instance }}"
        runbook_url:
        summary: "CPU load on Host is over 50%"
     - alert: KubernetesPodCrashLooping
       expr: kube_pod_container_status_restarts_total > 5
       for: 0m
       labels:
         severity: critical
         namespace: monitoring
       annotations:
          description: "Pod {{ $labels.pod }} is crash looping\n Value = {{ $value }}"
          summary: "Kubernetes pod crash looping"
```

I will create another `alert` and this alert for alerting whenever pod cannot be started or when a pod is in a crash looping status 

Going back to Prometheus UI we have a metric called `kube_pod_container_status_restarts_total` this basically tell us how many times a container has restarted with this value . 

- Whenever we have a pod that has a container inside that basically keeps restarting on and on and that number or total number of restarts exceeds five `kube_pod_container_status_restarts_total > 5` then we want to send an Alert saying hey there is a container thet keep restarting bcs there is some problem with the application and it cannot start  `expr: kube_pod_container_status_restarts_total > 5`

Let's say we want to trigger it right away when this happen `for: 0m`

I will add some `labels` bcs this super critical we can add `severity: critical` bcs we can not afford application downtime 

Then I can have description to describe a bit more detail . 

- We can even send the name of the pod that has this issue by grabbing the value of the pod label which we in the UI . For each cube pod container status restart total metric we can access it using labels `{{ $labels.pod }}` . We can send the value of how many time it has restarted already . And that's going to be the value `{{ $value }}`

- `summary` which is like title of the issue when it arrives in my email or slack channel or whatever notification channel 

With that we have configuration ready to add new alert rules . However for Prometheus Operatir to automatically pick up our Prometheus rule we need to add some labels in the `metadata` level. Using these `labels` Prometheus operator will be able to identify the Prometheus rules that we want to add to Prometheus and basically pick them up automatically and Prometheus will try to match any Prometheus rule resources that have these `labels` to automatically pick them up

- As you already know from the Kubernetes module one of the main usage of `labels` is to identify or to label some of the resources so other Kubernetes resources can match those `labels`

- The same concept here we are labeling our Prometheus rule and then Prometheus will match those 2 labels and know okay this is a alert rule then I need to pick up and basically add it to my configuration 

#### Apply Alert Rules 

To apply the alert rule file `kubectl apply -f alert-rule.yaml` . We don't need to specify `namespace` bcs we already defined in YAML file 

Now I do `kubectl get PrometheusRule -n monitoring` I should see a bunch of Prometheus rule  resources in a Cluster which actually correspond to this  out of the box alert rule in the UI like `alertmanager.rules`

And on top of that we have `main-rules` is the one that we just created . This is a acutal resource that I can get edit and so on inside the Cluster 

Now since we created this Prometheus rule we acutally expect that Prometheus operator which is actively listening for any new Reosurces like Prometheus rule or whatever is on the list, Created inside the Cluster to automatically pickup our new rules and tell Prometheus to reload the configuration and add these alert rules to its list and that mean we should actually be able to see our alert rules . It may take some time but enventually they will appear on this list . 

There is also a way to debug that Prometheus reloaded the configuration with new rules to make sure that operator picked up or addition and basically told Prometheus to do that and we can do that by checking logs of Prometheus pod and this will be very interesting place to troubeshoot or check to make sure configuration is reloaded properly so if I do `kubectl get pod -n monitoring` I have the Prometheus monitoring pod here with 2 containers and as you remember we have the Prometheus application itself and we have this helper or sidecar container that basically its only purpose is to reload the configuration automatically whenever these kind of changes happen 

If I do `kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring` and we gonna have 2 container I have to specify which containers logs i am interested in `kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader` and check the logs of config-reloader to see whether this application actually reloaded the configuration

- `-c` is for container

- And I can see today the reload was triggerd at the date of today

- The Prometheus rule file which is mounted inside the container was basically reloaded that means that Prometheus should actually have our Alert Rules we can also check the logs of the Prometheus itself `kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c prometheus` for today and I can see the msg completing loading of configuration file

So in the Prometheus pod container logs we saw that configuration was reloaded successfully since our Prometheus rule resource was matched on those labels and now we should actually be able to see those 2 rules that we added in Prometheus UI alerts . 

- Now I can see `main.rules` group which is the name of the group And we have `HostHighCpuLoad` and `KubernetesPodCrashLooping`. If I expand this I can see the configuration

So whenever these condigtion are met which means either we have a high CPU load over 50% or we have a pod in a Cluster that is crashlooping the alert  the alert will be fired 

#### Test Alert Rule 

Now I will trigger the `HostHighCpuLoad` rules to see that my Alert actually get fired or get into the firing state 

I will simulate a CPU load in my Cluster to trigger this alert . I will do that by deploying a pod in our Cluster that has an Application inside that simulates CPU loads 

Before we do that I will update our dashboard so that we can see the CPU utilization as a `%` to make it easier to see when the alert is going to be triggered 

Now I will go to DockerHub to get a `cpustress` image . It simulates a CPU stress inside a Docker container (https://hub.docker.com/r/containerstack/cpustress)

We will deploy this container in a pod in my Cluster that will generate a CPU stress, which should be over 50% . 

I have the Docker run command `docker run -it --name cpustress --rm containerstack/cpustress --cpu 4 --timeout 30s --metrics-brief`. I will translate this command into a kubectl command bcs I want to create a Kubernetes pod . `kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 30s --metrics-brief`

- ` --cpu 4 --timeout 30s --metrics-brief` These are options to this application which is running inside that container

- To pass any parameter or options in `kubectl` to an application which will run inside the pod . `--` whatever come after `--` is then being interpreted as options or parameters for the application inisde the container

- This example of taking Docker command and translate that to a kubectl run command can actually be very helpful whenever I want to quickly deploy some test or debugging or troubleshooting application as a Kubernetes pod inside cluster 

Once I executed that command this will create a pod in default namespace . And soom we will strt seeing the CPU load increasing and will get over 50% 

Now going back to alert rules I see that the `HostHighCpu` load is pending, which mean that the aler condition was actually met and I see that value is over 50% . So that's the CPU load on one of the instances 

Now it is waiting for 2 minutes in a pending state until it actually fires the alert . So if this problem doesn't come below 50% within 2 minutes, the alert will become firing 

## Alert Manager 

#### Firing State

Firing State mean . What happended is that Prometheus actually detected, there is high CPU load in the Cluster, so I will fire an alert bcs this condition is met , so Prometheus acutally fired or sent the alert to Alert Manager that what firing state mean 

So Prometheus handed over the alert to Alert Manager and now Alert Manager is the one that needs to acutally send out an alert to an email or messaging service or whatever notification channel I have configured . 

In our case bcs we don't have anything configured in Alert Manager . Prometheus fired an Alert and Alert Manager just ignored it  

Next step I will configure alert manager so that when it acutally gets an alert that is firing, it delivers that message to us through an email  

#### Alertmanager Configuration File 

Alert Manager is its own separate application so Prometheus is one Application, Alert Manager is another application . And that means alert manager has its own configuration configured through a configuration file

Where is Alert Manager configuration and how can we see what currently configured in there just like we see for Prometheus. 

First of all Alert  Manager also has a very simplistic UI which get deployed in the stack . To check on the UI I will do port forwarding `kubectl port-forward service/prometheus-kube-prometheus-alertmanager -n monitoring 9093:9093 &`

It give me a read only view of the configuration as well as a way to filter the Alerts 

If I go to `Status` it give me this will give me configuration and some other information of my Alert Manager 

```
global:
  resolve_timeout: 5m
  http_config:
    follow_redirects: true
    enable_http2: true
  smtp_hello: localhost
  smtp_require_tls: true
  smtp_tls_config:
    insecure_skip_verify: false
  pagerduty_url: https://events.pagerduty.com/v2/enqueue
  opsgenie_api_url: https://api.opsgenie.com/
  wechat_api_url: https://qyapi.weixin.qq.com/cgi-bin/
  victorops_api_url: https://alert.victorops.com/integrations/generic/20131114/alert/
  telegram_api_url: https://api.telegram.org
  webex_api_url: https://webexapis.com/v1/messages
  rocketchat_api_url: https://open.rocket.chat/
route:
  receiver: "null"
  group_by:
  - namespace
  continue: false
  routes:
  - receiver: "null"
    matchers:
    - alertname="Watchdog"
    continue: false
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
inhibit_rules:
- source_matchers:
  - severity="critical"
  target_matchers:
  - severity=~"warning|info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - severity="warning"
  target_matchers:
  - severity="info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - alertname="InfoInhibitor"
  target_matchers:
  - severity="info"
  equal:
  - namespace
- target_matchers:
  - alertname="InfoInhibitor"
receivers:
- name: "null"
templates:
- /etc/alertmanager/config/*.tmpl
```

This is a default configuration that Alert Manager currently is operating on

We have 5 main section in the Alert configuration 

- `global` configuration basically just define global configuration for all the receviers for all the `routes`, that's what this section about so we don't have to repeat the same value. So these are kind of like global vairables that apply or can be used throughout the whole configuration 

- We have `route` .

  - We need to tell alert manager which alerts should be sent to which receviers, bcs we may not want to send everything that comes in to 1 recevier . We may want to ignored some of them while sending alert that are from monitoring namespace to an email and maybe send every alert that is coming from a job Prometheus to WeChat channel etc ... So we need some kind of routing for our Alert inisde the Alert manager configuration so we can flexibly decide what alert goes where and for that we have `route`
 
  - The way we target individual Alerts or Groups of Alerts are using this `matchers` attribute
 
  - `matchers` expression for matching alerts with certain labels
 
  - ```
     - receiver: "null"
      matchers:
      - alertname="Watchdog"
      continue: false
    ```
    
  - This here mean that whenever we have an alert coming from Prometheus inside the alert manager, that has a label called `alertname="Watchdog"` should be sent to `receviver: "null"` which mean should be ignored . This for specifically configuring specifically configuring specific alerts and whatever is defined 
 
  - And then we may have another same section inside the `routes` list that said match any alert that has any label `alertname="test"` that should be send to receiver email .
 
  - And then outsides the `routes` we have the `recevier` again . We have `group_by` and so on . What are these and how do they impact alerts ?
 
    - These are like Global Configuration that apply to any Alert coming in from Prometheus . This Level apply for all the alert
   
  ```
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  ```

  - This configuration bascially just defines how often the notifications should be sent to the receiver and then we have this group attributes, which basically means maybe we don't want to send every alert one by one, maybe we want to group them and bascially send one alert message for a group of alerts to be more efficent

- `inhibit_rules` : are used to make sure we can stop certain alerts from being sent if other alerts are already firing . Let's say we have a critical alert indicating a major outage, if this critical alert is active we might want to inhibit or suppress related less critical alerts that would only add noise .

  - We can configure `source` alert which if triggered will then stop the target alerts that follow from being triggered. We can use these alerts to make sure we don't get overwhelmed during any outages 

- `receivers` . This is a main part of the Alert Configuration . This basically configures alert manager receivers

  - `receivers` are the notification channel or basically where is alert manager sending the alert that are coming from Prometheus and some of the receivers are email, WeChat, PagerDuty, Victor ops
 
  - `null` receiver means that whatever alert comes in from Prometheus will be ignored. That's why even though we were firing an alert from Prometheus, it was being ignored bcs the alert was being sent to the receiver `null`
 
  - Whatever `receiver` we configured here we can then use in `route` section . And here we have aflexibility to define which alert goes to which receiver 

- `templates`


`receivers` is a configuration we need to adjust now and add an email receiver here, as well as configure a `route` here for our 2 alerts and we see that we have a pod crash looping which is our CPU test container which is restarting . So we want these `HostHighCpu` and `KubernetesPodCrashLooping` to be sent to email 

## Configure Alertmanager with Email Receiver

Where is the Alert Manager Configuration come from inside the Cluster? 

In the `alert-manager.yaml` file I generated before . In the `Mounts` section I can see the the Alertmanager config point to  `config-volume` -> `/etc/alertmanager/config from config-volume (ro)` . `config-volume` is a `Secret` with a name is `alertmanager-prometheus-kube-prometheus-alertmanager-generated` (This is where the configuraton come from) 

To check that `kubectl get secret alertmanager-prometheus-kube-prometheus-alertmanager-generated -n monitoring -o yaml | less `

![Screenshot 2025-06-07 at 09 31 26](https://github.com/user-attachments/assets/4e450b0b-2af0-4e7c-ae3f-1ea3805f2237)

This is a zipped or archived version of my `alert-manager.yaml` file and the `base-64 encode content` with it 

To decode and unarchiving the content in `echo H4sIAAAAAAAA/7ySy2r0MAyF934KMcsf5k8vdBOYB5gn6DJoXMUR+JLKcoaB0mcvTtppCg3tqlvpWN+xjpxPJ/StARDKyU/UKQdKRVt4CEZSUVqalngiaWEXi/c7A+AklbE7XWp7DxED5REtVXF9lZf6Nw8BAqodSGZJFaEn0ToBDrB7rM2n5D4ZZ2Rt4f4mXysclWRCP5us7kZCXVVv7wbDceATayfFVzN7UBRH2q3he8g0kbBe4PAKZ5TI0b1w7JMByKmIpU09WGFli94A0HNZlvh1Eauf/cyHX2LfXf4FdZ3LMfbpuKw0yRZ8A7c95+M85oCq4nonSmH0qEt0Dalt5ikBIzqSxqbYs2v| base64 -D | gunzip | less`

- `gunzip` to unarchive to content

- `base64 -D` to decode

- `less` is a command pipeline that helps you view large outputs one screen at a time

Once executed I can see my `alertmanager` configuration 

The same way as for Prometheus Configuration file . This is acutally managed by the `Operator` . We could not adjust this value `H4sIAAAAAAAA/7ySy2r0MAyF934KMcsf5k8vdBOYB5gn6DJoXMUR+JLKcoaB0mcvTtppCg3tqlvpWN+xjpxPJ/StARDKyU/UKQdKRVt4CEZSUVqalngiaWEXi/c7A+AklbE7XWp7DxED5REtVXF9lZf6Nw8BAqodSGZJFaEn0ToBDrB7rM2n5D4ZZ2Rt4f4mXysclWRCP5us7kZCXVVv7wbDceATayfFVzN7UBRH2q3he8g0kbBe4PAKZ5TI0b1w7JMByKmIpU09WGFli94A0HNZlvh1Eauf/cyHX2LfXf4FdZ3LMfbpuKw0yRZ8A7c95+M85oCq4nonSmH0qEt0Dalt5ikBIzqSxqbYs2v` even if we overwrote the contents of the file here and set that inside the `Secret`, `Operator` will automatically reload . And also this acutally looks like  a really tedious and not nice way to adjust configuration 

The same as for Prometheus Alert rules we can actually adjust or create configuration for alertmanager using the custom Kubernetes resources from the `monitoring API`  (https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/monitoring_apis/alertmanagerconfig-monitoring-coreos-com-v1beta1) . This is another custom Component from monitoring API that actually lets us add or adjust alert manager configuration 

#### Condfigure Email Notification 

I will create another file `touch alert-manager-configuration.yaml`

```
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
  namespace: monitoring
  name: main-rules-alert
spec:
```

This part of `alert-manager-configuration` file pretty much the same with the Kubernetes Components . We have `apiVersion`, `kind`, `metadata`, `spec`

For `spec` we will need the docs bcs each kind has its own definition or list of attribute 

First we will need `email recevier`. It is an array of of object . Where we have the name of `receiver` and `emailConfigs[]` 

Let's check the `emailConfigs[]` docs for that . Each Object in the Email list or email configs list will have these attribute . We have `to` and `from`

To be able to send the email from my account, the alertmanager needs to have access to my email account. So we need to give credentials, but also we need to say which email `SMTP` host should be used . Bcs this is a Gmail email it will be `SMTP` and the attribute for that is `smarthost`

We need username and password for our email account . Bcs obviously not everyone can send an email from our account . That will be `authUsername` and `authPassword`. This is a configuration file properly will get check into the Repository and we don't want to hardcode the password 

- Instead I want to reference a secret in Kubernetes that has password value inside

- In the Docs `emailConfigs[].authPassword` we have the way to reference the secret of passowrd . `name` and `key`

- Then I will create a secret yaml file (Never check in the Repository.)

- Very important . We want the secret to be in the same `namespace` as the `alermanager` config otherwise we can not access it 

Also I haveone more attribute which is `AuthIdentity` which is also will be my email address. This is configure a receiver for email address for Gmail in my case where it will send an email 

One Important thing alway consider when sending email using an application . So automated script or application or service accessing my email account and sending email programatically . Most email providers have some kind of security mechanism against this . By defaul the block all this kind of activity. 

- To ensure this email will be send correctly is by using method that is available to me when I have 2 step authentication enabled in my email account, which is bes pratice

- In Gmail specifically I will create and use what's called an `application password` . This `application password` will then be use to access my email account programmatically log into it and send out an email from that email account

- If I do not have 2 steps authentication enabled, I will need to enable it in my account in order for this to be able to work

- Once it is enabled, bcs we are using Gmail, you will need to create an app password . Go to `myaccount.google.com/apppasswords` . From there I can create a dedicated password to be used with Prometheus . This will be a password that we encode and use it in the secret file 

```
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
  namespace: monitoring
  name: main-rules-alert
spec:
  receiver
  - name: 'email'
    emailConfigs:
    - to : 'trinhnguyen511998@gmail'
      from: 'trinhnguyen511998@gmail.com'
      smarthost: smtp.gmail.com:587
      authUsername: 'trinhnguyen511998@gmail.com'
      authIdentity: 'trinhnguyen511998@gmail.com'
      authPassword:
        name: gmail-auth
        key: password
```

----

```
email-secret.yaml

apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: gmail-auth
  namespace: monitoring
data:
  password: base64-encoded-value-of-my-password
```

Continously . Now we have to configure the `route` section 

We will configure the child `routes` using `matchers`

`matchers` are basically a label name and value pairs 

In the section above we create an `alert rules` has the name and name of the alert itself . This we are matching both of our alerts and we can now configure what we want to do with them 

In this case I can decide I want to send both of them to a receiver called an `email` which I have configured above 

If I have a bunch of receivers here you can basically select them by name and say this alert should be routed to that `receiver`

I can also define other attributes that apply for both of the routes like : `repeatInterval: 30m` Alert will be send to an email and if issue hasn't resolved, we will send another one in 30mins 

And depending on how urgent or the issue is or what kind of issue we have, we can adjust this value and basically send it more ofent or less often 

We also can configure these shared attributed or configuration for each alert specifically . for example I can put the `repeatInterval: 10m` in the `matchers` with `value: KubernetesPodCrashLooping` to send this alert every 10mins

```
route:
  receiver: 'email'
  repeateInterval: 30m
  routes:
  - matchers:
    - name: alertname # this is a label they automatically get
      value: HostHighCpuLoad # Name of the alert rule
  - matchers:
    - name: alertname
      value: KubernetesPodCrashLooping
    repeatInterval: 10m 
```

#### Apply the configurations 

To apply `kubectl apply -f email-secret.yaml` then `kubectl apply -f alert-manager-configuration.yaml`

Just like with Prometheus rule, This is a resource in Kubernetes which we can get and describe and execute all these kubectl commands on . `kubectl get alertmanagerconfig -n monitoring`

And the same way as our Prometheus rule was automatically discovered and picked up and reloaded inside the Prometheus application our alert manager configuration should also be automatically picked up and reloaded inside the alert manager application 

We can also check the logs inside the alertmanager application `kubectl logs alertmanager-monitoring-kube-prometes-alermanager-0 -n monitoring -c config-reloader` . If we notice it also has 2 containers One of them is the alert manager application and it has its own config reloader application just like Prometheus  

If we go back to the Prometheus UI, We will see the existing configuration was merged with our configuration 

If we notice the name of the `receiver` isn't `email` as we specified . It acutally monitoring `main rules-alert-config-email` which is in the `metadata` section. The name of the alertmanager config taken as a prefix and then the name we specified 

`send_resolved: false` mean that when the alert happen we will send it to this email address . However we can configure whether we want to send an email when the issue get resolved. Basically saying hey now everything back to normal here is an email notification that you don't have to take any action 

- This could be very useful in some cases where you want to notify the team that the issue got resolved 


And the `headers`, `html` and `require_tls` parts which is configuration for my email structure with from subject to and HTML template . (I can override that)

Go to the `route` section we will see that our `route` and the child route `routes` with matchers were merged also with the existing one . 

Basically the `receiver`, `route`, `matcher` parts were taken from the configuration I defined above . We also see that the `match` automatically get created whenever you add a route into this configuration file . By defaults it adds a `matcher` for a `namesapce` and whatever `namespace` that alert manager configuration was created in . 

- The parts mean that when the host high CPU load alert come from Prometheus to alert manager bcs it's firing, aler manager will check does this alert have a label `alertname: HostHighCPULoad` . Does this alert also have a label `namespace: monitoring`  . If both of these labels match so alert has both of these, then it will be sent to this `receiver: monitoring/main-rules-alert-config/email`. That's  why we have to set this label on both of our alerts . Otherwise this condition or this matcher would be false for both of our alerts and that will mean alert manager would send to receiver `null`

## Trigger Alerts for Email Receiver

#### Test Email Notification 

Right now both of our Alert are green. I will delete the `cpu-test pod` and we will restart it to regenerate the CPU stress, and if the 30s timeout is not enough to generate enough CPU load, we can also increase this, so I will set this at 60s . `kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief`. Soon enough I will see the Alert rule firing 

Now let's actually logically follow the alert as it makes its ways all the way to an email account where we configured Alert Manager to send it to 

First of all Alert Manager has an `endpoint` where we can see all the alerts that came in from Prometheus . So we see that is firing, so inside the Alert Manager, we have an endppoint `127.0.0.1:9090/api/v2/alerts` . This basically show us in the JSON format all the Alert that has been fired . If I copy and search this `HostHighCpuLoad` on the endpoint I should see it's `alertname` 

If we display this is more user-friendly way in the `VScode editor` . We should see some of the `arrtributes` that alert that arrives at the Alert Manager application container 

First of all we have these `annotations` with `description`, `summary` all the stuff that we set, plus auto-generated new ones, 

Then we have a `receiver` that match this alert so we know that Alert Manager `routed` this alert to this `receiver` . So that could be a good way to troubleshoot or debug our alerts and how they are routed, and we see all these . Then we see all the `labels` that our alert contain like 

- `alertname: HostHighCpuLoad` this is the one we configured in the matcher

- We also have the `instance`, the `namespace` and so on

Now that we know that this alert was routed to a correct receiver, now we can go and check our email

You can group multiple alert into 1 notific ation or one email, right now we just have one alert that we are receving . We have the name of the alert `isntance` and basically some key data, from these labels inside the subject, and we have other information in `Labels`, Annotaion 

If this alert arrive in our email inbox without us knowing what's going on in the cluster, we will know that this instance is having a high CPU load, which is currently above the threshold of 50% , so we need to take action bcs obviously something weird is happening in the cluster 

If you see the correct `receiver` for my alert, but email is not arriving, you may acutally have some authentication problems with the emails provider, so to debug that or troubleshoot that, you can also check `Alert Manager` logs `kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c alertmanager`, bcs if you have any authentication error, if the email can not be sent, then you will see an error message in the logs. 

Finally if you also want to test the pod crash looping alert being triggered and the sent to my email, then you can actually leave the CPU test pod running in the cluster for some time, bcs it acutally keep restarting and having this `CrashLoopBackOff` status, which is exactly what we are checking and when the restart count becomes greater than 5 the Alert will be fired as well  

And again you can make sure that the correct `receiver` was matched by alert manager for this alert for this alert . And then we have `"name": "monitoring/main-rules-alert-config/email"` 

So again Alert was fired by Prometheus, Alert Manager received it and basically checked the alert and its labels, which is a list of labels, `alert name`, `container`, `namespace` . So alert manager basically matched those 2 `labels` (`alertname` and `namespace`) and decided that alert should be routed to email receiver. and then using the configuration of the email configs, it will then send an email to our email account . So that how the flow of alert happen  

















