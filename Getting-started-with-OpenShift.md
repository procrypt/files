#Getting started with OpenShift

[OpenShift](https://www.openshift.org) is Red Hat's PaaS (Platform-as-a-Service) that allow developers to quickly develop, host, and scale applications in cloud.
Based on top of Docker containers and the Kubernetes container cluster manager, OpenShift adds developer and operational centric tools to enable rapid application development, easy deployment and scaling, and long-term lifecycle maintenance for small and large teams and applications.

There are a couple of ways by which you can setup your OpenShift cluster like [minishift](https://github.com/minishift/minishift) , [oc cluster up](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md).
We will be using `oc cluster up` for this blog.

#### Setting up single node Openshift cluster using `oc cluster up`

##### Install docker-daemon on your machine
`$ sudo dnf install -y docker   engine`

##### Enable docker service
`$ sudo systemctl enable docker.service`

##### Start the docker daemon
`$ sudo systemctl start docker`
 
##### To run docker without sudo add user to docker group and reboot the system
  
```bash
    $ sudo gpasswd -a <user> docker
    $ init 6
```
 
  Configure the Docker daemon with an insecure registry parameter of 172.30.0.0/16
  Edit your `/etc/sysconfig/docker` file and add the following line
  
```bash
     $ INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16' `
     $ sudo systemctl enable docker.service
```
  
**NOTE: We can run `oc cluster up` without the insecure registry by using `--skip-registry-flag=true` flag.**

#####  Start OpenShift single node cluster
  
```bash
     $ oc cluster up
    -- Checking OpenShift client ... OK
    -- Checking Docker client ... OK
    -- Checking Docker version ... OK
    -- Checking for existing OpenShift container ... OK
    -- Checking for openshift/origin:v1.5.0-alpha.0 image ... OK
    -- Checking for available ports ... OK
    -- Checking type of volume mount ... 
       Using nsenter mounter for OpenShift volumes
    -- Creating host directories ... OK
    -- Finding server IP ... 
       Using 192.168.121.96 as the server IP
    -- Starting OpenShift container ... 
       Creating initial OpenShift configuration
       Starting OpenShift using container 'origin'
       Waiting for API server to start listening
       OpenShift server started
    -- Adding default OAuthClient redirect URIs ... OK
    -- Installing registry ... OK
    -- Installing router ... OK
    -- Importing image streams ... OK
    -- Importing templates ... OK
    -- Login to server ... OK
    -- Creating initial project "myproject" ... OK
    -- Removing temporary directory ... OK
    -- Server Information ... 
       OpenShift server started.
       The server is accessible via web console at:
           https://192.168.121.96:8443
    
       You are logged in as:
           User:     developer
           Password: developer
    
       To login as administrator:
           oc login -u system:admin
```

This will start you single node OpenShift cluster on your system. 

#### Deplyoing application on OpenShift

We will deploy guestbook application to our single node cluster, using the [docker-compose](https://github.com/procrypt/kompose-example/blob/master/docker-compose.yml) file.

```bash
    $ cat docker-compose.yml
    version: "2"

    services:
      redis-master:
        image: gcr.io/google_containers/redis:e2e 
        ports:
          - "6379"
      redis-slave:
        image: gcr.io/google_samples/gb-redisslave:v1
        ports:
          - "6379"
        environment:
          - GET_HOSTS_FROM=dns
      frontend:
        image: gcr.io/google-samples/gb-frontend:v4
        ports:
          - "80:80"
        environment:
          - GET_HOSTS_FROM=dns
```

For creating the OpenShift artifacts from Dockerfile we will you [Kompose](https://github.com/kubernetes-incubator/kompose)

##### Login to OpenShift
 
```bash
   $ oc login
   Authentication required for https://192.168.121.96:8443 (openshift)
   Username: developer
   Password: 
   Login successful.
   
   You don't have any projects. You can try to create a new project, by running
   
       oc new-project <projectname>
```

##### Create new project
 
```bash
    $ oc new-project guestbook
    Now using project "guestbook" on server "https://192.168.121.96:8443".
    
    You can add applications to this project with the 'new-app' command. For example, try:
    
        oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git
    
    to build a new example application in Ruby.
```
 
We need to change the security context in OpenShift since by default OpenShift doesn't allow any container to run with root access. We need to login with OpenShift admin to make the changes.

```bash
    $ oc login -u system:admin
    $ oc edit scc restricted
```

 Change the `runAsUser` strategy to `RunAsAny`.

##### Login again as developer user

  `$ oc login -u developer`
    
##### We can now see the project in the web console.
![Image](https://github.com/procrypt/images/blob/master/web-ui.png?raw=true)

##### Create OpenShift Artifacts from [docker-compose](https://github.com/procrypt/kompose-example/blob/master/docker-compose.yml) file.

```bash
    $ kompose -f docker-compose.yml --provider=openshift convert -y
    INFO[0000] file "frontend-service.yaml" created         
    INFO[0000] file "redis-master-service.yaml" created     
    INFO[0000] file "redis-slave-service.yaml" created      
    INFO[0000] file "frontend-deploymentconfig.yaml" created 
    INFO[0000] file "frontend-imagestream.yaml" created     
    INFO[0000] file "redis-master-deploymentconfig.yaml" created 
    INFO[0000] file "redis-master-imagestream.yaml" created 
    INFO[0000] file "redis-slave-deploymentconfig.yaml" created 
    INFO[0000] file "redis-slave-imagestream.yaml" created  
```

##### Create a folder and move all the generated files to that folder

```bash
   $ mkdir guestbook
   $ mv frontend-* guestbook
   $ mv redis-* guestbook
```   
   
##### Deploy the application

```bash
   $ oc create -f guestbook/
    deploymentconfig "frontend" created
    imagestream "frontend" created
    service "frontend" created
    deploymentconfig "redis-master" created
    imagestream "redis-master" created
    service "redis-master" created
    deploymentconfig "redis-slave" created
    imagestream "redis-slave" created
    service "redis-slave" created
```

##### Your application has been deployed to OpenShift, for details

```bash
    $ oc get dc,svc,is,pvc,pods
    NAME              REVISION   DESIRED   CURRENT   TRIGGERED BY
    dc/frontend       1          1         0         config,image(frontend:v4)
    dc/redis-master   1          1         0         config,image(redis-master:e2e)
    dc/redis-slave    0          1         0         config,image(redis-slave:v1)
    
    NAME               CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    svc/frontend       172.30.10.147   <none>        80/TCP     40s
    svc/redis-master   172.30.8.180    <none>        6379/TCP   39s
    svc/redis-slave    172.30.28.207   <none>        6379/TCP   39s
    
    NAME              DOCKER REPO                                 TAGS      UPDATED
    is/frontend       172.30.44.119:5000/guestbook/frontend       v4        25 seconds ago
    is/redis-master   172.30.44.119:5000/guestbook/redis-master   e2e       24 seconds ago
    is/redis-slave    172.30.44.119:5000/guestbook/redis-slave    v1        
    
    NAME                      READY     STATUS    RESTARTS   AGE
    po/frontend-1-y9bca       1/1       Running   0          2m
    po/redis-master-1-meqjr   1/1       Running   0          2m
```

##### Web console of the application being deployed

![Image](https://github.com/procrypt/files/blob/master/detail.png?raw=true)

##### We now need to expose the service so that application to make it accessible to public.

```bash
   $ oc get svc
    NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    frontend       172.30.10.147   <none>        80/TCP     12m
    redis-master   172.30.8.180    <none>        6379/TCP   12m
    redis-slave    172.30.28.207   <none>        6379/TCP   12m

   $ oc expose svc frontend
    route "frontend" exposed
    
   $ oc get route
    NAME       HOST/PORT                                  PATH      SERVICES   PORT      TERMINATION
    frontend   frontend-guestbook.192.168.121.96.xip.io             frontend   80        
```    

##### Access the application.

![Image](https://github.com/procrypt/files/blob/master/output.png?raw=true)

We have successfully deployed the guestbook application on OpenShift.

##### OpenShift Cheat Sheet.

`$ oc type`

It will give you a quick summary of different conceptual types and definitions user in OpenShift.
Also check out `oc help` and `oc --help`  to see description and examples and learn how to use the commands. 
