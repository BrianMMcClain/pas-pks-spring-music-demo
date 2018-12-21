PAS+PKS Spring Music Demo
===

In this demo, we take a look at how PAS and PKS can work in tandum, where PAS will run a frontend application and PKS will run our backend database. While PCF does provide access to MySQL specifically (and many other services!), this is also applicable to those situations where an application may fit the cloud-native, [12 Factor](https://12factor.net/) Application model, but the backing service might have special requirements that don't fit that model.

![Architecture of our demo](img/arch.png)

All of these components were configured and installed by [Ops Manager](https://pivotal.io/platform/pcf-components/pcf-ops-manager), which provides an easy way to install and upgrade things like PAS, PKS, services, etc. Below you can see all of the components installed for this demo:

![Installed components on Ops Manager](img/opsman.png)

Refer to the documentation for steps to install [PCF Healthwatch](https://docs.pivotal.io/pcf-healthwatch/1-4/installing.html), [PCF Metrics](https://docs.pivotal.io/pcf-metrics/1-5/installing.html), and [MySQL for PCF](https://docs.pivotal.io/p-mysql/1-10/installing.html).

Deploying the Spring Music Application
---
The Spring Music application presents a demo of an app that could be used for tracking an album collection. By default, it populates a dataset with a handful of examples, but can optionally be connected to a backend database to persist changes.

From the `spring-music` directory, we can build our Spring Music application and then push it with a single command:

```
./gradlew clean assemble
cf push
```

This will use the [manifest.yml](spring-music/manifest.yml) for configuration, specifying basic options like the application's name and memory requirements. The PAS platform will detect that we're deploying a Java Spring application, and automatically use the [Java Buildpack](https://github.com/cloudfoundry/java-buildpack) to supply the runtime, framework, and dependencies to package and run our application. Once the buildpack completes, we'll see our application soon up and running:

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

A few moments later, as expected, our .Net application is up and running on a Windows Server machine!

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

[Apps Manager](https://docs.pivotal.io/pivotalcf/2-4/console/index.html) is a great tool for PAS devlopers and administrators to manage users and applications. It gives a great at-a-glance view at the applications running on the platform, including the ability to scale apps and stream logs. Additionally, Apps Manager works great with [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready) to get even more information specifically from your Spring applications, such as request traces and thread dumps.

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

```
$ kubectl describe service mysql-service

Name:                     mysql-service
Namespace:                default
Labels:                   app=dev-mysql
                          chart=mysql-0.11.0
                          heritage=Tiller
                          release=dev-mysql
Annotations:              <none>
Selector:                 app=dev-mysql
Type:                     LoadBalancer
IP:                       1.1.1.1
LoadBalancer Ingress:     1.2.3.4
Port:                     <unset>  3306/TCP
TargetPort:               3306/TCP
NodePort:                 <unset>  30633/TCP
Endpoints:                1.1.1.1:3306
Session Affinity:         None
External Traffic Policy:  Cluster
```

We can verify our database is up and running by using the MySQL CLI:

```
mysql --host=1.2.3.4 -u devUser -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 39775
Server version: 5.7.14 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| spring-music       |
+--------------------+
2 rows in set (0.06 sec)
```

Connecting Spring Music to MySQL
---

So now we have our application (Spring Music on PAS) and our database (MySQL on PKS) up and running, all that's left is to tell our application how to connect to MySQL. For this, we can use a feature in PAS called a ["User Provided Service"](https://docs.pivotal.io/tiledev/2-4/user-provided.html). We can create a user-provided service, supplying the connection string to our MySQL database, and PAS will provide this to our application on the backend. We'll create this service and name it `k8s-db`.

```
cf create-user-provided-service k8s-db -p '{"uri":"mysql://devUser:pass@1.2.3.4:3306/spring-music"}'
```

Next, we'll need to bind this service to our application:

```
cf bind-service spring-music k8s-db
```

Finally, we'll need to restart our application. Previously, we could either incur a very small amount of downtime (a few seconds), or stand up a second instance of our application and rebind the URL from the old to the new version to avoid downtime, PCF 2.4 ships with an experimental feature to address this directly. While not quite production-ready, we can use the new `v3-zdt-restart` command to perform a zero-downtime restart on our application:

```
cf v3-zdt-restart spring-music
```

This will ensure our application is restarted (or if we use the `v3-zdt-push` command, updated) without incurring any downtime. If multiple instances are running, PCF will update them one-by-one.

Finally, let's add an album to our Spring Music application and ensure these changes are written to the MySQL database.

![Adding an album to Spring Music](img/add-album.png)

And if we check in MySQL, we'll see our new album added to our database:

```
mysql> select title, artist, release_year from album where artist = "AC/DC";
+--------------+--------+--------------+
| title        | artist | release_year |
+--------------+--------+--------------+
| Back In Back | AC/DC  | 1980         |
+--------------+--------+--------------+
1 row in set (1.42 sec)
```

2-Layer Autorecovery
---

With our app up and running, happily serving requests, we're then faced with everything that comes with _keeping_ it up and running. The platform knows, though, what it means to be healthy. So if, say, one of the VMs running our application instances goes down (or is fully deleted), we have two things to remediate:

1. I've suddenly lost the VM running an instance of my application, so I need a new one spun up in its place on another VM.
2. I've suddenly lost a large chunk of my capacity to run future application instances, so I need to replace that VM.

Luckily, PCF knows that these two things should be true. As such, it will spring into action in a couple of ways:

1. PAS will immediately spin up new application instances that suddenly found themselves without a VM to run on, spreading the load across the remaining VMs.
2. [BOSH](http://bosh.io/) (the underlying operations automation layer) will notice the VM has gone down and create a new one in its place. It knows this has happened because it maintains a constant heartbeat with every VM that it's deployed and manages. Once this machine is back up, it enters back in the "pool" of eligible machines to run application isntances, and as new ones are spun up, PAS will use this new machine to even out the load.

Automatic Patching 
---

Finally, the question remains: how do I easily patch my infrastructure? How do I get the latest security patch to address that big CVE that just hit? While Ops Manager boils this down to just a couple of clicks, we can take this a bit further and even fully automate it. How more and more people are doing this is to use [Concourse](https://concourse-ci.org/), a CI/CD solution based on the idea of "pipelines" (one task feeding into the next).

![PKS upgrade pipeline](img/pipeline.png)

With these pipelines, we can bring in and install patches into our environment. A common scenario we've seen is that operators will have their pipeline automatically install patches in their lower environments (dev and test), then when they're happy with the stability, apply them to staging by manually running another pipeline. 

Read more about the pcf-pipelines [here](https://pivotal.io/concourse/operators) and on [GitHub](https://github.com/pivotal-cf/pcf-pipelines), or on the [Pivotal Network](https://network.pivotal.io/products/pcf-automation/) release. _(Note: To get access to this product on Pivotal Network, contact your Pivotal Support/Sales team.)

![Pivotal's Oded upgrades his PKS install from his Apple Watch](img/upgrade-tweet.png)

https://twitter.com/odedia/status/1072228525002903552
