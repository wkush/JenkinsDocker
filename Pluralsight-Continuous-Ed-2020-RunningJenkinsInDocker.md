# Running Docker in Jenkins
Chris Behrens 8 Apr 2020  2h 06m
https://app.pluralsight.com/library/courses/running-jenkins-docker/table-of-contents

## Notes on my setup 
Docker for Desktop > Linux containers Hyper-V backend.  Not using WSL2.

## Overview
* Running jenkins Master in Docker
* Isolation state of Master from the container
* Create docker images and containers with Jenkins
* Multi-arch builds

## Understanding Docker and Jenkins

Need to understand Docker under-the-covers for security reasons.

What is the diff between VMs and Docker.
VMs have individual kernels.
Docker has shared kernel.

What is the kernel?  We will run a windows kernel and a docker instance with a Linux kernel inside of that docker instance! 

Demo:

* Runs "systeminfo" in CMD.exe to show we are in windows.  Looks at forward or backward slashses as marker that we are using particular OS.  Uses Docker-for-Desktop.
OF> Can run systeminfo in PS also.
### Refine understanding of "kernel"

What is a kernel?  HAL..no  BIOS...definitely no.

Types of kernels:  mono, micro.

Kernel:  a core for memory and process management that gives ACCESS TO EVERYTHING.  Yes it does HAL of a type, etc.

#### Linux container does NOT share kernel with HOST.
In fact the containers share a kernel on a HyperV Linux VM.  That VM sits on top of the host NT kernel of the machine.

### Performance
Run natively on Linux to improve performance
### Security
Patch the host OS.  Hyper-jacking is an attack on the guest VM from a compromised host.
### Portability
Can move the container to another Win HOST (with VM hyperv enabled) or same container on a Linux host.

### WSL2
We should do this on WSL2. This course is hybrid hyperV.  WSL2 means we will be using a Linux kernel to host the containers.  WSL2 mounts the win filesystem to a linux container in a different way.  We will start with the VM/HyperV approach and then move to WSL2.

### The Vision and the WHy:  Jenkisn on Docker
How does the Azure build system work in Azure DevOps

1. Queue a build; 2. Azure reads build defintion (compiles demands: OS, deps,other); 3 matches build with build agent meeting demands; spins up a VM.
Subsequent builds are idempotent.

Wants to change e.g.   one "Universal Jenkins Agent" servicing 3 queued builds  into  3 separate parallel "Specialized Jenkins" agents specific to each type of build.

#### Jenkins queue models
Main Controller: labelled demands to jenkins agents
Dedicated machines etc.  Or dedicated VMs -- that's wasteful.  Better use containers.
Thats what azure devops does. 
All this is implementable as s first class artifact via the docker plug-in.  
We will have a jenkins Master in a Linux container on Windws.  We will assign jenkins agents to that master.

Demo: pull jenkins LTS from DockerHub, run it, modify docker run with tewaks.
https://app.pluralsight.com/library/courses/docker-fundamentals for help if cannot get it running by self.

Run JenkinsMaster in container.
Launch cmd.exe with admin permissions.
docker search jenkins  <--- lots of options  jenkins/jenkins is what we want. Specificlly the lts
docker pull jenkins/jenkins:lts
docker image ls   <--- lists the images we have pulled down for docker
```
  C:\UshDockerTests>docker image ls
  REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
  jenkins/jenkins                        lts                 190554e5446b        2 days ago          708MB
  node                                   latest              7aef30ae6655        6 months ago        943MB
  gerritcodereview/gerrit                latest              a6871596978b        9 months ago        576MB
  2019wkush/pswebapi                     latest              7f73c43bc696        9 months ago        254MB
  2019wkush                              pswebapi            7f73c43bc696        9 months ago        254MB
  pswebapi                               latest              7f73c43bc696        9 months ago        254MB
  <none>                                 <none>              ed60fd1ecf0d        9 months ago        1.74GB
  pswebapi                               dev                 bc2d106e8491        9 months ago        253MB
  2019wkush/pswebapi                     <none>              e6bc531932d3        9 months ago        254MB
  <none>                                 <none>              190b1992b320        9 months ago        1.74GB
  mcr.microsoft.com/dotnet/core/sdk      2.1-stretch         d7ec4dc56612        9 months ago        1.74GB
  mcr.microsoft.com/dotnet/core/aspnet   2.1-stretch-slim    b3df7864b3e1        9 months ago        253MB
  jenkins/jenkins                        <none>              f32b4bb22e4d        12 months ago       571MB
  jenkins/jenkins                        <none>              95bf220e341a        15 months ago       566MB
  hello-world                            latest              fce289e99eb9        20 months ago       1.84kB
```

docker run -p 80809:8080 jenkins/jenkins:lts   <-------  Simplet command we can do with that new container.  The log on the console contains an admin secret

```
  Jenkins initial setup is required. An admin user has been created and a password generated.
  Please use the following password to proceed to installation:

  8997bf84ba3c44ca8ce679bd94da59e4

  This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

  *************************************************************
  *************************************************************
  *************************************************************
```

Open chrome  localhost:8080.  


#### Changing and viewing the port mapping
-p 8080:8080  is where we specifiy the port mapping.  Jenkins runs on 8080 by default.  We could change that Jenkins config to a different port.  We would ahve to if it were a VM.  But we can tell docker to remap this container port to whatever we want.
We can check the mapping with 
```
  docker ps
```

```
  > docker ps
  CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                               NAMES
  f0a4ac494d03        jenkins/jenkins:lts   "/sbin/tini -- /usr/…"   8 minutes ago       Up 8 minutes        0.0.0.0:8080->8080/tcp, 50000/tcp   focused_brattain
```

Let us stop the container, remove it and re-run with the mapping so that on our machine the port 2112 is directed to the jenkins container which thinks it is running on 8080

```
  PS> docker stop focused_brattain
  focused_brattain
  PS C:\Users\Oisin.Feeley\Ush-Learning\2020-Continuous-Ed\Running Jenkins in Docker> docker rm focused_brattain
  focused_brattain
  PS C:\Users\Oisin.Feeley\Ush-Learning\2020-Continuous-Ed\Running Jenkins in Docker> docker run -p 2112:8080 jenkins/jenkins:lts
```

We are redirecting external 2112 to internal 8080
localhost:2112

```
  docker ps
  CONTAINER ID        IMAGE                 COMMAND                  CREATED              STATUS              PORTS                               NAMES
  b60b9ac6df4b        jenkins/jenkins:lts   "/sbin/tini -- /usr/…"   About a minute ago   Up About a minute   50000/tcp, 0.0.0.0:2112->8080/tcp   naughty_johnson
```

#### Change the name and add a port for the jenkins agent

```
  C:\UshDockerTests>docker run -p 2119:8080 -p 50000:50000 --name jenkins-master jenkins/jenkins:lts
```

#### Execute a shell command TO the container

```
  > docker exec -it jenkins-master /bin/bash
  jenkins@34f78bc4dbf2:/$ cd /var/jenkins_home/

  jenkins@34f78bc4dbf2:~$ ls
  config.xml                     jenkins.install.UpgradeWizard.state  nodeMonitors.xml  secret.key.not-so-secret  userContent
  copy_reference_file.log        jenkins.telemetry.Correlator.xml     nodes             secrets                   users
  hudson.model.UpdateCenter.xml  jobs                                 plugins           tini_pub.gpg              war
  identity.key.enc               logs                                 secret.key        updates

  ## NOTE:  No jobs here yet
  jenkins@34f78bc4dbf2:~$ ls jobs/

  jenkins@34f78bc4dbf2:~$ cat secrets/initialAdminPassword
  7eb6746ab0ec4a46b2e7ea3fea18fd7c
```


Put secret into chrome window prompt.  Select default plugins.
Create an admin user  "ushAdmin"  pwd "defecto"
In the final "Instance Configuration" he puts in the boxname of his windows laptop and the port "windows-tp27joi:2119".  How do I find that out for this machine?  Dumb way: search > Computer > (in results) This PC > right-click Properties

Smarter way? (See also ref section at end for other powershell network ops commands) 
```
  PS C:\Users\Oisin.Feeley> (Get-WmiObject win32_computersystem).DNSHostName
  QE64WS000000406
  PS C:\Users\Oisin.Feeley> (Get-WmiObject win32_computersystem).Domain
  na.wkglobal.com
```

Even before I entered FQDN found above into the Instance Configuration I got the login screen:

jenkins-in-docker-using-naglobal-FQDN.jpg

So the JenkinsURL must be for something else.....
and in fact it says on that screen "root URL for absolute links to various Jenkins resources" including BUILD_URL

Had a brief bit of confusion when it would not let me enter the root URL. Turned out that my credential had expired b/c I had been reading other material.  Reload fixed it.

Create a new Freestyle build named TEST via the jenkisn GUI.  Then look again at the container's "jobs" folder which ws previously empty.

```
  jenkins@34f78bc4dbf2:/$ ls /var/jenkins_home/jobs/TEST/
  builds  config.xml
  jenkins@34f78bc4dbf2:/$ ls /var/jenkins_home/jobs/TEST/builds/
  legacyIds  permalinks
  jenkins@34f78bc4dbf2:/$ cat /var/jenkins_home/jobs/TEST/config.xml
  <?xml version='1.1' encoding='UTF-8'?>
  <project>
    <description></description>
    <keepDependencies>false</keepDependencies>
    <properties/>
    <scm class="hudson.scm.NullSCM"/>
    <canRoam>true</canRoam>
    <disabled>false</disabled>
    <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
    <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
    <triggers/>
    <concurrentBuild>false</concurrentBuild>
    <builders/>
    <publishers/>
    <buildWrappers/>
  </project>jenkins@34f78bc4dbf2:/$
```


So, now we have the basic MVP of jenkins running in a container.

### Maintaing state outside the container
We get thsi because of docker layers to some extent.

They are read-only:  OS > Application > Configuration
and then a topmost writeable layer.   Our container puts a writeable layer ontop of Jenkins:LTS.  We want to manage that layer outside of the container so we can monitor,back-up, move elsewhere.   

Most essential form of isolating state is :.... Pipeline ScriptThey are read-only:  OS > Application > Configuration
and then a topmost writeable layer.   Our container puts a writeable filesystem layer ontop of Jenkins:LTS.  We want to manage that layer outside of the container so we can monitor,back-up, move elsewhere.   

Filelayer has plugins, our admin user, single job created.  We want to take control of that toplayer. And keep it outside of the container.

Most essential form of isolating state is :.... Pipeline Scripts. Classic FreeSytle build has its state stored as a config.xml file in jenkins filesystem.  It is tied to the filesystem (of container, vm whatever). If you use jenkinsfiles instead you can store those jenkins files in VCS.  The build definitions are the most important thing to use.  Do not use classic builds. 

### The DOcker Filesystem
Could either copy it all or have a base image and copy the top layer.
Copy-on-Write.
Store the deltas, copy to the Top Writeable Layer (TWL). Write/persist the changes.  TWL _is_ the state of the image.  He suggersts backup/mirror/raid the docker container files.  Suggests using volumes mounted on the host machine.

Demo: Mounting a volumen.
Modify the run statement to mount a volume.

Can't easily add a new volume to an existing container (TODO: investigate this). So we get rid of old container and then build a new one.

#### Create a volume for use with Docker

```
  PS> new-item -type d -path C:\Users\Oisin.Feeley\DockerVolumes\ -name jenkins-master
```

#### Create new container that mounts that volume

THIS DID NOT WORK.  Because it is missing the actual name of the mount
```
  > docker run -p 2119:8080 -p 50000:50000 -v C:\Users\Oisin.Feeley\DockerVolumes\jenkins-master --name jenkins-master jenkins/jenkins:lts
```

THIS DID WORK
```
  docker run -p 2120:8080 -p 50000:50000 -v c:/Users/Oisin.Feeley/DockerVolumes/jenkins-master2:/var/jenkins_home jenkins/jenkins:lts
```

#### Check the mount mapped to a container
Can use the container id or name

```
  > PS C:\Users\Oisin.Feeley\Ush-Learning\2020-Continuous-Ed\Running Jenkins in Docker> docker inspect -f "{{ .Mounts }}" fervent_feynman
  [{bind  /host_mnt/c/Users/Oisin.Feeley/DockerVolumes/jenkins-master2 /var/jenkins_home   true rprivate}]
```

Continuing with the tutorial we will see that if we enter a change to the config.xml for a running jenkins it will not pick it up.  It is not supposed to.  
Go through the login as admin, install plugins, enter root URL as before
Freestyle job "HelloDocker"   Explore the config.xml file created in the new DockerVolumes.  Enter a description into it.

Examine the job. No change.  Restart jenkins by appending  /restart to the URL.  (Note there is also safe restart).  Now we see the change.

Now stop and remove the docker container.  Then re-run the docker run.  When we login we should see our build.  We saved the state!

We could save this state to RAID, NAS whatever.   We should next be using PIpelines instead of these classic jenkins builds.
It's great to have this saved in a container, but pipelines allow us to save the files off to DVCS.  We should containerize.

## Creating a Jenkins Build Farm with Docker.
Ephemeral agents.

### Demo 1 single agent.  
* Ensure that the Docker Remote API is enabled
* COnfigure Docker in our Jenkins-Main
* Create a single containerized build agent (s-c-b-a)
* Attach s-c-b-a to our Jenkins-Main
* Review results

In this we use the HOST docker daemon's message passing abilities by contacting the daemon on it's reserved REST API port for remote message passing.  Alternatives discussed later include using JNLP(WebStart) or SSH.  

Want the jenkins-master to be able to control the docker host.  It will then be able to fire up container instances.

Host's RemoteAPI must be accessible to jenkins container on port 2375.  Go to docker desktop settings and GEneral? Expose daemon on 2375 without TLS.   Hyper-V reserves 2375.  There can be a race between hyper-v and docker.  

Restart docker-desktop.  We can examine the container config using a web-browser, e.g. Chrome
http://localhost:2375/containers/json:
```
  [{"Id":"8b513657cbda1f753ce657b88cd08916fb2b80ecb1becf9338444a55fe8a7810","Names":["/wizardly_snyder"],"Image":"jenkins/jenkins:lts","ImageID":"sha256:190554e5446bf00487790d326aeaaf5caab74529f0c977e4e6f99fb891de6565","Command":"/sbin/tini -- /usr/local/bin/jenkins.sh","Created":1599942430,"Ports":[{"IP":"0.0.0.0","PrivatePort":8080,"PublicPort":2120,"Type":"tcp"},{"IP":"0.0.0.0","PrivatePort":50000,"PublicPort":50000,"Type":"tcp"}],"Labels":{},"State":"running","Status":"Up 12 minutes","HostConfig":{"NetworkMode":"default"},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"ca46e720a34f36d61254e1e1019dd5948e38a948949d16b0fd27833091a6db3c","EndpointID":"fb7c61d55a499ce6be41117d8e40c124c627bb40966ec8a1bdd42dcebf473301","Gateway":"172.17.0.1","IPAddress":"172.17.0.2","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"02:42:ac:11:00:02","DriverOpts":null}}},"Mounts":[{"Type":"bind","Source":"/host_mnt/c/Users/Oisin.Feeley/DockerVolumes/jenkins-master2","Destination":"/var/jenkins_home","Mode":"","RW":true,"Propagation":"rprivate"}]}]
```
The above JSON output shows a correctly functioning Docker Remote API

localhost:2120 (or whatever Master-Jenkins) > Manage Jenkins > Manage Plugins > Available > 
search for Docker
install it, then restart, log back in 

Manage Jenkins > Manage Nodes and Clouds > COnfigure Clouds > Add a new cloud > Docker > Docker cloud details
this is where we specify our docker host api URI.  What is that?  It is the host docker daemon IP as it is exposed _inside_ the container.  There is a special FQDN setup for that within the container (can see inside container if do a "ping host.docker.internal). 
```
  docker exec -it wizardly_snyder /bin/bash
  jenkins@8b513657cbda:/$ ping host.docker.internal
  PING host.docker.internal (192.168.65.2) 56(84) bytes of data.
  64 bytes from 192.168.65.2 (192.168.65.2): icmp_seq=1 ttl=37 time=0.685 ms
```
Enter "tcp://host.docker.internal:2375"
Test Connection  should return the API version of the Remote Docker API
image: test-jenkins-remote-Docker-API.jpg

Enable;  Expose docker host

Then Docker Agent templates > Add Docker Template

Labels: Agent
Enabled: on
Name: Jenkins Agent
Docker image: jenkins/jenkins:lts   (for the moment this is just the same as our master)
Instance capacity: 10
Remote File System Root:  /var/jenkins_home
Save

Jenkins > Test-mount > Configure > General > Restrict  where this project can be run > Label expression: Agent
Save
Build Now.  Will see pending, jenkins does not have label agent.  That is because the container is not yet set up.

In the log should see "We now have 2 computers".
Docker Desktop logs.  ALso Docker Desktop should show the container popping up briefly when we get a new build started.
I do not see the ephemeral container being created as he does in the demo.  It works, but I cannot see it.



##### What happens when we trigger tbe build.

Jenkins looks for a label. If a label is present (can be absent which means any agent will do), Jenkins looks for an agent that can satisfy that label specifically and only.
The jenkins instance contacts docker using REST calls and tells it to create a NEW container using the instance parameters we specified for the agent. 
Jenkins provisions that running container as an agent using Jenkins remoting and executes the build.
Build succeed/fail.
Jenkins tears down container.

Problems: 
any meaningful changes to the jenkins image would have to be saved to a public dockerhub.  We can download some attractive images which seem to offer functionality we can just use and use them as a base.  But they are not trustworthy.  Prefer minimal base images.  https://snyk.io/blog/10-docker-image-security-best-practices

### Create our own image for DotNetCore to show how to build up an image.

Uses a dockerfile he found on stackoverlflow. I had to modify it to explicitly add the key with apt-key add

#### Dockerfile based on jenkins-LTS that also has dotnet tooling
```
  # https://stackoverflow.com/a/48609805
  # Dennis Faekas
  # https://stackoverflow.com/a/48609805
  # https://stackoverflow.com/questions/48104954/adding-net-core-to-docker-container-with-jenkins

  # For Pluralsight Running Jenkins in Docker course. Demo building ontop of jenkins/jenkins:lts

  FROM jenkins/jenkins:lts
  RUN apt-get update && apt-get install -y --no-install-recommends \
    curl libunwind8	gettext	apt-transport-https && \
    curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg && \
    apt-key add microsoft.gpg && \
    sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-debian-stretch-prod stretch main" > /etc/apt/sources.list.d/dotnetdev.list' && \
    apt-get update

  RUN apt-get install -y dotnet-sdk-3.1 && \
    export PATH=$PATH:$HOME/dotnet && \
    dotnet --version

  USER jenkins
```

He saves this to dotnetcore.Dockerfile (in my case in this C:\Users\Oisin.Feeley\UshLearning\2020-ContinuousEd\Running Jenkins in Docker. Then open cmd.exe in that directory (which contains the Dockerfile) and:

#### Build the new agent which contains dotnetcore and is based on Jenkins-LTS
```
  > docker build -t agent-dnc:v1 - < dotnetcore.Dockerfile
```
agent-dnc  is the name to give the image.  v1   is the tag

#### Verify that this new image "agent-dnc:v1" can build a test dotnet class
Then can list the image with "docker images"  and then run the image and then open a shell on it (docker exec -it asdfasdfa)
After checking that the "jenkins" user has permissions on all the /var/jenkins\_home he creates a new dotnet class directly with:
```
  C:\Users\Oisin.Feeley\Ush-Learning\2020-Continuous-Ed\Running Jenkins in Docker>docker exec -it sleepy_newton /bin/bash
  jenkins@3ea76fc7fe6f:/$ cd /var/jenkins_home/
  jenkins@3ea76fc7fe6f:~$ ls -l
  total 76
  -rw-r--r--  1 jenkins jenkins 1643 Sep 12 22:08 config.xml
  -rw-r--r--  1 jenkins jenkins   50 Sep 12 22:08 copy_reference_file.log
  -rw-r--r--  1 jenkins jenkins  156 Sep 12 22:08 hudson.model.UpdateCenter.xml
  -rw-------  1 jenkins jenkins 1712 Sep 12 22:08 identity.key.enc
  -rw-r--r--  1 jenkins jenkins    7 Sep 12 22:08 jenkins.install.UpgradeWizard.state
  -rw-r--r--  1 jenkins jenkins  171 Sep 12 22:08 jenkins.telemetry.Correlator.xml
  drwxr-xr-x  2 jenkins jenkins 4096 Sep 12 22:08 jobs
  drwxr-xr-x  3 jenkins jenkins 4096 Sep 12 22:08 logs
  -rw-r--r--  1 jenkins jenkins  907 Sep 12 22:08 nodeMonitors.xml
  drwxr-xr-x  2 jenkins jenkins 4096 Sep 12 22:08 nodes
  drwxr-xr-x  2 jenkins jenkins 4096 Sep 12 22:08 plugins
  -rw-r--r--  1 jenkins jenkins   64 Sep 12 22:08 secret.key
  -rw-r--r--  1 jenkins jenkins    0 Sep 12 22:08 secret.key.not-so-secret
  drwx------  4 jenkins jenkins 4096 Sep 12 22:08 secrets
  -rw-rw-r--  1 root    root    7152 Sep  9 20:21 tini_pub.gpg
  drwxr-xr-x  2 jenkins jenkins 4096 Sep 12 22:08 updates
  drwxr-xr-x  2 jenkins jenkins 4096 Sep 12 22:08 userContent
  drwxr-xr-x  3 jenkins jenkins 4096 Sep 12 22:08 users
  drwxr-xr-x 11 jenkins jenkins 4096 Sep 12 22:08 war
  jenkins@3ea76fc7fe6f:~$ dotnet new classlib -o TestLib

  Welcome to .NET Core 3.1!
  ---------------------
  SDK Version: 3.1.402

  Telemetry
  ---------
  The .NET Core tools collect usage data in order to help us improve your experience. The data is anonymous. It is collected by Microsoft and shared with the community. You can opt-out of telemetry by setting the DOTNET_CLI_TELEMETRY_OPTOUT environment variable to '1' or 'true' using your favorite shell.

  Read more about .NET Core CLI Tools telemetry: https://aka.ms/dotnet-cli-telemetry

  ----------------
  Explore documentation: https://aka.ms/dotnet-docs
  Report issues and find source on GitHub: https://github.com/dotnet/core
  Find out what's new: https://aka.ms/dotnet-whats-new
  Learn about the installed HTTPS developer cert: https://aka.ms/aspnet-core-https
  Use 'dotnet --help' to see available commands or visit: https://aka.ms/dotnet-cli-docs
  Write your first app: https://aka.ms/first-net-core-app
  --------------------------------------------------------------------------------------
  Getting ready...
  The template "Class library" was created successfully.

  Processing post-creation actions...
  Running 'dotnet restore' on TestLib/TestLib.csproj...
    Determining projects to restore...
    Restored /var/jenkins_home/TestLib/TestLib.csproj (in 2.22 sec).

  Restore succeeded.
```
```
  ## NOTE:  here we manually check that the Class was created correctly by the above post-creation action

  jenkins@3ea76fc7fe6f:~$ cd TestLib/
  jenkins@3ea76fc7fe6f:~/TestLib$ ls
  Class1.cs  TestLib.csproj  obj
  jenkins@3ea76fc7fe6f:~/TestLib$ cat Class1.cs
  ?using System;

  namespace TestLib
  {
      public class Class1
      {
      }
  }

  ## NOTE:  check that dotnet can run correctly on this Test class

  jenkins@3ea76fc7fe6f:~/TestLib$ dotnet build
  Microsoft (R) Build Engine version 16.7.0+7fb82e5b2 for .NET
  Copyright (C) Microsoft Corporation. All rights reserved.

    Determining projects to restore...
    All projects are up-to-date for restore.
    TestLib -> /var/jenkins_home/TestLib/bin/Debug/netstandard2.0/TestLib.dll

  Build succeeded.
      0 Warning(s)
      0 Error(s)

  Time Elapsed 00:00:01.66
  jenkins@3ea76fc7fe6f:~/TestLib$
```

So we now have a dotnet build server from our Dockerfile

### Demo: Attach YUour Dotnet Image to a Template for our CLoud
* Attach new image as a template for our cloud
* Create a C# CLI project
* Make quick Jenkinsfile to build C# project
   - Restrict to our dotnetcore agent
* Execute build
* Review results

Stop the running agent-dnc:v1 container. Remove it.
```
  docker stop sleepy\_newton; docker rm sleepy\_newton
```

##### Stop create cloud agent which depends on the docker image agent-dncv1
Master jenkins web gui > Mangage Jenkins > manage clouds > configure clouds > Docker Agent template

NOTE:  these details are important (see below)
We already had a "cloud" named docker with ONE template in it.  We shall now add another agent to our cloud with a new template

> Docker Agent templates >  give it the Remote File System Root: /var/jenkins_home (same as the previous one)
and the Label "DOTNETCORE" 
and use the Docker Image: agent-dncv1 
and give it the Name "agent-dnc"

#### Visual Studio 2019. Create a new test ConsoleApp

Create a new project >  C# ConsoleApp(.NET Core)

Location C:\...\Running Jenkins in Docker > Create

Modify the ConsoleApp1  to write our own message.
He has put this source up at https://github.com/FeynmanFan/JenkinsDocker

#### Create a Jenkinsfile for a Jenkins _Pipeline_ project
We will put this Jenkinsfile into GitHub and in Jenkins configure a new pipeline item that fetches this Jenkins file.
Wants a jenkins PIpeline project that will pull this project and magically build it on the correct agent.  Here is his Jenkinsfile that does exactly that:
```
  node('DOTNETCORE') {
    stage('SCM') {
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/FeynmanFan/JenkinsDocker']]])
      }
    stage('Build') {
      sh 'dotnet build ConsoleApp1'
    }
    stage('Test') {
      echo 'Execute unit tests here. This is the Jenkinsfile'
    }
    stage('Package') {
      echo 'Zip it up'
    }
    stage('Deploy') {
      echo 'Push to deployment'
    }
  }
```
See https://www.pluralsight.com/courses/automating-jenkins-groovy

#### In Jenkins create the pipeline new item (DNC-ConsoleApp)  manually
Jenkins Master > New Item > DNC-ConsoleApp > Pipeline > Pipeline script from SCM > Git > enter github URL to location of 

https://github.com/FeynmanFan/JenkinsDocker.git


The structure of the repository that we have entered in Jenkins (and also in the Jenkinsfile itself) on github is
```
  -----+
       | 
       +------ ConsoleApp1\
       | 
       +------ dotnetcore\
       | 
       +------ Jenkinsfile     <--- this is what we have detailed just above
```
When we build this job is does nothing for ages   we append  logs/all to the http://localhost:2120  and search for "label" until we find dotnetcore and a message saying that pull access is denied:
```
  Error during callback
  com.github.dockerjava.api.exception.NotFoundException: {"message":"pull access denied for agent-dnc, repository does not exist or may require 'docker login': denied: requested access to the resource is denied"}
```

This is because the agent image we created succesfully before (named  agent-dnc) relied on an image (jenkins/jenkins-lts) that exists on DockerHub (jenkins/jenkins:lts).  This agent (created in *Step create cloud agent which depends on the docker image agent-dncv1*)  will be trying to pull image agent-dnc  so we have to get that up to DockerHub using our own account.

##### Remove the old image from our host.
```
  docker image ls
```
select agent-dnc  tag v1
get its id or name

Remove and untag the image from the host node (docker RMI):
```
  docker rmi <id> -f
  docker rmi dd08a15d75c5 -f
  Untagged: agent-dnc:v1
  Deleted: sha256:dd08a15d75c5934d0f7bcf5411d381fce17504c3c54c43f257ba307dae388e40
  Deleted: sha256:82dfd733ef834d73341142a4807847c2229368f434f118e5dfd41e93f2fd05ed
  Deleted: sha256:cf1863c3cc045b952fe2838876cf301a6d4d41acb49de5200f6fa259d5081670
  Deleted: sha256:ed42669724beed88df211c6cb43daffdb363ba3429de9706a4e930dc74c90c4e
  Deleted: sha256:0751c02c09e59e855803a9e6efebbaf760c4e105608a932f6cc6eb80cc8c9901
```

##### Create a new image exactly the same except that we name it with our DockerHub account XXXXX
Recreate the image using our Dockerhub account name XXXXXX
```
  docker build -t XXXXXX/agent-dnc:v1 - < dotnetcore.Dockerfile
```

##### Push the new image to our account  [step push-docker-dns]
Upload to dockerhub using account XXXXX
```
  docker push XXXX/agent-dnc:v1
```

If the above fails then need to sign in to dockerhub using the whale icon.  Then after push look in dockerhub for the image 
Then update the agent template with the prepend XXXXX

Then can retrigger a build of DNC-ConsoleApp (he actually left the build running and showed that when the agent became available the build figured it out and succeeded.


image: templated-docker-agent-pulling-dotnet-image-build-works.jpg

### Demo Docker meta image

  * Create a docker image using:  Jenkins and Docker itself (how is that different from above?).
  * Make a new agent from this image
  * Attach this agent to Jenkins-Master
  * Make a Jenkinsfile that builds the Dotnetcore image and _pushes it to Dockerhub_ (we had to do that manually above)

#### jenkinsdocker.Dockerfile

Put this in my local experiment directory.  It does the ugly install docker curl-bash into a fresh jenkins-lts
```
  FROM jenkins/jenkins:lts

  USER root
  RUN apt-get update -qq && apt-get install -qqy \
    apt-transport-https \
    ca-certificates \
    curl \
    lxc \
    iptables

  RUN curl -sSL https://get-docker.com/ | sh

  RUN usermod -aG docker jenkins

  CMD dockerd
```

[step-building-jenkinsdocker-manually]
Build the above dockerfile locally with 
```
  >docker build -t garrumph/jenkinsdocker:v1 - < jenkinsdocker.Dockerfile
```

#### Run the new image and open a shell in it to check that dockerd is running and jenkins has permissions
Note:  we specify the parameter user as "jenkins" in the logon.

*Privileged mode has security implications.  It allows the container to do things it would not otherwise be allowed to do:
https://blog.trendmicro.com/trendlabs-security-intelligence/why-running-a-privileged-container-in-docker-is-a-bad-idea/*
```
  docker run --privileged garrumph/jenkinsdocker:v1

  PS> docker ps

  CONTAINER ID        IMAGE                       COMMAND                  CREATED              STATUS              PORTS                 NAMES
  4000998bde15        garrumph/jenkinsdocker:v1   "/sbin/tini -- /usr/…"   About a minute ago   Up About a minute   8080/tcp, 50000/tcp   wizardly_blackwell

  PS> docker exec -it --user jenkins wizardly_blackwell /bin/bash
  jenkins@4000998bde15:/$
```
##### check docker running
```
  jenkins@4000998bde15:/$ docker

  Usage:  docker [OPTIONS] COMMAND

  A self-sufficient runtime for containers

  Options:
        --config string      Location of client config files (default "/var/jenkins_home/.docker")
    -c, --context string     Name of the context to use to connect to the daemon (overrides DOCKER_HOST env var and default
                             context set with "docker context use")
    -D, --debug              Enable debug mode
              SNIP ETC

  jenkins@4000998bde15:/$ ps afx | grep docker
    164 pts/0    S+     0:00  \_ grep docker
      1 ?        Ss     0:00 /sbin/tini -- /usr/local/bin/jenkins.sh /bin/sh -c dockerd
      6 ?        S      0:00 /bin/sh -c dockerd
     10 ?        Sl     0:01  \_ dockerd
     17 ?        Ssl    0:01      \_ containerd --config /var/run/docker/containerd/containerd.toml --log-level info
```

So we are running  dockerd  in a docker container!

#### Check that we can manually execute the process we want to automate
Copy the dotnetcore.dockerfile into this dockerd container. NOTE:  new  "docker cp" command allows direct copying into named instance root directory
```
  > docker cp .\dotnetcore.Dockerfile wizardly_blackwell:/

  PS> docker exec -it --user jenkins wizardly_blackwell /bin/bash
  jenkins@4000998bde15:/$ ls
  bin  boot  dev  dotnetcore.Dockerfile  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
#### In this new dockerd container see if we can build the image within the container
```
  jenkins@4000998bde15:/$ docker build -t garrumph/agent-dnc:v2 - < dotnetcore.Dockerfile
   Welcome to .NET Core!
   ---------------------
   Step 5/5 : USER jenkins
     ---> Running in 61fa37567319
    Removing intermediate container 61fa37567319
     ---> f017b4d5c480
    Successfully built f017b4d5c480
    Successfully tagged garrumph/agent-dnc:v2
```

Having confirmed that the above sequence of steps (1. create a dockerfile that makes a dockerd container; 2) copy into it a dockerfile that creates a dotnetcore container) we should be able to put those steps into a Jenkinsfile and make it do the manual process for us. 

##### Create the Jenkinsfile
So we need a Jenkinsfile to push this image up to DockerHub. Modified what he supplied in order to use our own credentials for DOckerHub
NOTE:  this is the jenkins documention relevant to what is happening with the build and push
https://www.jenkins.io/doc/book/pipeline/docker/#building-containers
https://www.jenkins.io/doc/book/pipeline/docker/#custom-registry
For a Docker Registry which requires authentication, add a "Username/Password" Credentials item from the Jenkins home page and use the Credentials ID as a second argument to withRegistry():

NOTE: https://github.com/FeynmanFan/JenkinsDocker:  he created a new subdir in his git repo named "dotnetcore" and put into it dotnetcore.Dockerfile renamed as Dockerfile
NOTE: this is how to create dockerhubcreds which he did not specify in the video (no big deal, just a question about whether we use an "id" or not) 
https://appfleet.com/blog/building-docker-images-to-docker-hub-using-jenkins-pipelines/ 
Jenkins > Credentials > Global > Add credentials > 
image: set-dockerhub-creds-in-jenkins.jpg

buildDNC.Jenkinsfile:
```
  def dockerImage;

  environment   #### WRONG!

    node('docker') {
      stage('SCM') {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/wkush/JenkinsDocker']]]);
        }
        stage('build'){
          dockerImage = docker.build('garrumph/agent-dnc:v$BUILD_NUMBER','./dotnetcore');
        }
        stage('push'){
          docker.withRegistry('', 'dockerhubcredsgarrumph'){
            dockerImage.push();
          }
        }
      }
```
NOTE: the plugins necessary for these helper commands are from the plugins dockerpipeline and dockerbuildstep. Check they are installed. 
NOTE: this is a good explanation of Jenkinsfiles using Dockerfiles:  https://www.jenkins.io/doc/book/pipeline/docker/

###### Put this jenkinsfile in our own github.  He pulls it down from his, but I want to change it to use my creds, so I need to have it available on my github account as a public file. So modified the URL also.



Jenkins-Master > New Pipeline build "Build Agent-DNC" > Pipeline script from SCM > SCM:Git > Repositories/RepositoryURL:https://github.com/wkush/JenkinsDocker.git    Credentials:-none-   >  Script Path: buildDNC.Jenkinsfile

Build > Fails 
It is supposed to fail because of there being no docker agent available yet.  His failure console log shows "obtained buildDNC.jenkinsfile .... [Pipeline]Start of Pipeline  [Pipeline]node   Still waiting to schedule task  'Jenkins' doesn't have lable 'docker'. 
Mine fails with this:
```
  Started by user Ush F
  Obtained buildDNC.Jenkinsfile from git https://github.com/wkush/JenkinsDocker.git
  Running in Durability level: MAX\_SURVIVABILITY
  [Pipeline] Start of Pipeline (hide)
  [Pipeline] End of Pipeline
  groovy.lang.MissingPropertyException: No such property: environment for class: groovy.lang.Binding
    at groovy.lang.Binding.getVariable(Binding.java:63)
    at org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SandboxInterceptor.onGetProperty(SandboxInterceptor.java:270)
    at org.kohsuke.groovy.sandbox.impl.Checker$7.call(Checker.java:353)
    at org.kohsuke.groovy.sandbox.impl.Checker.checkedGetProperty(Checker.java:357)
    at com.cloudbees.groovy.cps.sandbox.SandboxInvoker.getProperty(SandboxInvoker.java:29)
    at com.cloudbees.groovy.cps.impl.PropertyAccessBlock.rawGet(PropertyAccessBlock.java:20)
    at WorkflowScript.run(WorkflowScript:3)
```

Fixed this by removing 'environment' from the buildDNC.Jenkinsfile and pushing the change to github.  (Actually was able to use the Jenkins "replay" on the job to modify the script first and test what it does:
```
  Started by user Ush F
  Replayed #1
  Running in Durability level: MAX\_SURVIVABILITY
  [Pipeline] Start of Pipeline
  [Pipeline] node
  Still waiting to schedule task
  ‘Jenkins’ doesn’t have label ‘docker’
```

##### Create the missing docker agent by adding a template to configure clouds

Needed to re-install the docker plugin for some reason.  Then ConfigureClouds > Docker Agent templates>
Add templates > Docker Agent templates >
labels: docker
Enabled: /
Name:  Docker builder image
Docker IMage: garrumph/jenkinsdocker:v1      WE NEED TO PUSH THIS IMAGE UP NOW!

###### Push up the image
We had created the image manually in [step-building-jenkinsdocker-manually]
We currently only have one image up:   image dockerhub-personal-agent-dncv1-pushed-manually.jpg                                
```
  PS > docker image ls
  REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
  garrumph/jenkinsdocker                 v1                  9dc1da18deda        3 hours ago         1.15GB
  garrumph/agent-dnc                     v1                  f028e7c2619d        17 hours ago        1.11GB
  <none>                                 <none>              f75f6aa740a0        19 hours ago        772MB
  jenkins/jenkins                        lts                 190554e5446b        3 days ago          708MB
```
```
  docker push garrumph/jenkinsdocker:v1
  The push refers to repository [docker.io/garrumph/jenkinsdocker]
  8c4dee3e2585: Pushed
```

Now we have 2 repositories, this new one has a single image in it
image dockerhub-repo-garrumph-jenkinsdocker.jpg

Remember we have separate repos in dockerhub :  garrumph/jenkinsdocker  and garrumph/agent-dnc

Back to Jenkins > Docker Agent templates>  Container Settings>
    Run Container privileged: /                           Here we are doing something dumb  
    Volumes: /var/run/docker.sock:/var/run/docker.sock    Here we are sharing the host dockerd socket with the child otherwise agent will not be able to connect.


Going to rebuild the Build Agent-DNC we could see it fail after it had pulled down the Jenkins file with a nessage about a missing docker method in groovy.Binding. 
```
Running on Docker builder image-0000488v8p30q on docker in /workspace/Build Agent-DNC
[Pipeline] {
[Pipeline] stage
[Pipeline] { (SCM)
[Pipeline] checkout
The recommended git tool is: NONE
No credentials specified
Cloning the remote Git repository
Cloning repository https://github.com/wkush/JenkinsDocker
 > git init /workspace/Build Agent-DNC # timeout=10
Fetching upstream changes from https://github.com/wkush/JenkinsDocker
 > git --version # timeout=10
 > git --version # 'git version 2.11.0'
 > git fetch --tags --progress -- https://github.com/wkush/JenkinsDocker +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
Checking out Revision 1882f3842898376529d1f4d25569946f25bf368c (refs/remotes/origin/master)
 > git config remote.origin.url https://github.com/wkush/JenkinsDocker # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 1882f3842898376529d1f4d25569946f25bf368c # timeout=10
Commit message: "Remove unwanted method"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (build)
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
groovy.lang.MissingPropertyException: No such property: docker for class: groovy.lang.Binding
```

Restart jenkins and retried:    

Keep getting the "docker"  property not available.  Restaring docker desktop.


### Private Registries.
We want private registry eitehr in dockerhub or inside our DMZ.
We need to control the registry URL and specify credentials.  These will be in the open over port 2375

* Pushing to a private repository in dockerbub via a build
* Pulling an image from a privatre repository for our build agent

####






Prepend with docker.io  to point to private repository

Use Dockerfiles only.  They show what happend to the image.

#### Working with Ephemaral agents.
Workspace results should be put somewhere.  That's why we do artifacting.

### Demo  adding archibving
We need to add a try - finally block around the build in order to get the archive step to run as the last step
As the last step
stage('Archive') {
  archiveArtifacts artifacts: (ConsoleApp1/*.*'
  }


## Working with Multi-Architecture Containers in Jenkins

This is all experimental.  Uses docker buildx.

We have now a manifest in the registry which lists the architectures available.


### BuildX in docker uses QEMU to emulate hardware architectures

On and on about MAME.  
BuildKit new engine for building docker images. Used by buildx. Executes image steps in parallel


## Maintaing Your Build Farm
Easy to get behind Jenkins updates
If something hurts do it often


## References

### Other useful articles

* Mounting volumes in windows to docker.
https://blog.sixeyed.com/docker-volumes-on-windows-the-case-of-the-g-drive/#:~:text=Docker%20volumes%20on%20Windows%20are,when%20you%20run%20a%20container.
Docker volumes on Windows are always created in the path of the graph driver, which is where Docker stores all image layers, writeable container layers and volumes. By default the root of the graph driver in Windows is C:\ProgramData\docker, but you can mount a volume to a specific directory when you run a container


### Specific issues



#### Tutorials
https://www.vogella.com/tutorials/Jenkins/article.html
https://www.youtube.com/watch?v=7KCS70sCoK0   Tech World with Nana "Complete Jenkins Pipeline Tutorial"
https://www.youtube.com/channel/UCTt7pyY-o0eltq14glaG5dg  "Automation Step by Step - Raghav Pal"  Full series on Jenkins
, also Postman, Docker, Selenium Beginner, Selenium Java Framework for Beginners,  Selenium Python, Kubernetes Beginner Tutorials, API Web Services, Git and GitHub 


##### Other useful PS commands for tracing network problems.  Move to own ref section at end.
```
> Get-ComputerInfo | Format-Table WindowsVersion,WindowsEditionId,WindowsProductName

WindowsVersion WindowsEditionId WindowsProductName
-------------- ---------------- ------------------
1809           Enterprise       Windows 10 Enterprise

```
```
Get-DnsClientServerAddress

InterfaceAlias               Interface Address ServerAddresses
                             Index     Family
--------------               --------- ------- ---------------
Ethernet                             6 IPv4    {8.8.8.8, 8.8.4.4}
Ethernet                             6 IPv6    {}
VMware Network Adapter V...1        25 IPv4    {}
VMware Network Adapter V...1        25 IPv6    {}
VMware Network Adapter V...8        22 IPv4    {}
VMware Network Adapter V...8        22 IPv6    {}
Loopback Pseudo-Interface 1          1 IPv4    {}
Loopback Pseudo-Interface 1          1 IPv6    {}
vEthernet (Default Switch)          26 IPv4    {}
vEthernet (Default Switch)          26 IPv6    {}

```

```
> Get-NetIPconfiguration


InterfaceAlias       : VMware Network Adapter VMnet8
InterfaceIndex       : 22
InterfaceDescription : VMware Virtual Ethernet Adapter for VMnet8
IPv4Address          : 192.168.87.1
IPv4DefaultGateway   :
DNSServer            :

InterfaceAlias       : VMware Network Adapter VMnet1
InterfaceIndex       : 25
InterfaceDescription : VMware Virtual Ethernet Adapter for VMnet1
IPv4Address          : 192.168.160.1
IPv4DefaultGateway   :
DNSServer            :

InterfaceAlias       : vEthernet (Default Switch)
InterfaceIndex       : 26
InterfaceDescription : Hyper-V Virtual Ethernet Adapter
NetProfile.Name      : Réseau non identifié
IPv4Address          : 192.168.43.145
IPv4DefaultGateway   :
DNSServer            :

InterfaceAlias       : Ethernet
InterfaceIndex       : 6
InterfaceDescription : Intel(R) Ethernet Connection (5) I219-LM
NetProfile.Name      : NETGEAR56-5G 2
IPv4Address          : 192.168.2.120
IPv4DefaultGateway   : 192.168.2.1
DNSServer            : 8.8.8.8
                       8.8.4.4
```

Test-NetConnection "Hostname" -Port #
Test-NetConnection "Hostname" -traceroute


