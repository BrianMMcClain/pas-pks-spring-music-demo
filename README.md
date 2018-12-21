PAS+PKS Spring Music Demo
===

![Architecture of our demo](img/arch.png)

![Installed components on Ops Manager](img/opsman.png)

Deploying the Spring Music Application
---
The Spring Music application presents a demo of an app that could be used for tracking an album collection. By default, it populates a dataset with a handful of examples, but can optionally be connected to a backend database to persist changes.

From the `spring-music` directory, we can build our Spring Music application and then push up it with a single command:

```
./gradlew clean assemble
cf push
```

This sill use the [manifest.yml](spring-music/manifest.yml) for configuration, specifying basic options like the application's name and memory requirements. The PAS platform will detect that we're deploying a Java Spring application, and automatically use the [Java Buildpack](https://github.com/cloudfoundry/java-buildpack) to supply the runtime, framework, and dependencies to package and run our application. Once the buildpack completes, we'll see our application soon up and running:

```
...
requested state: started
instances: 1/1
usage: 1G x 1 instances
urls: spring-music.cf.local
last uploaded: Fri Dec 21 17:08:05 UTC 2018
stack: cflinuxfs3
buildpack: client-certificate-mapper=1.8.0_RELEASE container-security-provider=1.16.0_RELEASE java-buildpack=v4.16.1-offline-https://github.com/cloudfoundry/java-buildpack.git#41b8ff8 java-main java-opts java-security jvmkill-agent=1.16.0_RELEASE open-jd...

     state     since                    cpu      memory         disk           details
#0   running   2018-12-21 12:09:04 PM   135.4%   221.3M of 1G   171.7M of 1G
```

Deploying the .Net Environment Application
---
With the installed support of PAS for Windows, we can just as easily deploy a .Net application on Windows as we did our Spring application on Linux. Likewise, our [NET-sample-app](NET-sample-app) will leverage the [Hosted Web Core Buildpack](https://docs.cloudfoundry.org/buildpacks/hwc/index.html), which will again provide the framework and runtime dependencies needed to run our application. The deployment process is the same, though, and from the `NET-sample-app` directory we can deploy our app with one command:

```
cf push
```

A few moments later, as espected, our .Net application is up and running on a Windows Server machine!

```
...
name:              net-environment
requested state:   started
routes:            net-environment.cf.local
last uploaded:     Fri 21 Dec 12:20:19 EST 2018
stack:             windows2016
buildpacks:        hwc

type:            web
instances:       1/1
memory usage:    1024M
start command:   .cloudfoundry\hwc.exe
     state     since                  cpu    memory        disk        details
#0   running   2018-12-21T17:20:33Z   0.0%   88.5M of 1G   52M of 1G
```

Interestingly enough, the CF CLI provides SSH access to your running applications, even if they're running on Windows!

```
$ cf ssh net-environment
Microsoft Windows [Version 10.0.17134.345]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\vcap> systeminfo
...
OS Name:                   Microsoft Windows Server Datacenter
OS Version:                10.0.17134 N/A Build 17134
OS Manufacturer:           Microsoft Corporation
...
```

Logging, Tracing, and Monitoring
---
We've also setup three products that help with visability into our applications and the platform as a whole:

**Apps Manager**

[Apps Manager](https://docs.pivotal.io/pivotalcf/2-4/console/index.html) is a great tool for PAS devlopers and adminstrators to manage users and applications. It gives a great at-a-glance view at the applications running on the platform, including the ability to scale apps and stream logs. Additionally, Apps Manager works great with [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready) to get even more information specifically from your Spring applications, such as request traces and thread dumps.

![Tacing calls made to Spring Music](img/apps-manager.png)


**PCF Metrics**

[PCF Metrics](https://pivotal.io/platform/services-marketplace/monitoring-metrics-and-logging/pcf-metrics) offers a deeper look into applications running on PAS, offering real-time metrics to help you troubleshoot. It offers up metrics like requests per minute, request latency, errors, and resource usage all correlated with application events such as scaling or crashes.

![Real-time metrics from our Spring Music application](img/metrics.png)


**PCF Healthwatch**

[PCF Healthwatch](https://docs.pivotal.io/pcf-healthwatch/1-4/index.html) allows for a deeper look into the PCF platform at the infrastructure layer and gives a way to monitor that everything is operating as expected. Even better, it ships with pre-configured dashboards for Pivotal's defined key performance indicators.

![PCF Healthwatch Dashboard](img/healthwatch.png)


Creating a Kubernetes Cluster with PKS
---

We have our Spring Music application up and running, but we don't have any services that it can connect to so that it can persist any changes. While we could absolutely use [MySQL for PCF](https://docs.pivotal.io/p-mysql/2-4/), for the purpose of this demo let's assume that we need to set up MySQL on our own. Kubernetes is a great place to do this, and with PKS we can create our own Kubernetes cluster with a single command:

```
pks create-cluster pks-demo --external-hostname pks-cluster.example.com --plan small --num-nodes 2
```

Since PKS isn't a fork of Kubernetes or a propritary build, and instead chooses to focus on making it easy to consume and operate Kubernetes, the above command will create a plain-old Kubernetes cluster. We can use the same tools and processes that we might be used to if we've used Kubernetes before, with a few additional tools in our toolbelt (such as [Harbor](https://docs.pivotal.io/partners/vmware-harbor/index.html), a container registry that can scan containers for vulnerabilities).

Since this _is_ just Kubernetes, we can use something like [Helm](https://helm.sh/) to automatically deploy and configure MySQL for us:

```
helm install --name dev-mysql --set mysqlRootPassword=pass,mysqlUser=devUser,mysqlPassword=pass,mysqlDatabase=spring-music stable/mysql
```

And for the purpose of our demo, we can expose this MySQL service just like we would in any other Kubernetes installation:

```
kubectl expose deployment dev-mysql --type=LoadBalancer --name=mysql-service
```

Once the service is exposed, we can get the Ingress IP that we'll eventually tell our application to use to connect to our database.


