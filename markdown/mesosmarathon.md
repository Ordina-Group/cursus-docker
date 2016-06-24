## Mesos and Marathon
*Docker at scale!*

![jenkins-docker](images/dockermesosmarathon-logo.jpg)


## Say What!?

### Apache Mesos
Apache Mesos is an open-source cluster manager that was developed at the University of California, Berkeley. It "provides efficient resource isolation and sharing across distributed applications, or frameworks". The software enables resource sharing in a fine-grained manner, improving cluster utilization.

Apache Mesos lets us treat a cluster of nodes as on big computer. Not as individual machines, not as VMs, but as computational resources like cores, memory, disks, etc.


## Say What!?

### Marathon
Marathon is a popular PaaS or container orchestration system scaling to thousands of physical servers. It is fully REST based and allows canary style deploys and deployment topologies. It is written in Scala and is highly available, elastic, and distributed.

Marathon is designed to launch long-running applications, and, in Mesosphere, serves as a replacement for a traditional init system. It has many features that simplify running applications in a clustered environment, such as high-availability, node constraints, application health checks, an API for scriptability and service discovery, and an easy to use web user interface.


# Let's get started

## 
- Because installing a complex stack like this is not the goal of this course, we will provide you with your own Mesos and Marathon stack.
- Execute the commands below to clean up and get the cluster up and running
- Building the stack will take some time. Continue in the next slide if you do not want to follow the progress of building the stack

```
cd /mnt/sda1/cursus-docker/mesos-marathon

# Stop all Containers
docker stop $(docker ps -a -q)

# If you need to install docker-compose:
sudo /mnt/sda1/cursus-docker/basics-docker/compose/install-compose.sh

# Change IP's in the docker-compose.yml file in case your docker-machine has an IP other then 192.168.99.100
sed -i s/192.168.99.100/<< docker-machine-ip >>/g docker-compose.yml

# Build the Mesos and Marathon cluster
docker-compose build

# Start the cluster
docker-compose up

```


### Stack ingredients
The stack consists of 7	Docker images
<small>
- **zookeeper:** <br>Software that is used to coordinate the master nodes

- **mesosmaster:** <br>The Mesos Master is a node in the cluster which orchestrates the running of tasks on slaves

- **mesosslave:** <br>A Mesos Slave is a Mesos instance which offers resources to the cluster. They are the worker instances - tasks are allocated to the slaves by the Mesos master.

- **marathon:** <br> Organizes and manages services deployed (what is deployed where, what ports, what configuration, etc)

- **marathonsubmit:** <br> A helper image for submitting jobs to marathon

- **gateway:** <br> A proxy that forwards TCP and UDP connections to where services are currently running on the clusters

- **secretary:** <br> A little tool to distrubute keys and password across the cluster in a secure manner

</small>


## Stack ingedients - Mesos
Remember that Apache Mesos lets us treat a cluster of nodes as on big computer? We've only got one slave in this setup (sorry, your hardware resources are limited)... so we will not be running a full swing cluster. 

That taken into account, the management of the cluster remains fairly the same. And this is exceactly what were we want to introduce you to.


## Stack ingedients - Marathon
Marathon manages the services that will manage deployed on the Mesos cluster. This tool will get the most attention during this course. 


## Stack ingedients - Others
The Zookeeper, Gateway and Secretary images are here support Mesos or Marathon, but are not within our scope of interrest. They are needed to get the show on the road, but will not require your attention.

The marathonsubmit image however does require some attention. More on that later in this course.


## Once all is running
Once everything is up and running you can start exploring. Feel free to click around in Mesos and Marathon and see how the two relate to each other.

- **http://<< docker-machine-ip >>:5050** - webinterface of Mesos

- **http://<< docker-machine-ip >>:8080** - webinterface of Marathon


## The demo application

You've got a simple site that is running on <br> http://<< docker-machine-ip >>:1234 

#### ... Boring ... 

This demo application was deployed by the *marathon-submit* container which is part of our docker-compose file. That's no fun, we want to try this out for ourself. We will in a couple of minutes.


## Now what the @#!& happend !?
Take a look at the Dockerfile of marathon-submit. You should notice this:
```
# Default entrypoint submits all files into Marathon
COPY bin/launch.sh /launch.sh
RUN chmod u+x /launch.sh
ENTRYPOINT ["/launch.sh"]
```
The **launch.sh** script is defined as entrypoint for the container. This get's executed once the containers spins up. 
Take a look at that script, and go to the next slide.


## Now what the @#!& happend !?
#### marathon-submit/bin/launch.sh
```
#!/bin/sh
set -x

# Prefer locally build lighter version
LIGHTER="/usr/bin/lighter"
DEVLIGHTER="/lighter/dist/lighter-$(uname -s)-$(uname -m)"
if [ -e  "$DEVLIGHTER" ]; then
    LIGHTER="$DEVLIGHTER"
fi

for file in /submit/json/*.json; do
  while true; do
    curl -X POST -H "Content-Type: application/json" "${MARATHON_URL}/v2/apps?force=true" -d "@${file}"
    if [ "$?" == "0" ]; then
      break
    fi
    sleep 1 & wait
  done
done

# Run lighter to deploy yaml files
function DEPLOY {
    mkdir -p /submit/target
    chmod -R a+rwX /submit/site/keys
    cd /submit/site

    "$LIGHTER" -t /submit/target deploy -f -m "${MARATHON_URL}" $(find . -name \*.yml -not -name globals.yml)
    chmod -R a+rwX /submit/target
}

DEPLOY;
EVENTS="CREATE,CLOSE_WRITE,DELETE,MODIFY,MOVED_FROM,MOVED_TO"

inotifywait -m -q -e "$EVENTS" -r --format '%:e %w%f' "/submit/site" "$LIGHTER" | while read file
  do
    DEPLOY;
  done
```
As you might notice, we can use JSON and YAML files to pass instructions to Marathon. Let's dig some deeper!


## Now what the @#!& happend !?
If we look at marathon **http://<< docker-machine-ip >>:8080/** we can see that there are two applications defined
- demo
- myproduct

Click around in these applications and try to detect which of the two is our demo application.


## Now what the @#!& happend !?
Ok, that should not have been to hard. Our demo application is of course the "demo" application. 

Try to find the corresponding YAML or JSON file that got deployed by **marathon-submit**.
Note:
The file that corresponds to the demo webapp is **marathon-submit/site/demo-webapp.yml**


## Now what the @#!& happend !?
If you've found the file that corresponds to our demo application, then you could have seen that it deploys multiple instances of the same application. If you check Mesos you will see that they all end up on the same slave node. In a real life cluster with multiple nodes they would have been spread across different nodes.


## What does this button do ??
The **marathon-submit** container redeploys the ***.yml** files once you change them. Try scaling the demo web application down. 
- Change the yml file to scale down to 1 instance
- Once you save the file, return to the marathon dashboard and observate.


## Let's spice it up
We are going to run a different (web)application "srenkens/wolfenstein3d"


## Ah... the good old times!
![pipeline](images/wolfenstein3d.png)
#### Ok admitted. It's not that badass, but hey... you get to play games during your course!


## Instructions
- Create a new **wolf3d.yml** file for the application
  - You can use the yml from the demo webapp as a starting point
- Change the networkmode from HOST to BRIGE (mandatory!).
  - You need to change more then just the mode, make sure you read: https://mesosphere.github.io/marathon/docs/ports.html
- Deploy at least 2 instances
- Application should be made available at: 
  - http://<< docker-machine-ip >>:8888/
- Hint: http://www.yamllint.com/

Note:
same as demo app asside from the following
- Change imagename to srenkens/wolfenstein3d
- Change networkmode from HOST to BRIDGE
- Add portMappings option to service --> container --> docker (containerport = 80, serviceport 8888)
- Remove the ports option
- Remove env options
- Change healthchecks path to /

<slides-url>docker-workshop/solutions/mesosmarathon/wolf3d.yml


## Did you defeat the boss!?
Got the game running? Well done!
![pipeline](images/wolfenstein3d-finshed.jpg)


## More information
If you wanna know more about Docker Clustering on Mesos with Marathon then the following video might be interesting. <br>
<iframe class="stretch" src="https://www.youtube.com/embed/hZNGST2vIds" frameborder="0" allowfullscreen></iframe>


## END of chapter
The Mesos and Marathon stack which you have seen in this chapter is a basic setup of the stack. In a datacenter it could scale out to control large number of servers with even more running containers.

Next to Kubernetes, Mesos and Marathon have long been the stacks to look at if you wanted to build a Docker cluster with orchstration. Slowly but surely other competitors are arising. Some alternatives to this stack are:
 - Mesosphere DCOS (https://mesosphere.com/) <br>  -    essentially the Mesos Marathon successor
 - Rancher (http://rancher.com/)
 - Or other cloud based solutions like CodeFresh (https://codefresh.io/)