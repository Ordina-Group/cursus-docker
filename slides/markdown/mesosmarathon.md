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


## Some internals
- TODO
- TODO
- TODO
- TODO
- TODO


# Let's get started

## 
- Because installing a complex stack like this is not the goal of this course, we will provide you with your own Mesos and Marathon stack.
- Execute the commands below to clean up and get the cluster up and running
- Building the stack will take some time. Continue in the next slide if you do not want to follow the progress of building the stack

```
# Stop all Containers
docker stop $(docker ps -a -q)

# If you need to install docker-compose:
sudo /mnt/sda1/cursus-docker/basics-docker/compose/install-compose.sh

# Change IP's in the docker-compose.yml file in case your docker-machine has an IP other then 192.168.99.100
sed -i s/192.168.99.100/<< docker-machine-ip >>/g docker-compose.yml

# Build the Mesos and Marathon cluster
cd /mnt/sda1/cursus-docker/mesos-marathon
docker-compose build

# Start the cluster
docker-compose up

```


### Stack ingredients
The stack consists of 7	 docker images
<small>
- **zookeeper:** <br>Software that is used to coordinate the master nodes

- **mesosmaster:** <br>The master a node in the cluster and orchestrates the running of tasks on slaves

- **mesosslave:** <br>A Mesos slave is a Mesos instance which offers resources to the cluster. They are the ‘worker’ instances - tasks are allocated to the slaves by the Mesos master.

- **marathon:** <br> Organizes and manages services deployed (what is deployed where, what ports, what configuration, etc)

- **marathonsubmit:** <br> A helper image for submitting jobs to marathon

- **gateway:** <br> A proxy that forwards TCP and UDP connections to where services are currently running on the clusters

- **secretary:** <br> A little tool to distrubute keys and password across the cluster

</small>


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

marathon-submit/bin/launch.sh


## Container overview
When you are done, you will have these containers running. Some will be maniputed by Jenkins and some by using docker-compose.
 
![pipeline](images/container-overview-ci-all.png)


## The Pipeline
These are the jobs that you will configure in Jenkins.
![pipeline](images/pipeline-complete.png)


# ..But first... 
#...some preperations...


## Windows and less then 20GB on the C disk ?? (1/3)
### You're screwed... but there is a solution
We need to move the docker-machine data to a disk with more free space
- Exit from the SSH session of the `workshop` machine

```
docker@workshop:~$ exit
```
- Stop the workshop machine

```
docker-machine stop workshop
```
- Do the same for other running docker-machines (you can list them with docker-machine ls)
- Continue on next slide


## Windows and less then 20GB on the C disk  (2/3)
- Using Windows explorer, create a folder on a disk that does have more then 20GB available (ex: d:\docker-machines)
- Using Windows explorer, move the folder `C:\Users\<username>\.docker\machine` to your created directory.
- Open a Command prompt with administrator rights 
  - (startmenu, type `cmd`, right-click `cmd`, run as administrator)
- Create a symbolic link on the original location and point it to the location where your data is stored
```
mklink /D  c:\Users\<username>\.docker\machine d:\docker-machines\machine
```


## Windows and less then 20GB on the C disk  (3/3)
- Return to your quickstart terminal
- Start the workshop machine

```
docker-machine start workshop
```
- SSH back into the workshop

```
docker-machine ssh workshop
```
- Reinstall docker-compose

```
# Navigate to the basics-docker/ directory.  
$ cd /mnt/sda1/cursus-docker/basics-docker/compose
$ sudo ./install-compose.sh
```
- Done


# Let's get started


## Setting up Git server
- We will setup a GitLAB as a GIT server. 
- This GitLAB server will be used as a server to host our REPO to Jenkins. 
- You can use GitLAB to check the sources and (if you even want to) make changes to the sources.


### Setting up GitLab (docker-compose)	
- The containers that we need for our Continuous Delivery environment will be managed by Docker Compose. Later we will add more.
- You will find a docker-compose.yml file in /mnt/sda1/cursus-docker/continuous. For now this file only contains one container definition for GitLab, our Git server.
- (Setup instructions on the next slide)

```	
git:
  image: gitlab/gitlab-ce:8.0.4-ce.1
  volumes:
    - ./.data/gitlab/config:/etc/gitlab
    - ./.data/gitlab/logs:/var/log/gitlab
    - ./.data/gitlab/data:/var/opt/gitlab
  ports:
    - "2222:22"
    - "80:80"
```


### Setting up GitLab - Start
- To test our compose file we will
 - Start the GitLAB container by using the docker-compose tool
 - Check the logs of GitLAB container
- Execute the following commands:

```
# navigate to /mnt/sda1/cursus-docker/continuous
docker-compose up -d

# to inspect the logs
docker-compose logs git

# Wait until gitlab server has started
# Look for 'The server is now ready to accept connections' (part of the redis logging)
# Use ctrl-c to quit the logging
```
You now have a running GitLAB server


### Setting up GitLab - Configure
Let's perform the basic configuration of GitLAB 

- Start a webbrowser and browse to: `http://<< docker-machine-ip >>/`
- Login with the default user and password.
  - username: root
  - password: 5iveL!fe
- Next you have to set a new password
 - If you wanna be lazy.. you can change it to the original `5iveL!fe` password
- After setting the password login again.


### Setting up GitLab - Configure
We will clone a remote repo which contains our StickyNote sources. That cloned repo will be served by GitLAB. 

- Go to create new project, fill in the form as follows:
  - Project path: stickynote
  - Import project from: "git Any repo by URL" `https://github.com/OrdinaNederland/cursus-docker-sampleapp.git`
  - Visibility Level: Public
- Create project

You can now check the sources (and change them if you like) using the GitLAB webinterface.


## Setting up Jenkins
Jenkins will be used as our main Continuous Delivery tool. It will build our code, our docker images and manage our Docker containers


### Setting up Jenkins - Dockerfile
Jenkins is available as Docker image but for this workshop we build our own Jenkins image. We use the Jenkins base image.
- Go to the `/mnt/sda1/cursus-docker/continuous` directory and create a folder called `jenkins`
- In the newly created folder, create a file named `Dockerfile` with the following content.

```docker
FROM jenkins:1.642.1

USER root
RUN apt-get update && apt-get install \  
     wget curl apt-transport-https -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER jenkins

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt
```


### Setting up Jenkins - plugins
- Since we try to automate most of the things we are doing, we also automate and manage our Jenkins plugins by specifying them in the plugins.txt file.
- Create a file `plugins.txt` in the same dir as the `Dockerfile` and the following plugins to the file.  

```
scm-api:0.2
jquery:1.7.2-1
parameterized-trigger:2.17
authentication-tokens:1.1
credentials:1.22
docker-build-step:1.33
docker-commons:1.2
git-client:1.19.0
git:2.4.0
gradle:1.24
token-macro:1.9
delivery-pipeline-plugin:0.9.8
```


### Setting up Jenkins - docker-compose
The containers for our Continuous Delivery environment are managed by Docker Compose, as seen for GitLab. Now we will add the Jenkins container to the configuration.
- Create a dir for Jenkins data

```
# Navigate to the /mnt/sda1/cursus-docker/continuous directory first
cd /mnt/sda1/cursus-docker/continuous
sudo mkdir -p .data/jenkins && sudo chmod 777 .data/jenkins
```
- Edit the file `docker-compose.yml`, add the following content at the end.

```
jenkins:
  build: ./jenkins
  volumes:
    - ./.data/jenkins:/var/jenkins_home
    - /var/run/docker.sock:/var/run/docker.sock
  ports:
    - "8080:8080"
  user: jenkins:100
  extra_hosts:
    mavenrepository: 52.30.98.133
```


### Jenkins - Add link to gitlab container
We need to make the GitLab container accessible from the Jenkins container. We do this by adding a `link` in the Jenkins container configuration.
- Add the `links:` section and `git:git` to that section, so the Jenkins container can refer to the git host, using the name "git".

```
jenkins:
  build: ./jenkins
  volumes:
     - ./.data/jenkins:/var/jenkins_home
     - /var/run/docker.sock:/var/run/docker.sock
  ports:
    - "8080:8080"
  user: jenkins:100
  extra_hosts:
    mavenrepository: 52.30.98.133
  links:
    - git:git
```


### Setting up Jenkins - docker-compose (RECAP)
A little recap:	
- We have defined our Jenkins container in a docker-compose file.
  - The container will be build based on a Dockerfile in the directory `./jenkins`.
  - The container will mount the `jenkins_home` directory to the directory `.data/jenkins` on the host.
  - The container maps port `8080` from to container to `8080` on the localhost.
  - The container will have an extra entry in the host file to resolve the dockerhost for building images.


### Setting up Jenkins - LAUNCH!
Ok, time to get Jenkins up and running.
- Start the docker container(s).

  ```
  docker-compose up -d
  ```
- Open Jenkins in a browser (http://<< docker-machine-ip >>:8080).
- Jenkins might not be available straight away. Give it some time to start before entering panic mode.

You might have noticed that only Jenkins got started after the docker-compose command. Docker-compose is smart enough to notice that the GitLAB container was already running.


## Configuring Jenkins
Jenkins requires some configuration before it can interact with Docker. 


### Jenkins - Global configuration
- First we have to set some global configurations in Jenkins.
  - Open the Jenkins application in your browser
  - Go to: Manage Jenkins -> Configure system
  - Search for: `Docker Builder` and provide the url: `unix:///var/run/docker.sock`
  - !! SAVE BEFORE TESTING THE CONNECTION !!
  - Go to: Manage Jenkins -> Configure system
  - Search for: `Docker Builder`
  - `Test Connection`


## Jenkins - Setting up delivery pipeline
Now we are ready to set up our delivery pipeline. The pipeline is an aggregation of several small builds to build, test and deliver our service.

![pipeline](images/pipeline-complete.png)


## The main job
![pipeline](images/pipeline-main.png)


### Jenkins - Job main (1/2)
The main job is a simple job that is responsible for checking out the StickyNote GIT repo. Everytime you want to start the pipeline, you will do so by starting this job.


### Jenkins - Job main (2/2)
- Create new item
 - Item name: `service-main`
 - Type: `Free style project`
 - Delivery pipeline configuration: `Stage Name=build, Task Name=clean`
 - Source Code Management Git: http://git/root/stickynote.git (remember the linked container named git)
 - Add post build action: Trigger parameterized build on other projects
   - projects to build: `service-build` (ignore non-existing error)
   - Trigger when build is: stable
   - Add parameters: predefined parameters
    ```
    SOURCE_BUILD_NUMBER=${BUILD_NUMBER}
    SOURCE_BUILD_WORKSPACE=${WORKSPACE}
    VERSION=1.0.${SOURCE_BUILD_NUMBER}
    ```
 - Save


## The Build job
![pipeline](images/pipeline-build.png)


### Jenkins - Job build (1/2)
The build job runs the Gradle "build" task. This compiles our code and run the unit tests.


### Jenkins - Job build (2/2)
- Create new item
 - Item name: `service-build`
 - Type: `Free style project`
 - Delivery pipeline: `Stage Name=build, Task Name=build`
 - Advanced Project Options
   - Use custom workspace directory: `${SOURCE_BUILD_WORKSPACE}`
 - Add build step: Invoke Gradle script
   - Use gradle wrapper
   - Tasks: `build`
 - Save, and test by starting the `service-main` job.


## The package job
![pipeline](images/pipeline-package.png)


### Jenkins - Job package (1/3)
The package job creates a Docker image for our StickyNote service. The definition of this image is declared in a Dockerfile.

- If you like to see how we build our docker image, you can inspect the `build.gradle` in GitLAB at this location:
 - /build.gradle  
- The `Dockerfile` is located in the GIT repo at this location: 
 - /src/main/resources/docker/Dockerfile

```
FROM java:8-jdk

RUN echo 127.0.2.1 mongodb > /etc/hosts; cat /etc/hosts

ADD stickynote-service.jar /

EXPOSE 8080

CMD ["java","-jar","/stickynote-service.jar"]
```


### Jenkins - Job package (2/3)
- Create new item
- Item name: `service-package`
- Type: `Copy service-build`
- Delivery pipeline: `Stage Name=build, Task Name=package`
- Change build step: Invoke Gradle script
  - Use gradle wrapper
  - change: Tasks: `buildImage`
- Save


### Jenkins - Job package (3/3)
- Configure job `service-build` to add a trigger.
  - Add post build action: Trigger parameterized build on other projects
    - projects to build: `service-package`
    - Trigger when build is: `stable`
    - Add parameters: current build parameters
- Save and test by starting the `main` project.


## DockerUI
In the next part we are creating many docker containers. Just for the fun we install a DockerUI on our host.

DockerUI is a web interface for the Docker Remote API. Is gives us a view of running containers and allows us to perform commands like stop, start, remove, etc. 


### Adding docker ui

- Edit the docker-compose.yml (in /mnt/sda1/cursus-docker/continuous) and add the dockerui section

```
dockerui:
  image: dockerui/dockerui
  privileged: true
  ports:
    - "9090:9000"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```
- Use `docker-compose up -d` to start the dockerui container.
- Browse to http://<< docker-machine-ip >>:9090 to see which containers are running.


## The start job
![pipeline](images/pipeline-start.png)


### Jenkins - Job service-start (1/5)
The start job will be creating and starting the StickyNote Docker container and the MongoDB database container.


### Jenkins - Job service-start (2/5)
- Create new item
 - Item name: `service-start`
 - Type: `Copy service-build`
 - Delivery pipeline: `Stage Name=QA, Task Name=start`
 - Remove build step gradle.
 - Add build step: Execute docker command
   - Docker command: pull image
   - Image name: `mongo`
   - Tag: `3.0.6`
 - Add build step: Execute docker command
   - Docker command: create container
   - Image name: `mongo:3.0.6`
   - Container name: `test_db`


### Jenkins - Job service-start (3/5)
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `stickynote/stickynote-service:latest`
  - Container name: `test_service`
  - Advanced - Links: `test_db:mongodb`
- Add build step: Execute docker command
  - Docker command: start container(s)
  - Container ID(s): `test_db, test_service`
  - Advanced - Port bindings: `8888:8080`
  - Advanced - Wait for ports:
  ```
  test_db 27017
  test_service 8080
  ```


### Jenkins - Job service-start (4/5)
- Change post build action: Trigger parameterized build on other projects
  - projects to build: `service-stop`
  - Don't mind the error about non-existing project, we'll fix that later.
- Save


### Jenkins - Job service-start (5/5)  
- Configure job `service-package` to add a trigger.
    - Add post build action: Trigger parameterized build on other projects
      - projects to build: `service-start`
      - Add parameters: current build parameters
- Save


### The start job (RECAP)
This job effectivly performs these Docker CLI command but then through the the Docker API.
```
# Pull and create MongoDB container
docker pull mongodb:3.0.6
docker create --name test_db mongodb:3.06

# StickyNote service: No need to pull, since we've build the image on the same Dockerhost. 
# Therefore it's already available for us.
docker create --name test_service -p 8888:8000 --link test_db:mongodb stickynote/stickynote-service:latest 

# Start containers
docker start test_db test_service
```


## The stop job
![pipeline](images/pipeline-stop.png)


### Jenkins - Job service-stop (1/3)
This jobs will stop and removing the earlier created containers.


	
### Jenkins - Job service-stop (2/3)
- Create new item
 - Item name: `service-stop`
 - Type: `Copy service-start`
 - Delivery pipeline: `Stage Name=QA, Task Name=stop`
 - Remove all existing build steps.
 - Remove Trigger from Post-build Actions.


### Jenkins - Job service-stop (3/3)
- Add build step: Execute docker command
  - Docker command: Stop container(s)
  - Container ID(s): `test_service, test_db`
- Add build step: Execute docker command
  - Docker command: Remove container(s)
  - Container ID(s): `test_service, test_db`
  - Advanced - Ignore if not found to TRUE
  - Advanced - Remove volumes to TRUE
  - Advanced - Force remove to TRUE
- Save and test by starting the `main` project.


### The stop job (RECAP)
This job effectivly performs these Docker CLI command but then through the the Docker API.
```
# Stop the containers
docker stop test_service test_db

# Remove the containers
docker rm -f -v test-service test_db 
```


## The tests job
![pipeline](images/pipeline-tst.png)


### Jenkins - Job service-tests (1/4)
The test job perform tests against our running container. Gradle will start a JMeter test and JMeter will use our test_service container as its target.


### Jenkins - Job service-tests (2/4)
- Create new item
 - Item name: `service-tests`
 - Type: `Copy service-build`
 - Delivery pipeline: `Stage Name=QA, Task Name=tests`
 - Change build step: Invoke Gradle script
   - Use gradle wrapper
   - Tasks (multiline!)
     - line 1: `integrationTest`
     - line 2: `jmeterRun`
 - Change post build action: Trigger parameterized build on other projects
   - projects to build: `service-stop`
   - Trigger when build is: Complete (always trigger)
   - Add parameters: current build parameters


### Jenkins - Job service-tests (3/4)
- Configure job `service-tests`
  - Add build step: Execute shell script
    - Move this step above the Gradle build step
    - Commando:
```
#!/bin/bash
dockerhostip=$(/sbin/ip route|awk '/default/ { print $3 }')
find src/it-test -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
find build -type f -print0 | xargs -0 sed -i "s/dockerhost/$dockerhostip/g"
```
- Save
<hr>
Background: <br>This piece of bash script replaces every occurence of 'dockerhost' in your job's workspace files with the IP of your docker host machine.
<hr>


### Jenkins - Job service-tests (4/4)
- Configure job `service-start` to change the post build trigger.
  - Change post build action: Trigger parameterized build on other projects
    - projects to build: `service-tests`
    - Add parameters: current build parameters
- Save and test by starting the `main` project.


## The deploy job
![pipeline](images/pipeline-deploy.png)


### Jenkins - Job service-deploy (1/5)
The deploy job deploys (no shit) our's service to a "production (demo)" environment. After the tests have succeeded this job will:
- Tag the image created in the `package` job.
- Remove the current production containers
- Create new production containers based on the new image
- Start the newly created containers


### Jenkins - Job service-deploy (2/5)
- Create new item
 - Item name: `service-deploy`
 - Type: `Copy service-start`
 - Delivery pipeline: `Stage Name=Release, Task Name=deploy`
 - Remove all build steps (4).
 - Remove post build action: Trigger parameterized build on other projects
 - Add build step: Execute docker command
   - Docker command: Tag image
   - Name of image: `stickynote/stickynote-service:latest`
   - Target repository of the new tag: `stickynote/stickynote-service`
   - The tag to set: `${VERSION}`


### Jenkins - Job service-deploy (3/5)
- Add build step: Execute docker command
  - Docker command: Remove container(s)
  - Container ID(s): `demo_db, demo_service`
  - Advanced - Ignore if not found to TRUE
  - Advanced - Remove volumes to TRUE
  - Advanced - Force remove to TRUE
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `mongo:3.0.6`
  - Container name: `demo_db`


### Jenkins - Job service-deploy (4/5)
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `stickynote/stickynote-service:${VERSION}`
  - Container name: `demo_service`
  - Advanced - Links: `demo_db:mongodb`
- Add build step: Execute docker command
  - Docker command: start container(s)
  - Container ID(s): `demo_db, demo_service`
  - Advanced - Port bindings: `8887:8080`
  - Advanced - Wait for ports:

  ```
  demo_db 27017
  demo_service 8080
  ```
- Save


### Jenkins - Job service-deploy (5/5)
- Configure `service-stop` job
- Add post build action: Trigger parameterized build on other projects
  - projects to build: `service-deploy`
  - Add parameters: current build parameters
- Save and test by starting the `main` project.


### The deploy job (RECAP)
This job effectivly performs these Docker CLI command but then through the the Docker API.
```
# Give the image a TAG
docker tag stickynote/stickynote-service:latest stickynote/stickynote-service:<SOMEVERSIONNUMBER>

# Stop and remove existing containers
docker rm -f -v demo_db demo_service

# Create the new containers
docker create --name demo_db mongodb:3.06
docker create --name demo_service -p 8887:8000 --link demo_db:mongodb stickynote/stickynote-service:<SOMEVERSIONNUMBER>

# Start containers
docker start demo_db demo_service
```
The sticky note (REST)application is available at: 

`http://<<machine-ip>>:8887/notes/`



## Delivery pipeline
Well... until now we did not get some visual feedback of our pipeline other then single failing jobs. Let's make that a bit more attractive.


## Delivery pipeline view
- On the Jenkins main page (where the jobs are listed)
 - Click on the (+) to add an extra tab.
 - Choose Delivery Pipeline view
 - Pick a name for your view
 - Components - Add
 - Choose a Name
 - Initial Job `service-main`


### NICE!
![pipeline](images/pipeline-view.png)


## The sonar job
![pipeline](images/pipeline-sonar.png)


### Sonar (1/4)
We will add Sonar to keep track of our code quality.


### Jenkins - Sonar (2/4)
- Create new item
 - Item name: `service-sonar`
 - Type: `Copy service-build`
 - Delivery pipeline: `Stage Name=QA, Task Name=sonar`
 - Change build step: Invoke Gradle script
   - Use gradle wrapper
   - Tasks: sonarRunner
 - Save


### Docker - Sonar (3/4)
First we stop all of our docker services, and add the Sonar docker image to our `docker-compose` file.
- Stop the docker services using docker-compose (in /mnt/sda1/cursus-docker/continuous)
```
docker-compose stop
```


### Docker - Sonar (4/8)
- Edit the docker-compose.yml file.
- Add a container for the database:

```
sonardb:
  image: postgres:9
  environment:
    - POSTGRES_USER=sonar
    - POSTGRES_PASSWORD=secret
  volumes:
    - ./.data/sonardb/data:/var/lib/postgresql/data
```


### Docker - Sonar (5/8)
- Add a container for Sonar:

```
sonar:
  image: sonarqube:5.1.1
  links:
    - sonardb:db
  environment:
    - SONARQUBE_JDBC_USERNAME=sonar
    - SONARQUBE_JDBC_PASSWORD=secret
    - SONARQUBE_JDBC_URL=jdbc:postgresql://db/sonar
  ports:
    - "9000:9000"

```


### Docker - Sonar (6/8)
- And link the two Sonar containers to the Jenkins container
- In the `links:` section of the Jenkins configuration add links from Jenkins to Sonar (see below).
- Save the docker-compose.yml file

```
jenkins:
  ...
  links:
    - git:git
    - sonar:sonar
    - sonardb:sonardb
  ...
```


### Docker - Sonar
- Restart compose

```
docker-compose up -d
```


### Jenkins - Sonar (7/8)
- Configure `service-sonar` job
- Edit post build action: Trigger parameterized build on other projects
  - projects to build: `service-deploy`
- Save


### Jenkins - Sonar (8/8)
- Configure job `service-stop`
  - Edit post build action: Trigger parameterized build on other projects
    - projects to build: `service-sonar`
- Save and test by starting the `main` project.
- Contact Sonar dashboard at `http://<< docker-machine-ip >>:9000/`


And you are..
# DONE!
This is just one way of setting up a simple Continuous Delivery pipeline. 
You might encounter clusters of Docker engines and other ways of controlling where and how containers will be created, started, stopped, etc...

This solution is a simple one, but gives you some idea of how a CD like solution might look like and the various Docker commands it relates to.

We provided a base level of knowledge for you to investigate and discover more of Docker by yourself. The last chapter provides you some interesting links with more useful Docker material. 