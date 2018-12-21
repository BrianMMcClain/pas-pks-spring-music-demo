PAS+PKS Spring Music Demo
===

![Architecture of our demo](img/arch.png)

![Installed components on Ops Manager](img/opsman.png)

Deploying the Spring Music Application
---
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

