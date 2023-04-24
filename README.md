# Using Podman to Generate and Test a Kubernetes YAML Manifest
**Tom Dean - 3/22/2023**

## Introduction

A couple of weeks back, in a previous article, [Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://www.linkedin.com/pulse/deploying-kubernetes-in-docker-kind-cluster-using-podman-tom-dean-1c/), we took a look at how to use Podman to deploy a Kubernetes cluster as a KIND container.

Recently, I was discussing what a good use case might be for an article on deploying an application in a Podman Pod with my friend Jeff Kaleth, and he suggested Atlassian Confluence, as he does quite a bit of work documenting things with it.  Confluence tends to be consumed as SaaS these days, but sometimes you might want to run a local instance.

Traditionally, this would mean installing software either on a bare metal server or virtual machine, on top of an operating system.  This would be messy, and take time and effort to get everything configured properly.  Yuck!

*There has to be a better way.  There is!*

In my previous article, [Deploying a Confluence Server in a Podman Pod Using Containers](https://www.linkedin.com/pulse/deploying-confluence-server-podman-pod-using-containers-tom-dean/), I showed you how to get a minimal instance of Confluence, with a Postgres instance backing it up, in a Podman Pod, up and running in minutes.

In this tutorial, I'm going to show you how we can use Podman and our Confluence pod, running in Podman, to generate a YAML manifest that we can then deploy to Kubernetes.  We're going to focus on using Podman to generate the YAML manifest, with a detour to generate a set of `systemd` unit files in the process!

***Buckle up!  Let's hit the road!***

## References

[Deploying a Confluence Server in a Podman Pod Using Containers](https://www.linkedin.com/pulse/deploying-confluence-server-podman-pod-using-containers-tom-dean/)

[Deploying a Kubernetes-In-Docker (KIND) Cluster Using Podman on Ubuntu Linux](https://www.linkedin.com/pulse/deploying-kubernetes-in-docker-kind-cluster-using-podman-tom-dean-1c/)

[Getting Started with Podman ](https://podman.io/getting-started/)

[dockerhub: atlassian/confluence-server](https://hub.docker.com/r/atlassian/confluence-server)

[dockerhub: postgres](https://hub.docker.com/_/postgres)

[Podman: Managing pods and containers in a local container runtime](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods#)

[Podman can now ease the transition to Kubernetes and CRI-O](https://developers.redhat.com/blog/2019/01/29/podman-kubernetes-yaml#)

## Prerequisites

You will need an x64 Linux instance (physical or virtual) to deploy the Confluence container we will be using.  You're also going to need Podman installed and configured on your Linux instance.  If you need to accomplish this, see the **References** section above.

## Deploy a Confluence Server in a Podman Pod

For those who followed the tutorial in my previous article, [Deploying a Confluence Server in a Podman Pod Using Containers](https://www.linkedin.com/pulse/deploying-confluence-server-podman-pod-using-containers-tom-dean/), this will look familiar.

Before we can generate a YAML manifest, we will need to create our pod, so we have something to grab a configuration snapshot from.

First, let's create directories *(if needed)* for our Confluence server data and our Postgres database data:
```bash
mkdir -p ~/confluence/site1/data
mkdir -p ~/confluence/site1/database
```

If you still have these directories from the first tutorial, and don't care about the site data in the `site1` directory, feel free to delete the existing directories and create a fresh set of `site1` directories:
```bash
sudo rm -rf ~/confluence/*
```

```bash
mkdir -p ~/confluence/site1/data
mkdir -p ~/confluence/site1/database
```

### Preserving Existing Confluence Data - IF DESIRED

You can create a second set of directories instead, to preserve any data from the first tutorial.

Create a new set of `site2` directories:
```bash
mkdir -p ~/confluence/site2/data
mkdir -p ~/confluence/site2/database
```

***Moving forward, use `site2` in place of `site1` when creating your containers, as appropriate.***

### Refresh Container Images

Next, let's pull our container images, so we have them.  If you've done this before, do it again to refresh your container images.

Pull the `atlassian/confluence-server` container image:
```bash
podman pull atlassian/confluence-server
```

Pull the `postgres` container image:
```bash
podman pull postgres
```

Checking our work:
```bash
podman images
```

We should see something like the following:
```bash
REPOSITORY                             TAG         IMAGE ID      CREATED       SIZE
docker.io/atlassian/confluence-server  latest      3144ae928531  12 hours ago  1.42 GB
docker.io/atlassian/confluence         latest      52ead2ad4991  2 weeks ago   1.42 GB
docker.io/library/postgres             latest      3b6645d2c145  3 weeks ago   387 MB
k8s.gcr.io/pause                       3.5         ed210e3e4a5b  2 years ago   690 kB
```

First, we'll create our `confluence-pod` to put our `atlassian/confluence-server` and `postgres` containers in.  We'll publish our Confluence server port, which is `8090`, to our host on `8290`, as some folks use port `8090` for Cockpit.  We're going to keep the database traffic local to our pod.

Create our `confluence-pod` pod:
```bash
podman pod create --name confluence-pod -p 8290:8090
```

Next, we'll add our `atlassian/confluence-server` and `postgres` containers.

Create our `confluence-postgres` container in the `confluence-pod` pod:
```bash
podman run --name confluence-postgres -e POSTGRES_USER=confluenceUser -e POSTGRES_PASSWORD=confluencePW -e POSTGRES_DB=confluenceDB -v ~/confluence/site1/database:/var/lib/postgresql/data --pod confluence-pod -d postgres
```

Note the `postgres` user, password and the Confluence database password above.

Next, create our `confluence-server` container in the `confluence-pod` pod:
```bash
podman run -v ~/confluence/site1/data:/var/atlassian/application-data/confluence --name confluence-server --pod confluence-pod -d atlassian/confluence
```

Checking our work:
```bash
podman ps -a --pod
```

We should see our `confluence-pod` pod, with the `confluence-postgres` and `confluence-server` containers:
```bash
CONTAINER ID  IMAGE                                  COMMAND         CREATED         STATUS             PORTS                   NAMES                POD ID        PODNAME
04ed5ee269f8  k8s.gcr.io/pause:3.5                                   46 seconds ago  Up 27 seconds ago  0.0.0.0:8290->8090/tcp  27e702e352bd-infra   27e702e352bd  confluence-pod
821be37b766b  docker.io/library/postgres:latest      postgres        26 seconds ago  Up 27 seconds ago  0.0.0.0:8290->8090/tcp  confluence-postgres  27e702e352bd  confluence-pod
3a344dd8b800  docker.io/atlassian/confluence:latest  /entrypoint.py  7 seconds ago   Up 8 seconds ago   0.0.0.0:8290->8090/tcp  confluence-server    27e702e352bd  confluence-pod
```

If we point our web browser at our host, on port `8290`, we should get our Confluence licensing page as shown below.

![Mission Accomplished!](images/Screen%20Shot%202023-03-09%20at%202.22.01%20PM.png)

***You now have a working instance of Confluence Server!***

## Use Podman to Generate and Test a Kubernetes YAML Manifest

We can use Podman to export the running configuration of our `confluence-pod` pod in a YAML manifest that we can then use to create a copy of our `confluence-pod` pod in Kubernetes.

### Exporting Our Pod Configuration to YAML

Now that we have a running `confluence-pod` pod with our `confluence-postgres` and `confluence-server` containers, we can capture the configuration in a YAML manifest, using the `podman generate kube` command.

First, let's take a look at the `podman generate` command:
```bash
podman generate --help
```

We should see:
```bash
Generate structured data based on containers, pods or volumes

Description:
  Generate structured data (e.g., Kubernetes YAML or systemd units) based on containers, pods or volumes.

Usage:
  podman generate [command]

Available Commands:
  kube        Generate Kubernetes YAML from containers, pods or volumes.
  systemd     Generate systemd units.
```

We can use the `podman generate` command to create both YAML manifests for Kubernetes, as well as `systemd` unit files that we can use to manage Podman containers in `systemd`.  We're going to take a look at generating a YAML manifest first.

Trying the `podman generate kube` command:
```bash
podman generate kube confluence-pod
```

We get the following output:
```bash
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.4.4
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-03-22T19:41:43Z"
  labels:
    app: confluence-pod
  name: confluence-pod
spec:
  containers:
...
```

We can use redirection to create a file for our YAML configuration:
```bash
podman generate kube confluence-pod > confluence-pod.yaml
```

Checking the contents of our file:
```bash
cat confluence-pod.yaml
```

We should see something similar to the following:
```bash
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.4.4
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-03-22T19:43:59Z"
  labels:
    app: confluence-pod
  name: confluence-pod
spec:
  containers:
...
```

We need to edit our YAML manifest.  I'm using `vi`, but you can use your favorite editor.

Edit manifest:
```bash
vi confluence-pod.yaml
```

We want to change two things in the manifest:

- Remove any timestamp or other unneeded metadata
- Change all instances of `site1` to `site2`, if you are using the `site2` directories, *if needed*

We should end up with something like the following:
```bash
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.4.4
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: confluence-pod
  name: confluence-pod
spec:
  containers:
  - args:
    - postgres
    image: docker.io/library/postgres:latest
    name: confluence-postgres
    ports:
    - containerPort: 8090
      hostPort: 8290
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/lib/postgresql/data
      name: home-tdean-confluence-site1-database-host-0
  - args:
    - /entrypoint.py
    image: docker.io/atlassian/confluence:latest
    name: confluence-server
    resources: {}
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    volumeMounts:
    - mountPath: /var/atlassian/application-data/confluence
      name: home-tdean-confluence-site1-data-host-0
  restartPolicy: Never
  volumes:
  - hostPath:
      path: /home/tdean/confluence/site1/database
      type: Directory
    name: home-tdean-confluence-site1-database-host-0
  - hostPath:
      path: /home/tdean/confluence/site1/data
      type: Directory
    name: home-tdean-confluence-site1-data-host-0
status: {}
```

***We now have a YAML manifest for our `confluence-pod` pod!***

### Exporting Our Pod Configuration to a `systemd` Unit File

While we have a running `confluence-pod` pod , we're going to use Podman to generate a `systemd` unit file, which we will set aside for now.

Let's take a look at the `podman generate` command again:
```bash
podman generate --help
```

Once again, we should see:
```bash
Generate structured data based on containers, pods or volumes

Description:
  Generate structured data (e.g., Kubernetes YAML or systemd units) based on containers, pods or volumes.

Usage:
  podman generate [command]

Available Commands:
  kube        Generate Kubernetes YAML from containers, pods or volumes.
  systemd     Generate systemd units.
```

Let's give the `podman generate systemd` command a try:
```bash
podman generate systemd confluence-pod
```

We should get output similar to the following:
```bash
# pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service
# autogenerated by Podman 3.4.4
# Wed Mar 22 19:48:11 UTC 2023

[Unit]
Description=Podman pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=
Requires=container-3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0.service container-821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d.service
Before=container-3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0.service container-821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d.service

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman start 04ed5ee269f8a558c66d7d4bb821653b54586d3d8d87b53dd732b09738cd8412
ExecStop=/usr/bin/podman stop -t 10 04ed5ee269f8a558c66d7d4bb821653b54586d3d8d87b53dd732b09738cd8412
ExecStopPost=/usr/bin/podman stop -t 10 04ed5ee269f8a558c66d7d4bb821653b54586d3d8d87b53dd732b09738cd8412
PIDFile=/run/user/1000/containers/overlay-containers/04ed5ee269f8a558c66d7d4bb821653b54586d3d8d87b53dd732b09738cd8412/userdata/conmon.pid
Type=forking

[Install]
WantedBy=default.target
# container-3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0.service
# autogenerated by Podman 3.4.4
# Wed Mar 22 19:48:11 UTC 2023

[Unit]
Description=Podman container-3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/run/user/1000/containers
BindsTo=pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service
After=pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman start 3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0
ExecStop=/usr/bin/podman stop -t 10 3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0
ExecStopPost=/usr/bin/podman stop -t 10 3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0
PIDFile=/run/user/1000/containers/overlay-containers/3a344dd8b80039e92aa830ff37e98bc096878a1829f90c3842bb6f57588177f0/userdata/conmon.pid
Type=forking

[Install]
WantedBy=default.target
# container-821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d.service
# autogenerated by Podman 3.4.4
# Wed Mar 22 19:48:11 UTC 2023

[Unit]
Description=Podman container-821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/run/user/1000/containers
BindsTo=pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service
After=pod-27e702e352bdadc67a302d163fdd46c6abd5bc4c2357ad5178b00e84c924c33d.service

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman start 821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d
ExecStop=/usr/bin/podman stop -t 10 821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d
ExecStopPost=/usr/bin/podman stop -t 10 821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d
PIDFile=/run/user/1000/containers/overlay-containers/821be37b766ba9f4b502956db4a0675cb2b305900e1463b253c684d7eea3993d/userdata/conmon.pid
Type=forking

[Install]
WantedBy=default.target
```

We can see that we get three unit files, one for the `confluence-pod` pod and one each for our `confluence-postgres` and `confluence-server` containers.

Let's put that into a file:
```bash
podman generate systemd confluence-pod > confluence-pod-systemd.txt
```

***We're going to set our consolidated `systemd` unit file aside for now.  Look for an article on how to put this to use in the future!***

### Tearing Down Our Confluence Server

Before we test our YAML manifest, we need to delete the existing `confluence-server` containers and the `confluence-pod` pod.  When we test our YAML manifest, we will be creating objects with the same name in Podman, and if these object already exist, we will get an error and the process will fail.

Delete the containers:
```bash
podman rm -f confluence-server confluence-postgres
```

Delete the pod:
```bash
podman pod rm -f confluence-pod
```

Checking our work:
```bash
podman ps -a --pod
```

We should not see any containers or pods:
```bash
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
```

***Looking great!  Now we're ready to test our manifest.***

### Testing Our Confluence Pod Manifest Using Podman

We can use the `podman play kube` command to test our YAML manifest:
```bash
podman play kube --help
```

We should see:
```bash
Play containers, pods or volumes from a structured file

Description:
  Play structured data (e.g., Kubernetes YAML) based on containers, pods or volumes.

Usage:
  podman play [command]

Available Commands:
  kube        Play a pod or volume based on Kubernetes YAML.
```

Let's give it a try with our YAML manifest:
```bash
podman play kube confluence-pod.yaml
```

We should get output similar to the following:
```bash
Trying to pull docker.io/atlassian/confluence:latest...
Getting image source signatures
Copying blob d6adb981df4e [--------------------------------------] 0.0b / 0.0b
Copying blob b9cabe75b440 [--------------------------------------] 0.0b / 0.0b
Copying blob adb8bdfab6a9 [--------------------------------------] 0.0b / 0.0b
Copying blob 7c134c18e00f skipped: already exists
Copying blob 026ca17296e7 skipped: already exists
Copying blob cff6df0cca35 [--------------------------------------] 0.0b / 0.0b
Copying blob 2d76e33286b3 [--------------------------------------] 0.0b / 0.0b
Copying blob 342f22ed35dc skipped: already exists
Copying blob ac1184f53f8b skipped: already exists
Copying blob 74ac377868f8 [--------------------------------------] 0.0b / 0.0b
Copying config 3144ae9285 done
Writing manifest to image destination
Storing signatures
Pod:
c9f0e15b3548dce303a2d41e4cdc721882a4f10358d0c62525b3e592510920d6
Containers:
ea61b70bfecfaa8558bf968e059fd7882e0cdbc5b4d3005c9da1609f7de89fb0
3afce24471e61edd14595a50b2a477399a161fc0551620302535ec8bf4a9500a
```

Checking our work:
```bash
podman ps -a --pod
```

We should see something similar to the following:
```bash
CONTAINER ID  IMAGE                                  COMMAND         CREATED             STATUS                 PORTS                   NAMES                               POD ID        PODNAME
73694bf971bc  k8s.gcr.io/pause:3.5                                   About a minute ago  Up About a minute ago  0.0.0.0:8290->8090/tcp  c9f0e15b3548-infra                  c9f0e15b3548  confluence-pod
ea61b70bfecf  docker.io/library/postgres:latest      postgres        About a minute ago  Up About a minute ago  0.0.0.0:8290->8090/tcp  confluence-pod-confluence-postgres  c9f0e15b3548  confluence-pod
3afce24471e6  docker.io/atlassian/confluence:latest  /entrypoint.py  About a minute ago  Up About a minute ago  0.0.0.0:8290->8090/tcp  confluence-pod-confluence-server    c9f0e15b3548  confluence-pod
```

If we point our web browser at our host, on port `8290`, we should get our Confluence licensing page as shown below.

![Mission Accomplished!](images/Screen%20Shot%202023-03-09%20at%202.22.01%20PM.png)

***Our YAML manifest works great with Podman!  Success!***

### Tearing Down Our Confluence Server

Before we wrap up, we need to delete the test `confluence-server` containers and the test `confluence-pod` pod.

Delete the containers:
```bash
podman rm -f --all
```

Delete the pod:
```bash
podman pod rm -f --all
```

Checking our work:
```bash
podman ps -a --pod
```

We should not see any containers or pods:
```bash
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES       POD ID      PODNAME
```

***Our manifest is looking good and passes the Podman test!  Kubernetes, here we come!***

## Summary

So, we've shown that deploying a self-hosted Confluence server doesn't have to be a pain.  We stood up a hosted instance of Confluence in short order, using the power of Podman, pods and containers.  And we used that to generate a YAML manifest, based on our pod's running configuration in Podman, and used Podman to test our manifest.

*Shall we take our Confluence pod manifest to Kubernetes?  Why yes!*

In the next article, we're going to see what happens when we feed our YAML manifest for our Confluence pod into a KIND (Kubernetes-In-Docker) cluster!

Enjoy!

*Tom Dean*
