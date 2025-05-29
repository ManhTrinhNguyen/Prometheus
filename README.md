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

To go through how the dashboards are actually made up and I can create all of these myself as well, on a dashboard view I can have multiple rows, so if I add 2 rows here, I can see that a row is one grouping of different data . I can then group the data into each row based on what the data is, so we can have 



